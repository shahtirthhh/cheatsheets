# Fine-Tuning — Complete Staff-Level Guide

_When to fine-tune · LoRA/QLoRA · Data prep · Training · Evaluation · Serving_

---

# Part 1: The Decision Framework (Most Important Section)

## Prompt vs RAG vs Fine-Tune

```
THE QUESTION TO ASK: what is actually failing?

  ❌ "The model doesn't know our internal data"        → RAG, not fine-tuning
  ❌ "The model's info is outdated"                     → RAG
  ❌ "We need citations and auditability"               → RAG
  ✅ "The model won't follow our output format"         → Fine-tune
  ✅ "It doesn't match our brand voice"                  → Fine-tune
  ✅ "It fails at our domain-specific reasoning"        → Fine-tune
  ✅ "Prompts are 3000 tokens of instructions/examples" → Fine-tune (cost/latency)
  ✅ "We need a small model to match a large one"       → Fine-tune (distillation)

THE RULE:
  RAG teaches the model WHAT to say (knowledge).
  Fine-tuning teaches the model HOW to say it (behavior, format, style, reasoning).

  Fine-tuning is a TERRIBLE way to inject facts:
    • Facts get entangled in weights — you can't cite or verify them
    • Updating one fact means retraining
    • The model still hallucinates, just more confidently
    • Costs orders of magnitude more than re-indexing a document
```

## The Escalation Ladder

```
Always climb in this order. Stop at the first rung that works.

  1. BETTER PROMPT           Cost: hours     Try: clearer instructions, structure
  2. FEW-SHOT EXAMPLES       Cost: hours     Try: 3-10 examples in the prompt
  3. RAG                     Cost: days      Try: retrieve relevant context
  4. PROMPT + RAG TUNED      Cost: days      Try: reranking, query rewriting
  5. LoRA FINE-TUNE          Cost: 1-2 weeks Try: 500-5000 examples
  6. FULL FINE-TUNE          Cost: weeks     Try: 10K+ examples, multi-GPU
  7. CONTINUED PRETRAINING   Cost: months    Try: new domain/language entirely

Most teams that "need fine-tuning" actually need rung 1 or 2.
Interviewers respect a candidate who says "I'd exhaust prompting and RAG first."
```

## Cost/Benefit Reality Check

```
                      Setup    Per-query    Update       Explainability
                      ─────    ─────────    ──────       ──────────────
Prompt engineering    Hours    High (long   Instant      Full
                               prompt)
RAG                   Days     Medium       Re-index     Full (citations)
LoRA fine-tune        1-2 wks  LOW (short   Retrain      None
                               prompt)      (hours)
Full fine-tune        Weeks    LOW          Retrain      None
                                            (days)

FINE-TUNING'S REAL WIN: inference cost and latency.
  Before: 3000-token prompt with 15 few-shot examples → $0.045/call, 4s
  After:  200-token prompt on a fine-tuned 8B model  → $0.0004/call, 0.8s
  At 1M calls/month, that's $45,000 → $400. THAT justifies the effort.
```

---

# Part 2: LoRA — The Math

## The Core Insight

```
Full fine-tuning updates every weight matrix W (d × d).
For Llama-3-8B that's 8 billion parameters — needs ~64GB just for optimizer states.

LoRA's observation: the UPDATE ΔW during fine-tuning has LOW INTRINSIC RANK.
You're not learning a fundamentally new function; you're nudging an existing one.
So ΔW can be approximated by the product of two skinny matrices.

  W_new = W_frozen + ΔW
  ΔW    = B × A          where A is (r × d) and B is (d × r), r << d

  Original W:  4096 × 4096         = 16,777,216 params  (FROZEN ❄️)
  LoRA A:      16 × 4096           =     65,536 params  (trainable 🔥)
  LoRA B:      4096 × 16           =     65,536 params  (trainable 🔥)
  ─────────────────────────────────────────────────────
  Trainable:   131,072 / 16,777,216 = 0.78%

  ┌──────────────────┐     ┌────┐   ┌──────────────────┐
  │                  │     │    │   │                  │
  │   W (frozen)     │  +  │ B  │ × │        A         │
  │   4096 × 4096    │     │4096│   │    16 × 4096     │
  │                  │     │ ×16│   │                  │
  └──────────────────┘     └────┘   └──────────────────┘

  Forward pass: h = Wx + (B(Ax)) × (α/r)
  A is initialized with random Gaussian, B with ZEROS.
  → At step 0, BA = 0, so the model behaves EXACTLY like the base model.
    Training starts from a known-good state rather than a perturbed one.
```

