---
title: Learning to rank and ML pipelines
date: 2022-05-11 21:45:00
tags: [data, python]
---

Ranking is a core task in information retrieval (IR) and recommender systems. In previous posts, I talked about {% post_link Building-a-full-text-search-engine "search engines"%} and {% post_link Machine-learning-basics "machine learning (ML)"%}, and now we will see how to apply ML to rank (or re-rank) search results. Additionally, I will discuss why ML workflows should be automated and look into a few alternatives to build ML pipelines.

## Search relevance

Even when using existing technologies like Elasticsearch, engineers still have to tune their index/queries to optimize results for relevancy. For example, when searching news, the title is more relevant than the body and recent news are more relevant than older ones. [Tuning](https://www.elastic.co/guide/en/elasticsearch/guide/current/relevance-conclusion.html) a search engine for optimal results can be quite challenging and requires proper instrumentation and and validation. You can find a presentation about using Logistic regression to select boost parameters [here](https://haystackconf.com/us2021/talk-6/).

Although machine learning is computationally expensive, it can be applied for ranking problems and return highly relevant results using plugins like [Elasticsearch Learning to Rank (LTR)](https://opensourceconnections.com/blog/2017/04/03/test-drive-elasticsearch-learn-to-rank-linear-model/), which supports linear models and xgboost, a tree-based model. Numerical features are required to train ML models and significantly impact performance metrics. The [MSLR dataset](https://www.microsoft.com/en-us/research/project/mslr/) is often used to benchmark LTR models and contains query/relevance judgments, with features like tf-idf statistics, pagerank, bm25 and so on.

## LTR workflow

As you can't improve what you can't measure, let's assume you already have some infrastructure in place, like an analytics solution to measure search relevancy (manual/crowdsourced relevance judgments or click/session data). Afterwards, you can gather the data, build features, train different models/hyperparameters and deploy the best one. CI/CD for ML projects is harder to get right as additional artifacts should be tracked and the models need retraining as performance drifts over time.

Python, Scala and R are among the most popular languages for ML and they're all suitable for NLP/Ranking as they have a healthy ecosystem of numerical packages with efficient sparse matrix representations, text processing techniques and ML algorithms. Python has the edge, especially with [transformers](https://huggingface.co/docs/transformers/index), while Scala is popular for data streaming (Flink, Kafka streams, Spark streaming) and R is the best for data visualization while more limited for production workloads. In [this repo](https://github.com/ruial/ltr) you will find a Python notebook to train different LTR models on the first fold of MSLR-WEB10K, using pontwise/pairwise/listwise approaches, along with a comparison of Python and Scala for prediction.

These are the metrics I got on the test set with ~240k samples:

```
Linear model with standard scaler:
  - ndcg@10:        0.4
  - training:       7s
  - python predict: 80ms
  - scala predict:  50ms

XGBoost model:
  - ndcg@10:        0.5
  - cpu training:   52s
  - gpu training:   16s
  - python predict: 800ms
  - scala predict:  400ms
```

The batch inference times in Python are very fast, as most code executed is actually native. I haven't made tests in a real-time inference scenario (e.g.: REST API or queue/stream), but even if Python is slightly slower serializing/deserializing or transforming data, it is trivial to increase the number of instances and setup load balancing. The development effort of deploying workloads on more performant languages, such as Scala, does not seem worth it for the small latency gains. LTR is also usually applied as only a re-ranking strategy on the top K results to reduce loading times.

For more info on LTR:

- https://tech.olx.com/ranking-ads-with-machine-learning-ee03d7734bf4
- https://everdark.github.io/k9/notebooks/ml/learning_to_rank/learning_to_rank.html
- https://www.manning.com/books/ai-powered-search

## ML Pipelines

CI/CD systems like Jenkins or Github Actions allow workflows to be triggered via API/Cron, but they are not suitable for ML workflows, since they lack some features/integrations and agents may not have the required resources. A possible alternative is [Airflow](https://docs.aws.amazon.com/mwaa/latest/userguide/what-is-mwaa.html), which is commonly used for scheduled ETL workflows and has operators to run tasks in Kubernetes (k8s) and cloud services like AWS Batch/ECS. I didn't find the development experience to be the fastest (build docker image, upload image, update dag, trigger dag) and would require maintaining more glue code for ML tasks. [Prefect Orion](https://orion-docs.prefect.io) is a mature successor to Airflow, with a simpler architecture and better development experience, but since it is a generic workflow orchestration platform, it requires additional work to integrate with the ML ecosystem.

A viable alternative, which has a low maintenance cost, though higher lock-in, are the could offerings like AWS Sagemaker and Databricks with MLflow. However, my favorite alternative so far is [Metaflow](https://metaflow.org), a modular framework to build ML pipelines. Tasks can be run locally, in AWS Batch or in k8s, allowing data scientists to develop and test the pipelines from their machine, handles dependency management with Conda, has caching, integrates with existing schedulers (AWS Step Functions, Argo Workflows), provides a web UI, and the Python client can be used to retrieve models or experiment metrics. Metaflow does not handle model deployment or monitoring, but it can be done with a specialized platform like [Seldon](https://github.com/SeldonIO/seldon-core) or by reusing the current organization's infrastructure. I recommend reading the excellent book [Data Science Infrastructure](https://www.manning.com/books/effective-data-science-infrastructure) if you are interested in learning more about the topic.

These articles cover ML pipelines in more detail:

- https://huyenchip.com/2021/09/13/data-science-infrastructure.html
- https://ljvmiranda921.github.io/notebook/2021/05/15/navigating-the-mlops-landscape-part-2/
- https://datatalks.club/blog/mlops-10-minutes.html


## The Shift to Agentic RAG and LLM-Based Re-ranking

While classic LTR relied on gradient-boosted trees and manual feature engineering (like BM25 or PageRank), the modern pipeline has shifted toward Retrieval-Augmented Generation (RAG) and Agentic Workflows. Nowadays it is common to index document embeddings in vector databases and use semantic search to find relevant context. Using frameworks like [LangChain](https://github.com/langchain-ai/langchain), it is possible to orchestrate this process and even create custom agents that use a ReAct (Reasoning and Acting) loop to evaluate search results. These agents can for example summarize the top-K results, call specialized tools to validate facts, generate alternative queries, route queries to specific retrieval engines, and use LLMs as high-precision cross-encoders to re-rank documents based on contextual relevance. While the cost for orchestrating these new pipelines is still high, there is no denying that recent developments (more powerful models, longer context, tool calling, better frameworks, higher efficiency through quantization and other techniques) are impressive and with the massive investments being made these technologies will keep improving and are already viable for many production use cases. 
