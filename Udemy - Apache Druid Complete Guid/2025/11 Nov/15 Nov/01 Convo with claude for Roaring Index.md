New chat
what is roaring or concise compressed bitmap indexex that apache druid uses to enable fast filtering and searching accross multiple columns ?

Roaring Bitmaps (also called Concise Compressed Bitmap Indexes) are a highly efficient data structure that Apache Druid uses for fast filtering and searching across multiple columns.

What They Are
Roaring bitmaps are compressed bitmap indexes that efficiently represent sets of integers (like row numbers). They're called "roaring" because they use the Roaring Bitmap compression algorithm, which is an evolution of earlier approaches like CONCISE (Compressed 'n' Composable Integer Set).

How They Work in Druid
Basic Concept:

For each distinct value in a column, Druid maintains a bitmap where each bit represents a row
If a row contains that value, the corresponding bit is set to 1, otherwise 0
These bitmaps are heavily compressed using the Roaring algorithm
Example:

Column: Country
Row 0: USA
Row 1: India  
Row 2: USA
Row 3: Japan

Bitmap for "USA":    [1, 0, 1, 0]  → rows 0, 2
Bitmap for "India":  [0, 1, 0, 0]  → row 1
Bitmap for "Japan":  [0, 0, 0, 1]  → row 3
Why They're Fast
Compressed Storage: Roaring bitmaps use intelligent compression that adapts based on data density, achieving 10-100x compression ratios
Fast Boolean Operations: Filtering queries like WHERE country = 'USA' AND age > 25 become fast bitwise AND/OR operations on compressed bitmaps
Cache-Friendly: The compressed format fits more data in CPU cache, improving performance
Direct Operations on Compressed Data: Can perform operations without full decompression
Roaring vs CONCISE
Roaring is the modern standard and typically outperforms CONCISE. It uses a hybrid approach:

Sparse regions: Stores as sorted arrays of integers
Dense regions: Stores as uncompressed bitmaps
Very dense regions: Uses run-length encoding
This adaptive strategy makes Roaring fast across all data distributions, which is why Druid defaults to it for bitmap indexing.

in the example that you share , what if the number of rows are very high , that means the bitvalue cannot be stored in bigInt what needs to be done in that case ?

Great question! You've identified a key challenge. When you have billions of rows, a single BigInt (64 bits) can't represent all row positions. Here's how Roaring Bitmaps elegantly solve this:

The Roaring Bitmap Solution: Chunking
Roaring Bitmaps don't store bitmaps as single continuous bit arrays. Instead, they use a two-level hierarchical structure:

1. Chunk by High Bits (Containers)
The 32-bit row number is split into:

High 16 bits: Used as a "chunk key" (65,536 possible chunks)
Low 16 bits: Position within that chunk (65,536 positions per chunk)
Row Number: 4,294,967,295 (max 32-bit)
            ↓
Binary: 11111111 11111111 | 11111111 11111111
        ←─── High 16 ───→   ←─── Low 16 ───→
        (Chunk ID)          (Position in chunk)
2. Three Container Types
Each chunk (container) stores its 65,536 possible values using one of three formats:

Array Container (sparse data):

Stores actual values as sorted array
Used when < 4,096 values present
Example: [5, 127, 1024, 5000]
Bitmap Container (medium density):

Traditional bitmap: 8,192 bytes (65,536 bits)
Used when 4,096-60,000 values present
Each bit represents presence/absence
Run Container (consecutive runs):

Stores start/length pairs
Example: [100, 500] means rows 100-599
Used for consecutive sequences
Example with Large Numbers
Let's say you have 10 billion rows:

Row 0:           Chunk 0, Position 0
Row 65,535:      Chunk 0, Position 65,535
Row 65,536:      Chunk 1, Position 0
Row 4,000,000:   Chunk 61, Position 4,000
Row 10B:         Chunk 152,587, Position 58,368
For "Country = USA" across 10 billion rows:

