# Project: Smart Customer Support Knowledge Graph & AI Assistant

## One-line idea

We will build a system that takes customer, product, ticket, document, and transaction data from different sources, converts it into a **Neo4j knowledge graph**, analyses relationships using **graph algorithms**, powers a **Graph RAG AI assistant**, automates ingestion using **Spark and Airflow**, and finally displays insights in **NeoDash, Bloom, and Power BI**.

This fits the TOC very well because the course already covers Neo4j graph modelling, Graph Data Science, vector search, Graph RAG, Spark ingestion, Airflow pipelines, NeoDash, Bloom, Power BI, FastAPI, and end-to-end integration. 

---

# Final project on Day 15: what it will look like

By Day 15, the learner will have built a mini production-style platform called:

## “SupportGraph Intelligence Platform”

It will look like this:

Oracle / PostgreSQL / CSV sample data

        ↓

Apache Spark cleans and transforms data

        ↓

Apache Airflow schedules and monitors the pipeline

        ↓

Neo4j stores the customer-support-product-document graph

        ↓

Neo4j GDS calculates influence, importance, and communities

        ↓

Neo4j Vector Index stores document embeddings

        ↓

FastAPI exposes graph search and Graph RAG APIs

        ↓

LLM answers user questions using graph context

        ↓

NeoDash shows operational dashboards

        ↓

Bloom enables visual graph investigation

        ↓

Power BI shows executive reporting

The TOC’s final day expects a full walkthrough across Oracle/PostgreSQL ingestion, Spark processing, Airflow orchestration, Neo4j graph analytics, Graph RAG API, NeoDash dashboard, Bloom investigation, Power BI reporting, tests, data quality checks, and system architecture documentation. This project directly maps to that final objective. 

---

# Business story for the project

Imagine a company has:

- Customers
- Products
- Support tickets
- Agents
- Knowledge base articles
- Error codes
- Product categories
- Regions
- Transactions
- Escalations

The company wants to answer questions like:

- Which customers are most affected by recurring product issues?
- Which products create the most support burden?
- Which agents handle the most complex cases?
- Which knowledge articles are most useful?
- Are some issues connected even if they appear in different tickets?
- Can an AI assistant answer support questions using our company’s own support knowledge?
- Can managers view dashboards in Power BI?
- Can analysts visually explore customer-product-ticket relationships?

This is a very beginner-friendly project because most people understand customers, products, tickets, and support documents even if they do not know graph databases yet.

---

# The graph model we build gradually

By the end, the Neo4j graph could contain nodes like:

(:Customer)

(:Product)

(:Ticket)

(:Agent)

(:KnowledgeArticle)

(:Issue)

(:ErrorCode)

(:Region)

(:Transaction)

(:DocumentChunk)

And relationships like:

(:Customer)-[:RAISED]->(:Ticket)

(:Ticket)-[:ABOUT]->(:Product)

(:Ticket)-[:ASSIGNED_TO]->(:Agent)

(:Ticket)-[:MENTIONS]->(:Issue)

(:Ticket)-[:HAS_ERROR]->(:ErrorCode)

(:KnowledgeArticle)-[:SOLVES]->(:Issue)

(:Customer)-[:LOCATED_IN]->(:Region)

(:Customer)-[:PURCHASED]->(:Product)

(:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)

(:DocumentChunk)-[:SIMILAR_TO]->(:DocumentChunk)

This matches the TOC’s focus on graph modelling, node labels, relationship types, properties, schema design, advanced Cypher, graph algorithms, vector indexes, Graph RAG, and dashboards.

---

# Day-wise project development plan

## Day 1 – Build the first graph model

### What the learner learns

The learner first understands what a graph is: things connected to other things. Instead of explaining Neo4j with abstract theory, we start with a simple support example.

### Project contribution

On Day 1, we create the first version of the graph:

Customer → Ticket → Product

Ticket → Agent

Ticket → Issue

The learner creates a small dataset manually or from CSV:

Customer: Asha

Ticket: T001

Product: Mobile App

Issue: Login Failure

