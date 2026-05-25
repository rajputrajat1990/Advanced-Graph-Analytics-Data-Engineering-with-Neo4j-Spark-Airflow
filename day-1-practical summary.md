# Day 1 Summary — Support Graph Project

## Overall result

Day 1 is complete. 🎉

We built and validated the beginner support graph foundation in Neo4j, enabled APOC in your Docker-based Neo4j environment, and finished the first query-planning checks with `EXPLAIN` and `PROFILE`.

---

## 1. Graph model and starter data completed

We created and worked with a basic support-domain graph involving entities such as:

Customer

Ticket

Product

Issue

Agent

We also worked with relationships such as:

Customer -[:RAISED]-> Ticket

Ticket -[:ABOUT]-> Product

Ticket -[:HAS_ISSUE]-> Issue

Agent -[:ASSIGNED_TO]-> Ticket

By the end of Day 1, your graph contained the expected starter support cases, including issues such as:

I001 → Login Failure

I002 → Payment Failure

I003 → App Crash

The `App Crash` issue was added safely using `UNWIND + MERGE`, so we avoided duplicate issue creation.

---

## 2. Basic Cypher querying completed

We practised basic `MATCH` queries against the support graph and verified that the graph structure was working.

Examples included:

MATCH (c:Customer)-[:RAISED]->(t:Ticket)

RETURN c.name, t.ticketId;

and later:

MATCH (c:Customer)-[:RAISED]->(t:Ticket)-[:ABOUT]->(p:Product)

RETURN c.name, t.ticketId, p.name;

This confirmed that customer-ticket-product paths were working correctly.

---

## 3. Health checks completed

We ran checks to confirm that important graph data existed and was connected properly.

We verified things like:

Customers exist ✅

Tickets exist ✅

Products exist ✅

Issues exist ✅

Relationships exist ✅

This gave us confidence that the graph was not only created, but usable.

---

## 4. ID uniqueness constraints completed

We added and verified uniqueness constraints for key identifiers such as:

Customer.customerId

Ticket.ticketId

Product.productId

Issue.issueId

Agent.agentId

This helps protect the graph from duplicate business IDs and supports better modelling practice.

---

## 5. Variable-length path practice completed

We practised variable-length paths to understand how Neo4j can traverse relationship chains beyond a single hop.

This helped establish the idea that graph queries can explore connected structures flexibly, not just fixed one-step relationships.

---

## 6. `UNWIND` completed

We completed several `UNWIND` exercises:

UNWIND simple list ✅

UNWIND list of maps ✅

UNWIND + MERGE ✅

Verify Issue nodes ✅

The main practical outcome was adding or matching:

I003 → App Crash

using a list-of-maps pattern and `MERGE`.

That was an important real-world data-loading pattern.

---

## 7. `WITH` chaining completed

We consulted the official Neo4j `WITH` documentation and then practised `WITH` in project-relevant ways. The documentation explains that `WITH` can create variables, control variable scope, bind expression results, perform aggregations, remove duplicates, order/paginate, and filter results. 

Completed `WITH` examples:

Basic WITH chaining from Customer → Ticket → Product ✅

WITH aggregation: ticket count per Product ✅

WITH + collect(): ticket IDs per Product ✅

WITH ... WHERE: high-priority ticket filtering ✅

Key learning:

WITH lets us pass selected variables from one query stage to the next.

We also learned that variables not passed through `WITH` drop out of scope, which is an important Cypher concept. 

---

## 8. APOC procedures enabled and tested

This was one of the biggest Day 1 wins.

Initially, this query returned no APOC procedures:

SHOW PROCEDURES

YIELD name

WHERE name STARTS WITH "apoc."

RETURN name;

So we investigated your Docker environment.

We confirmed:

Container name: supportgraph-neo4j

Image:          neo4j:2026.04.0

Then we found the APOC jar here:

