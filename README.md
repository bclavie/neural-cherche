<div align="center">
  <h1>SparsEmbed - Splade</h1>
  <p>Neural search</p>
</div>

This repository presents a replication of both models Splade and SparseEmbed:

- *[SPLADE: Sparse Lexical and Expansion Model for First Stage Ranking](https://arxiv.org/abs/2107.05720)* authored by Thibault Formal, Benjamin Piwowarski, Stéphane Clinchant, SIGIR 2021.

- *[SPLADE v2: Sparse Lexical and Expansion Model for Information Retrieval](https://arxiv.org/abs/2109.10086)* authored by Thibault Formal, Carlos Lassance, Benjamin Piwowarski, Stéphane Clinchant, SIGIR 2022.

- *[SparseEmbed: Learning Sparse Lexical Representations with Contextual Embeddings for Retrieval](https://research.google/pubs/pub52289/)* authored by Weize Kong, Jeffrey M. Dudek, Cheng Li, Mingyang Zhang, and Mike Bendersky, SIGIR 2023.

## Installation

We can install sparsembed using:

```
pip install sparsembed
```

If we plan to evaluate our model while training install:

```
pip install "sparsembed[eval]"
```

## Retriever

### Splade

We can initialize a Splade Retriever directly from the `splade_v2_max` checkpoint available on HuggingFace. Retrievers are based on PyTorch sparse matrices, stored in memory and accelerated with GPU. We can reduce the number of activated tokens via the `n_tokens` parameter in order to reduce the memory usage of those sparse matrices. 

```python
from sparsembed import model, retrieve
from transformers import AutoModelForMaskedLM, AutoTokenizer

device = "cuda" # cpu

batch_size = 10

# List documents to index:
documents = [
 {'id': 0,
  'title': 'Paris',
  'url': 'https://en.wikipedia.org/wiki/Paris',
  'text': 'Paris is the capital and most populous city of France.'},
 {'id': 1,
  'title': 'Paris',
  'url': 'https://en.wikipedia.org/wiki/Paris',
  'text': "Since the 17th century, Paris has been one of Europe's major centres of science, and arts."},
 {'id': 2,
  'title': 'Paris',
  'url': 'https://en.wikipedia.org/wiki/Paris',
  'text': 'The City of Paris is the centre and seat of government of the region and province of Île-de-France.'
}]

model = model.Splade(
    model=AutoModelForMaskedLM.from_pretrained("naver/splade_v2_max").to(device),
    tokenizer=AutoTokenizer.from_pretrained("naver/splade_v2_max"),
    device=device
)

retriever = retrieve.SpladeRetriever(
    key="id", # Key identifier of each document.
    on=["title", "text"], # Fields to search.
    model=model # Splade retriever.
)

retriever = retriever.add(
    documents=documents,
    batch_size=batch_size,
    k_tokens=256, # Number of activated tokens.
)

retriever(
    ["paris", "Toulouse"], # Queries 
    k_tokens=20, # Maximum number of activated tokens.
    k=100, # Number of documents to retrieve.
    batch_size=batch_size
)
```

```python
[[{'id': 0, 'similarity': 11.481657981872559},
  {'id': 2, 'similarity': 11.294965744018555},
  {'id': 1, 'similarity': 10.059721946716309}],
 [{'id': 0, 'similarity': 0.7379149198532104},
  {'id': 2, 'similarity': 0.6973429918289185},
  {'id': 1, 'similarity': 0.5428210496902466}]]
```

### SparsEmbed 

We can also initialize a retriever dedicated to SparseEmbed model. The checkpoint `naver/splade_v2_max` is not a SparseEmbed trained model so we should train one before using it as a retriever.

```python
from sparsembed import model, retrieve
from transformers import AutoModelForMaskedLM, AutoTokenizer

device = "cuda" # cpu

batch_size = 10

# List documents to index:
documents = [
 {'id': 0,
  'title': 'Paris',
  'url': 'https://en.wikipedia.org/wiki/Paris',
  'text': 'Paris is the capital and most populous city of France.'},
 {'id': 1,
  'title': 'Paris',
  'url': 'https://en.wikipedia.org/wiki/Paris',
  'text': "Since the 17th century, Paris has been one of Europe's major centres of science, and arts."},
 {'id': 2,
  'title': 'Paris',
  'url': 'https://en.wikipedia.org/wiki/Paris',
  'text': 'The City of Paris is the centre and seat of government of the region and province of Île-de-France.'
}]

model = model.SparsEmbed(
    model=AutoModelForMaskedLM.from_pretrained("naver/splade_v2_max").to(device),
    tokenizer=AutoTokenizer.from_pretrained("naver/splade_v2_max"),
    device=device
)

retriever = retrieve.SparsEmbedRetriever(
    key="id", # Key identifier of each document.
    on=["title", "text"], # Fields to search.
    model=model # Splade retriever.
)

retriever = retriever.add(
    documents=documents,
    batch_size=batch_size,
    k_tokens=256, # Number of activated tokens.
)

retriever(
    ["paris", "Toulouse"], # Queries 
    k_tokens=20, # Maximum number of activated tokens.
    k=100, # Number of documents to retrieve.
    batch_size=batch_size
)
```

## Training

Let's fine-tune Splade and SparsEmbed.

### Dataset

Your training dataset must be made out of triples `(anchor, positive, negative)` where anchor is a query, positive is a document that is directly linked to the anchor and negative is a document that is not relevant for the anchor.

```python
X = [
    ("anchor 1", "positive 1", "negative 1"),
    ("anchor 2", "positive 2", "negative 2"),
    ("anchor 3", "positive 3", "negative 3"),
]
```

### Models

Both Splade and SparseEmbed models can be initialized from the `AutoModelForMaskedLM` pretrained models.

```python
from transformers import AutoModelForMaskedLM, AutoTokenizer

model = model.Splade(
    model=AutoModelForMaskedLM.from_pretrained("naver/splade_v2_max").to(device),
    tokenizer=AutoTokenizer.from_pretrained("naver/splade_v2_max"),
    device=device,
)
```

```python
from transformers import AutoModelForMaskedLM, AutoTokenizer

model = model.SparsEmbed(
    model=AutoModelForMaskedLM.from_pretrained("naver/splade_v2_max").to(device),
    tokenizer=AutoTokenizer.from_pretrained("naver/splade_v2_max"),
    embedding_size=64,
    k_tokens=96,
    device=device,
)
```

### Splade

The following PyTorch code snippet illustrates the training loop to fine-tune Splade:

```python
from transformers import AutoModelForMaskedLM, AutoTokenizer, optimization
from sparsembed import model, utils, train, retrieve, losses
import torch

device = "cuda" # cpu or cuda
batch_size = 8
epochs = 1 # Number of times the model will train over the whole dataset.

model = model.Splade(
    model=AutoModelForMaskedLM.from_pretrained("naver/splade_v2_max").to(device),
    tokenizer=AutoTokenizer.from_pretrained("naver/splade_v2_max"),
    device=device
)

optimizer = torch.optim.AdamW(model.parameters(), lr=1e-5)

scheduler = optimization.get_linear_schedule_with_warmup(
    optimizer=optimizer, 
    num_warmup_steps=6000,
    num_training_steps=4_000_000,
)

flops_scheduler = losses.FlopsScheduler(weight=1e-4, steps=50_000) 

X = [
    ("anchor 1", "positive 1", "negative 1"),
    ("anchor 2", "positive 2", "negative 2"),
    ("anchor 3", "positive 3", "negative 3"),
]

for anchor, positive, negative in utils.iter(
        X,
        epochs=epochs,
        batch_size=batch_size,
        shuffle=True
    ):
        loss = train.train_splade(
            model=model,
            optimizer=optimizer,
            anchor=anchor,
            positive=positive,
            negative=negative,
            flops_loss_weight=flops_scheduler.get(),
        )

        scheduler.step()
        flops_scheduler.step()

# Save the model.
model.save_pretrained("checkpoint")

# Beir benchmark for evaluation.
documents, queries, qrels = utils.load_beir("scifact", split="test")

retriever = retrieve.SpladeRetriever(
    key="id",
    on=["title", "text"],
    model=model
)

retriever = retriever.add(
    documents=documents,
    batch_size=batch_size,
    k_tokens=96,
)

utils.evaluate(
    retriever=retriever,
    batch_size=batch_size,
    qrels=qrels,
    queries=queries,
    k=100,
    k_tokens=96,
    metrics=["map", "ndcg@10", "ndcg@10", "recall@10", "hits@10"]
)
```

After having saved the model with `save_pretrained`, we can load the checkpoint using:

```python
from sparsembed import model

device = "cuda"

model = model.Splade(
    model_name_or_path="checkpoint",
    device=device,
)
```

## SparsEmbed

The following PyTorch code snippet illustrates the training loop to fine-tune SparseEmbed:

```python
from transformers import AutoModelForMaskedLM, AutoTokenizer, optimization
from sparsembed import model, utils, train, retrieve, losses
import torch

device = "cuda" # cpu or cuda
batch_size = 8
epochs = 1 # Number of times the model will train over the whole dataset.

model = model.SparsEmbed(
    model=AutoModelForMaskedLM.from_pretrained("distilbert-base-uncased").to(device),
    tokenizer=AutoTokenizer.from_pretrained("distilbert-base-uncased"),
    device=device
)

optimizer = torch.optim.AdamW(model.parameters(), lr=1e-5)

scheduler = optimization.get_linear_schedule_with_warmup(
    optimizer=optimizer, 
    num_warmup_steps=6000, # Number of warmup steps.
    num_training_steps=4_000_000 # Length training set.
)

flops_scheduler = losses.FlopsScheduler(weight=1e-4, steps=50_000)

X = [
    ("anchor 1", "positive 1", "negative 1"),
    ("anchor 2", "positive 2", "negative 2"),
    ("anchor 3", "positive 3", "negative 3"),
]

for anchor, positive, negative in utils.iter(
        X,
        epochs=epochs,
        batch_size=batch_size,
        shuffle=True
    ):
        loss = train.train_sparsembed(
            model=model,
            optimizer=optimizer,
            k_tokens=96,
            anchor=anchor,
            positive=positive,
            negative=negative,
            flops_loss_weight=flops_scheduler.get(),
            sparse_loss_weight=0.1,
        )

        scheduler.step()
        flops_scheduler.step()

# Save the model.
model.save_pretrained("checkpoint")

# Beir benchmark for evaluation.
documents, queries, qrels = utils.load_beir("scifact", split="test")

retriever = retrieve.SparsEmbedRetriever(
    key="id",
    on=["title", "text"],
    model=model
)

retriever = retriever.add(
    documents=documents,
    k_tokens=96,
    batch_size=batch_size
)

utils.evaluate(
    retriever=retriever,
    batch_size=batch_size,
    qrels=qrels,
    queries=queries,
    k=100,
    k_tokens=96,
    metrics=["map", "ndcg@10", "ndcg@10", "recall@10", "hits@10"]
)
```

After having saved the model with `save_pretrained`, we can load the checkpoint using:

```python
from sparsembed import model

device = "cuda"

model = model.SparsEmbed(
    model_name_or_path="checkpoint",
    device=device,
)
```

## Utils

We can get the activated tokens / embeddings of a sentence with:

```python
model.encode(["deep learning, information retrieval, sparse models"])
```

We can evaluate similarities between pairs of queries and documents without the use of a retriever: 

```python
model.scores(
    queries=["Query A", "Query B"], 
    documents=["Document A", "Document B"],
    batch_size=32,
)
```

```python
tensor([5.1449, 9.1194])
```

Wen can visualize activated tokens:

```python
model.decode(**model.encode(["deep learning, information retrieval, sparse models"]))
```

```python
['deep sparse model retrieval information models depth fuzzy learning dense poor memory recall processing reading lacy include remember knowledge training heavy retrieve guide vague type small learn data']
```

We can save the model and load it back.

```python
model.save_pretrained("checkpoint")


from sparsembed import model
model = model.SparsEmbed(
    model_name_or_path="checkpoint",
    device=device
)

# or 

from sparsembed import model
model = model.Splade(
    model_name_or_path="checkpoint",
    device=device
)
```

## Evaluation
<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-baqh{text-align:center;vertical-align:top}
.tg .tg-7r3q{background-color:rgba(130, 130, 130, 0.08);border-color:inherit;color:#000000;font-weight:bold;text-align:center;
  vertical-align:middle}
.tg .tg-c4ze{color:#000000;font-weight:bold;text-align:center;vertical-align:middle}
.tg .tg-59ps{background-color:rgba(130, 130, 130, 0.08);color:#000000;text-align:center;vertical-align:top}
.tg .tg-f8tx{color:#000000;text-align:center;vertical-align:top}
.tg .tg-en8h{background-color:rgba(130, 130, 130, 0.08);color:#000000;font-weight:bold;text-align:center;vertical-align:middle}
.tg .tg-1pye{color:#000000;font-weight:bold;text-align:center;vertical-align:top}
.tg .tg-vpkt{border-color:inherit;color:#000000;font-weight:bold;text-align:center;vertical-align:top}
.tg .tg-amwm{font-weight:bold;text-align:center;vertical-align:top}
.tg .tg-o1k1{background-color:rgba(130, 130, 130, 0.08);color:#000000;font-weight:bold;text-align:center;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-vpkt">Metrics</th>
    <th class="tg-vpkt" colspan="4">hits@1</th>
    <th class="tg-c4ze" colspan="4">hits@10</th>
    <th class="tg-c4ze" colspan="4">map</th>
    <th class="tg-c4ze" colspan="4">ndcg@10</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-7r3q">dataset</td>
    <td class="tg-7r3q">arguana</td>
    <td class="tg-7r3q">fiqa</td>
    <td class="tg-en8h">scidocs</td>
    <td class="tg-en8h">scifact</td>
    <td class="tg-en8h">arguana</td>
    <td class="tg-en8h">fiqa</td>
    <td class="tg-en8h">scidocs</td>
    <td class="tg-en8h">scifact</td>
    <td class="tg-en8h">arguana</td>
    <td class="tg-en8h">fiqa</td>
    <td class="tg-en8h">scidocs</td>
    <td class="tg-en8h">scifact</td>
    <td class="tg-en8h">arguana</td>
    <td class="tg-en8h">fiqa</td>
    <td class="tg-en8h">scidocs</td>
    <td class="tg-en8h">scifact</td>
  </tr>
  <tr>
    <td class="tg-baqh"><a href="https://raphaelsty.github.io/cherche/retrieve/tfidf/" target="_blank" rel="noopener noreferrer">tfidf</a></td>
    <td class="tg-baqh">0.000</td>
    <td class="tg-baqh">0.154</td>
    <td class="tg-baqh">0.157</td>
    <td class="tg-baqh">0.467</td>
    <td class="tg-baqh">0.666</td>
    <td class="tg-baqh">0.509</td>
    <td class="tg-baqh">0.713</td>
    <td class="tg-baqh">0.853</td>
    <td class="tg-baqh">0.214</td>
    <td class="tg-baqh">0.135</td>
    <td class="tg-baqh">0.094</td>
    <td class="tg-baqh">0.566</td>
    <td class="tg-baqh">0.310</td>
    <td class="tg-baqh">0.172</td>
    <td class="tg-baqh">0.138</td>
    <td class="tg-baqh">0.612</td>
  </tr>
  <tr>
    <td class="tg-baqh"><a href="https://raphaelsty.github.io/cherche/retrieve/lunr/" target="_blank" rel="noopener noreferrer">lunr</a></td>
    <td class="tg-baqh">0.000</td>
    <td class="tg-baqh">0.194</td>
    <td class="tg-amwm">0.195</td>
    <td class="tg-baqh">0.513</td>
    <td class="tg-baqh">0.723</td>
    <td class="tg-baqh">0.633</td>
    <td class="tg-amwm">0.850</td>
    <td class="tg-baqh">0.833</td>
    <td class="tg-baqh">0.236</td>
    <td class="tg-baqh">0.183</td>
    <td class="tg-amwm">0.110</td>
    <td class="tg-baqh">0.591</td>
    <td class="tg-amwm">0.344</td>
    <td class="tg-baqh">0.223</td>
    <td class="tg-amwm">0.163</td>
    <td class="tg-baqh">0.627</td>
  </tr>
  <tr>
    <td class="tg-59ps"><a href="https://huggingface.co/raphaelsty/distilbert-splade" target="_blank" rel="noopener noreferrer">raphaelsty/distilbert-splade</a></td>
    <td class="tg-59ps">0.002</td>
    <td class="tg-59ps">0.250</td>
    <td class="tg-59ps">0.180</td>
    <td class="tg-o1k1">0.557</td>
    <td class="tg-59ps">0.693</td>
    <td class="tg-59ps">0.748</td>
    <td class="tg-59ps">0.729</td>
    <td class="tg-59ps">0.880</td>
    <td class="tg-59ps">0.232</td>
    <td class="tg-59ps">0.227</td>
    <td class="tg-59ps">0.097</td>
    <td class="tg-o1k1">0.628</td>
    <td class="tg-59ps">0.332</td>
    <td class="tg-59ps">0.277</td>
    <td class="tg-59ps">0.144</td>
    <td class="tg-o1k1">0.666</td>
  </tr>
  <tr>
    <td class="tg-f8tx"><a href="https://huggingface.co/raphaelsty/distilbert-sparsembed" target="_blank" rel="noopener noreferrer">raphaelsty/distilbert-sparsembed</a></td>
    <td class="tg-1pye">0.003</td>
    <td class="tg-f8tx">0.208</td>
    <td class="tg-f8tx">0.154</td>
    <td class="tg-f8tx">0.480</td>
    <td class="tg-f8tx">0.600</td>
    <td class="tg-f8tx">0.648</td>
    <td class="tg-f8tx">0.664</td>
    <td class="tg-f8tx">0.797</td>
    <td class="tg-f8tx">0.202</td>
    <td class="tg-f8tx">0.186</td>
    <td class="tg-f8tx">0.085</td>
    <td class="tg-f8tx">0.556</td>
    <td class="tg-f8tx">0.286</td>
    <td class="tg-f8tx">0.232</td>
    <td class="tg-f8tx">0.129</td>
    <td class="tg-f8tx">0.593</td>
  </tr>
  <tr>
    <td class="tg-59ps"><a href="https://huggingface.co/naver/splade_v2_max" target="_blank" rel="noopener noreferrer">naver/splade_v2_max</a></td>
    <td class="tg-59ps">0.002</td>
    <td class="tg-59ps">0.259</td>
    <td class="tg-59ps">0.180</td>
    <td class="tg-59ps">0.553</td>
    <td class="tg-o1k1">0.705</td>
    <td class="tg-o1k1">0.790</td>
    <td class="tg-59ps">0.760</td>
    <td class="tg-59ps">0.863</td>
    <td class="tg-o1k1">0.234</td>
    <td class="tg-o1k1">0.235</td>
    <td class="tg-59ps">0.100</td>
    <td class="tg-59ps">0.616</td>
    <td class="tg-59ps">0.337</td>
    <td class="tg-59ps">0.286</td>
    <td class="tg-59ps">0.149</td>
    <td class="tg-59ps">0.649</td>
  </tr>
  <tr>
    <td class="tg-59ps"><a href="https://huggingface.co/raphaelsty/splade-max" target="_blank" rel="noopener noreferrer">raphaelsty/splade-max</a></td>
    <td class="tg-59ps">0.002</td>
    <td class="tg-o1k1">0.261</td>
    <td class="tg-59ps">0.174</td>
    <td class="tg-59ps">0.553</td>
    <td class="tg-59ps">0.698</td>
    <td class="tg-59ps">0.785</td>
    <td class="tg-59ps">0.756</td>
    <td class="tg-o1k1">0.873</td>
    <td class="tg-59ps">0.232</td>
    <td class="tg-o1k1">0.235</td>
    <td class="tg-59ps">0.098</td>
    <td class="tg-59ps">0.620</td>
    <td class="tg-59ps">0.334</td>
    <td class="tg-o1k1">0.287</td>
    <td class="tg-59ps">0.147</td>
    <td class="tg-59ps">0.655</td>
  </tr>
  <tr>
    <td class="tg-59ps"><a href="https://huggingface.co/raphaelsty/sparsembed-max" target="_blank" rel="noopener noreferrer">raphaelsty/sparsembed-max</a></td>
    <td class="tg-59ps">0.001</td>
    <td class="tg-59ps">0.236</td>
    <td class="tg-59ps">0.174</td>
    <td class="tg-59ps">0.553</td>
    <td class="tg-59ps">0.597</td>
    <td class="tg-59ps">0.681</td>
    <td class="tg-59ps">0.676</td>
    <td class="tg-59ps">0.860</td>
    <td class="tg-59ps">0.195</td>
    <td class="tg-59ps">0.206</td>
    <td class="tg-59ps">0.092</td>
    <td class="tg-59ps">0.618</td>
    <td class="tg-59ps">0.278</td>
    <td class="tg-59ps">0.248</td>
    <td class="tg-59ps">0.134</td>
    <td class="tg-59ps">0.651</td>
  </tr>
</tbody>
</table>