## Hyperparameters That Matter

```
r (rank)             8-64.   Capacity of the adapter.
                     8-16  → style, tone, format changes
                     32-64 → new domain knowledge, complex reasoning
                     Higher r ≠ always better. Overfits on small datasets.

lora_alpha           Scaling factor. Effective scale = alpha / r.
                     Convention: alpha = 2 × r (so scale = 2).
                     Think of alpha/r as the adapter's learning rate multiplier.

lora_dropout         0.05-0.1. Regularization on the adapter path.

target_modules       WHICH layers get adapters. THE most impactful choice.
                     Minimal:  ["q_proj", "v_proj"]              (original paper)
                     Better:   ["q_proj","k_proj","v_proj","o_proj"]
                     Best:     + ["gate_proj","up_proj","down_proj"]  (MLP layers too)
                     Adding MLP layers roughly triples trainable params but
                     consistently improves quality on domain adaptation.

bias                 "none" (default), "all", or "lora_only". "none" is fine.
```

## QLoRA — Fitting 70B on One GPU

```
QLoRA = 4-bit quantized base model + LoRA adapters in bf16.

  THREE INNOVATIONS:
  1. NF4 (4-bit NormalFloat) — a quantization datatype that's information-
     theoretically optimal for normally-distributed weights (which NN weights are).
     Better than naive int4 for the same bit budget.

  2. Double Quantization — quantize the quantization constants themselves.
     Saves another ~0.4 bits/param.

  3. Paged Optimizers — use NVIDIA unified memory to page optimizer states
     to CPU RAM during memory spikes, preventing OOM crashes.

  MEMORY FOR LLAMA-3-8B:
    Full fine-tune (bf16 + Adam):  8B × (2 + 2 + 4 + 4) = ~96 GB   ❌
    LoRA (bf16 base):              8B × 2 + small        = ~18 GB   ⚠️
    QLoRA (nf4 base):              8B × 0.5 + small      = ~6 GB    ✅

  Trains on a single RTX 4090 (24GB) with room to spare.
  Quality cost vs LoRA: typically <1% on downstream benchmarks.
```

---

# Part 3: Data Preparation (Where Projects Actually Fail)

## Data Quality Beats Data Quantity

```
Empirically, for instruction tuning:
  1,000 carefully curated examples  >  50,000 scraped noisy examples

WHY: the model learns the DISTRIBUTION of your examples, including the mistakes.
     Ten inconsistent examples teach the model to be inconsistent.

MINIMUM VIABLE DATASET SIZES:
  Format/style adherence:    200-500 examples
  Domain-specific tone:      500-2,000
  Complex reasoning tasks:   2,000-10,000
  New task capability:       10,000+
```

## Format: ChatML / Conversational