Chunk 0:     ArrayContainer([0, 15, 1024, ...])
Chunk 1:     RunContainer([0-5000])
Chunk 61:    BitmapContainer([bit array])
...
Chunk 152587: ArrayContainer([58368, ...])
Key Advantages
Scalability: Can handle 2^32 (4.3 billion) row IDs efficiently, and implementations exist for even larger datasets
Memory Efficiency: Only stores chunks that have data (sparse support)
Fast Operations: Boolean operations (AND/OR) work chunk-by-chunk in parallel
Adaptive: Each chunk automatically picks the most efficient storage format
In Apache Druid Specifically
Druid uses this exact Roaring Bitmap implementation through the RoaringBitmap library. When filtering:

Druid looks up the relevant chunks for the filter value
Performs fast bitwise operations on those chunks
Combines results across chunks
Returns matching row IDs
This is why Druid can filter billions of rows in milliseconds - it's not working with giant monolithic bitmaps but efficiently compressed, chunked containers!

with an example of large scale show how druid finds the row numbers which it needs to display

Great! Let me walk through a concrete example of how Druid uses Roaring Bitmaps to find matching rows at scale.

Scenario Setup
Dataset: 5 billion web events

Columns: timestamp, country, device_type, user_id, page_views
Query: SELECT user_id, page_views WHERE country = 'USA' AND device_type = 'mobile'
Step 1: Bitmap Index Lookup
Druid maintains Roaring Bitmaps for each distinct value:

Country Column Bitmaps:

"USA" bitmap:
  Chunk 0:     RunContainer([0-45000])           // Rows 0-45,000
  Chunk 15:    BitmapContainer([...])            // Rows 983,040-1,048,575
  Chunk 1200:  ArrayContainer([500, 1024, ...])  // Rows 78,643,200+
  ...
  Chunk 76293: RunContainer([10000-50000])       // Rows 4,999,950,000+
  