Agent: Rajat

Then they write simple Cypher queries such as:

MATCH (c:Customer)-[:RAISED]->(t:Ticket)-[:ABOUT]->(p:Product)

RETURN c, t, p;

The TOC says Day 1 covers graph modelling principles, node labels, relationship types, properties, schema design patterns, variable-length paths, UNWIND, WITH chaining, APOC procedures, and query profiling with EXPLAIN and PROFILE. 

### Output of Day 1

A small working Neo4j graph with customers, tickets, products, agents, and issues.

### Official documentation to consult during implementation

- Neo4j Cypher Manual
- Neo4j Graph Data Modelling documentation
- Neo4j APOC documentation
- Neo4j query tuning documentation for EXPLAIN and PROFILE

---

## Day 2 – Add graph analytics

### What the learner learns

Now the learner sees that a graph is not only for storing data. It can also calculate importance, influence, and groups.

### Project contribution

We run graph algorithms to answer:

- Which products are most central to support issues?
- Which issues connect many tickets?
- Which customers are heavily affected?
- Are there issue communities?

We add calculated properties like:

Product.pageRankScore

Issue.communityId

Customer.degreeScore

The TOC says Day 2 covers GDS setup, graph projection, PageRank, Betweenness Centrality, Degree Centrality, Louvain, Label Propagation, and writing algorithm results back as node properties. 

### Output of Day 2

The support graph now has intelligence scores. For example, the system can say:

> “Login Failure is the most central issue because it connects many customers, tickets, and products.”

### Official documentation to consult during implementation

- Neo4j Graph Data Science Manual
- Neo4j GDS graph projection documentation
- Neo4j PageRank documentation
- Neo4j Louvain and Label Propagation documentation

---

## Day 3 – Add knowledge articles and vector search

### What the learner learns

The learner now understands the difference between exact search and semantic search.

Example:

- Exact search: “login error”
- Semantic search: “I cannot access my account”

Both should find similar knowledge articles.

### Project contribution

We add knowledge base articles and split them into document chunks:

KnowledgeArticle → DocumentChunk

DocumentChunk has text and embedding

Example article:

Article: How to fix login failure

Chunk: If a customer cannot sign in, ask them to reset password...

We generate embeddings and store them in Neo4j vector indexes.

The TOC says Day 3 covers Neo4j Vector Index setup, embedding storage, similarity search, and hybrid queries combining vector search with Cypher graph traversal.

### Output of Day 3

The system can find useful support articles even when the user asks questions in natural language.

### Official documentation to consult during implementation

- Neo4j Vector Index documentation
- Neo4j semantic search/vector search documentation
- Neo4j Cypher documentation for combining vector results with graph traversal

---

## Day 4 – Build the Graph RAG retrieval layer

### What the learner learns

The learner now understands the first half of Graph RAG: retrieving the right information before asking an AI model to answer.

### Project contribution

We build a retrieval pipeline:

User asks:

Why are many customers facing login problems?

System does:

1. Converts question into embedding

2. Searches similar document chunks

3. Traverses graph to related tickets, products, and issues

4. Uses PageRank or centrality scores to rank important context

5. Formats context for the LLM

The TOC says Day 4 covers RAG architecture, retriever, context assembler, generator, subgraph extraction, relevance ranking using GDS scores, and formatting graph output as LLM-ready context. 

### Output of Day 4

A working retrieval function that returns graph-aware context, but not yet a full AI answer.

### Official documentation to consult during implementation

- Neo4j GraphRAG-related documentation and examples
- Neo4j Vector Index documentation
- Neo4j GDS documentation for using graph scores in ranking
- LangChain or LlamaIndex retrieval documentation, depending on selected framework

---

## Day 5 – Add LLM and expose through FastAPI

### Day 5 – Add dynamic retrieval, LLM reasoning, and expose through FastAPI

### What the learner learns

The learner now sees how the graph becomes an AI-powered assistant that can respond to **different types of user questions**, not only a few hard-coded retrieval patterns. Instead of assuming in advance what graph context is needed, the system learns to decide:

