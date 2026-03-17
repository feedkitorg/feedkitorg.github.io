---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Political Content Classifier
description: A fast, accurate classifier that detects whether a social media post is about politics. Designed for researchers running feed experiments or analyzing social media data at scale.


# # Author box
# author:
#     title: About Author
#     title_url: '#'
#     external_url: true
#     description: Author description

# Micro navigation
micro_nav: true

# Page navigation
# page_nav:
#     prev:
#         content: Previous page
#         url: '#'
#     next:
#         content: Next page
#         url: '#'
---

## What it does

Given a social media post, the model returns a probability that the post is political. It uses a broad definition of political content from the [Pew Research Center](https://www.pewresearch.org/):

> *Political content includes mentions of officials and activists, social issues, news, and current events.*

This covers a wide range of posts — from election news and policy debates to social movements and current events — while excluding entertainment, personal updates, and other non-political content.

## Where it comes from

The classifier was developed as part of a preregistered field experiment on X (Twitter) involving 1,256 participants during the 2024 U.S. presidential campaign. In the experiment, the model ran in real time inside a browser extension to identify political posts before applying a second-stage content scorer. It needed to be fast (under 500 ms per batch on GPU) and accurate enough to reliably separate political from non-political content in naturalistic feeds.

The model is a fine-tuned version of [`cardiffnlp/twitter-roberta-base`](https://huggingface.co/cardiffnlp/twitter-roberta-base) — a RoBERTa model pretrained on 58 million tweets, making it well-suited for the informal language of social media posts.

## Training

**Data.** The model was trained on 50,000 English posts collected from X in July 2023. Posts were sampled from the reverse-chronological feeds of 300 users spanning six political-leaning strata, defined by their follow patterns toward U.S. Congress members (from far-left to far-right). Bot accounts were filtered out using the Botometer API. The dataset was split 80% train / 10% validation / 10% test, stratified by class.

**Labels.** Posts were annotated by GPT-5.2 using the Pew Research Center political content definition. The same annotation prompt was used in the original Science paper (which used GPT-4); this release uses GPT-5.2 labels on the same training corpus.

**Fine-tuning.** Key settings: maximum sequence length 256 tokens (covers Bluesky, Facebook, and thread-style content), weighted cross-entropy loss to handle the 34%/66% class imbalance, AdamW optimizer with a cosine learning rate schedule, and early stopping on validation Macro-F1.

## Performance

Evaluated on the held-out test set of 5,000 posts:

| Metric | Value |
|---|---|
| ROC-AUC | 0.983 |
| Average Precision | 0.973 |
| Macro F1 | 0.934 |
| Default threshold | 0.5 |
| Optimal threshold (F1-maximizing) | 0.67 |

The model produces highly confident predictions: the vast majority of posts receive a probability close to 0 or close to 1, with very few ambiguous cases in between.

### Confidence score

The model returns a **confidence score** — the raw probability P(political) — alongside the binary label. This is a value between 0 and 1: a score close to 1 means the model is highly confident the post is political, a score close to 0 means it is highly confident it is not. The binary label (`is_politics`) is derived by applying a threshold to this probability: posts above the threshold are labeled political, posts below are labeled non-political. Because the model produces well-separated probabilities, confidence scores can also be used directly as a continuous measure of "how political" a post is, which is useful for ranking or filtering by degree rather than applying a hard cutoff.

## Download

```
https://piccardi-jhu.s3.us-east-1.amazonaws.com/roberta-politics-base.zip
```

The zip contains the model weights, tokenizer files, and a `model_metadata.json` file with the optimal threshold and test metrics.

---

## How to use it

### 1. Install dependencies

```bash
pip install transformers torch
```

### 2. Download and unzip the model

```bash
wget https://piccardi-jhu.s3.us-east-1.amazonaws.com/roberta-politics-base.zip
unzip roberta-politics-base.zip
```

This creates a `roberta-politics-base/` directory with the model weights and tokenizer.

### 3. Load the model

```python
import torch
from transformers import AutoTokenizer, AutoModelForSequenceClassification

MODEL_DIR = "roberta-politics-base"

device = torch.device("cuda" if torch.cuda.is_available() else
                      "mps"  if torch.backends.mps.is_available() else "cpu")

tokenizer = AutoTokenizer.from_pretrained(MODEL_DIR)
model = AutoModelForSequenceClassification.from_pretrained(MODEL_DIR).to(device)
model.eval()
print(f"Model loaded on {device}")
```

### 4. Define the predict function

```python
def predict(texts, threshold=0.5):
    """
    Classify one or more social media posts.

    Parameters
    ----------
    texts : str or list of str
        One post or a list of posts to classify.
    threshold : float
        Minimum P(political) to label a post as political.
        Use 0.5 (default) for balanced precision/recall,
        or 0.67 (optimal) for higher precision.

    Returns
    -------
    list of dicts, each with:
        'label'        : 'political' or 'non-political'
        'is_politics'  : 1 or 0
        'confidence'   : float, P(political) in [0, 1]
    """
    if isinstance(texts, str):
        texts = [texts]

    inputs = tokenizer(
        texts,
        truncation=True,
        padding=True,
        max_length=256,
        return_tensors="pt",
    ).to(device)

    with torch.no_grad():
        logits = model(**inputs).logits

    probs = torch.softmax(logits, dim=-1)[:, 1].cpu().tolist()

    return [
        {
            "label"      : "political" if p >= threshold else "non-political",
            "is_politics": int(p >= threshold),
            "confidence" : round(p, 4),
        }
        for p in probs
    ]
```

### 5. Classify posts

**Single post:**

```python
result = predict("The Senate voted today to pass the new climate bill.")
print(result[0])
# {'label': 'political', 'is_politics': 1, 'confidence': 0.9996}
```

**Batch of posts:**

```python
posts = [
    "The Prime Minister announced new austerity measures.",
    "Just finished the best book I've read all year!",
    "Protests erupt as lawmakers debate immigration reform.",
    "My sourdough starter is finally active after 5 days 🍞",
]

results = predict(posts)

for post, r in zip(posts, results):
    print(f"{r['label']:<15} ({r['confidence']:.2f})  {post[:60]}")

# political       (1.00)  The Prime Minister announced new austerity measures.
# non-political   (0.00)  Just finished the best book I've read all year!
# political       (1.00)  Protests erupt as lawmakers debate immigration reform.
# non-political   (0.00)  My sourdough starter is finally active after 5 days 🍞
```

**Large dataset (CSV):**

For large files, process in chunks to avoid memory issues.

```python
import pandas as pd

df = pd.read_csv("posts.csv")  # must have a column named 'text'

CHUNK_SIZE = 64
all_results = []

for i in range(0, len(df), CHUNK_SIZE):
    batch = df["text"].iloc[i : i + CHUNK_SIZE].tolist()
    all_results.extend(predict(batch))

df["is_politics"] = [r["is_politics"] for r in all_results]
df["confidence"]  = [r["confidence"]  for r in all_results]

df.to_csv("posts_classified.csv", index=False)
```

### Using the optimal threshold

The training notebook saves an F1-maximizing threshold alongside the model weights. Use it instead of the default 0.5 when you want higher precision — for example, when the classifier feeds into a more expensive downstream model.

```python
import json

with open("roberta-politics-base/model_metadata.json") as f:
    metadata = json.load(f)

threshold = metadata["optimal_threshold"]  # 0.67
print(f"Optimal threshold : {threshold}")
print(f"Test Macro-F1     : {metadata['test_metrics']['macro_f1']:.4f}")
print(f"Test ROC-AUC      : {metadata['test_metrics']['roc_auc']:.4f}")

results = predict(posts, threshold=threshold)
```

---

## Choosing a threshold

| Threshold | When to use |
|---|---|
| **0.5** | General use; balanced precision and recall |
| **0.67** | Prefiltering before a slower downstream model (e.g., LLM-based scorer); reduces false positives |
| **Custom** | Tune on your own labeled sample if your platform or topic distribution differs from X |

---

## Citation

If you use this classifier in your research, please cite:

```bibtex
@article{piccardi2025reranking,
  title   = {Reranking partisan animosity in algorithmic social media feeds
             alters affective polarization},
  author  = {Piccardi, Tiziano and Saveski, Martin and Jia, Chenyan and
             Hancock, Jeffrey T. and Tsai, Jeanne and Bernstein, Michael},
  journal = {Science},
  year    = {2025},
  doi     = {10.1126/science.adu5584},
  url     = {https://www.science.org/doi/10.1126/science.adu5584}
}
```
