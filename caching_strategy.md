# Caching Strategy: The Cache-Aside Pattern

To handle **900k+ foods (28M+ records)** with a high read-to-write ratio (99:1), relying solely on MySQL was not feasible. I implemented a strict **Cache-Aside (Lazy Loading)** pattern using Redis.

## 📐 The Pattern

In this pattern, the application code, not the database, is responsible for managing the cache.

```text
      [ Application ]
             │
             ▼
      [ Check Redis ] 
      │             │
(Hit) │             │ (Miss)
      ▼             ▼
[ Return Data ]   [ MySQL Query ]
                    │
                    ▼
              [ Write to Redis ]
                    │
                    ▼
              [ Return Data ]
```

## 💻 Implementation (Pseudocode)

I abstracted this logic into a reusable helper function to ensure consistency across all endpoints.

```php
/**
 * Generic Cache-Aside Wrapper
 * 
 * @param string $key      Unique Redis key (e.g., 'v2:product:barcode:12345')
 * @param int    $ttl      Time-To-Live in seconds
 * @param callable $callback  Function to execute on Cache Miss
 */
function get_from_cache_or_db($redis, $key, $ttl, $callback) {
    // 1. Attempt to fetch from Redis
    $cached_data = $redis->get($key);
    
    if ($cached_data) {
        return json_decode($cached_data, true);
    }
    
    // 2. Cache Miss: Execute the expensive database/logic callback
    $fresh_data = $callback();
    
    // 3. Store in Redis if data is valid
    if ($fresh_data) {
        $redis->setex($key, $ttl, json_encode($fresh_data));
    }
    
    return $fresh_data;
}
```

### Usage Example

```php
$product = get_from_cache_or_db($redis, "product:{$barcode}", 1209600, function() use ($barcode, $pdo) {
    // 1. Fetch raw data from MySQL
    $raw_data = fetch_product_from_db($pdo, $barcode);

    // 2. "Hydrate" / Transform data (Procedural)
    // The diet compatibility engine runs here, BEFORE caching.
    // This transformation is expensive, so I cache the result, not just the row.
    $processed_product = process_raw_data($raw_data);
    $final_product = generate_diet_flags($processed_product);

    return $final_product;
});
```

## ♻️ Invalidation Strategy

Since food data changes infrequently (packaging updates, formula changes), I use a hybrid invalidation strategy:

1.  **Passive Expiry (TTL)**: Keys have a default TTL of **14 days** (1,209,600 seconds) since food metadata is static.
2.  **Versioning**: I namespace keys with API versions (e.g., `api-v2:...`). This allows me to "invalidate" the entire cache instantly during major deployments by simply changing the key prefix in the configuration.
3.  **Write-Through(ish)**: When a product is updated:
    *   The `update_product()` function calculates the specific Redis key.
    *   It immediately calls `$redis->del($key)`.
    *   The next user request triggers a fresh DB fetch (Step 2 in the diagram above).

## 🚀 Why this worked for Chomp

*   **Predictable Performance**: Redis consistently returns data in a few milliseconds.
*   **Trade-off Acknowledgement**: The first user to request a specific food after the cache expires *does* pay the performance penalty of the complex SQL join (Cache Miss). However, with a 14-day TTL, this penalty is amortized over thousands of subsequent hits.
*   **Cost Efficiency**: I avoided the need for complex replication or sharding, as Redis absorbed 95% of read traffic, allowing a single MySQL instance to handle the load.
*   **Resilience**: If the Database goes down, the API continues to serve hot content from Redis.