```python
# Standard conversational format (what most models expect)
{
  "messages": [
    {"role": "system", "content": "You are a medical coding assistant. Output ICD-10 codes only."},
    {"role": "user", "content": "Patient presents with acute bronchitis, unspecified."},
    {"role": "assistant", "content": "J20.9"}
  ]
}

# Multi-turn
{
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "What's the code for type 2 diabetes?"},
    {"role": "assistant", "content": "E11.9 for type 2 diabetes mellitus without complications."},
    {"role": "user", "content": "With diabetic neuropathy?"},
    {"role": "assistant", "content": "E11.40 for type 2 diabetes mellitus with diabetic neuropathy, unspecified."}
  ]
}

# Tool calling
{
  "messages": [
    {"role": "user", "content": "What's the weather in Mumbai?"},
    {"role": "assistant", "content": null,
     "tool_calls": [{"id": "c1", "type": "function",
                     "function": {"name": "get_weather", "arguments": "{\"city\":\"Mumbai\"}"}}]},
    {"role": "tool", "tool_call_id": "c1", "content": "{\"temp\": 31, \"condition\": \"humid\"}"},
    {"role": "assistant", "content": "It's 31°C and humid in Mumbai right now."}
  ]
}
```

## Data Quality Pipeline

```python
import json, hashlib
from collections import Counter

def build_dataset(raw_examples: list[dict]) -> list[dict]:
    seen_hashes = set()
    cleaned = []
    stats = Counter()

    for ex in raw_examples:
        msgs = ex["messages"]

        # 1. Structural validation
        if not msgs or msgs[-1]["role"] != "assistant":
            stats["bad_structure"] += 1
            continue

        # 2. Exact deduplication
        h = hashlib.sha256(json.dumps(msgs, sort_keys=True).encode()).hexdigest()
        if h in seen_hashes:
            stats["duplicate"] += 1
            continue
        seen_hashes.add(h)

        # 3. Length filtering (both directions)
        total_tokens = sum(len(tokenizer.encode(m["content"] or "")) for m in msgs)
        if total_tokens > 4096:
            stats["too_long"] += 1
            continue
        if len(msgs[-1]["content"]) < 10:
            stats["response_too_short"] += 1
            continue

        # 4. Reject refusals and boilerplate leaking from the teacher model
        resp = msgs[-1]["content"].lower()
        if any(p in resp for p in ["as an ai", "i cannot", "i'm sorry, but", "i don't have access"]):
            stats["refusal"] += 1
            continue

        cleaned.append(ex)
        stats["kept"] += 1

    print(stats)
    return cleaned

# 5. Near-duplicate removal via embeddings (catches paraphrases)
def dedupe_semantic(examples, threshold=0.95):
    vecs = embed([e["messages"][-2]["content"] for e in examples])
    keep, seen = [], []
    for i, v in enumerate(vecs):
        if not seen or max(cosine(v, s) for s in seen) < threshold:
            keep.append(examples[i]); seen.append(v)
    return keep
```

## The Splits

```python
# HOLD OUT BEFORE ANY TRAINING. Never tune on the test set.
train, temp = train_test_split(dataset, test_size=0.2, random_state=42)
val, test  = train_test_split(temp, test_size=0.5, random_state=42)
# 80% train / 10% validation (early stopping, LR selection) / 10% test (final, touched ONCE)

# For multi-source data, split by SOURCE not randomly, or near-duplicates
# across the split will inflate your metrics.
```

---

# Part 4: Training with HuggingFace + PEFT

