# Pyserini: A Conceptual Framework for Retrieval

This guide presents a conceptual framework for a representational approach to information retrieval that integrates dense and sparse representations into the same underlying (bi-encoder) architecture.

If you're a Waterloo student traversing the [onboarding path](https://github.com/lintool/guide/blob/master/ura.md),
make sure you've first done all the exercises leading up to this guide, starting [here](https://github.com/castorini/anserini/blob/master/docs/start-here.md).
In general, if you don't understand what it is that you're doing when following this guide, i.e., you're just [cargo culting](https://en.wikipedia.org/wiki/Cargo_cult_programming) (i.e., blindly copying and pasting commands into a shell), then you should back up to the previous step in the onboarding path.

**Learning outcomes** for this guide, building on previous steps in the onboarding path:

+ Understand how sparse and dense representations can be viewed as variations in a bi-encoder architecture.
+ Be able to identify correspondences between Lucene indexing and retrieval operations with the above conceptual framework.
+ Be able to extract the BM25 vector representation of a document from a Lucene index and compute its inner product with respect to a query.
+ Understand the difference between dense and sparse representations, and between supervised (learned) and unsupervised (heuristic) representations.

## Bi-Encoders

As a recap from [here](https://github.com/castorini/anserini/blob/master/docs/start-here.md), this is the "core retrieval" problem that we're trying to solve:

> Given an information need expressed as a query _q_, the text retrieval task is to return a ranked list of _k_ texts {_d<sub>1</sub>_, _d<sub>2</sub>_ ... _d<sub>k</sub>_} from an arbitrarily large but finite collection
of texts _C_ = {_d<sub>i</sub>_} that maximizes a metric of interest, for example, nDCG, AP, etc.

How might we tackle the challenge?
One approach, known as a bi-encoder (or dual-encoder) architecture, is presented below:

<img src="images/architecture-biencoder.png" width="400" />

The approach is conceptually quite simple:
Let's say we have two "encoders":

+ The **document encoder** takes a document and generates a representation of the document.
+ The **query encoder** takes a query and generates a representation of the query.

In addition, we have a **comparison function** that takes two representations (one from a document, the other from a query) and produces an estimate of relevance, i.e., the degree to which the document is relevant to the query.
In other words, the comparison function produces a relevance score; alternatively, we call this a query-document score.

Let's assume that the encoders generate representations in the form of **vectors**, and that the score is computed in terms of the **inner product** (or **dot product**) between the document and query vectors.

Let's further assume that the encoders have been designed or built in such a way that the larger the score (i.e., inner product), the more relevant the document is to the query.
(Exactly how, we'll see later.)

Given this setup, how would we build a retrieval system?
Well, here's one obvious way:

**Step (1).** Let's take the document collection (or corpus), i.e., _C_ = {_d<sub>i</sub>_}, and encode each document.
So we have a bunch of vectors now, each corresponding to one document.

**Step (2).** When a query arrives, we need to encode it, i.e., generate its vector representation.

**Step (3).** In the final step, we need to find the _k_ document vectors that have the highest query-document scores in terms of the inner product of their vector representations.
We say _k_ because in nearly all settings, _k_ is specified externally, i.e., the user says, give me the top 10 hits.
Hence, top-_k_ retrieval.

Step (1) and step (2) are relatively straightforward given the document and query encoders.
Encoding the document collection is embarrassingly parallel and encoding the query happens at search time.

Step (3) has a very naive implementation: take the query vector, compute its score with the vector of the first document, compute its score with the vector of the second document, compute its score with the vector of the third document, etc.
Repeat until we iterate through all documents, keeping track of the top-_k_ scores along the way (e.g., in a heap).
In other words, compute all inner products in a brute-force manner.

Don't laugh, this isn't as ridiculous as it sounds!
(For later, this is in fact what's happening with a `FlatIP` index in Faiss.)
However, researchers have developed more efficient data structures and top-_k_ retrieval algorithms for vectors of different types.
As a preview: for sparse vectors, we use inverted indexes, and for dense vectors, we use HNSW.

## BM25 as a Bi-Encoder

Now, okay, what does that have to do with retrieval using BM25?

Well, BM25 is simply an "instantiation" of the above bi-encoder framework, where BM25 is the document encoder and the query encoder generates a so-called multi-hot vector, like this:

<img src="images/architecture-bm25.png" width="400" />

Wait, seriously?

Yup!
Let's consider `docid` 7187158, the answer to the query about Paula Deen's brother:

```
Paula Deen and her brother Earl W. Bubba Hiers are being sued by a former general manager at Uncle Bubba'sâ¦ Paula Deen and her brother Earl W. Bubba Hiers are being sued by a former general manager at Uncle Bubba'sâ
```

This is the BM25 vector representation for that document:

```json
{
    "be": 2.637899875640869,
    "brother": 4.09124231338501,
    "bubba": 7.102361679077148,
    "bubba's\u00e2": 11.091651916503906,
    "deen": 7.4197235107421875,
    "earl": 5.663764953613281,
    "former": 3.8262834548950195,
    "gener": 2.2932770252227783,
    "her": 2.7393782138824463,
    "hier": 8.24051284790039,
    "manag": 2.832794189453125,
    "paula": 6.438521862030029,
    "su": 5.404428005218506,
    "uncl": 5.362298488616943,
    "w": 3.9339818954467773
}
```

This requires a bit of explanation.
As previously mentioned, BM25 is a so-called "bag-of-words" model, which means that the representation of the document is a **sparse** vector, which each dimension corresponds to a term in the vocabulary space.
Usually, we say "terms" (or sometimes "tokens") instead of "words" because there's some amount of processing to tokenize the input document.
Typically, words are converted into lower case, "stopwords" are discarded, and some form of stemming is applied (e.g., to remove suffixes).

In a bag-of-words representation, the dimension of the sparse vector is the size of the vocabulary space, i.e., the number of unique terms in the entire collection (i.e., all documents).
It's called a "bag of words" because we're only keeping track of the individual terms in a document, neglecting other important aspects of "meaning" such as the order of the terms, linguistic relationships, etc.
Sometimes, these are called sparse _lexical_ vectors, to emphasize that their dimensions correspond to lexical items (i.e., terms in the vocabulary).

Since most of the entries in a sparse vector (representing a document) are zero, it can be conveniently represented in JSON, with the keys denoting the terms that have non-zero weights (or scores).
This is what we've done above.
The weights (or scores) of those terms are determined by the [BM25 scoring function](https://en.wikipedia.org/wiki/Okapi_BM25), which is a function of various term statistics such as the term frequency (the number of times a term appears in a document), the term's document frequency (the number of documents in the collection that contains the term), the document length, etc.

However, the high-level idea is that "important" terms get high weights, and "unimportant" terms get low weights.
BM25 is just one (of many) scoring functions that attempt to capture this intuition.

Who came up with the BM25 scoring function?
Its origins date back to the 1970s, but you can find a good overview from 2009 [here](https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf).

So, the indexing phase with Lucene, as in the [previous exercise](experiments-msmarco-passage.md) you've done, corresponds to the green box below:

<img src="images/architecture-bm25a.png" width="400" />

Conceptually, you've computed the BM25 document vector of every document in the collection and stored them in a data structure called an inverted index.
(Actually, in reality, the inverted index only stores component statistics that allow you to reconstruct the BM25 document vectors.)

With the `IndexReader` class in Pyserini, you can materialize (i.e., reconstruct) the BM25 document vector for a particular document:

```python
from pyserini.index.lucene import IndexReader
import json

index_reader = IndexReader('indexes/lucene-index-msmarco-passage')
tf = index_reader.get_document_vector('7187158')
bm25_vector = {term: index_reader.compute_bm25_term_weight('7187158', term, analyzer=None) for term in tf.keys()}

print(json.dumps(bm25_vector, indent=4, sort_keys=True))
```

The above snippet of code will generate exactly the same JSON above.

Once you've built an index, the retrieval stage corresponds to the purple box below:

<img src="images/architecture-bm25b.png" width="400" />

The below snippet of code generates the query representation:

```python
from pyserini.analysis import Analyzer, get_lucene_analyzer

analyzer = Analyzer(get_lucene_analyzer())
query_tokens = analyzer.analyze('what is paula deen\'s brother')
```

The query tokens are:

```
['what', 'paula', 'deen', 'brother']
```

The query representation is simply a sparse vector where all the query tokens get a score (weight) of one.
The fancy way of saying this is a "multi-hot vector".

As described above, the top-_k_ retrieval problem is to find the _k_ documents from the collection that have the highest inner product (between a document vector and the query vector).
Without getting into details, the inverted index allows this top-_k_ to be computed efficiently.

Here, we can manually compute the inner product between the query vector and the document vector:

```python
score = sum({term: bm25_vector[term] for term in bm25_vector.keys() & query_tokens}.values())
```

Here, we are using a dictionary comprehension to generate a new dictionary that only contains the keys (terms) that appear in _both_ the query and the document.
Then we sum up the values (i.e., the term weights) to arrive at the query-document score.
This is exactly the inner product.

The query-document score you should get is `17.949487686157227`.

Let's try searching with the same query using Lucene:

```python
from pyserini.search.lucene import LuceneSearcher

searcher = LuceneSearcher('indexes/lucene-index-msmarco-passage')
hits = searcher.search('what is paula deen\'s brother')

for i in range(0, 10):
    print(f'{i+1:2} {hits[i].docid:7} {hits[i].score:.5f}')
```

You'll see that hit 1, `docid` 7187158, has a score of `17.94950`, which is the same as the score we computed by hand (modulo rounding).

Taking stock of where we're at:
This section explains how retrieval with BM25 (e.g., with Lucene), is an instantiation of the bi-encoder architecture.
Repeat the same exercise above for a few other queries (make them up) to convince yourself.

## Transformers in Bi-Encoders

Let's go back and look at the bi-encoder architecture again.

<img src="images/architecture-biencoder.png" width="400" />

With BM25, the document encoder generates sparse lexical (or bag-of-words) vectors.
We can imagine encoders generating other types of vector representations.

The two major axes of variations are:

+ The basis of the representation vectors (sparse vs. dense).
They could be sparse vectors (as in the case of BM25) or they could be dense vectors.
In the case of dense vectors, the dimensions of the vectors putatively capture some "latent semantic space". 
Thus, we often contrast sparse lexical vector with dense semantic vectors.
+ Whether the representations are learned (supervised vs. unsupervised).
In the case of BM25, there is no "learning" (modulo minor parameter tuning): the BM25 scoring function specifies the weights that each term should receive.
BM25 was derived from a probabilistic model of retrieval and has evolved over many decades (see [here](https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf)), but it is _not_ the product of a (supervised) machine-learning approach.
Alternatively, document and query representations can be _learned_ from large amounts of data.
So, we have a contrast between unsupervised (or heuristic) representations and supervised (or learned) representations.

Indeed, dense retrieval models usually refer to encoders that _learn_ to produce dense semantic representations using, you guessed it, transformers models!
Where do the training data come from?
Again, you got it: datasets such as MS MARCO.

So the picture for dense retrieval looks like this:

<img src="images/architecture-dense.png" width="400" />

Again, to summarize:
We often say that dense retrieval models generate learned dense representations.
In contrast, BM25 generates unsupervised (or heuristic) sparse representations.
To complete the possibilities, yes, there are learned sparse representations and unsupervised dense representations as well.

We'll save a more complete exploration of this design space for some other time, but they're sketched out in this article (on which this guide is based) if you want to learn more:

> Jimmy Lin. [A Proposed Conceptual Framework for a Representational Approach to Information Retrieval.](https://arxiv.org/abs/2110.01529) arXiv:2110.01529, October 2021. 

Okay, that's it for this lesson.
Before you move on, however, add an entry in the "Reproduction Log" at the bottom of this page, following the same format: use `yyyy-mm-dd`, make sure you're using a commit id that's on the main trunk of Pyserini, and use its 7-hexadecimal prefix for the link anchor text.

## Reproduction Log[*](reproducibility.md)
+ Results reproduced by [@sahel-sh](https://github.com/sahel-sh) on 2023-07-23 (commit [`9619502`](https://github.com/castorini/pyserini/commit/9619502f7c1b421ae86b59cafed137fc5eaafa10))


