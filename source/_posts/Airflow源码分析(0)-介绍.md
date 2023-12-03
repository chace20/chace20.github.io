---
title: Airflow源码分析(0)-介绍
date: 2019-09-01 12:06:46
tags: 
 - Airflow
 - 任务编排调度
---

[Airflow](https://airflow.apache.org/index.html)是一套分布式的任务编排和调度系统，核心概念是DAG，通过DAG编排任务，并通过scheduler调度任务到不同的worker上执行。

Airflow基于Python开发，所以在开发Airflow前要有Python基础。

开发环境的搭建可以参考：[Contributing](https://github.com/apache/airflow/blob/master/CONTRIBUTING.md)
