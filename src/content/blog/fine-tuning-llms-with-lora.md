---
author: Obed Macallums
pubDatetime: 2026-06-28T10:00:00Z
title: "Fine-tuning LLMs with LoRA: A Practical Guide"
slug: fine-tuning-llms-with-lora
featured: false
draft: false
tags:
  - python
  - machine-learning
  - llm
  - lora
  - peft
  - huggingface
  - fine-tuning
description:
  A practical guide to fine-tuning Large Language Models with LoRA (Low-Rank Adaptation). Learn how parameter-efficient fine-tuning works, why it dramatically reduces compute and memory, and how to implement it end-to-end using Hugging Face PEFT and Transformers, including QLoRA for consumer GPUs.
---

Fine-tuning a Large Language Model (LLM) used to mean updating billions of parameters, an effort that demanded expensive multi-GPU clusters and terabytes of memory. **LoRA (Low-Rank Adaptation)** changes that equation. By freezing the original model and training only a tiny set of new parameters, LoRA makes it possible to specialize massive models on a single GPU. In this guide we explore how LoRA works, why it is so efficient, and how to implement it end-to-end using the Hugging Face `peft` and `transformers` libraries.

## Table of Contents

## What Is Fine-tuning?

Pre-trained LLMs such as Llama, Mistral, or GPT-style models learn general language patterns from massive corpora. **Fine-tuning** adapts that general knowledge to a specific task or domain, for example legal document classification, customer-support responses, or code generation in a particular style.

Traditional **full fine-tuning** updates every weight in the model. For a 7-billion-parameter model this means:

- Storing optimizer states (often 2x the model size in memory).
- Keeping gradients for every parameter.
- Producing a full copy of the model for each task you train.

This quickly becomes impractical. A single 7B model in 16-bit precision already needs around 14 GB just to hold the weights, and full fine-tuning can push memory requirements past 60-80 GB. This is the problem **parameter-efficient fine-tuning (PEFT)** methods solve, and LoRA is the most popular among them.

## How LoRA Works

LoRA is built on a simple but powerful observation: the **weight updates** learned during fine-tuning have a low **intrinsic rank**. In other words, the change applied to a large weight matrix can be approximated well by the product of two much smaller matrices.

Consider a pre-trained weight matrix `W` of shape `(d, k)`. Instead of updating `W` directly, LoRA freezes it and learns an additive update `ΔW` expressed as a low-rank decomposition:

```text
W_new = W + ΔW = W + (B · A)
```

Where:

- `A` has shape `(r, k)`
- `B` has shape `(d, r)`
- `r` is the **rank**, and `r << min(d, k)`

Only `A` and `B` are trained, while `W` stays frozen. Because `r` is small (typically 4, 8, 16, or 32), the number of trainable parameters drops dramatically, often to **less than 1%** of the original model.

During the forward pass, the output combines the frozen path and the low-rank path:

```text
h = W · x + (B · A) · x · (α / r)
```

The scalar `α` (alpha) is a scaling factor that controls how strongly the LoRA update influences the model. A common heuristic is to set `alpha` to twice the rank.

### Why This Is So Efficient

LoRA delivers several practical advantages:

- **Tiny trainable footprint**: only the `A` and `B` matrices receive gradients, so optimizer states and gradients shrink accordingly.
- **No inference latency when merged**: after training, `B · A` can be merged back into `W`, producing a standard model with zero extra cost at inference time.
- **Portable adapters**: the trained weights are just a few megabytes. You can keep one base model and swap many task-specific adapters on top of it.
- **Reduced risk of catastrophic forgetting**: since the base weights stay frozen, the model retains its general capabilities.

## Setting Up the Environment

To follow the practical examples, install the core libraries from the Hugging Face ecosystem:

```bash
pip install transformers peft datasets accelerate
```

For QLoRA (covered later), you will also need `bitsandbytes`:

```bash
pip install bitsandbytes
```

> Note: `bitsandbytes` requires a CUDA-capable GPU for its 4-bit and 8-bit kernels.

## Fine-tuning with LoRA: A Practical Example

Let's walk through a complete fine-tuning workflow. We will load a base model, attach a LoRA configuration, train it on a dataset, and save the resulting adapter.

### Loading the Base Model and Tokenizer

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "mistralai/Mistral-7B-v0.1"

tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.bfloat16,
    device_map="auto",
)
```

### Configuring LoRA

The `LoraConfig` object defines the rank, scaling factor, dropout, and which modules to adapt. For decoder-only LLMs, the attention projection layers (`q_proj`, `k_proj`, `v_proj`, `o_proj`) are the most common targets.

```python
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,                       # rank of the update matrices
    lora_alpha=32,              # scaling factor (commonly 2 * r)
    target_modules=[
        "q_proj", "k_proj", "v_proj", "o_proj",
    ],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
```

The `print_trainable_parameters()` call reveals the efficiency of LoRA. A typical output looks like this:

```text
trainable params: 13,631,488 || all params: 7,255,453,696 || trainable%: 0.1879
```

We are training fewer than **0.2%** of the parameters.

### Preparing the Dataset

For this example we use a small instruction-style dataset. The key step is tokenizing each example into a format the model can learn from.

```python
from datasets import load_dataset

dataset = load_dataset("databricks/databricks-dolly-15k", split="train")

def format_example(example):
    prompt = (
        f"### Instruction:\n{example['instruction']}\n\n"
        f"### Response:\n{example['response']}"
    )
    return {"text": prompt}

