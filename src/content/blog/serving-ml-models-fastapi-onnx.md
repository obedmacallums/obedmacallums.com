---
author: Obed Macallums
pubDatetime: 2026-07-07T10:00:00Z
title: "Serving ML Models in Production with FastAPI + ONNX"
slug: serving-ml-models-fastapi-onnx
featured: false
draft: false
tags:
  - fastapi
  - onnx
  - machine-learning
  - python
  - nlp
  - mlops
description: Learn how to serve machine learning models in production using FastAPI and ONNX Runtime. Export a Hugging Face NLP model to ONNX, build a REST inference endpoint, benchmark performance against native PyTorch, and containerize the service with Docker.
---

Training a model in a notebook is only half the job. Getting it to answer real HTTP requests reliably, quickly, and without dragging a multi-gigabyte PyTorch install into every container is a different problem entirely. In this article we build a small but production-shaped inference service: we export a Hugging Face NLP model to **ONNX**, serve it through a **FastAPI** endpoint, compare its performance against native PyTorch, and package everything with **Docker**.

## Table of Contents

## From Notebook to Production

A model that works in a Jupyter notebook has none of the constraints a production service has to deal with:

- **Latency**: users expect a response in milliseconds, not seconds.
- **Throughput**: a single instance may need to handle many concurrent requests.
- **Footprint**: a full PyTorch + CUDA toolchain is large and slow to cold-start, which hurts container startup time and cost.
- **Stability**: the serving layer needs input validation, predictable errors, and observability.

ONNX and FastAPI address these concerns from two different angles: ONNX optimizes _how_ the model runs, and FastAPI structures _how_ it's exposed.

## Why ONNX?

**ONNX** (Open Neural Network Exchange) is an open format for representing machine learning models. Instead of shipping a PyTorch model together with the full PyTorch runtime, you export it once to a `.onnx` file and run it with **ONNX Runtime**, a lightweight, highly optimized inference engine.

The practical benefits for serving:

- **Portability**: the same `.onnx` file can run on ONNX Runtime regardless of whether the model was originally built with PyTorch, TensorFlow, or scikit-learn.
- **Performance**: ONNX Runtime applies graph-level optimizations (operator fusion, constant folding) and can use execution providers tuned for CPU or GPU, which often results in lower latency than eager-mode PyTorch inference.
- **Smaller runtime**: ONNX Runtime's Python package is considerably lighter than a full PyTorch installation, which means smaller Docker images and faster cold starts.

The trade-off is a conversion step: not every custom layer or dynamic control flow exports cleanly, so it's worth validating outputs after conversion.

## Why FastAPI?

**FastAPI** is a natural fit for wrapping a model as a service:

- **Async support**: request handling doesn't block on I/O, which matters once you add logging, metrics, or downstream calls.
- **Pydantic validation**: request and response schemas are declared as typed models, so malformed input is rejected before it reaches the model.
- **Automatic docs**: every endpoint gets an interactive Swagger UI for free, which is useful both for testing and for other teams consuming the API.

Together, ONNX handles the "make inference fast" problem and FastAPI handles the "expose it safely" problem.

## Exporting a Hugging Face Model to ONNX

