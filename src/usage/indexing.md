# Indexing

Indexing is the process of building a data structure that allows for efficient search. pgvecto.rs supports three indexing algorithms: brute force (FLAT), IVF, and HNSW. The default algorithm is HNSW.

Assuming there is a table `items` and there is a column named `embedding` of type `vector(n)`, you can create a vector index for squared Euclidean distance with the following SQL.

```sql
CREATE INDEX ON items USING vectors (embedding vector_l2_ops);
```

The `vector_l2_ops` is an operator class for squared Euclidean distance. There are all operator classes for each vector type and distance type.

| Vector type | Distance type              | Operator class |
| ----------- | -------------------------- | -------------- |
| vector      | squared Euclidean distance | vector_l2_ops  |
| vector      | negative dot product       | vector_dot_ops |
| vector      | negative cosine similarity | vector_cos_ops |
| vecf16      | squared Euclidean distance | vecf16_l2_ops  |
| vecf16      | negative dot product       | vecf16_dot_ops |
| vecf16      | negative cosine similarity | vecf16_cos_ops |

Now you can perform a KNN search with the following SQL again, this time the vector index is used for searching.

```sql
SELECT * FROM items ORDER BY embedding <-> '[3,2,1]' LIMIT 5;
```

::: details

pgvecto.rs constructs the index asynchronously. When you insert new rows into the table, they will first be placed in an append-only file. The background thread will periodically merge the newly inserted row to the existing index. When a user performs any search prior to the merge process, it scans the append-only file to ensure accuracy and consistency.

:::

## Options

We utilize TOML syntax to express the index's configuration. Here's what each key in the configuration signifies:

| Key        | Type  | Description                            |
| ---------- | ----- | -------------------------------------- |
| segment    | table | Options for segments.                  |
| optimizing | table | Options for background optimizing.     |
| indexing   | table | The algorithm to be used for indexing. |

Options for table `segment`.

| Key                      | Type    | Description                                                         |
| ------------------------ | ------- | ------------------------------------------------------------------- |
| max_growing_segment_size | integer | Maximum size of unindexed vectors. Default value is `20_000`.       |
| max_sealed_segment_size  | integer | Maximum size of vectors for indexing. Default value is `1_000_000`. |

Options for table `optimizing`.

| Key                | Type    | Description                                                                                                                                                |
| ------------------ | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| optimizing_threads | integer | Maximum threads for indexing. Default value is the sqrt of number of cores.                                                                                |
| sealing_secs       | integer | If a writing segment larger than `sealing_size` do not accept new data for `sealing_secs` seconds, the writing segment will be turned to a sealed segment. |
| sealing_size       | integer | See above.                                                                                                                                                 |

Options for table `indexing`.

| Key  | Type  | Description                                                             |
| ---- | ----- | ----------------------------------------------------------------------- |
| flat | table | If this table is set, brute force algorithm will be used for the index. |
| ivf  | table | If this table is set, IVF will be used for the index.                   |
| hnsw | table | If this table is set, HNSW will be used for the index.                  |

You can choose only one algorithm in above indexing algorithms. Default value is `hnsw`.

Options for table `flat`.

| Key          | Type  | Description                                |
| ------------ | ----- | ------------------------------------------ |
| quantization | table | The algorithm to be used for quantization. |

Options for table `ivf`.

| Key              | Type    | Description                                                     |
| ---------------- | ------- | --------------------------------------------------------------- |
| nlist            | integer | Number of cluster units. Default value is `1000`.               |
| least_iterations | integer | Least iterations for K-Means clustering. Default value is `16`. |
| iterations       | integer | Max iterations for K-Means clustering. Default value is `500`.  |
| quantization     | table   | The quantization algorithm to be used.                          |

Options for table `hnsw`.

| Key             | Type    | Description                                        |
| --------------- | ------- | -------------------------------------------------- |
| m               | integer | Maximum degree of the node. Default value is `12`. |
| ef_construction | integer | Search scope in building. Default value is `300`.  |
| quantization    | table   | The quantization algorithm to be used.             |

Options for table `quantization`.

| Key     | Type  | Description                                         |
| ------- | ----- | --------------------------------------------------- |
| trivial | table | If this table is set, no quantization is used.      |
| scalar  | table | If this table is set, scalar quantization is used.  |
| product | table | If this table is set, product quantization is used. |

You can choose only one algorithm in above indexing algorithms. Default value is `trivial`.

Options for table `product`.

| Key    | Type    | Description                                                                                                              |
| ------ | ------- | ------------------------------------------------------------------------------------------------------------------------ |
| sample | integer | Samples to be used for quantization. Default value is `65535`.                                                           |
| ratio  | string  | Compression ratio for quantization. Only `"x4"`, `"x8"`, `"x16"`, `"x32"`, `"x64"` are allowed. Default value is `"x4"`. |

## Progress View

We also provide a view `pg_vector_index_info` to monitor the progress of indexing.

| Column       | Type   | Description                                                                         |
| ------------ | ------ | ----------------------------------------------------------------------------------- |
| tablerelid   | oid    | The oid of the table.                                                               |
| indexrelid   | oid    | The oid of the index.                                                               |
| tablename    | name   | The name of the table.                                                              |
| indexname    | name   | The name of the index.                                                              |
| idx_status   | text   | Its value is `NORMAL` or `UPGRADE`. Whether this index is normal or needs upgrade.  |
| idx_indexing | bool   | Not null if `idx_status` is `NORMAL`. Whether the background thread is indexing.    |
| idx_tuples   | int8   | Not null if `idx_status` is `NORMAL`. The number of tuples.                         |
| idx_sealed   | int8[] | Not null if `idx_status` is `NORMAL`. The number of tuples in each sealed segment.  |
| idx_growing  | int8[] | Not null if `idx_status` is `NORMAL`. The number of tuples in each growing segment. |
| idx_write    | int8   | Not null if `idx_status` is `NORMAL`. The number of tuples in write buffer.         |
| idx_size     | int8   | Not null if `idx_status` is `NORMAL`. The byte size for all the segments.           |
| idx_config   | text   | Not null if `idx_status` is `NORMAL`. The configuration of the index.               |

## Examples

There are some examples.

```sql
-- HNSW algorithm, default settings.

CREATE INDEX ON items USING vectors (embedding vector_l2_ops);

--- Or using bruteforce with PQ.

CREATE INDEX ON items USING vectors (embedding vector_l2_ops)
WITH (options = $$
[indexing.flat]
quantization.product.ratio = "x16"
$$);

--- Or using IVFPQ algorithm.

CREATE INDEX ON items USING vectors (embedding vector_l2_ops)
WITH (options = $$
[indexing.ivf]
quantization.product.ratio = "x16"
$$);

-- Use more threads for background building the index.

CREATE INDEX ON items USING vectors (embedding vector_l2_ops)
WITH (options = $$
optimizing.optimizing_threads = 16
$$);

-- Prefer smaller HNSW graph.

CREATE INDEX ON items USING vectors (embedding vector_l2_ops)
WITH (options = $$
segment.max_growing_segment_size = 200000
$$);
```