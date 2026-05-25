## Day 1 — Graph Modelling & Advanced Cypher

- Graph modelling principles, node labels, relationship types, properties, and schema design patterns
- Variable-length paths, UNWIND, WITH chaining, and APOC procedures
- Query profiling and optimisation with EXPLAIN and PROFILE
- Lab: Design a graph schema for RAG document storage; write and optimise Cypher queries against the model

***

**Graph Modelling Principles, Node Labels, Relationship Types, Properties, and Schema Design Patterns**

- A **graph database** stores data as a network of nodes and relationships — not rows and columns. The fundamental idea is that for certain problem domains, the relationships between entities are as analytically important as the entities themselves, and a structure that treats those relationships as first-class citizens is the right tool.
- In a relational database, relationships live as foreign key columns or join tables — they are implied and computed on demand at query time. In Neo4j, a **relationship is a physically stored, typed, directional object** that permanently connects two nodes via a pointer. It is not computed; it is retrieved in constant time.
- **Why index-free adjacency matters:** in SQL, a two-hop traversal requires three JOINs; a five-hop traversal requires six. Each additional hop multiplies the number of intermediate rows the database must evaluate — query cost grows exponentially with depth. In Neo4j, each hop follows a stored pointer from a node directly to its neighbours — the cost per hop is constant regardless of how large the overall graph is. This property is called **index-free adjacency** and is the architectural reason graph databases outperform relational databases on deeply connected queries.
- **Nodes** are the entities in your graph — people, documents, products, locations, concepts. Every node can have zero or more **labels** and zero or more **properties**.
- **Node labels** classify nodes into categories.
  - A single node can carry **multiple labels simultaneously** — a node can be labelled both `:Author` and `:Reviewer`, inheriting the semantics and query-accessibility of both categories without any data duplication.
  - Labels serve as the primary filter in Cypher pattern matching — `MATCH (n:Document)` considers only Document-labelled nodes; unlabelled or differently-labelled nodes are never touched. This makes label selection a primary performance lever for large graphs.
  - Naming convention: **PascalCase** — `DocumentChunk`, `ResearchPaper`, `AuthorEntity`. No spaces, no underscores.
- **Relationship types** define the semantic meaning of a connection.
  - Every relationship has exactly **one type** — you cannot attach multiple types to a single relationship instance. If you need to express multiple connection semantics between two nodes, create multiple distinct relationships.
  - Relationships are always **directional** — `(a)-[:CITES]->(b)` means `a` cites `b`. The direction carries meaning. Cypher can traverse relationships in either direction in a query (`-[:CITES]-`), but the stored direction is always significant for the domain model.
  - Relationships can carry **properties** — `(a)-[:CITED_BY {year: 2023, context: "methodology"}]->(b)` — storing attributes of the connection itself, not just the fact of connection.
  - Naming convention: **SCREAMING_SNAKE_CASE** — `AUTHORED_BY`, `CONTAINS_CHUNK`, `SEMANTICALLY_SIMILAR_TO`.
