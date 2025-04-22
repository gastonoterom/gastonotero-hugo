---
id: satoshi-spark
title: Satoshi Spark 🚀 🔥
description: DDD Project for Bitcoin Crowdfunding.
author: Gaston Otero
date: "2025-03-08"
---

## Crowdfunding, via bitcoin lightning

Satoshi Spark is a Bitcoin crowdfunding application that lets users create and donate to campaigns using the Lightning Network. While functional, the project's main purpose is to demonstrate software architecture principles - specifically Domain-Driven Design (DDD), Command Query Responsibility Segregation (CQRS), and distributed systems concepts.


The application is structured into several separate domains (bounded contexts) like authentication, accounting, crowdfunding, and Bitcoin operations. Each domain handles its specific responsibilities and communicates with others through messages rather than direct calls. This separation makes the code clearer and easier to maintain.

Here is the Github repository for the full project: [Satoshi Spark Source Code](https://github.com/gastonoterom/Satoshi-Spark)

And here is a blog article explaining in more depth the project's architecture: [DDD, CQRS and Distributed Systems in Python](/blog/ddd-cqrs-distributed-systems)