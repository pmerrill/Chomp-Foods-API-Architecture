# Architecture Deep Dive

This document details the internal request processing flow of the Chomp Foods API, specifically the `GET barcode` endpoint.

## 🔄 Request Lifecycle

The API follows a strict "fail-fast" pipeline designed to protect database resources.

```text
1. Client Request ──▶ Nginx ──▶ PHP Application
                                     │
                                     ▼
2. Security Check ──▶ MySQL (Blocklist/Quota)
                                     │
                                     ▼
3. Caching Phase  ──▶ Redis (Check Key)
                       │          │
           (Hit) ◀─────┘          └─────▶ (Miss)
             │                              │
             ▼                              ▼
4. Return JSON (200 OK)           5. Execution Phase
                                     │
                                     ├──▶ MySQL (Lookup ID from Barcode)
                                     ├──▶ MySQL (Fetch Metadata)
                                     ├──▶ PHP (Run Diet Engine)
                                     ├──▶ Redis (Cache Result)
                                     │
                                     ▼
                                  Return JSON (200 OK)
```

## 🛡️ 1. Security & Rate Limiting

Before touching any business logic, the API performs lightweight checks:
1.  **IP Denylist**: Checks the security access table for IPs attempting to brute-force keys.
2.  **Tiered Rate Limiting**: 
    - Queries the rate limit tracking table to track `views_per_minute`.
    - Enforces hard quotas based on subscription tier (Free, Standard, Premium).
    - Returns `429 Too Many Requests` or `401 Unauthorized` immediately if limits are breached.

## 🧠 2. Dietary Compatibility Engine

The core value proposition of Chomp is not just raw data, but *interpreted* data. The **Dietary Compatibility Engine** is a proprietary service that determines if a product is Vegan, Vegetarian, Gluten-Free, etc.

### Abstracted Logic Flow

1.  **Input Normalization**: Ingredient lists are parsed, and multi-word ingredients ("Milk and Cream") are split.
2.  **Banned Ingredient Matching**:
    - The engine loads cached lists of "Banned Ingredients" for each supported diet.
    - It uses **Levenshtein Distance** algorithms to fuzzy-match product ingredients against banned lists (handling typos or slight naming variations).
3.  **Exception Handling**:
    - Checks for "Allowed Alternatives". For example, "Butter" is banned for Vegans, but "Cocoa Butter" is explicitly allowed. The engine distinguishes these nuances.
4.  **Confidence Scoring**:
    - Assigns a compatibility score:
        - **0 (Incompatible)**: Exact match on banned ingredient.
        - **1 (Uncertain)**: Probable match, or data suggests cross-contamination risk.
        - **2 (Compatible)**: No banned ingredients found.

## ⚡ Infrastructure Optimizations

To achieve sub-100ms response times on a single relational database node:
*   **Optimized Lookup Tables**: I created a lightweight, indexed copy of the product table containing only essential columns. This acts as an internal search index, allowing fast initial identification before joining against the heavier data tables.
*   **Compound Indexes**: Configured on lookup tables to ensure instant results even as the dataset grows to millions of records (effectively constant time lookups).
*   **PHP-FPM Tuning**: Optimized `pm.max_children` and `pm.start_servers` to match the DigitalOcean droplet's CPU core count, preventing thrashing under load.

---

## ☁️ Migration: From Monolith to Serverless

I am currently re-architecting this system to eliminate the operational overhead of managing Nginx/PHP servers.

| Component | Legacy (Current) | Future (Cloudflare Workers) |
|-----------|------------------|-----------------------------|
| **Compute** | PHP on a single Droplet | V8 (Edge Network) |
| **Scaling** | Vertical (Resize Droplet) | Auto-scaling (Serverless) |
| **Caching** | Centralized Redis | Distribution KV Store (Edge) |
| **Database** | MySQL (Self-hosted) | Supabase (Managed Postgres) |

**Key Advantage**: The "Dietary Compatibility Engine" is being ported to TypeScript, allowing it to run directly on the Edge. This means a user in Tokyo hits a worker in Tokyo, determining Vegan status in single-digit milliseconds without ever hitting my US-based primary database.