- what the question is about,
- which part of the graph is relevant,
- whether the answer needs knowledge-only context,
- whether it needs operational ticket context,
- whether it needs customer, product, issue, or agent context,
- and how to handle questions where some context is missing.

This makes Day 5 more realistic and production-like, because real users do not ask only pre-defined demo questions. They ask unpredictable questions, and the system must dynamically choose the correct graph retrieval path before generating an answer. This expands the original Day 5 objective of connecting the retrieval layer to an LLM and exposing it through FastAPI.

### Project contribution

We extend the Day 4 retrieval layer into a **dynamic GraphRAG answer-generation service**.

The system should now support a workflow like this:

User asks a question  
↓  
System interprets the question type  
↓  
System selects the correct graph retrieval strategy  
↓  
System retrieves the relevant support subgraph and document context  
↓  
System formats the retrieved context for the LLM  
↓  
Local or external LLM generates an answer grounded in graph context  
↓  
FastAPI exposes the response as an API

The retrieval planner should dynamically decide which graph projection or retrieval path to use based on the user’s question. Example question categories could include:

- **Issue troubleshooting questions**  
    Example: _Why are customers facing login failure?_  
    Retrieval focus: `Issue`, `Ticket`, `KnowledgeArticle`, `DocumentChunk`, `Product`, `Agent`
    
- **Customer summary questions**  
    Example: _Summarise customer Asha Sharma’s support situation._  
    Retrieval focus: `Customer`, related `Ticket`, `Product`, `Issue`, assigned `Agent`, related knowledge coverage
    
- **Product issue questions**  
    Example: _What are the main issues affecting the Mobile App?_  
    Retrieval focus: `Product`, related `Ticket`, `Issue`, `Customer`, `KnowledgeArticle`
    
- **Knowledge-first support recommendation questions**  
    Example: _What should we recommend for app crash issues?_  
    Retrieval focus: `KnowledgeArticle`, `DocumentChunk`, `Issue`, and operational context if available
    
- **Agent or workload questions**  
    Example: _Which issues is Rajat Support handling most often?_  
    Retrieval focus: `Agent`, assigned `Ticket`, `Issue`, `Product`, score/context properties
    

The dynamic retrieval layer should also handle missing operational context gracefully. For example, if a question maps to an issue that has knowledge coverage but no linked tickets, the system should still answer using knowledge context and explicitly mention the operational-data limitation. This reflects the Day 3 / Day 4 App Crash scenario, where knowledge exists but ticket coverage is missing. 

The LLM answer stage should use retrieved evidence only. The answer should be grounded in:

- retrieved document chunks,
- related knowledge article details,
- related issue details,
- optional operational context:
    - tickets
    - customers
    - products
    - assigned agents
- and warnings when operational data is absent.

The FastAPI layer should expose this behaviour through a production-style API, beginning with endpoints such as:

- `GET /health`
- `GET /neo4j/health`
- `GET /llm/health`
- `POST /ask`

Later endpoints can expand to:

- `GET /customer/{customer_id}/summary`
- `GET /product/{product_id}/issues`
- `GET /issue/{issue_id}/knowledge`

This keeps Day 5 directly connected to the broader platform vision, where FastAPI acts as the service layer between Neo4j, GraphRAG, the LLM, and downstream dashboards/reports.

### Output of Day 5

By the end of Day 5, the learner should have a working **Ask SupportGraph API prototype** that can:

- accept a natural-language question,
- determine the most relevant retrieval path dynamically,
- query Neo4j for graph and knowledge context,
- assemble an LLM-ready context payload,
- call the LLM,
- return a grounded answer,
- and emit warnings when operational graph coverage is incomplete.

This means Day 5 no longer ends with only a static retrieval-to-LLM demo. It ends with the first version of a **dynamic production-style GraphRAG API**. That is the correct preparation for the later platform stages in Spark, Airflow, NeoDash, Bloom, and Power BI.

### Suggested production-style success criteria for Day 5

Day 5 should be considered successful when all of the following work:

