---
title: "I Started an Audio Learning Channel: EarLearner"
date: 2026-06-18 00:00:00 -0700
categories: [Announcement]
tags: [youtube, learning, system-design, chinese, audio]
lang: en
---

I just published the first three episodes on my new YouTube channel, [EarLearner](https://www.youtube.com/@earlearner). Here is what it is and why I built it.

## The idea

I spend a lot of time in transit — driving, commuting, waiting — where my hands and eyes are occupied but my mind is free. Podcasts fill that window well, but I kept wanting something more structured: content that actually teaches rather than just discusses.

Most technical learning assumes you are sitting at a desk with a screen. That format does not fit a 30-minute drive. I wanted audio that works as the primary medium, not as a fallback.

EarLearner is exactly that: clear, focused, audio-only lessons designed for busy people who want to grow their knowledge without staring at a screen. No slides, no visuals — just high-quality explanations you can absorb while doing something else. Every moment counts.

## The first three episodes

The first batch focuses on classic system design problems — topics that come up constantly in engineering interviews and in real architectural work. They are well-suited to audio because the core ideas are conceptual and self-contained.

**设计一个速率限制器（API 网关）** — A deep dive into rate limiting and the API gateway that hosts it. Covers the four main algorithms — token bucket, leaky bucket, fixed window counter, and sliding window — with clear comparisons of when each breaks down. Then goes into the engineering reality: how to implement atomic operations using Redis Lua scripts, how distributed rate limiting works across multiple servers, circuit breakers, and what a well-behaved 429 response looks like. Reference implementations include Nginx, AWS API Gateway, Cloudflare, and Stripe.

[![设计一个速率限制器](https://img.youtube.com/vi/coMhUDmH-cU/hqdefault.jpg)](https://www.youtube.com/watch?v=coMhUDmH-cU)

**设计一个密钥管理系统** — A thorough walkthrough of secret management systems, with HashiCorp Vault as the primary reference. Explains why hard-coding credentials in repositories is a permanent liability, then builds up the design layer by layer: envelope encryption with DEK and KEK, hardware security modules, Shamir's secret sharing for the unseal ceremony, static versus dynamic secrets (including how Vault generates short-lived database credentials on demand), path-based access policies, authentication methods for machines and Kubernetes pods, lease management and auto-rotation, audit logging under compliance constraints, and Kubernetes integration patterns. AWS Secrets Manager and GCP Secret Manager are compared at the end.

[![设计一个密钥管理系统](https://img.youtube.com/vi/CZ6aogP1kt8/hqdefault.jpg)](https://www.youtube.com/watch?v=CZ6aogP1kt8)

**设计一个分布式键值存储系统** — A systematic treatment of distributed key-value stores, tracing the design from requirements through every major subsystem. Consistent hashing with virtual nodes for data distribution; replication factor three and tunable quorum (W + R > N) for consistency; vector clocks, last-write-wins, and CRDTs for conflict resolution; LSM trees, write-ahead logs, Memtables, SSTables, compaction, and tombstone resurrection for the write path; Bloom filters, sparse indexes, and row caches for reads; Gossip protocol and Phi Accrual Failure Detector for fault detection; and a three-layer recovery stack of hinted handoff, read repair, and Merkle tree anti-entropy. Closes with CAP and PACELC. Reference systems: DynamoDB, Cassandra, RocksDB, and Bigtable.

[![设计一个分布式键值存储系统](https://img.youtube.com/vi/P4ikJJwWTq8/hqdefault.jpg)](https://www.youtube.com/watch?v=P4ikJJwWTq8)

## Subscribe

The channel is at [youtube.com/@earlearner](https://www.youtube.com/@earlearner). More episodes are coming — system design is just the starting point.