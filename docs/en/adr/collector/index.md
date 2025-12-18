---
title: Collector ADR
---

# Collector Repository ADR

> 🇰🇷 [한국어 버전](/ko/adr/collector/)

Architecture Decision Records for the [specvital/collector](https://github.com/specvital/collector) repository (Collector Service).

## ADR Index

| #   | Title                                                                                                  | Date       |
| --- | ------------------------------------------------------------------------------------------------------ | ---------- |
| 01  | [Scheduled Re-collection Architecture](./01-scheduled-recollection.md)                                 | 2024-12-18 |
| 02  | [Clean Architecture Layer Introduction](./02-clean-architecture-layers.md)                             | 2024-12-18 |
| 03  | [Graceful Shutdown and Context-Based Lifecycle Management](./03-graceful-shutdown.md)                  | 2024-12-18 |
| 04  | [Distributed Lock-Based Scheduler Concurrency Control](./04-distributed-lock-scheduler-concurrency.md) | 2024-12-18 |
| 05  | [OAuth Token Graceful Degradation](./05-oauth-token-graceful-degradation.md)                           | 2024-12-18 |
| 06  | [Asynq Logging with slog Adapter](./06-asynq-slog-adapter.md)                                          | 2024-12-18 |
| 07  | [Worker-Scheduler Process Separation](./07-worker-scheduler-separation.md)                             | 2024-12-18 |
| 08  | [Semaphore-Based Clone Concurrency Control](./08-semaphore-clone-concurrency.md)                       | 2024-12-18 |
| 09  | [Repository Pattern Data Access Abstraction](./09-repository-pattern.md)                               | 2024-12-18 |

## Related

- [All ADRs](/en/adr/)
- [Collector PRD](/en/prd/04-collector-service.md)