- **Properties** are key-value pairs attached to nodes or relationships.
  - Supported types: `String`, `Integer`, `Float`, `Boolean`, `Date`, `DateTime`, `Point` (spatial), and **arrays** of any of those types.
  - Properties represent **intrinsic attributes** of the entity. The rule of thumb: if a piece of information describes the entity itself (a person's name, a document's title, a chunk's embedding), it is a property. If it describes how one entity relates to another (who authored a document, which chunk belongs to which document), it is a relationship.
  - Avoid storing lists of IDs as array properties — `author_ids: [1, 2, 3]` — that pattern is a relational habit. In a graph, model those as actual relationships to Author nodes.
- **Schema design patterns for graph modelling:**
  - **Avoid super-nodes:** a node with millions of outgoing relationships (a "hub") creates traversal bottlenecks because every query touching it must evaluate all its relationships. Break super-nodes up using intermediate category or segment nodes.
  - **Reify relationships into nodes when they carry attributes or need further connections:** an employment relationship between Person and Company that has a start date, end date, and title is better modelled as an `:Employment` node connected to both, rather than an overloaded relationship with many properties.
  - **Model for your queries:** unlike a relational schema designed to minimise redundancy, a graph schema is designed for the traversal patterns your application needs. Think about "what are the most common graph traversal paths?" and model the schema to make those paths cheap.
  - **Graph schema for RAG document storage:** a production-grade schema for Retrieval-Augmented Generation (RAG) typically includes:
    - `(:Document {id, title, source, ingested_at})` — represents the source document
    - `(:DocumentChunk {id, text, embedding, chunk_index, token_count})` — a segment of the document; the unit of retrieval
    - `(:Entity {name, type})` — a named entity extracted from text (person, organisation, location, concept)
    - `(:Topic {name})` — an abstract subject cluster
    - Relationships: `(Document)-[:HAS_CHUNK]->(DocumentChunk)`, `(DocumentChunk)-[:MENTIONS]->(Entity)`, `(Entity)-[:BELONGS_TO]->(Topic)`, `(DocumentChunk)-[:PRECEDES]->(DocumentChunk)` (sequential ordering within a document), `(Entity)-[:RELATED_TO {score}]->(Entity)` (entity co-occurrence or semantic similarity)
  - This combined structure supports both vector-based semantic search (searching on `DocumentChunk.embedding`) and graph traversal-based context expansion (following entity and topic connections to retrieve structurally related chunks that a pure vector search would miss).

***

**Variable-Length Paths, UNWIND, WITH Chaining, and APOC Procedures**

- **Variable-length paths** are Cypher's mechanism for traversing a graph to an unknown or bounded depth in a single pattern expression — a capability that has no clean SQL equivalent and is one of the most distinctive features of Cypher.
- Syntax: `(a)-[:RELATIONSHIP_TYPE*min..max]->(b)` — traverse between `min` and `max` hops.
  - `(a)-[:CITES*1..3]->(b)` — match all nodes `b` reachable from `a` via 1 to 3 `CITES` hops.
  - `(a)-[:CITES*]->(b)` — unlimited depth traversal. Use with extreme care on dense graphs — unbounded traversals can visit every node in the graph.
  - `(a)-[:CITES*3]->(b)` — exactly 3 hops. A fixed-depth traversal.
  - Variable-length paths are used for finding all downstream citations of a paper, all reachable communities from a starting node, or all documents connected to an entity within two hops — patterns that require flexible depth.
- **The `path` variable:** you can bind the traversed path to a variable for further analysis.
  ```cypher
  MATCH path = (a:Document)-[:CITES*1..3]->(b:Document)
  RETURN length(path) AS hops, nodes(path) AS path_nodes
  ```
  `nodes(path)` returns the ordered list of nodes along the path; `relationships(path)` returns the relationships; `length(path)` returns the number of hops.
- **Shortest path:** `shortestPath()` and `allShortestPaths()` are built-in functions for shortest path traversal.
  ```cypher
  MATCH (a:Author {name: "Alice"}), (b:Author {name: "Bob"})
  MATCH path = shortestPath((a)-[:COLLABORATED_WITH*]-(b))
  RETURN path
  ```
- **UNWIND** is the operation that takes a list (an array property, a collected result, or a literal list) and expands it into individual rows — essentially the opposite of `collect()`.
  - Use case 1 — iterating over an array property: a `DocumentChunk` node has an array of extracted keywords. To process each keyword as a separate row, `UNWIND chunk.keywords AS keyword`.
  - Use case 2 — processing a collected list further: after `collect(n)` aggregates nodes into a list, `UNWIND` re-expands them for further per-row operations.
  - Use case 3 — batch creation from a list: passing a list of parameter maps and using `UNWIND` to create nodes for each map entry in a single query — the standard pattern for bulk data loading in Cypher.
  ```cypher
  UNWIND [{name: "Alice", age: 30}, {name: "Bob", age: 25}] AS person_data
  CREATE (:Person {name: person_data.name, age: person_data.age})
  ```
- **WITH chaining** is the mechanism for piping results from one Cypher clause to the next, creating a multi-stage query pipeline — analogous to using CTEs (WITH clauses) in SQL.
  - `WITH` passes selected variables (and optionally renames, transforms, or aggregates them) to the subsequent clause. Variables not listed after `WITH` are dropped from scope.
  - `WITH` allows intermediate filtering (`WHERE` after `WITH`), intermediate aggregation (`count()`, `sum()`, `collect()`), and intermediate limiting or ordering of results before further processing.
  ```cypher
  MATCH (a:Author)-[:AUTHORED]->(d:Document)
  WITH a, count(d) AS doc_count
  WHERE doc_count > 5
  MATCH (a)-[:AFFILIATED_WITH]->(i:Institution)
  RETURN a.name, doc_count, i.name
  ```
  This query first finds authors and counts their documents, filters to only prolific authors, then continues to find their institutional affiliations — something that would require a subquery or CTE in SQL but reads as a natural pipeline in Cypher.
- **APOC (Awesome Procedures On Cypher)** is the standard Neo4j plugin that extends Cypher with hundreds of utility procedures covering data import, string manipulation, graph algorithms, JSON handling, metadata inspection, and more.
  - **`apoc.do.when` / `apoc.when`** — conditional branching in Cypher (Cypher has no native IF/ELSE for data operations).
  - **`apoc.periodic.iterate`** — the standard mechanism for processing large datasets in batches without hitting memory limits. Specify an outer query to produce rows and an inner query to process each row; APOC batches the inner operation in configurable transaction sizes.
    ```cypher
    CALL apoc.periodic.iterate(
      "MATCH (c:DocumentChunk) WHERE c.embedding IS NULL RETURN c",
      "SET c.embedding = generateEmbedding(c.text)",
      {batchSize: 100, parallel: false}
    )
    ```
  - **`apoc.load.json`** / **`apoc.load.csv`** — load data from external JSON or CSV sources into the graph directly from a Cypher query.
  - **`apoc.path.subgraphNodes` / `apoc.path.expand`** — controlled graph expansion with filtering on relationship types, node labels, and depth — more flexible than variable-length path syntax for complex traversal constraints.
  - **`apoc.merge.node`** / **`apoc.merge.relationship`** — idempotent node and relationship creation with dynamic labels and types (Cypher's native `MERGE` requires static labels and types).
  - **`apoc.meta.graph`** — returns a summary of the graph's schema — what node labels exist, what relationship types connect them. Invaluable for exploring an unfamiliar graph database.

***

**Query Profiling and Optimisation with EXPLAIN and PROFILE**

- Writing a Cypher query that returns correct results is the first step. Writing one that returns those results efficiently at scale is the second — and EXPLAIN and PROFILE are the tools that show you exactly what Neo4j is doing under the hood to execute your query.
- **EXPLAIN** prepends to any Cypher query and returns the **query execution plan** without actually executing the query. It shows you what Neo4j *would* do — which indexes it would use, what operations it would perform, and in what order — without touching any data and without any execution cost.
  ```cypher
  EXPLAIN MATCH (a:Author {name: "Alice"})-[:AUTHORED]->(d:Document)
  RETURN d.title
  ```
  The output is a tree of **query operators** — `NodeIndexSeek`, `Expand(All)`, `Filter`, `Projection`, etc. — each representing a step in the execution plan. Read the plan from the bottom up (innermost operations execute first).
- **PROFILE** executes the query in full and returns the execution plan annotated with actual runtime statistics — **rows produced**, **db hits** (disk/cache reads), and **elapsed time** for each operator.
  - `db hits` is the key metric — it represents how many reads Neo4j performed. A high `db hits` count on a Filter operator indicates a label scan with a subsequent property filter rather than an index lookup — a common performance problem.
  - A `NodeByLabelScan` in the plan means Neo4j is scanning all nodes with a given label. If that label has millions of nodes, this is a full scan — almost always a sign that a property index is missing.
  - A `NodeIndexSeek` means Neo4j is using an index to jump directly to matching nodes — this is what you want for property-based lookups.
- **Creating indexes:** the most impactful optimisation for most Cypher queries is ensuring that frequently-queried properties have an index.
  ```cypher
  CREATE INDEX author_name_idx FOR (a:Author) ON (a.name)
  CREATE INDEX chunk_id_idx FOR (c:DocumentChunk) ON (c.id)
  ```
  After creating an index, re-run `EXPLAIN` — you should see the operator change from `NodeByLabelScan` to `NodeIndexSeek`.
- **Composite indexes:** when queries frequently filter on multiple properties of the same node label, a composite index covers both properties.
  ```cypher
  CREATE INDEX doc_source_date_idx FOR (d:Document) ON (d.source, d.ingested_at)
  ```
- **Query optimisation rules — common patterns:**
  - **Filter early using WHERE:** place `WHERE` clauses as early as possible in the query — immediately after the first `MATCH` that introduces the relevant variable, not at the end of the query. This reduces the number of rows that subsequent operations must process.
  - **Use LIMIT for exploration:** when exploring an unfamiliar graph or developing a query, always add `LIMIT 25` to prevent accidentally triggering a full graph traversal that returns millions of rows.
  - **Avoid Cartesian products:** two consecutive `MATCH` clauses with no shared variable produce a Cartesian product (every combination of rows from both matches). This is almost always a logic error. Always ensure consecutive MATCH clauses share at least one variable or use appropriate `WITH` filtering between them.
  - **Parameterise queries:** use Cypher parameters (`$param_name`) rather than string-interpolated query values. Parameterised queries are compiled once and their execution plans are cached — string-interpolated queries are recompiled on every execution, adding overhead and bypassing the plan cache.
  ```cypher
  // Bad — string interpolation, no plan caching
  MATCH (a:Author {name: "Alice"}) RETURN a
  
  // Good — parameterised, plan is cached
  MATCH (a:Author {name: $author_name}) RETURN a
  ```
  - **Use MERGE carefully:** `MERGE` is a read-then-write operation (find existing or create new). On large graphs without appropriate indexes on the merge key, each `MERGE` performs a full label scan. Always ensure the property you are merging on has an index.

***

**Wrap-Up**

- Day 1 established the conceptual and practical foundation for all graph work in this programme — every day that follows builds on these principles.
- You now understand why Neo4j's index-free adjacency makes it fundamentally faster than relational databases for connected data queries, how to model a graph schema with correct use of labels, relationship types, and properties, and how to design a RAG-specific graph schema that supports both semantic retrieval and structural context expansion.
- Variable-length paths give you depth-flexible traversal; UNWIND gives you list-to-row expansion; WITH chaining gives you multi-stage query pipelines; APOC gives you the procedural extensions that Cypher's declarative syntax alone cannot express.
- EXPLAIN and PROFILE are your performance debugging tools — use them on every non-trivial query before promoting it to production.
- **Looking ahead to Day 2:** the graph schema you designed today is the substrate on which Day 2's Graph Data Science algorithms will run. Tomorrow you project that graph into the GDS in-memory layer and run PageRank, Betweenness Centrality, and community detection — producing scores that will be written back as node properties and used in the RAG retrieval ranking logic on Day 4.

***