```python
import torch
from transformers import (
    AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig,
    TrainingArguments, EarlyStoppingCallback,
)
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training, TaskType
from trl import SFTTrainer, SFTConfig
from datasets import load_dataset

MODEL = "meta-llama/Meta-Llama-3.1-8B-Instruct"

# ── 1. Quantization config (QLoRA) ──
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",                    # NormalFloat4, not plain int4
    bnb_4bit_compute_dtype=torch.bfloat16,        # compute in bf16, store in nf4
    bnb_4bit_use_double_quant=True,               # quantize the quantization constants
)

# ── 2. Load model and tokenizer ──
model = AutoModelForCausalLM.from_pretrained(
    MODEL,
    quantization_config=bnb_config,
    device_map="auto",
    attn_implementation="flash_attention_2",       # 2-3x faster, less memory
    torch_dtype=torch.bfloat16,
)
model.config.use_cache = False                     # incompatible with gradient checkpointing
model = prepare_model_for_kbit_training(model, use_gradient_checkpointing=True)

tokenizer = AutoTokenizer.from_pretrained(MODEL)
tokenizer.pad_token = tokenizer.eos_token
tokenizer.padding_side = "right"                   # "left" only for batch generation

# ── 3. LoRA config ──
peft_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=32,
    lora_alpha=64,                                  # 2 × r
    lora_dropout=0.05,
    bias="none",
    target_modules=["q_proj","k_proj","v_proj","o_proj","gate_proj","up_proj","down_proj"],
)
model = get_peft_model(model, peft_config)
model.print_trainable_parameters()
# trainable params: 83,886,080 || all params: 8,113,197,056 || trainable%: 1.03

# ── 4. Training config ──
args = SFTConfig(
    output_dir="./out",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=8,                  # effective batch = 4 × 8 = 32
    gradient_checkpointing=True,                    # trade compute for memory
    optim="paged_adamw_8bit",                       # paged prevents OOM spikes
    learning_rate=2e-4,                             # LoRA uses 10-100x higher LR than full FT
    lr_scheduler_type="cosine",
    warmup_ratio=0.03,
    max_grad_norm=0.3,
    weight_decay=0.001,
    bf16=True,
    max_seq_length=2048,
    packing=True,                                   # pack short examples together — big speedup
    logging_steps=10,
    eval_strategy="steps",
    eval_steps=50,
    save_strategy="steps",
    save_steps=50,
    save_total_limit=3,
    load_best_model_at_end=True,
    metric_for_best_model="eval_loss",
    greater_is_better=False,
    report_to="wandb",
    seed=42,
)

# ── 5. Train ──
trainer = SFTTrainer(
    model=model,
    args=args,
    train_dataset=train_ds,
    eval_dataset=val_ds,
    processing_class=tokenizer,
    callbacks=[EarlyStoppingCallback(early_stopping_patience=3)],
)
trainer.train()

# ── 6. Save the adapter (tiny — ~150MB, not 16GB) ──
trainer.model.save_pretrained("./adapter")
tokenizer.save_pretrained("./adapter")
```

## Hyperparameter Guidance

```
learning_rate     LoRA: 1e-4 to 3e-4  (much higher than full FT because
                                       you're training few params from zero-init B)
                  Full FT: 1e-5 to 5e-5
                  Too high → loss spikes, catastrophic forgetting
                  Too low  → underfits, no behavior change

epochs            1-3 for most instruction tuning.
                  Beyond 3, memorization dominates and generalization drops.
                  Watch eval_loss: the epoch where it turns upward is your stopping point.

effective batch   16-64 via gradient_accumulation_steps.
                  Larger = more stable gradients, slower wall-clock per step.

max_seq_length    Set to the 95th percentile of your data, not the max.
                  Memory scales quadratically with sequence length (attention).

packing=True      Concatenates short examples to fill the context window.
                  2-5x throughput gain on datasets with short examples.
                  ⚠️ Ensure your collator masks cross-example attention properly.

warmup_ratio      0.03-0.1. Prevents early large gradients from destabilizing training.
```

---

# Part 5: Alignment Beyond SFT (DPO)

```
SFT teaches the model what a GOOD answer looks like.
It does NOT teach what a BAD answer looks like.

PREFERENCE OPTIMIZATION uses PAIRS: (prompt, chosen, rejected).

  RLHF (PPO):  train a reward model on human rankings, then use RL to
               optimize the policy against it. Powerful but complex —
               four models in memory, unstable, hard to tune.

  DPO:         skip the reward model entirely. Directly optimize the policy
               using a closed-form loss derived from the same objective.
               Two models in memory, stable, far simpler. Now the default.
```

