# Today's study notes: Embeddings and Vector Databases (Chapter 4)

9 November 2026 · Based on Chapter 4: Embeddings and Vector Databases of the Local & Private AI Engineering Handbook (Part 1).

## 1. Embeddings (Section 4.1, Listing 4.1)

- An embedding model turns text into a fixed-length vector (dimension `d`, often 384, 768 or 1,024). Texts with similar meaning land close together, even if they share no words.
- Retrieval becomes geometry: embed every chunk once, embed the query, return the nearest chunks.
- Use the **same model** for documents and queries, record the model name next to the index, and expect to re-embed everything if you change it.

Example local embedding models:

| Model             | Dimension | Notes                                       |
| ----------------- | --------- | ------------------------------------------- |
| all-MiniLM-L6-v2  | 384       | Small and fast, English, short max input    |
| bge-small-en-v1.5 | 384       | Small retrieval-oriented English model      |
| bge-base-en-v1.5  | 768       | Better quality, about 3x the compute        |
| nomic-embed-text  | 768       | Long input, also available through Ollama   |
| bge-m3            | 1,024     | Multilingual, long input, heavier           |

How to choose: document languages, the model's max input length vs. chunk size (longer input is silently cut off), speed and memory, and above all retrieval quality on your own data.

## 2. Similarity metrics (Section 4.2, Listing 4.2)

- **Cosine:** the angle between vectors; ignores length.
- **Dot product:** grows with both angle and length.
- **Euclidean (L2):** straight-line distance; smaller is closer.

The key rule: **normalise vectors to length 1 and use inner product.** Then `||a - b||^2 = 2 - 2cos(theta)`, so all three metrics rank neighbours identically. On raw (unnormalised) vectors the dot-product ranking differs (Listing 4.2 showed a different top 5).

Watch the conventions, since a "distance" is often misread as a "similarity":

- FAISS L2 returns squared distance (smaller is closer); IP returns larger is closer.
- Chroma returns a distance (cosine distance = 1 - similarity).
- Qdrant cosine and dot: larger is closer; Euclid is a distance.

## 3. FAISS: Flat, IVF and HNSW (Section 4.3, Listings 4.3 and 4.4)

FAISS is an index only: vectors, no text, no metadata, no persistence of documents. You map vector positions back to documents yourself.

| Index | How it works                                                        | Trade-off                                                  |
| ----- | ------------------------------------------------------------------- | ---------------------------------------------------------- |
| Flat  | Compares the query with every vector; exact (recall 100%)           | Nothing to tune; fine up to a few hundred thousand vectors |
| IVF   | k-means into `nlist` cells; scans only the `nprobe` nearest cells   | Needs training; raise `nprobe` for more recall, less speed |
| HNSW  | Layered graph with `M` links per node; `efSearch` sets beam width   | No training, great recall, but more memory and slower build |

Measured in the book's build (100,000 synthetic vectors, d = 128, one CPU thread, so a demo of the shape, not a benchmark for my hardware):

- Flat (exact): recall 1.000, 463 queries/s.
- IVF `nprobe = 4`: recall 0.999, 15,414 queries/s (about 33x faster).
- HNSW `efSearch = 64`: recall 0.999, 12,419 queries/s (about 27x faster).
- Build time: IVF about 2.5 s, HNSW about 20 s. HNSW buys query speed and recall with build time and memory.

Practical notes: FAISS returns positions (and -1 when there are fewer results than requested), and has no real metadata filtering. Start with Flat and move on only when measurement says so.

## 4. Chroma, Qdrant, Weaviate and pgvector (Section 4.4)

A vector database adds what an app needs on top of an index: text and metadata storage, filtering, updates and deletes, persistence, often a network API.

| Store    | Runs as                       | Good for                                           | Watch out for                                      |
| -------- | ----------------------------- | -------------------------------------------------- | -------------------------------------------------- |
| FAISS    | Library in your process       | Fastest pure vector search, many index types       | Vectors only, you manage IDs and persistence       |
| Chroma   | Embedded, or client-server    | Simplest API, prototypes                           | Default embedder downloads a model                 |
| Qdrant   | Server (Docker) or local mode | Rich payload filters, HNSW, snapshots, hybrid      | A service to run and monitor                       |
| Weaviate | Server (Docker)               | Built-in hybrid search, multi-tenancy              | Heaviest to run and learn                          |
| pgvector | Extension inside PostgreSQL   | One database for metadata and vectors, SQL filters | Can trail dedicated stores at very large scale     |

Private-deployment trap: if you add documents to Chroma without embeddings, it downloads a default embedding model. In an air-gapped setup, always compute embeddings yourself and pass them in.

## 5. Choosing a store (Section 4.5, Listing 4.5)

- Prototype or notebook: FAISS Flat, or Chroma if I want text and metadata stored for me.
- On-prem service with filters and several users: Qdrant, or pgvector if Postgres is already in the system.
- Built-in hybrid search or multi-tenancy: Weaviate, or Qdrant with sparse and dense vectors.
- Static corpus, no filters, maximum speed: FAISS with IVF or HNSW.
- Tens of millions of vectors or more: a server database with compression and sharding, benchmarked on my own data.

Hide the store behind a small `VectorStore` interface (`add`, `search`, `delete`) so backends can be swapped with one adapter class. Store the embedding model name, chunking settings and a content hash with each chunk to detect staleness.

## 6. Hands-on lab (Listings 4.6 and 4.7)

Indexed the same 8-document toy corpus in FAISS, Qdrant and Chroma using a toy word-hashing embedder (no model download). Expected result: all three return the same top hit, `To send an item back, print the return label from your account page.` with scores 0.41, 0.12 and 0.05. With a real model (all-MiniLM-L6-v2, DIM = 384) they should still agree with each other and also find paraphrases the toy embedder cannot.

## Common mistakes to avoid

- Embedding documents with one model and queries with another.
- Using inner product on unnormalised vectors and reading scores as similarities.
- Chunks longer than the embedding model's max input (silently truncated).
- Letting Chroma download its default embedder in an air-gapped deployment.
- Tuning `nprobe` or `efSearch` without measuring recall on my own data.
- Treating FAISS as a database.

## The big picture

Embeddings turn meaning into distance. Normalise and use inner product so every metric agrees, pick an index by the recall, speed, build time and memory trade-off, and pick a store by the filtering, persistence and scale you actually need, behind an interface so it can be swapped.

## Check yourself (answers)

1. Same model for documents and queries: vectors from different models live in different spaces, so distances are meaningless.
2. Cosine, dot and L2 rank identically when all vectors are normalised to length 1.
3. `nprobe` is the number of nearest IVF cells scanned per query; raising it increases recall and cost.
4. A vector database adds, for example, text and metadata storage, filtering, updates and deletes, persistence, a network API.
5. Already on PostgreSQL with moderate scale: pgvector, since metadata, SQL filters and vectors live in one database.

## Next time

Part 2 of the handbook: Chapter 5 onwards (production RAG).