"India" bitmap:
  Chunk 0:     ArrayContainer([45001, 45002...])
  Chunk 20:    RunContainer([...]
  ...

"Japan" bitmap:
  ...
Device_type Column Bitmaps:

"mobile" bitmap:
  Chunk 0:     BitmapContainer([...])
  Chunk 15:    RunContainer([5000-60000])
  Chunk 1200:  ArrayContainer([200, 800, ...])
  ...
  Chunk 76293: BitmapContainer([...])

"desktop" bitmap:
  ...
Step 2: Bitmap Intersection (AND Operation)
Druid performs chunk-by-chunk intersection:

Chunk 0 Intersection:

USA Chunk 0:      RunContainer([0-45000])
                      ∩ (AND)
mobile Chunk 0:   BitmapContainer([bits: 0,1,0,1,1...])
                      ↓
Result Chunk 0:   ArrayContainer([0, 3, 4, 7, 11, ...])
                  // Only rows where BOTH conditions true
Chunk 15 Intersection:

USA Chunk 15:     BitmapContainer([983040: 1, 983041: 0, 983042: 1...])
                      ∩
mobile Chunk 15:  RunContainer([5000-60000])
                      ↓
Result Chunk 15:  ArrayContainer([988040, 988041, ...])
                  // Converted 5000-60000 to absolute rows:
                  // 983,040 + 5000 = 988,040
Chunk 76293 Intersection:

USA Chunk 76293:     RunContainer([10000-50000])
                         ∩
mobile Chunk 76293:  BitmapContainer([...])
                         ↓
Result Chunk 76293:  ArrayContainer([5000138240, 5000138245, ...])
Step 3: Materialized Row Numbers
After processing all chunks, Druid has:

Final Matching Row Numbers:
[0, 3, 4, 7, 11, 25, 30, ..., 988040, 988041, ..., 5000138240, ...]
                ↓
Total: 1.2 billion matching rows (24% of 5B)
Step 4: Fetch Actual Data
Now Druid retrieves the actual column values:

Row 0:

Segment File Location: segment_2024-01-01/data
Offset calculation: Row 0 → Block 0, Position 0
Read: user_id = "user_12345", page_views = 5
Row 988040:

Segment File Location: segment_2024-01-15/data  
Offset calculation: Row 988,040 → Block 15,438, Position 40
Read: user_id = "user_67890", page_views = 12
Row 5000138240:

Segment File Location: segment_2024-12-31/data
Offset calculation: Row 5,000,138,240 → Block 76,293,598, Position 0
Read: user_id = "user_99999", page_views = 3
Step 5: Return Results
Results:
┌──────────────┬────────────┐
│ user_id      │ page_views │
├──────────────┼────────────┤
│ user_12345   │ 5          │
│ user_45678   │ 8          │
│ user_67890   │ 12         │
│ ...          │ ...        │
│ user_99999   │ 3          │
└──────────────┴────────────┘
1.2B rows (limited to top 10,000 in response)
Query time: 234ms
Performance Breakdown
Why it's fast:

Bitmap operations: 1.2 billion row IDs identified in ~150ms
Chunk-parallel processing
Compressed operations (no decompression needed)
Only 76,294 chunks to process (not 5B individual checks)
Column scanning: Only reads matching rows (~80ms)
Direct offset access (not sequential scan)
Columnar storage means only reading 2 columns
Skip 3.8B non-matching rows entirely
Memory efficiency:
Roaring bitmaps: ~50MB compressed (vs 625MB uncompressed)
Only active chunks loaded into memory
Result set aggregated efficiently
Real-World Optimization
Druid also employs:

Segment pruning:

If query has: WHERE timestamp BETWEEN '2024-06-01' AND '2024-06-30'
Skip segments: 2024-01, 2024-02, ..., 2024-05, 2024-07, ...
Only process: 2024-06 segments (reduces from 5B to ~400M rows)
Parallel processing:

Chunk 0-1000:     Thread 1
Chunk 1001-2000:  Thread 2
Chunk 2001-3000:  Thread 3
...
Results merged
This is why Druid can query billions of rows in sub-second times - it never actually "scans" billions of rows, but uses bitmap math to instantly identify the needle in the haystack!

oh so each unique value has its own bitmap ?

YES, exactly! Each unique value in a column has its own separate Roaring Bitmap. This is a crucial insight.

The Inverted Index Structure
Think of it like an inverted index (similar to how search engines work):

Country Column (5 billion rows):

"USA" → Roaring Bitmap → [rows: 0, 3, 4, 7, 11, 25, 30, ...]
"India" → Roaring Bitmap → [rows: 1, 2, 5, 6, 8, 9, 12, ...]
"Japan" → Roaring Bitmap → [rows: 13, 14, 19, 22, 28, ...]
"Germany" → Roaring Bitmap → [rows: 15, 16, 17, 18, 20, ...]
...
(~200 unique countries total)
Device_type Column (5 billion rows):

"mobile" → Roaring Bitmap → [rows: 0, 1, 4, 5, 9, 11, ...]
"desktop" → Roaring Bitmap → [rows: 2, 3, 6, 7, 8, 10, ...]
"tablet" → Roaring Bitmap → [rows: 12, 13, 14, 15, 16, ...]
(3 unique device types)
Memory Implications
For low-cardinality columns (few unique values):

Very efficient!
Example: device_type with 3 values = 3 bitmaps
Example: country with 200 values = 200 bitmaps
For high-cardinality columns (many unique values):

Can become expensive
Example: user_id with 100M unique users = 100M bitmaps! 😱
This is why Druid doesn't create bitmap indexes for ALL columns by default
Cardinality Trade-offs
Good for Bitmap Indexing:
✅ country (200 unique values)
✅ device_type (3-5 unique values)
✅ browser (10-20 unique values)
✅ status_code (50 unique values)
✅ category (1000 unique values)
Bad for Bitmap Indexing:
❌ user_id (100M unique values) → 100M bitmaps!
❌ session_id (1B unique values) → 1B bitmaps!
❌ timestamp (continuous) → Millions of bitmaps
❌ email (unique per user) → Too many bitmaps
Real Memory Example
5 billion rows, Country column (200 unique values):

Without compression (theoretical):
- 200 bitmaps × 5 billion bits each
- = 200 × 625 MB = 125 GB

With Roaring Bitmap compression:
- Depends on data distribution
- Typical: 10-100x compression
- Realistic: 1-12 GB for all 200 bitmaps
5 billion rows, user_id column (100M unique values):

Without compression (theoretical):
- 100M bitmaps × 5 billion bits each
- = Impossible! (6.25 PB)

With Roaring Bitmap compression:
- Still huge! Each bitmap might compress to 1-10 KB
- 100M × 5 KB = 500 GB minimum
- This is why you DON'T index high-cardinality columns!
Druid's Strategy
Druid lets you configure which columns get bitmap indexes:

json
{
  "dimensionsSpec": {
    "dimensions": [
      {
        "name": "country",
        "type": "string",
        "createBitmapIndex": true  // ✅ Low cardinality
      },
      {
        "name": "user_id", 
        "type": "string",
        "createBitmapIndex": false  // ❌ High cardinality
      }
    ]
  }
}
Query Performance Impact
With bitmap index:

sql
WHERE country = 'USA'  
→ Instant lookup: Get bitmap for 'USA'
→ ~1ms to find matching rows
Without bitmap index:

sql
WHERE user_id = 'user_12345'
→ Full column scan required
→ ~100-1000ms to scan 5B rows (still columnar, so faster than row-based)
Summary
✅ Yes, each unique value = one bitmap
📊 Works brilliantly for low-cardinality columns (< 10K unique values)
💾 Becomes memory-intensive for high-cardinality columns
🎯 Druid intelligently chooses which columns to index based on cardinality
⚡ This is why filtering on country is instant, but filtering on unique IDs requires scanning
This design is what makes Druid so fast for typical OLAP queries that filter on dimensions like country, device type, product category, etc., while avoiding the memory explosion that would come from indexing every possible column!

lets say i want to query by userid , which will obviously not have a bit map does that mean that , 
querying by userid will be slow ?

Great question! Yes, querying by user_id will be slower than querying by indexed columns like country, BUT it's not as bad as you might think. Let me explain why:

Speed Comparison
With Bitmap Index (country = 'USA'):

Time: ~1-5ms
Process: Direct bitmap lookup → instant row identification
Without Bitmap Index (user_id = 'user_12345'):

Time: ~50-500ms (depending on data size and storage)
Process: Columnar scan of user_id column
So yes, it's 10-100x slower, but still quite fast compared to traditional databases!

Why It's Still Reasonably Fast
1. Columnar Storage
Druid stores data in columnar format:

Traditional Row Storage:
[row0: ts, country, device, user_id, views] → Scan entire row
[row1: ts, country, device, user_id, views] → Scan entire row
[row2: ts, country, device, user_id, views] → Scan entire row
...

Druid Columnar Storage:
user_id column: [user_12345, user_67890, user_11111, ...]
→ Only scan this one column, skip all others!
Benefit: Reading 1 column from 5B rows is much faster than reading 10 columns from 5B rows.

2. Compression
The user_id column is compressed (dictionary encoding, LZ4, etc.):

Raw data: 5B rows × 20 bytes/user_id = 100 GB
Compressed: ~10-20 GB
→ Less disk I/O, more data fits in cache
3. Segment Pruning
If your query has other filters, Druid can skip entire segments:

sql
SELECT * FROM events 
WHERE user_id = 'user_12345' 
  AND timestamp BETWEEN '2024-11-01' AND '2024-11-30'
```

**Process:**
```
Total segments: 365 (one per day for a year)
After time filter: Only scan November segments (30 segments)
Data reduced: 5B rows → ~410M rows
Scan time: 500ms → 40ms
```

### 4. **Parallel Scanning**
Druid scans segments in parallel across multiple threads:
```
Thread 1: Scan segment 2024-11-01
Thread 2: Scan segment 2024-11-02
Thread 3: Scan segment 2024-11-03
...
Thread 30: Scan segment 2024-11-30

All running simultaneously!
Real-World Query Pattern
Bad Query (slow):

sql
SELECT * FROM events 
WHERE user_id = 'user_12345'
Scans ALL 5B rows across ALL segments
Time: ~500-2000ms
Better Query (much faster):

sql
SELECT * FROM events 
WHERE user_id = 'user_12345' 
  AND timestamp >= CURRENT_TIMESTAMP - INTERVAL '7' DAY
Scans only last 7 days of data
Time: ~50-100ms
Best Query (fastest):

sql
SELECT * FROM events 
WHERE country = 'USA'           -- Bitmap index: instant filtering
  AND device_type = 'mobile'    -- Bitmap index: instant filtering
  AND user_id = 'user_12345'    -- Scan only matching rows from above
Bitmap filters reduce to 100M rows
Then scan user_id on only those 100M rows
Time: ~10-20ms
Optimization Strategies
Strategy 1: Bloom Filters
Druid can use Bloom filters for high-cardinality columns:

json
{
  "type": "bloom",
  "column": "user_id"
}
```

**How it helps:**
- Probabilistic data structure
- Can quickly answer: "Is user_12345 definitely NOT in this segment?"
- Skips entire segments without scanning
- False positive rate: ~1-5%

**Result:**
```
Before: Scan 365 segments
After Bloom filter: Scan ~10-20 segments (where user might exist)
Speed improvement: 20-30x faster
Strategy 2: Partitioning
Partition data by a related dimension:

json
{
  "partitionsSpec": {
    "type": "hashed",
    "partitionDimensions": ["country"]
  }
}
Query:

sql
WHERE country = 'USA' AND user_id = 'user_12345'
Only scan USA partitions
Reduces scan from 5B rows → 1.5B rows
Strategy 3: Pre-aggregation
Create a rollup table for user-level queries:

sql
-- Instead of querying raw events table
-- Create user summary table
INSERT INTO user_summary
SELECT user_id, 
       COUNT(*) as event_count,
       SUM(page_views) as total_views
FROM events
GROUP BY user_id

-- Now query the summary table
SELECT * FROM user_summary WHERE user_id = 'user_12345'
-- Time: <10ms (much smaller table, can be fully indexed)
When to Index user_id
You might consider indexing user_id if:

Lower cardinality subset: You have 100K power users (not 100M)
Memory available: You have enough RAM for the bitmaps
Query pattern: 90% of queries filter by user_id
Hybrid approach: Index active users from last 30 days only
json
{
  "dimensions": [
    {
      "name": "user_id",
      "type": "string",
      "createBitmapIndex": false  // Default: no index
    },
    {
      "name": "active_user_id",
      "type": "string", 
      "createBitmapIndex": true   // Only for recent active users
    }
  ]
}
Summary
Query Type	Speed	Method
Indexed column (country)	1-5ms	Bitmap lookup
Non-indexed + time filter	50-100ms	Columnar scan (pruned)
Non-indexed + Bloom filter	100-200ms	Skip segments
Non-indexed alone	500-2000ms	Full columnar scan
Key Takeaway: Querying by user_id is slower but still acceptable for many use cases, especially when combined with:

Time-based filters (most important!)
Bloom filters
Partitioning strategies
Pre-aggregated summary tables
The real secret is: Don't query 5 billion rows directly. Always add time boundaries or other indexed filters to narrow down the search space first!








Roaring bitmap indexes in Apache Druid - Claude