1. **LLM connectivity works**
    
    - local or configured LLM responds from Python
2. **Neo4j connectivity works**
    
    - Python can execute Cypher through the Neo4j driver
3. **Dynamic retrieval planning works**
    
    - the system can choose different retrieval paths based on the question
4. **GraphRAG context assembly works**
    
    - the system can retrieve knowledge and operational context together
5. **Warning behaviour works**
    
    - if no operational context exists, the API still returns a useful knowledge-grounded answer with a clear warning
6. **FastAPI endpoint works**
    
    - `POST /ask` returns a structured response with:
        - user question
        - retrieval plan or retrieval type
        - answer
        - supporting context summary
        - warnings (if any)

These criteria make Day 5 feel like a real-world backend milestone rather than only a classroom demo.

### Official documentation to consult during implementation

- FastAPI official documentation
- Neo4j Python Driver documentation
- Neo4j Vector Index documentation
- Neo4j Cypher Manual for:
    - `MATCH`
    - `OPTIONAL MATCH`
    - `RETURN`
    - `WITH`
    - `CASE`
    - aggregating functions such as `collect()`
- LangChain or LlamaIndex documentation, depending on selected framework, especially for:
    - routing
    - retrieval chains
    - prompt orchestration
    - structured output handling
- LLM provider documentation (local or remote), depending on the selected serving path

This expands the original Day 5 documentation references so they better support dynamic retrieval planning and production-style API behaviour.

---

## Day 6 – Introduce Spark for scalable data processing

### What the learner learns

The learner now understands that manually loading small data is fine for practice, but real companies have larger files and tables.

### Project contribution

We use Apache Spark to read raw support data from CSV or Parquet:

customers.csv

tickets.csv

products.csv

agents.csv

knowledge_articles.csv

Spark cleans and transforms it into graph-ready structures:

Customer nodes

Ticket nodes

Product nodes

RAISED relationships

ABOUT relationships

ASSIGNED_TO relationships

Then Spark loads the data into Neo4j using the Neo4j Spark Connector.

The TOC says Day 6 covers Spark architecture, Driver, Executors, DataFrames, SparkSession, transformations, joins, aggregations, Parquet/CSV, and Neo4j Spark Connector for reading and writing to Neo4j.

### Output of Day 6

The graph can now be built from processed datasets instead of manual Cypher inserts.

### Official documentation to consult during implementation

- Apache Spark SQL/DataFrame documentation
- Apache Spark programming guide
- Neo4j Connector for Apache Spark documentation

---

## Day 7 – Add Oracle and PostgreSQL style sources

### What the learner learns

The learner now understands that data in real organisations usually lives in databases, not just CSV files.

### Project contribution

We simulate or connect two relational sources:

Oracle: customer and transaction data

PostgreSQL: support tickets and knowledge article metadata

Spark reads from both using JDBC, joins the data, and produces graph-ready output.

Example transformation:

Oracle customers table + PostgreSQL tickets table

        ↓

Customer nodes + Ticket nodes + RAISED relationships

The TOC says Day 7 covers reading from Oracle and PostgreSQL using JDBC in Spark, partitioned reads, write strategies, incremental loading patterns, and transforming relational data into graph-ready node and relationship structures.

### Output of Day 7

The project now has a realistic multi-source ingestion layer.

### Official documentation to consult during implementation

- Apache Spark JDBC data source documentation
- Oracle JDBC documentation
- PostgreSQL JDBC documentation
- Neo4j Spark Connector documentation

---

## Day 8 – Add Airflow orchestration

### What the learner learns

The learner now understands that pipelines should not be run manually every time.

### Project contribution

We introduce Airflow to run the project workflow:

Task 1: Check source availability

Task 2: Extract Oracle data

Task 3: Extract PostgreSQL data

Task 4: Run Spark transformation

Task 5: Load into Neo4j

Task 6: Validate graph counts

The TOC says Day 8 covers Airflow Scheduler, Workers, Metadata DB, Executors, CeleryExecutor, LocalExecutor, KubernetesExecutor, Docker Compose setup, Redis broker, and configuring a CeleryExecutor cluster.