```python
from trl import DPOTrainer, DPOConfig

# Data format
{
  "prompt":   "Explain our refund policy.",
  "chosen":   "Refunds are available within 30 days of purchase. Contact support with your order ID.",
  "rejected": "I think you can probably get a refund, maybe try emailing someone?"
}

dpo_config = DPOConfig(
    output_dir="./dpo_out",
    beta=0.1,                        # KL penalty strength — how far the policy may drift
                                     # higher beta = stay closer to the SFT model
    learning_rate=5e-7,              # MUCH lower than SFT
    num_train_epochs=1,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=8,
    max_length=1024,
    max_prompt_length=512,
    bf16=True,
)

trainer = DPOTrainer(
    model=sft_model,                 # start from your SFT checkpoint, not the base
    ref_model=None,                  # with PEFT, the frozen base acts as the reference
    args=dpo_config,
    train_dataset=preference_ds,
    processing_class=tokenizer,
    peft_config=peft_config,
)
trainer.train()

# THE PIPELINE: base → SFT (learn the task) → DPO (learn preferences)
# Running DPO on a base model without SFT first usually underperforms.
```

---

# Part 6: Evaluation (Loss Is Not Enough)

```
⚠️ eval_loss tells you almost nothing about whether the model is USEFUL.
   A model can have lower perplexity and still be worse at your task.

EVALUATE ON FOUR AXES:
  1. TASK METRICS       exact match, F1, BLEU, or a domain-specific score
  2. LLM-AS-JUDGE       pairwise comparison against the base model
  3. REGRESSION CHECK   did it get worse at things it used to do well?
  4. HUMAN REVIEW       sample 50 outputs and actually read them
```

```python
# ── 1. Task-specific metric ──
def evaluate_exact_match(model, tokenizer, test_ds):
    correct = 0
    for ex in test_ds:
        prompt = tokenizer.apply_chat_template(
            ex["messages"][:-1], tokenize=False, add_generation_prompt=True
        )
        out = generate(model, tokenizer, prompt, max_new_tokens=128, temperature=0.0)
        if normalize(out) == normalize(ex["messages"][-1]["content"]):
            correct += 1
    return correct / len(test_ds)

# ── 2. LLM-as-judge, pairwise vs the base model ──
from pydantic import BaseModel
from typing import Literal

class Verdict(BaseModel):
    winner: Literal["A", "B", "tie"]
    reasoning: str

judge = ChatOpenAI(model="gpt-4o", temperature=0).with_structured_output(Verdict)

def pairwise_eval(test_ds, base_model, tuned_model):
    wins = {"tuned": 0, "base": 0, "tie": 0}
    for ex in test_ds:
        q = ex["messages"][-2]["content"]
        a = generate(base_model, q)
        b = generate(tuned_model, q)
        # Randomize position to avoid position bias in the judge
        first, second, swapped = (b, a, True) if random.random() < 0.5 else (a, b, False)
        v = judge.invoke(
            f"Question: {q}\n\nResponse A:\n{first}\n\nResponse B:\n{second}\n\n"
            f"Which response is more accurate, complete, and appropriately formatted?"
        )
        winner = v.winner
        if winner != "tie":
            is_tuned = (winner == "A") == swapped
            wins["tuned" if is_tuned else "base"] += 1
        else:
            wins["tie"] += 1
    return wins

# ── 3. Catastrophic forgetting check ──
# Run a general benchmark BEFORE and AFTER fine-tuning.
# If MMLU/HellaSwag drops more than a couple points, you over-trained.
# Fixes: lower LR, fewer epochs, mix 5-10% general instruction data into training.
```

```
CATASTROPHIC FORGETTING — the interview question people miss:

  Fine-tuning narrowly on one task degrades unrelated capabilities.
  A model tuned hard on SQL generation may lose conversational ability.

  MITIGATIONS:
    • LoRA inherently forgets less than full FT (base weights are frozen)
    • Lower learning rate and fewer epochs
    • Mix 5-10% general instruction data into your training set
    • Keep a "capability regression suite" and run it every training run
```

