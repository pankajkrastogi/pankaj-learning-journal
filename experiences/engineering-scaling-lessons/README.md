# Scaling Matters: A Real Production Lesson

This documents a real-world scaling issue I encountered while building backend systems using Java, PostgreSQL, and cloud infrastructure.  
The goal is to share **practical lessons learned from production**, not theoretical best practices.

---

## Background

I worked as a backend developer on a system where **hundreds of devices** interacted with a Java-based API backed by **PostgreSQL**.

At first glance, the scale appeared modest. However, synchronized behavior during device boot and near–real-time interactions exposed significant **concurrency and performance challenges**.

---

## The Problem

All devices booted at roughly the same time and called multiple APIs to load configuration and reference data. This resulted in:

- Sudden spikes in concurrent API requests
- Increased latency
- Multiple retries from devices
- 5xx errors
- Device crashes due to repeated failures

A similar issue occurred during near–real-time operations, where latency directly impacted user experience.

---

## Initial Architecture

- Java web services running on **Jetty**
- Deployed on a **single, high-capacity VM**
- PostgreSQL database hosted on a **separate VM**
- Adequate CPU and memory headroom on both machines
- Database schema normalized (~2NF) with proper indexing

Despite this, the system struggled under load.

---

## Investigation and Findings

### Network
- Both services ran in the same private network
- Network latency was not the issue

### Web Server
- Load testing showed Jetty could handle the expected concurrency

### Application Layer
- The **c3p0 connection pool** degraded under high concurrency
- Configuration tuning improved latency slightly but did not solve the root issue

---

## The Real Bottleneck: Database Concurrency

Although well-designed for data consistency, the database became the primary bottleneck:

- High read/write concurrency
- Data assembled from multiple normalized tables
- Lock contention under load
- Query latency increased despite indexing

Tuning helped marginally, but the fundamental issue remained.

---

## Trade-offs and Optimizations

- Proposal to flatten parts of the schema (trade consistency for performance) was rejected
- Removed **foreign key constraints**, resulting in noticeable performance gains
- Identified slow searches caused by **long string-based primary keys**
- Optimized and normalized search keys to improve lookup performance

These changes improved throughput but did not fully address scaling needs.

---

## Scaling the Database

Migrating to a different database technology (e.g., NoSQL) could have helped, but time and cost constraints made it impractical.

Instead, we implemented horizontal read scaling:

- Added a **read replica**
- Used **Slony** for replication
- Introduced **pgpool** for routing
- Invested in external training due to limited in-house replication experience

This resolved performance issues during mass device boot scenarios.

---

## Handling Strong Consistency Requirements

Some workflows required immediate consistency and could not tolerate replication lag.

Final solution:

- Separate database connection pools:
    - **Primary database** for writes and strongly consistent reads
    - **Read replica** for high-concurrency, read-heavy workloads

This architecture successfully scaled the system to handle expected load.

---

## Key Learnings

- Normalization alone does not guarantee performance at scale
- Database locking behavior is critical under concurrency
- Connection pooling can become a hidden bottleneck
- Read replicas solve specific problems, not all scaling challenges
- Architectural decisions must balance theory with operational reality
- In hindsight, a **NoSQL datastore or search cluster** may have been better suited for high-concurrency read workloads

---

## Why This Exists

Scaling problems often arise not from “big numbers,” but from:

- Synchronized client behavior
- Concurrency spikes
- Hidden architectural bottlenecks

These lessons are difficult to learn from tutorials and usually surface only in production systems.

---