### Output of Day 8

A basic Airflow pipeline exists, even if it is still simple.

### Official documentation to consult during implementation

- Apache Airflow core concepts documentation
- Apache Airflow executor documentation
- Apache Airflow Docker Compose documentation
- Apache Airflow CeleryExecutor documentation

---

## Day 9 – Make the pipeline production-like

### What the learner learns

The learner now understands that real pipelines fail, retry, alert, and need monitoring.

### Project contribution

We improve the Airflow DAG:

- Add TaskGroups

- Add retry logic

- Add failure alerts

- Add sensors

- Add timeout handling

- Add SLA-style checks

- Add XCom for passing metadata

Example:

If PostgreSQL is unavailable, retry the task.

If Spark job fails, send an alert.

If Neo4j node count is lower than expected, mark validation as failed.

The TOC says Day 9 covers TaskGroups, dynamic DAG generation, XCom, custom operators, sensors, retry logic, timeout policies, SLA configuration, and email or Slack alerts.

### Output of Day 9

The pipeline becomes safer, more reliable, and easier to troubleshoot.

### Official documentation to consult during implementation

- Apache Airflow DAG documentation
- Apache Airflow TaskGroup documentation
- Apache Airflow XCom documentation
- Apache Airflow sensors documentation
- Apache Airflow notifications/callbacks documentation

---

## Day 10 – Complete automated data pipeline

### What the learner learns

The learner now sees the full backend pipeline working end to end.

### Project contribution

We combine Spark and Airflow properly:

Airflow triggers Spark job

Spark extracts from Oracle/PostgreSQL

Spark transforms data

Spark loads nodes and relationships into Neo4j

Airflow validates and logs the pipeline

Monitoring dashboard tracks success/failure

The TOC says Day 10 covers SparkSubmitOperator, extracting from Oracle with cx_Oracle and PostgreSQL with psycopg2, transforming with Spark, loading into Neo4j, monitoring, logging with StatsD and Prometheus, CI/CD deployment strategies, and Grafana dashboard monitoring. 

### Output of Day 10

A working automated data engineering pipeline:

Source DBs → Spark → Neo4j → Logs/Monitoring

### Official documentation to consult during implementation

- Apache Airflow Spark provider documentation
- SparkSubmitOperator documentation
- Apache Spark JDBC documentation
- Neo4j Spark Connector documentation
- Prometheus, StatsD, and Grafana documentation

---

## Day 11 – Build NeoDash dashboard

### What the learner learns

The learner now understands how technical graph data becomes a useful dashboard.

### Project contribution

We create a NeoDash dashboard showing:

- Total customers

- Total tickets

- Top products by ticket count

- Top issues by PageRank

- Tickets by region

- Graph visualisation of Customer → Ticket → Product

The TOC says Day 11 covers NeoDash widget types, Cypher-driven charts, parameterised filters, report layout, global dashboard parameters, linked widgets, KPI cards, bar charts, and graph visualisation widgets.

### Output of Day 11

A working operational dashboard for support managers.

### Official documentation to consult during implementation

- NeoDash documentation
- Neo4j Cypher documentation
- Neo4j Browser/visualisation-related documentation

---

## Day 12 – Add advanced dashboard and API integration

### What the learner learns

The learner now understands that dashboards can interact with APIs and not just query the database directly.

### Project contribution

We enhance the dashboard:

- Click a product to see related tickets

- Click an issue to see related customers

- Add map/region view if region data exists

- Connect to FastAPI endpoint

- Show AI answer from Graph RAG API

Example dashboard widget:

Ask AI: Why is product X getting many complaints?

The widget calls the FastAPI endpoint built earlier.

The TOC says Day 12 covers drill-down interactivity, map widgets, linked parameters between reports, connecting NeoDash to the Graph RAG API, and serving Neo4j results through FastAPI middleware. 

### Output of Day 12

The dashboard becomes interactive and AI-enabled.

### Official documentation to consult during implementation

