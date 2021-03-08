---
title: "AI, System: Distributed Machine Learning"
categories:
  - AI
  - System
---

Spent few days getting into some classic designs of parallel, distributed machine learning systems. It is often simple to parallelize AI applications since most of them are "embarrassingly parallel": processes in progress rarely communicate with each other. Therefore distributed learning systems focus on scalability, easy development and deployment, rather than extreme efficiency.

[Federated learning](https://lan-jing.github.io/ai/system/math/securityProjects/) is quite a different thing. Based on previous work, researchers are now interested in training better models in a collaborative effort, while keeping each participant's dataset secure and private.

## MPI: Simple Collective Operations

Teo, Choon Hui, et al. "A scalable modular convex solver for regularized risk minimization." Proceedings of the 13th ACM SIGKDD international conference on Knowledge discovery and data mining. 2007.

Agarwal, Alekh, et al. "A reliable effective terascale linear learning system." The Journal of Machine Learning Research 15.1 (2014): 1111-1133.

## Mapreduce: Efficient Abstraction, Fault-tolerance

Dean, Jeffrey, and Sanjay Ghemawat. "MapReduce: simplified data processing on large clusters." Communications of the ACM 51.1 (2008): 107-113.

Chu, Cheng, et al. "Map-reduce for machine learning on multicore." Advances in neural information processing systems 19 (2007): 281.

Ranger, Colby, et al. "Evaluating mapreduce for multi-core and multiprocessor systems." 2007 IEEE 13th International Symposium on High Performance Computer Architecture. Ieee, 2007.

## Spark: In-memory, Iterative, Functional Programming

Zaharia, Matei, et al. "Spark: Cluster computing with working sets." HotCloud 10.10-10 (2010): 95.

Vavilapalli, Vinod Kumar, et al. "Apache hadoop yarn: Yet another resource negotiator." Proceedings of the 4th annual Symposium on Cloud Computing. 2013.