We'll use `distilbert-base-uncased-finetuned-sst-2-english`, a small sentiment analysis model, as our example. Hugging Face's [`optimum`](https://huggingface.co/docs/optimum) library provides a direct export path to ONNX.

```bash
pip install "optimum[onnxruntime]" transformers fastapi uvicorn
```

Export the model with the `optimum-cli`:

```bash
optimum-cli export onnx \
  --model distilbert-base-uncased-finetuned-sst-2-english \
  --task text-classification \
  onnx_model/
```

This downloads the model, converts it to ONNX, and writes the result (model graph, weights, and tokenizer files) into `onnx_model/`. You can sanity-check the export directly in Python:

```python
from optimum.onnxruntime import ORTModelForSequenceClassification
from transformers import AutoTokenizer, pipeline

tokenizer = AutoTokenizer.from_pretrained("onnx_model")
model = ORTModelForSequenceClassification.from_pretrained("onnx_model")

classifier = pipeline("text-classification", model=model, tokenizer=tokenizer)
print(classifier("This library made deployment so much easier."))
# [{'label': 'POSITIVE', 'score': 0.9998}]
```

`ORTModelForSequenceClassification` is a drop-in replacement for the usual `AutoModelForSequenceClassification`, except inference runs on ONNX Runtime instead of PyTorch.

## Building the Inference Endpoint with FastAPI

With the exported model in place, wrapping it in an API is straightforward. We define typed request/response models with Pydantic and load the model once at startup, not per request.

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel
from optimum.onnxruntime import ORTModelForSequenceClassification
from transformers import AutoTokenizer, pipeline

MODEL_DIR = "onnx_model"

tokenizer = AutoTokenizer.from_pretrained(MODEL_DIR)
model = ORTModelForSequenceClassification.from_pretrained(MODEL_DIR)
classifier = pipeline("text-classification", model=model, tokenizer=tokenizer)

app = FastAPI(title="Sentiment Analysis API")


class TextInput(BaseModel):
    text: str


class Prediction(BaseModel):
    label: str
    score: float


@app.post("/predict", response_model=Prediction)
def predict(payload: TextInput) -> Prediction:
    result = classifier(payload.text)[0]
    return Prediction(label=result["label"], score=result["score"])


@app.get("/health")
def health() -> dict:
    return {"status": "ok"}
```

Run it with:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

And call it:

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "FastAPI and ONNX make a great combo."}'
```

```json
{ "label": "POSITIVE", "score": 0.9997 }
```

Because the model and tokenizer are loaded once at import time, `/predict` only pays the cost of tokenization and a forward pass, not model loading.

## Benchmarking: PyTorch vs ONNX Runtime

To see whether the conversion was worth it, we can compare the ONNX pipeline against the equivalent PyTorch pipeline on the same inputs:

```python
import time
from transformers import AutoModelForSequenceClassification, AutoTokenizer, pipeline
from optimum.onnxruntime import ORTModelForSequenceClassification

MODEL_NAME = "distilbert-base-uncased-finetuned-sst-2-english"
text = "Benchmarking inference performance is essential before shipping to production."
n_runs = 50

def benchmark(clf, n=n_runs):
    clf(text)  # warm-up run, excluded from timing
    start = time.perf_counter()
    for _ in range(n):
        clf(text)
    elapsed = time.perf_counter() - start
    return (elapsed / n) * 1000  # ms per inference

tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)

pt_model = AutoModelForSequenceClassification.from_pretrained(MODEL_NAME)
pt_classifier = pipeline("text-classification", model=pt_model, tokenizer=tokenizer)

onnx_model = ORTModelForSequenceClassification.from_pretrained(
    "onnx_model"
)
onnx_classifier = pipeline("text-classification", model=onnx_model, tokenizer=tokenizer)

print(f"PyTorch:  {benchmark(pt_classifier):.2f} ms/inference")
print(f"ONNX:     {benchmark(onnx_classifier):.2f} ms/inference")
```

On a typical CPU-only machine, results look roughly like this (your numbers will vary with hardware, batch size, and sequence length):

| Backend         | Avg. latency (single input, CPU) |
| --------------- | -------------------------------- |
| PyTorch (eager) | ~18 ms                           |
| ONNX Runtime    | ~9 ms                            |

ONNX Runtime commonly cuts CPU latency by close to half for small transformer models like DistilBERT, mostly thanks to graph optimizations and a leaner execution path. The gap tends to shrink on GPU, where PyTorch's eager mode is already close to hardware limits, but the smaller runtime footprint remains valuable for deployment.

## Dockerizing the Service

To ship this as a portable service, we containerize the FastAPI app together with the exported ONNX model.

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY onnx_model/ ./onnx_model/
COPY main.py .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```text
# requirements.txt
fastapi
uvicorn[standard]
optimum[onnxruntime]
transformers
```

Build and run it:

```bash
docker build -t sentiment-api .
docker run -p 8000:8000 sentiment-api
```

Because we install `optimum[onnxruntime]` instead of full PyTorch, and bake in the already-exported ONNX model rather than downloading and converting it at container startup, the resulting image is smaller and starts faster than an equivalent PyTorch-based service — a meaningful difference when running multiple replicas behind a load balancer.

## Conclusion

FastAPI and ONNX solve complementary problems when moving a model from notebook to production: ONNX Runtime gives you a lighter, faster inference engine, and FastAPI gives you a typed, documented, async-ready way to expose it over HTTP. The pattern shown here — export once, load once at startup, validate inputs with Pydantic, containerize the result — scales to most single-model services.

From here, natural next steps include dynamic batching for higher throughput, GPU execution providers for larger models, and adding structured logging and metrics for observability in production. Those are worth their own deep dive, but the foundation above is enough to take a model from a notebook experiment to a service you can actually deploy.