- NeoDash documentation
- FastAPI documentation
- Neo4j Python Driver documentation
- Neo4j Cypher documentation

---

## Day 13 – Add Neo4j Bloom investigation

### What the learner learns

The learner now understands how non-technical users can explore graphs visually without writing Cypher.

### Project contribution

We create Bloom perspectives such as:

Support Investigator View

Customer Risk View

Product Issue View

Knowledge Article View

Example Bloom search phrases:

Show tickets for customer Asha

Show issues related to Mobile App

Show customers affected by Login Failure

Find path between Customer A and Error Code E101

The TOC says Day 13 covers Bloom perspectives, search phrases, scene setup, styling rules, rule-based highlighting, path expansion, and sharing investigations.

### Output of Day 13

A visual investigation interface for analysts and support leads.

### Official documentation to consult during implementation

- Neo4j Bloom documentation
- Neo4j Bloom perspectives documentation
- Neo4j Bloom search phrases documentation
- Neo4j Bloom styling documentation

---

## Day 14 – Add Power BI reporting

### What the learner learns

The learner now understands how graph insights can be consumed by business executives.

### Project contribution

We build Power BI reports using FastAPI as a middleware layer.

Power BI report pages:

Page 1: Support Overview

Page 2: Product Issue Analysis

Page 3: Customer Risk Analysis

Page 4: Agent Performance

Page 5: Knowledge Article Effectiveness

FastAPI endpoints might expose:

GET /metrics/ticket-summary

GET /metrics/top-products

GET /metrics/customer-risk

GET /metrics/agent-performance

GET /metrics/issue-communities

The TOC says Day 14 covers connecting Power BI to Neo4j via FastAPI middleware and Python scripting, dynamic filtering with Cypher parameter binding, graph-powered Power BI reports, and refreshing reports from live Neo4j graph data.

### Output of Day 14

An executive reporting layer connected to the graph system.

### Official documentation to consult during implementation

- Microsoft Power BI documentation
- Power BI Python scripting documentation
- Power BI web/API connector-related documentation
- FastAPI documentation
- Neo4j Python Driver documentation

---

## Day 15 – Final integration, testing, and architecture review

### What the learner learns

The learner now sees how all pieces fit into one production-style system.

### Project contribution

We run the full project:

1. Source data is available in Oracle/PostgreSQL/CSV.

2. Airflow starts the DAG.

3. Spark extracts and transforms data.

4. Neo4j receives updated nodes and relationships.

5. GDS algorithms update scores and communities.

6. Vector index supports semantic retrieval.

7. FastAPI exposes Graph RAG and reporting APIs.

8. NeoDash displays operational dashboards.

9. Bloom supports graph investigation.

10. Power BI displays executive reports.

11. Tests validate data quality and pipeline health.

12. Architecture is documented.

The TOC says Day 15 covers full pipeline walkthrough, Oracle/PostgreSQL ingestion, Spark processing, Airflow orchestration, Neo4j graph analytics, Graph RAG API, NeoDash dashboard, Bloom investigation, Power BI reporting, testing strategies, unit tests for DAGs, integration tests, data quality checks, and system architecture documentation.

### Output of Day 15

A complete, demo-ready platform:

SupportGraph Intelligence Platform

It includes:

- Automated ingestion pipeline
- Graph database
- Graph analytics
- Semantic search
- AI assistant
- FastAPI middleware
- NeoDash dashboard
- Bloom investigation view
- Power BI report
- Monitoring and validation
- Architecture documentation

### Official documentation to consult during implementation

- Apache Airflow testing documentation
- Neo4j operations and Cypher documentation
- Neo4j GDS documentation
- FastAPI testing documentation
- Power BI documentation
- Spark documentation
- Grafana/Prometheus documentation

---

# Journey

## Day 1 to Day 5: Build the brain

We first build the graph brain.

Neo4j stores connected data.

GDS finds important things.

Vector search finds semantically similar text.

Graph RAG uses the graph to help AI answer questions.

FastAPI exposes the AI assistant.