---

# Part 7: Serving Fine-Tuned Models

```python
# ── Option A: load base + adapter at runtime (flexible) ──
from peft import PeftModel

base = AutoModelForCausalLM.from_pretrained(MODEL, torch_dtype=torch.bfloat16, device_map="auto")
model = PeftModel.from_pretrained(base, "./adapter")
model.eval()

# Multiple adapters on ONE base model — the big operational win
model.load_adapter("./adapter_support", adapter_name="support")
model.load_adapter("./adapter_sales", adapter_name="sales")
model.set_adapter("support")     # switch per request, ~milliseconds
# One 16GB base in memory + N × 150MB adapters, instead of N × 16GB models.

# ── Option B: merge for maximum inference speed ──
merged = model.merge_and_unload()          # folds BA back into W
merged.save_pretrained("./merged_model")
# No adapter overhead at inference (~5-10% faster), but you lose hot-swapping
# and now store a full-size model per variant.

# ── Serving with vLLM (production) ──
# vllm serve ./merged_model --max-model-len 4096 --gpu-memory-utilization 0.9
#
# Or with LoRA hot-swapping:
# vllm serve meta-llama/Meta-Llama-3.1-8B-Instruct \
#     --enable-lora --lora-modules support=./adapter_support sales=./adapter_sales
```

```
DEPLOYMENT DECISION:
  Merge      → single-purpose model, maximum throughput, simplest serving
  Adapters   → many variants, per-tenant customization, rapid iteration
               (a 150MB adapter deploys in seconds; a 16GB model does not)
```

---

# Part 8: OpenAI Fine-Tuning (Managed Alternative)

```python
from openai import OpenAI
client = OpenAI()

# JSONL, one conversation per line
# {"messages": [{"role":"system",...},{"role":"user",...},{"role":"assistant",...}]}

file = client.files.create(file=open("train.jsonl", "rb"), purpose="fine-tune")

job = client.fine_tuning.jobs.create(
    training_file=file.id,
    validation_file=val_file.id,
    model="gpt-4o-mini-2024-07-18",
    hyperparameters={"n_epochs": 3, "batch_size": "auto", "learning_rate_multiplier": "auto"},
    suffix="support-bot-v1",
)

client.fine_tuning.jobs.retrieve(job.id)
resp = client.chat.completions.create(
    model="ft:gpt-4o-mini-2024-07-18:org:support-bot-v1:abc123",
    messages=[{"role": "user", "content": "..."}],
)
```

```
MANAGED vs SELF-HOSTED:
  OpenAI FT:   no infra, fast to try, but higher per-token cost than base,
               no weight ownership, limited to their base models
  Self-hosted: full control, cheapest at scale, own the weights, any base model,
               but you own the GPUs, serving stack, and ops
```

---

# Part 9: 🧩 Interview Q&A

**Q: When would you fine-tune instead of using RAG?**
A: They solve different problems. RAG injects knowledge — facts, documents, anything that changes. Fine-tuning changes behavior — output format, tone, domain-specific reasoning patterns, or following instructions the base model ignores. Fine-tuning is a poor way to teach facts: the knowledge gets entangled in weights so you can't cite or verify it, updating one fact requires retraining, and the model still hallucinates, just more confidently. The strongest signal for fine-tuning is a prompt that's grown to thousands of tokens of instructions and examples — distilling that into weights cuts per-call cost and latency dramatically. In production I'd often use both: fine-tune for behavior, RAG for knowledge.

**Q: Explain LoRA mathematically and why it works.**
A: Full fine-tuning learns an update ΔW to each weight matrix W. LoRA's insight is that this update has low intrinsic rank — you're nudging an existing function, not learning a new one — so ΔW can be factored as B×A where A is r×d and B is d×r with r much smaller than d. Only A and B are trained; W stays frozen. For a 4096×4096 matrix with r=16, that's 131K trainable parameters instead of 16.7M, under 1%. Crucially, B is initialized to zeros so BA=0 at step zero, meaning training starts from exactly the base model's behavior rather than a random perturbation. The forward pass computes Wx + (B(Ax))·(α/r), where α/r scales the adapter's contribution.

