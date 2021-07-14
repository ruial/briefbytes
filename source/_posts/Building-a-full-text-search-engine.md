---
title: Building a full text search engine
date: 2021-07-14 21:15:00
tags: [go, information retrieval, nlp]
---

Information Retrieval (IR) is the activity of finding resources that have relevant information, usually documents, from a large collection, in response to a query. I've always been fascinated by large-scale search engines like Google. In this post I will talk a bit about the IR process and how to build a full text search engine in Go. The code is available in [GitHub](https://github.com/ruial/busca).

## Indexing

Indexing is an operation to facilitate searches, by avoiding the need to do full scans over the entire dataset. With indexes, faster retrievals are traded for additional storage and higher update times because the index has to be maintained. A simple index example in JSON format is presented bellow:

```json
{
  "inverted-index": {
    "example": ["doc1", "doc2"],
    "content": ["doc1"]
  },
  "documents": {
    "doc1": "example content",
    "doc2": "example"
  }
}
```

As the index stores a list of words and the documents in which they appear, matches are quickly retrieved. Tokenization is the process of separating text into smaller units (words, n-grams, sentences). Other text preprocessing techniques like lower casing, removing stop words, stemming (trimming suffixes), among others, can be useful to improve search results. On [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html), these text analysis steps are performed by an Analyzer.

## Querying

An IR process starts when a user enters a query into the system, which is then evaluated, and a set of relevant results are displayed. Query expansion techniques like replacing synonyms and fixing spelling errors are often used to improve relevance. Variations of the [tf–idf](https://en.wikipedia.org/wiki/Tf%E2%80%93idf) weighting scheme are able to rank how well each document matches a given query. Other examples of ranking algorithms are Okapi BM25, PageRank and [machine learning models](https://en.wikipedia.org/wiki/Learning_to_rank).

There are many [metrics](https://en.wikipedia.org/wiki/Evaluation_measures_(information_retrieval)) to evaluate search engines. Precision, recall and F1-score are popular when dealing with unordered sets. On huge web search engines, it is not possible to calculate absolute recall as the number of relevant results for a given query is usually unknown, so the mean precision of the top results on a set of queries can be considered. To take order into account there are metrics like the discounted cumulative gain. When an IR system is already deployed, new versions can be evaluated with online metrics and AB testing.

## A minimal search engine in Go

Go is a great language to build web services. Its concurrency mechanisms are easy to use, it is statically typed, compiles quickly, dependency management is not a disaster (looking at you, Python) and tooling is also great (race detector, profiler, benchmark tests and no need for test libraries).

The data structures chosen for the index consist of two maps. One with the terms as keys and document IDs as values, and another with the document IDs as keys and the documents and term frequencies as values. Storing the term frequencies at index time allows faster ranking of search results, as they do not have to be calculated later.

```go
type DocumentData struct {
  Doc         Document
  TermsCount  float64
  Frequencies map[string]float64
}

type Index struct {
  terms      map[string][]core.DocumentID
  documents  map[core.DocumentID]core.DocumentData
  analyzer   analysis.Analyzer
  docsMutex  sync.RWMutex
}
```

The RWMutex is important to prevent race conditions. The lock can be held by an arbitrary number of readers or a single writer. To maximize concurrency, the operations done while the lock is held should be fast. I've created a snippet with the most important parts:

```go
func (i *Index) AddDocument(document core.Document) error {
  // text analysis outside the lock
  terms := i.analyzer.Analyze(document.Text())
  frequencies := core.NewTermFrequency(terms)

  i.docsMutex.Lock()
  defer i.docsMutex.Unlock()
  id := document.ID()
  if _, ok := i.documents[id]; ok {
    return errors.New("Document already exists")
  }
  // update the maps if the document does not already exist
  var termsCount float64
  for t, f := range frequencies {
    i.terms[t] = append(i.terms[t], id)
    termsCount += f
  }
  i.documents[id] = core.DocumentData{Doc: document, Frequencies: frequencies, TermsCount: termsCount}
  return nil
}

func (i *Index) SearchDocuments(query string, filterFn search.Filter, rankFn search.Ranker) []core.DocumentScore {
  // use the same analyzer to get search terms, filter the documents and rank them
  terms := i.analyzer.Analyze(query)
  docs := i.filterDocuments(terms, filterFn)
  return rankFn(terms, docs)
}

func (i *Index) filterDocuments(terms []string, filterFn search.Filter) map[core.DocumentID]core.DocumentData {
  // readers must also lock to prevent race conditions
  i.docsMutex.RLock()
  defer i.docsMutex.RUnlock()
  docs := make(map[core.DocumentID]core.DocumentData)
  // filter the documents using the search/document terms and a function that has a condition (AND, OR)
  docIds := filterFn(terms, i.terms)
  for _, id := range docIds {
    docs[id] = i.documents[id]
  }
  return docs
}

// Usage
idx := index.New(index.Opts{Analyzer: analysis.WhitespaceAnalyzer{}})
idx.AddDocument(core.NewBaseDocument("doc1", "some text"))

// there are variations to the original tf idf formula, check the code for implementation details
// TfWeightLog - when it is more important to find a term, than the number of times it appears on a document
// IdfWeightSmooth - when a word appears on all documents, idf will be 0, smoothing will cause it to be higher than zero,
// which can be useful when filtering documents and they all contain a term
tfidfRanker := search.TfIdfRanker(search.TfWeightLog, search.IdfWeightSmooth)
results := idx.SearchDocuments("sample query", search.OrFilter, tfidfRanker)
```

After implementing the index data structure and ranking algorithm, it was trivial to add an HTTP API on top of it. However, there are some complex topics like Unicode Text Segmentation, spelling correction (Levenshtein automaton, Norvig algorithm), autocomplete (finite state transducers, n-grams), handling multiple text fields, efficient disk persistence and of course, distributing indexes across different machines for scale (sharding). This is why using an existing solution like Elasticsearch, which is built on top of Lucene, is a good idea.