This corresponds to Week 1 of the TOC, which covers Neo4j advanced analytics, graph algorithms, vector indexes, hybrid graph search, Graph RAG retrieval, LLM integration, and FastAPI deployment.

---

## Day 6 to Day 10: Build the data factory

Then we build the data pipeline.

Spark processes data.

Oracle/PostgreSQL provide source data.

Airflow schedules and monitors the workflow.

Neo4j receives clean graph-ready data.

This corresponds to Week 2 of the TOC, which covers Apache Spark fundamentals, Neo4j integration, JDBC extraction from Oracle/PostgreSQL, Airflow architecture, production DAG design, error handling, alerting, and end-to-end Oracle/Postgres-to-Neo4j pipelines.

---

## Day 11 to Day 15: Build the user experience

Finally, we build dashboards and business-facing interfaces.

NeoDash for graph dashboards.

Bloom for visual investigation.

Power BI for executive reporting.

Final day for integration and testing.

This corresponds to Week 3 of the TOC, which covers NeoDash, advanced NeoDash and FastAPI integration, Neo4j Bloom, Power BI integration, and complete end-to-end system review.

---

# What the final demo would look like

On Day 15, learner should be able to:

## Demo Part 1: Run the pipeline

Open Airflow and trigger the DAG:

supportgraph_daily_pipeline

Show tasks:

extract_oracle_customers

extract_postgres_tickets

spark_transform_to_graph

load_to_neo4j

run_gds_scores

update_vector_index

validate_counts

This aligns with the TOC’s requirement for Airflow orchestration, Spark processing, Oracle/PostgreSQL ingestion, Neo4j loading, monitoring, and end-to-end testing. 

---

## Demo Part 2: Show the graph

Open Neo4j and show:

Customer → Ticket → Product → Issue → KnowledgeArticle

Show query:

MATCH path = (c:Customer)-[:RAISED]->(t:Ticket)-[:ABOUT]->(p:Product)

RETURN path

LIMIT 25;

This demonstrates the Neo4j graph modelling, Cypher querying, and graph traversal parts of the TOC. 

---

## Demo Part 3: Show graph analytics

Show top issues by PageRank:

Login Failure: 0.89

Payment Failure: 0.73

App Crash: 0.68

Show communities:

Community 1: Login/OTP/Password issues

Community 2: Payment/Card/Refund issues

Community 3: App Crash/Device compatibility issues

This maps to the TOC’s graph data science coverage, including centrality algorithms and community detection.

---

## Demo Part 4: Ask the AI assistant

Ask:

Why are customers facing login issues, and what should agents recommend?

The AI assistant responds using graph context, ticket history, important issues, and relevant knowledge articles.

This maps to the TOC’s Graph RAG retrieval pipeline, vector search, hybrid graph search, LLM integration, prompt engineering, FastAPI endpoint deployment, and evaluation. 

---

## Demo Part 5: Show NeoDash

Show:

Total tickets

Top products by issue volume

Top issues by centrality score

Customers at risk

Graph visualisation

This maps to the TOC’s NeoDash dashboarding, KPI cards, charts, graph widgets, parameterised filters, and dynamic Cypher queries.

---

## Demo Part 6: Show Bloom

Search in Bloom:

Show customers affected by Login Failure

Then expand paths:

Customer → Ticket → Issue → KnowledgeArticle → Product

This maps to the TOC’s Bloom perspectives, search phrases, scene setup, styling rules, path expansion, and visual investigation. 

---

## Demo Part 7: Show Power BI

Show executive-level pages:

Support Overview

Product Risk

Customer Impact

Agent Performance

Knowledge Base Usage

This maps to the TOC’s Power BI integration through FastAPI middleware, dynamic Cypher-bound filters, graph-powered reports, and live graph data refresh.

---

# Why this project works well for a complete beginner

This project is good for a learner because it tells one continuous story:

Day 1: I created connected data.

Day 2: I found important things in that data.

Day 3: I searched documents by meaning.

Day 4: I prepared useful context for AI.

Day 5: I built an AI API.

Day 6: I processed bigger data.

