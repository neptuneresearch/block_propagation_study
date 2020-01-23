# Block Propagation Study
Author: Neptune Research & Isthmus (Noncesense Research Lab, noncesense.org)  
Date: January 8 2020

# Overview
- Node Received Timestamp (NRT) is the value of a Monero node's system clock at the time it receives a block.
- NRT data, recorded from [monerod-archive](https://github.com/neptuneresearch/monerod-archive) nodes in different world regions, is imported into PostgreSQL.
- Propagation Time Lower Bound, a formula for the MAX-MIN of all NRT timestamps from all nodes, is computed on the data set and graphed.

# Data in this repository
| Filename | Description | Size |
| - | - | - |
| ```archive_nrt_synced.csv``` | Base data set ```archive_nrt_synced``` | 392,666 rows, 14,999,588 B |
| ```prop_time_lower_bound.csv``` | Result view ```archive_nrt_synced__proptimelowerbound``` | 191,597 rows, 3,547,092 B |

# Source data
## Table: ```archive```
This table supports all fields output by monerod-archive. These columns are relevant to this study:

| Column | Description |
| - | - |
| ```node_id``` | Source node |
| ```height``` | Block height |
| ```is_node_synced``` | Is the node's main blockchain fully synced? (True/false)
| ```nrt``` | NRT timestamp (Milliseconds scale) |
| ```mrt``` | MRT timestamp (Seconds scale) |
| ```deltart``` | ```= (NRT / 1000) - MRT``` |


## Data subset for this study
We filter the archive data set as follows.
- ```is_node_synced = TRUE```, to include only blocks received while the node was fully synced.
- ```height > 1```, to include only valid data (some incomplete records were recorded with height 0, and there happens to be an alt block at height 1 in this data set).

How much data do we have? Query:

```
SELECT
  node_id,
  MIN(height) AS height_min,
  MAX(height) AS height_max,
  MAX(height) - MIN(height) AS height_range
FROM archive
WHERE is_node_synced = TRUE AND height > 1
GROUP BY node_id
ORDER BY node_id ASC
```

Results, in order of descending height range:

| node_id | height_min | height_max | height_range |
| - | - | - | - |
| NYC | 1,787,564 | 1,989,130 | 201,566 |
| London | 1,822,549 | 1,989,130 | 166,581 |
| Frankfurt | 1,872,531 | 1,989,130 | 116,599 |
| Tokyo | 1,977,879 | 1,989,130 | 11,251 |

Bug: Alt blocks were included in this study. There are not many overall, but they are not relevant and likely introduced outliers. Future versions should include ```WHERE is_alt_block = FALSE```.

| node_id | alt_block_count |
| - | - |
| Tokyo | 24 (0.002%) |
| Frankfurt | 311 (0.002%) |
| NYC | 489 (0.002%) |
| London | 372 (0.002%) |

## Views
We make 2 views for the source data, which we will combine in the final query.

1. ```archive_nrt_synced```: The data set, NRT per height per node (also has MRT and deltaRT for future analysis).

```
CREATE OR REPLACE VIEW archive_nrt_synced
AS
  SELECT
    node_id,
    height,
    nrt,
    mrt,
    deltart
  FROM archive
  WHERE is_node_synced = TRUE AND height > 1
  ORDER BY node_id ASC, height ASC
```

2. ```archive_nrt_synced__height_nodecount```: A utility view for counting the number of nodes that recorded each height.

```
CREATE OR REPLACE VIEW archive_nrt_synced__height_nodecount
AS
  SELECT
    height,
    COUNT(node_id) AS node_count
  FROM archive_nrt_synced
  GROUP BY height
  ORDER BY height ASC
```

# Propagation Time Lower Bound
Let ```NRT(h,x)``` be the node receipt timestamp of block with hash ```h``` on archival node ```x```.

```
prop_time_lower_bound(h) = MAX[NRT(h,1), NRT(h,2), NRT(h,3), NRT(h,4)] - MIN[NRT(h,1), NRT(h,2), NRT(h,3), NRT(h, 4)]
```

We run this formula ```prop_time_lower_bound``` at each recorded block height, and add on the number of nodes per height ```node_count``` and the block size ```size``` (from separate data set ```monero``` which is the real and complete blockchain).

## Query

```
CREATE OR REPLACE VIEW archive_nrt_synced__proptimelowerbound
AS
  WITH p AS (
      SELECT 
          height,
          MAX(nrt) - MIN(nrt) AS prop_time_lower_bound
      FROM archive_nrt_synced
      GROUP BY height
  )
  SELECT
      p.*,
      nc.node_count,
      b.size
  FROM p
  JOIN archive_nrt_synced__height_nodecount nc ON nc.height = p.height
  JOIN monero b ON b.height = p.height
  ORDER BY height ASC
```

## Results

1. log(prop_time_lower_bound) x number
![https://i.imgur.com/JgJwray.png](https://i.imgur.com/JgJwray.png)

2. log(size) x log(prop_time_lower_bound)
![https://i.imgur.com/WlTVYQM.png](https://i.imgur.com/WlTVYQM.png)