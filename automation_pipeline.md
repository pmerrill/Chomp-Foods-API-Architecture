# Automation & Data Partitioning

One of the unique challenges of maintaining a 28M+ record dataset myself is ensuring data freshness without manual data entry. I built a **Self-Healing Data Pipeline** that automatically sources missing products.

## 🔄 The Self-Healing Loop

When a user scans a barcode that doesn't exist in our database, it doesn't just return a 404; it triggers a background discovery job.

```text
[ User Scan ]
     │
     ▼
(404 Not Found) ──▶ [ Log "Missing Barcode" ]
                             │
                             ▼
                    [ Background Import Task ]
                             │
                             ▼
              [ Query Open Food Facts / USDA ]
                     │                │
            (Found)  │                │ (Not Found)
               ▼                      ▼
    [ Normalization Engine ]    [ Mark Invalid ]
               │
               ├──▶ [ Insert into MySQL ]
               │
               ▼
    [ Update Redis Cache ]
```

## ⚙️ Worker Logic (PHP-CLI)

The automation logic handles the messy work of interacting with third-party APIs:

1.  **Ingestion**: Fetches raw JSON from Open Food Facts or USDA.
2.  **Normalization**:
    *   Standardizes distinct brand names (e.g., "Kroger Value" and "Kroger Premium" -> "Kroger").
    *   Formats ingredient lists for the Diet Engine.
    *   Downloads and resizes product images (Thumb/Small/Display sizes).
3.  **Integration**: inserts the clean data into the product table and related lookup tables, making it immediately available for the next scan.

## 🧩 Data Partitioning & Lookup Tables

To keep the application fast, data is split into **Hot** and **Cold** storage:

*   **Hot Data (Lookup Tables)**:
    *   A lightweight "master" product table.
    *   **Goal**: Extremely fast `SELECT` for initial search/existence checks.
    
*   **Cold Data (Detail Tables)**:
    *  Product ingredients, nutrients, images, etc.
    *   **Goal**: Only queried when a specific product detail view is requested (and then cached in Redis).

This separation ensures that massive text blobs (like ingredient lists) don't weigh down simple search queries.