**Q: What is QLoRA and how does it differ from LoRA?**
A: QLoRA quantizes the frozen base model to 4-bit while keeping the LoRA adapters in bf16. Three techniques make it work: NF4, a quantization datatype that's information-theoretically optimal for normally-distributed weights; double quantization, which quantizes the quantization constants for another 0.4 bits per parameter; and paged optimizers, which page optimizer state to CPU memory during spikes to avoid OOM crashes. For Llama-3-8B this takes memory from roughly 18GB with LoRA to about 6GB, so it trains on a single consumer GPU. The quality cost versus standard LoRA is typically under one percent on downstream benchmarks.

**Q: How much data do you need, and what matters more — quantity or quality?**
A: Quality, decisively. A thousand carefully curated examples routinely outperform fifty thousand scraped noisy ones, because the model learns the distribution of your examples including their inconsistencies. Rough minimums: 200-500 examples for format and style adherence, 500-2000 for domain tone, 2000-10000 for complex reasoning, and 10000-plus for genuinely new capabilities. My data pipeline does structural validation, exact deduplication by hash, semantic deduplication by embedding similarity above 0.95, length filtering in both directions, and removal of refusals and boilerplate that leaked in from a teacher model.

**Q: What is catastrophic forgetting and how do you prevent it?**
A: Fine-tuning narrowly on one task degrades unrelated capabilities the model previously had — a model tuned hard on SQL generation may lose conversational fluency. It happens because gradient updates optimized for your narrow distribution overwrite representations that supported other behaviors. Mitigations, in order of effectiveness: use LoRA rather than full fine-tuning, since frozen base weights limit the damage; lower the learning rate and reduce epochs; mix five to ten percent general instruction data into the training set; and maintain a capability regression suite — a general benchmark run before and after every training run so a multi-point drop fails the build rather than shipping silently.

**Q: How do you evaluate a fine-tuned model?**
A: Evaluation loss is nearly useless on its own — a model can have lower perplexity and still be worse at the actual task. I evaluate on four axes. Task-specific metrics on a held-out test set touched exactly once. Pairwise LLM-as-judge comparison against the base model, with response positions randomized to counter the judge's position bias. A regression suite on general benchmarks to catch catastrophic forgetting. And human review of fifty sampled outputs, because automated metrics consistently miss failure modes that are obvious to a reader.

**Q: How do you serve multiple fine-tuned variants efficiently?**
A: Keep the base model loaded once and hot-swap LoRA adapters per request. A 16GB base plus N adapters of roughly 150MB each is vastly cheaper than N full 16GB models, and switching adapters takes milliseconds. vLLM supports this natively with `--enable-lora` and named adapter modules. Merging the adapter into the base weights gives maybe five to ten percent faster inference and simpler serving, but you lose hot-swapping and store a full model per variant — so I merge only for single-purpose high-throughput deployments and keep adapters separate whenever there are multiple variants or frequent iteration.

**Q: Walk me through the full fine-tuning pipeline you'd run in production.**
A: First, justify it — confirm that prompting and RAG genuinely can't solve the problem, and quantify the expected win, usually inference cost. Then build the dataset: collect from production traces or a teacher model, clean and deduplicate, and split into train, validation, and test before any training touches the data. Train with QLoRA on a strong instruct base, targeting attention and MLP projections, using an effective batch size around 32, a learning rate near 2e-4, cosine schedule, and early stopping on validation loss. Evaluate on all four axes including a regression check. If preference data exists, follow SFT with DPO starting from the SFT checkpoint. Serve via adapter hot-swapping behind vLLM, keeping the base model shared. Finally, monitor production outputs and periodically retrain as the data distribution drifts.