/var/lib/neo4j/labs/apoc-2026.04.0-core.jar

and copied it into:

/var/lib/neo4j/plugins/apoc.jar

The official APOC installation documentation says APOC is packaged with Neo4j in `$NEO4J_HOME/labs`, can be installed by moving the APOC jar to `$NEO4J_HOME/plugins`, and requires restarting Neo4j afterwards.

We also fixed file ownership to:

neo4j:neo4j

Then restarted the container and verified APOC successfully.

The official APOC docs also state that APOC should match the Neo4j version by year and month because it relies on Neo4j internal APIs; your Neo4j image and APOC jar both matched `2026.04.0`. 

---

## 9. APOC first usage completed

After APOC was enabled, we successfully ran:

RETURN apoc.version() AS apocVersion;

Then we tested APOC built-in help.

The official APOC overview lists **Built-in Help** and **Procedures & Functions** as main APOC documentation areas, and includes `apoc.help` and `apoc.version` in the APOC procedure/function navigation.

We discovered that in your APOC version, `apoc.help()` returns these columns:

type

name

text

signature

roles

writes

core

isDeprecated

So we corrected the query from using the wrong column:

description ❌

to the correct column:

text ✅

Final working APOC help query:

CALL apoc.help("text")

YIELD name, type, text, signature

RETURN

  name,

  type,

  text,

  signature

ORDER BY name

LIMIT 10;

This confirmed APOC is installed, callable, and useful.

---

## 10. `EXPLAIN` completed

We consulted the official Neo4j query-plan documentation. It says query plans can be shown by prepending a query with `EXPLAIN` or `PROFILE`; `EXPLAIN` plans the query but does not run it and only returns estimated rows. 

You ran:

EXPLAIN

MATCH (c:Customer)-[:RAISED]->(t:Ticket)

RETURN

  c.name AS customer,

  t.ticketId AS ticketId;

Neo4j returned a plan with operators:

ProduceResults

Projection

Filter

DirectedRelationshipTypeScan

We learned to read the plan from bottom to top, because the official documentation says query plans are read from the leaf operator at the bottom upward to the root operator, usually `ProduceResults`.

Your `EXPLAIN` estimated:

Estimated Rows: 3

---

## 11. `PROFILE` completed

Then we safely ran the same query with `PROFILE`:

PROFILE

MATCH (c:Customer)-[:RAISED]->(t:Ticket)

RETURN

  c.name AS customer,

  t.ticketId AS ticketId;

The official documentation says `PROFILE` runs the query and shows real runtime measurements such as rows and DB hits. It also warns that `PROFILE` uses more resources and can write data if used with write queries, so we used it only on a read-only query.

Your `PROFILE` result showed:

Estimated Rows: 3

Actual Rows:    2

DB Hits:        11 total

Memory:         64 total allocated memory

Key learning:

EXPLAIN = estimated plan, query not run

PROFILE = real execution, actual rows and DB hits

The official documentation explains that `Estimated Rows` is the planner’s estimate, while `Rows` in `PROFILE` shows actual rows produced during execution. It also explains that `DB Hits` represent low-level database access operations and are not the same as rows. 

---

# Final Day 1 checklist

Graph model                 ✅

Sample data                 ✅

Basic MATCH queries          ✅

Health checks                ✅

ID uniqueness constraints    ✅

Variable-length paths        ✅

UNWIND                       ✅

WITH chaining                ✅

APOC procedures              ✅

EXPLAIN / PROFILE            ✅

# Biggest wins from Day 1

1. You built a working Neo4j support graph.

2. You practised practical Cypher patterns instead of isolated syntax.

3. You used MERGE safely to avoid duplicate data.

4. You learnt WITH chaining and aggregation.

5. You enabled APOC manually inside Docker.

6. You debugged APOC procedure output columns correctly.

7. You learnt the difference between EXPLAIN and PROFILE.

8. You completed the full Day 1 learning path.