Day 7: I read data from databases.

Day 8: I automated the workflow.

Day 9: I handled failures.

Day 10: I built the full backend pipeline.

Day 11: I made a dashboard.

Day 12: I connected dashboard and API.

Day 13: I explored the graph visually.

Day 14: I built business reports.

Day 15: I integrated and presented the whole system.

That is much easier than teaching each tool separately.

---

# My recommended project name and modules

## Project name

SupportGraph Intelligence Platform

## Modules

01_data_sources

02_spark_processing

03_neo4j_graph_model

04_graph_algorithms

05_vector_search

06_graph_rag_api

07_airflow_pipeline

08_neodash_dashboard

09_bloom_investigation

10_powerbi_reporting

11_tests_and_docs

---

# Beginner-friendly dataset design

To keep it manageable, I would use a small starter dataset first:

## Customers

customer_id, name, region, segment

C001, Asha Sharma, North, Premium

C002, Ravi Mehta, West, Standard

C003, Neha Singh, South, Premium

## Products

product_id, name, category

P001, Mobile App, Digital

P002, Credit Card, Banking

P003, Web Portal, Digital

## Tickets

ticket_id, customer_id, product_id, agent_id, issue_type, priority

T001, C001, P001, A001, Login Failure, High

T002, C002, P002, A002, Payment Failure, Medium

T003, C003, P001, A001, App Crash, High

## Knowledge articles

article_id, title, issue_type, content

K001, Fix login failure, Login Failure, Steps to reset password and verify OTP...

K002, Resolve payment failure, Payment Failure, Check card status and retry transaction...

K003, Fix app crash, App Crash, Clear cache and update app version...

Later, the same dataset can be placed in CSV, PostgreSQL, and Oracle-style tables.

---

# Final architecture diagram

                 ┌────────────────────┐

                 │ Oracle Customers   │

                 │ Oracle Transactions│

                 └─────────┬──────────┘

                           │

                 ┌─────────▼──────────┐

                 │ PostgreSQL Tickets │

                 │ KB Metadata        │

                 └─────────┬──────────┘

                           │

                 ┌─────────▼──────────┐

                 │   Apache Spark     │

                 │ Clean + Transform  │

                 └─────────┬──────────┘

                           │

                 ┌─────────▼──────────┐

                 │ Apache Airflow     │

                 │ Schedule + Monitor │

                 └─────────┬──────────┘

                           │

                 ┌─────────▼──────────┐

                 │       Neo4j        │

                 │ Graph + Vectors    │

                 └──────┬─────┬───────┘

                        │     │

         ┌──────────────▼┐   ┌▼────────────────┐

         │ Neo4j GDS     │   │ Vector Search   │

         │ Scores/Groups │   │ Semantic Search │

         └──────┬────────┘   └──────┬──────────┘

                │                   │

                └─────────┬─────────┘

                          │

                 ┌────────▼─────────┐

                 │ FastAPI Graph RAG│

                 │ AI Assistant API │

                 └───┬──────┬──────┘

                     │      │

        ┌────────────▼┐   ┌─▼────────────┐

        │  NeoDash    │   │  Power BI    │

        │ Dashboards  │   │ Reports      │

        └─────────────┘   └──────────────┘

  

                 ┌────────────────────┐

                 │ Neo4j Bloom        │

                 │ Visual Investigation│

                 └────────────────────┘

---
## A customer support intelligence system powered by graph analytics and AI.

By Day 15, the learner would not just know the definitions of Neo4j, Spark, Airflow, FastAPI, NeoDash, Bloom, and Power BI. They would have built a complete working system where:

- Data comes from multiple sources.
- Spark prepares it.
- Airflow automates it.
- Neo4j stores it as a graph.
- GDS finds patterns.
- Vector search supports semantic retrieval.
- Graph RAG answers questions.
- FastAPI exposes services.
- NeoDash and Bloom help explore the graph.
- Power BI presents business reports.
- Tests and documentation make it production-like.

That would make the course much more understandable, because every day answers one simple question:

> “What new part of our real project are we building today?”