dataset = dataset.map(format_example)

def tokenize(example):
    tokens = tokenizer(
        example["text"],
        truncation=True,
        max_length=512,
        padding="max_length",
    )
    tokens["labels"] = tokens["input_ids"].copy()
    return tokens

tokenized_dataset = dataset.map(tokenize, remove_columns=dataset.column_names)
```

### Training the Adapter

We use the standard Hugging Face `Trainer`. Because only the LoRA parameters are trainable, this loop is far lighter than full fine-tuning.

```python
from transformers import Trainer, TrainingArguments, DataCollatorForLanguageModeling

training_args = TrainingArguments(
    output_dir="./lora-mistral",
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    num_train_epochs=1,
    learning_rate=2e-4,
    bf16=True,
    logging_steps=10,
    save_strategy="epoch",
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_dataset,
    data_collator=DataCollatorForLanguageModeling(tokenizer, mlm=False),
)

trainer.train()
```

### Saving and Loading the Adapter

After training, save only the adapter weights. They are typically just a few megabytes.

```python
model.save_pretrained("./lora-mistral-adapter")
```

To use the adapter later, load the base model again and attach the saved weights:

```python
from peft import PeftModel

base_model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.bfloat16,
    device_map="auto",
)

model = PeftModel.from_pretrained(base_model, "./lora-mistral-adapter")
```

### Merging for Inference

If you want a standalone model with no PEFT dependency at inference time, merge the adapter into the base weights:

```python
merged_model = model.merge_and_unload()
merged_model.save_pretrained("./mistral-merged")
```

The merged model behaves exactly like a normal Transformers model, with no additional latency.

## QLoRA: Fine-tuning on Consumer GPUs

LoRA already reduces the trainable parameters, but the **frozen base model** still occupies significant memory. A 7B model in 16-bit precision needs around 14 GB just to be loaded, which can exceed the capacity of consumer GPUs.

**QLoRA (Quantized LoRA)** solves this by loading the frozen base model in **4-bit precision** while keeping the LoRA adapters in higher precision for training. This combination makes it possible to fine-tune models that would otherwise not fit, sometimes training a 7B or even 13B model on a single 16 GB GPU.

QLoRA introduces three key techniques:

- **4-bit NormalFloat (NF4)**: a quantization data type optimized for the normally distributed weights of neural networks.
- **Double quantization**: quantizing the quantization constants themselves to save additional memory.
- **Paged optimizers**: using NVIDIA unified memory to handle memory spikes gracefully.

### QLoRA in Practice

The workflow is nearly identical to standard LoRA. The main difference is the `BitsAndBytesConfig` used when loading the model, plus a preparation step.

```python
import torch
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import prepare_model_for_kbit_training, LoraConfig, get_peft_model

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)

model = AutoModelForCausalLM.from_pretrained(
    "mistralai/Mistral-7B-v0.1",
    quantization_config=bnb_config,
    device_map="auto",
)

# Prepare the quantized model for training
model = prepare_model_for_kbit_training(model)

lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)

model = get_peft_model(model, lora_config)
```

From here, the `Trainer` setup and training loop are exactly the same as in the LoRA example. The only conceptual difference is that the base model lives in 4-bit memory while gradients flow only through the higher-precision LoRA adapters.

## Choosing Good Hyperparameters

A few practical guidelines help you get strong results:

- **Rank (`r`)**: start with 8 or 16. Higher ranks add capacity but also more parameters. Increase only if the model underfits.
- **Alpha (`lora_alpha`)**: a common starting point is `2 * r`. It scales the magnitude of the LoRA update.
- **Target modules**: adapting only attention projections is a solid default. For harder tasks, you can also target the MLP layers (`gate_proj`, `up_proj`, `down_proj`).
- **Learning rate**: LoRA tolerates higher learning rates than full fine-tuning. Values around `1e-4` to `3e-4` work well.
- **Dropout**: a small `lora_dropout` (0.05-0.1) helps regularize when data is limited.

## LoRA vs. Full Fine-tuning

| Aspect                 | Full Fine-tuning            | LoRA / QLoRA                     |
| ---------------------- | --------------------------- | -------------------------------- |
| Trainable parameters   | 100%                        | < 1%                             |
| GPU memory             | Very high                   | Low to moderate                  |
| Storage per task       | Full model copy             | A few MB per adapter             |
| Inference latency      | Baseline                    | Baseline (after merging)         |
| Catastrophic forgetting| Higher risk                 | Lower risk (base stays frozen)   |
| Best for               | Large datasets, deep shifts | Most task and domain adaptations |

For the vast majority of practical use cases, LoRA and QLoRA deliver results comparable to full fine-tuning at a fraction of the cost.

## Conclusion

LoRA has fundamentally lowered the barrier to customizing Large Language Models. By exploiting the low intrinsic rank of weight updates, it trains a tiny fraction of parameters while keeping the powerful base model frozen. The Hugging Face `peft` library makes this approach straightforward to adopt, and **QLoRA** extends it even further by enabling fine-tuning of large models on consumer-grade hardware through 4-bit quantization.

Whether you are adapting a model to a niche domain, teaching it a specific response style, or building multiple specialized adapters on top of a shared base, LoRA offers an efficient, portable, and production-friendly path. With just a few megabytes of trained weights, you can turn a general-purpose LLM into a focused expert, without ever touching the original billions of parameters.
