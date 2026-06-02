---
author: Obed Macallums
pubDatetime: 2026-06-01T10:30:00Z
title: "Introduction to Hugging Face Transformers"
slug: huggingface-transformers
featured: false
draft: false
tags:
  - huggingface
  - transformers
  - python
  - nlp
  - machine-learning
  - pytorch
description:
    A practical introduction to the Hugging Face Transformers library. Learn to use the high-level pipeline API for common NLP tasks, understand tokenizers and models, and run a basic fine-tuning workflow with the Trainer.
---

The **Hugging Face Transformers** library has become the de facto standard for working with state-of-the-art machine learning models, especially in Natural Language Processing (NLP). It provides a unified API to download, run, and fine-tune thousands of pretrained models from the **Hugging Face Hub**. In this article we explore the high-level `pipeline()` API, the most common NLP tasks, the tokenizers and models that power them, and a basic fine-tuning workflow.

## Table of Contents

## What Is Hugging Face Transformers?

**Transformers** is an open-source Python library that gives you access to thousands of pretrained models for text, vision, and audio. It abstracts away the complexity of loading model weights, tokenizing inputs, and running inference, while still letting you drop down to lower levels when you need full control.

Two pieces work together:

- **The Transformers library**: the code that defines model architectures, tokenizers, and training utilities.
- **The Hugging Face Hub**: a platform hosting hundreds of thousands of pretrained models and datasets, each identified by a name like `distilbert-base-uncased-finetuned-sst-2-english`.

The library integrates with **PyTorch**, TensorFlow, and JAX, though PyTorch is the most common backend.

## Installation

Install the library with pip. You will typically want a deep learning backend (PyTorch) and a couple of companion libraries:

```bash
# Core library
pip install transformers

# With PyTorch as the backend (recommended)
pip install transformers torch

# Companion libraries for datasets and accelerated training
pip install datasets accelerate
```

You can verify the installation by running a quick inference:

```python
from transformers import pipeline

classifier = pipeline("sentiment-analysis")
print(classifier("I love using the Transformers library!"))
```

The first run downloads the default model and caches it locally for future use.

## The `pipeline()` API

The fastest way to get started is the `pipeline()` function. It wraps three steps — preprocessing (tokenization), model inference, and postprocessing — behind a single call. You specify the **task**, and Hugging Face selects a sensible default model:

```python
from transformers import pipeline

# Create a pipeline for a specific task
classifier = pipeline("sentiment-analysis")

result = classifier("This tutorial is clear and helpful.")
print(result)
# [{'label': 'POSITIVE', 'score': 0.9998}]
```

You can also choose a specific model from the Hub by passing its name:

```python
classifier = pipeline(
    "sentiment-analysis",
    model="distilbert-base-uncased-finetuned-sst-2-english"
)
```

## Common NLP Tasks

The `pipeline()` API supports many tasks out of the box. Here is one example for each of the most common ones.

### Sentiment Analysis

```python
classifier = pipeline("sentiment-analysis")
print(classifier("The service was slow and disappointing."))
# [{'label': 'NEGATIVE', 'score': 0.9994}]
```

### Named Entity Recognition (NER)

```python
ner = pipeline("ner", grouped_entities=True)
print(ner("Hugging Face is based in New York City."))
# [{'entity_group': 'ORG', 'word': 'Hugging Face', ...},
#  {'entity_group': 'LOC', 'word': 'New York City', ...}]
```

### Question Answering

```python
qa = pipeline("question-answering")
result = qa(
    question="Where is the Eiffel Tower located?",
    context="The Eiffel Tower is a landmark located in Paris, France."
)
print(result)
# {'answer': 'Paris, France', 'score': 0.97, ...}
```

### Summarization

```python
summarizer = pipeline("summarization")
text = """Hugging Face Transformers provides thousands of pretrained
models to perform tasks on different modalities such as text, vision,
and audio. These models can be applied to a wide range of use cases."""
print(summarizer(text, max_length=30, min_length=10))
```

### Translation

```python
translator = pipeline("translation_en_to_fr")
print(translator("Machine learning is fascinating."))
# [{'translation_text': "L'apprentissage automatique est fascinant."}]
```

### Text Generation

```python
generator = pipeline("text-generation", model="gpt2")
print(generator("In the future, artificial intelligence will",
                max_length=30, num_return_sequences=1))
```

## Under the Hood: Tokenizers and Models

The `pipeline()` API is convenient, but understanding the two components it hides — the **tokenizer** and the **model** — gives you much more flexibility.

A **tokenizer** converts raw text into the numeric token IDs a model expects, and back again. A **model** takes those IDs and produces predictions. The `AutoTokenizer` and `AutoModel` classes load the right implementation automatically based on the model name:

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

model_name = "distilbert-base-uncased-finetuned-sst-2-english"

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)

# Tokenize the input text
inputs = tokenizer("I really enjoyed this movie!", return_tensors="pt")
print(inputs)
# {'input_ids': tensor([[...]]), 'attention_mask': tensor([[...]])}

# Run the model
with torch.no_grad():
    logits = model(**inputs).logits

# Convert logits to predicted label
predicted_class = logits.argmax().item()
print(model.config.id2label[predicted_class])
# POSITIVE
```

This is exactly what the sentiment-analysis pipeline does internally: tokenize, run the model, and map the highest-scoring logit back to a human-readable label.

## Basic Fine-tuning

Pretrained models are powerful, but you often get better results by **fine-tuning** them on your own data. The `Trainer` API handles the training loop for you. The example below fine-tunes a model for text classification using a dataset from the `datasets` library.

```python
from datasets import load_dataset
from transformers import (
    AutoTokenizer,
    AutoModelForSequenceClassification,
    TrainingArguments,
    Trainer,
)

# 1. Load a dataset from the Hub
dataset = load_dataset("imdb")

# 2. Load a tokenizer and tokenize the text
model_name = "distilbert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(model_name)

def tokenize(batch):
    return tokenizer(batch["text"], padding="max_length", truncation=True)

tokenized = dataset.map(tokenize, batched=True)

# Use small subsets to keep the example fast
train_dataset = tokenized["train"].shuffle(seed=42).select(range(1000))
eval_dataset = tokenized["test"].shuffle(seed=42).select(range(500))

# 3. Load the model with a classification head (2 labels)
model = AutoModelForSequenceClassification.from_pretrained(
    model_name, num_labels=2
)

# 4. Define the training arguments
training_args = TrainingArguments(
    output_dir="./results",
    eval_strategy="epoch",
    learning_rate=2e-5,
    per_device_train_batch_size=8,
    num_train_epochs=2,
)

# 5. Create the Trainer and train
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
)

trainer.train()
```

After training, you can save the fine-tuned model and reuse it later — or push it to the Hub to share with others:

```python
# Save locally
trainer.save_model("./my-finetuned-model")

# Reload it in a pipeline
from transformers import pipeline
classifier = pipeline("sentiment-analysis", model="./my-finetuned-model")
```

## Conclusion

Hugging Face Transformers lowers the barrier to using state-of-the-art models dramatically. The `pipeline()` API lets you solve common NLP tasks in a few lines of code, while `AutoTokenizer` and `AutoModel` give you fine-grained control when you need it. When pretrained models are not enough, the `Trainer` API makes fine-tuning on your own data straightforward.

From here, natural next steps include exploring the thousands of models on the Hub, experimenting with vision and audio tasks, and combining Transformers with the broader PyTorch ecosystem to build and deploy your own machine learning applications.
