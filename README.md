# Chomp Foods API Architecture

![Foods](https://img.shields.io/badge/foods-900k%2B-blue)
![Latency](https://img.shields.io/badge/latency-%3C100ms-brightgreen)

> **Architectural overview for the Chomp Foods Nutrition API, serving 28M+ records to health and fitness applications globally.**

## 📖 About Chomp API
The [Chomp Foods API](https://chompthis.com/api) is a specialized nutrition engine that goes beyond simple data retrieval. It interprets complex ingredient lists to determine dietary suitability (Vegan, Gluten-Free, etc.) for **900,000+ foods**. This repository documents the high-scale architecture that powers the platform.

## 🧐 The Problem

Chomp aggregates nutrition data from sources like **USDA** and **Open Food Facts**. As the dataset grew to **28M+ records**, standard SQL queries resulted in search latencies exceeding **2 seconds**, causing timeouts for client applications and poor user experience.

## 💡 The Solution

I architected a high-performance **LEMP stack** (Linux, Nginx, MySQL, PHP) optimized for read-heavy workloads, introducing a multi-layered caching strategy using **Redis**.

**Key Results:**
*   Reduced P95 search latency from **2s to <100ms** (meaning 95% of all requests complete in under 100 milliseconds).
*   Scaled to handle **high-concurrency traffic** on a single, optimized database instance.

---

## 🏗 System Architecture

```text
[ Client Application ]
        │
        ▼
[ Nginx Load Balancer ]
        │
        ▼
[ PHP-FPM Application ] ──▶ [ Redis Cache ]
        │
        ▼
[ MySQL Database ]
(Single Instance + Optimization Tables)
```

## 🛠 Tech Stack

*   **Infrastructure**: DigitalOcean
*   **Database**: MySQL (Self-Hosted, Single Instance), Redis
*   **Backend**: PHP-FPM, Nginx
*   **API Spec**: OpenAPI

## 📂 Documentation

*   [**Deep Dive Architecture**](./architecture.md): Detailed request flow, diet engine logic, and optimization techniques.
*   [**Caching Strategy**](./caching_strategy.md): Analysis of the "Cache-Aside" pattern and invalidation logic.
*   [**Automation Pipeline**](./automation_pipeline.md): How "Self-Healing" background jobs source missing data.
*   [**API Specification**](./openapi.yaml): Full OpenAPI 3.0 definition.
*   [**Nginx Configuration**](./nginx.conf): Sample production configuration.

---

## 🚀 Future Roadmap & Migration

While this legacy architecture served me well, scaling stateful PHP workers has its limits. I am currently **migrating to a Serverless Edge Architecture** using **Cloudflare Workers**.

**The New Stack:**
*   **Runtime**: V8 (Cloudflare Workers) - moving from PHP to TypeScript.
*   **Data**: Supabase (PostgreSQL) + KV Store.
*   **Benefits**: Zero cold starts, global edge caching, and reduced overhead.
