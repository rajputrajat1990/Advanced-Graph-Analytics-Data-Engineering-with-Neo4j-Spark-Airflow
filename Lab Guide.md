
# Objective

To build the **SupportGraph Intelligence Platform** project for the 15-day programme. The stack includes **Neo4j, Apache Spark, Apache Airflow, NeoDash, Neo4j Bloom, Power BI, Oracle/PostgreSQL-style ingestion, FastAPI, and Graph RAG/LLM integration**, as described in the TOC.

# Recommended machine size

For this project on a single Ubuntu 24.04 machine with 70 GB storage, I recommend:

## Best practical configuration

CPU: 6 cores
RAM: 24 GB
Storage: 70 GB

## Minimum workable configuration

CPU: 4 cores
RAM: 16 GB
Storage: 70 GB

## Comfortable configuration

CPU: 8 cores
RAM: 32 GB
Storage: 70 GB

# Why we need this much CPU and RAM?

This project is not just one application. By the end, your machine may run multiple services:

| Component | Why it needs resources |
| :--- | :--- |
| Neo4j | Graph database, graph algorithms, vector index |
| Neo4j GDS | Graph analytics such as PageRank/community detection |
| Apache Spark | Data processing jobs |
| Apache Airflow | Scheduler, webserver, workers, metadata DB |
| PostgreSQL | Airflow metadata DB and/or sample source system |
| Redis | Airflow CeleryExecutor broker if used |
| FastAPI | Graph RAG and reporting API |
| NeoDash | Dashboarding layer |
| Bloom | Visual graph investigation, usually via Neo4j environment |
| Power BI | Mostly outside Ubuntu if using Desktop, or via API/reporting layer |
| LLM/embedding service | May use external API or local lightweight model |

# Design of the project

| Parameter / Component | Configuration Details                                                 |
| :-------------------- | :-------------------------------------------------------------------- |
| **Mode**              | Local single-machine learning environment                             |
| **Container runtime** | Docker / Docker Compose                                               |
| **Dataset size**      | Small to medium                                                       |
| **Neo4j**             | Single instance                                                       |
| **Spark**             | Local mode                                                            |
| **Airflow**           | Docker Compose                                                        |
| **Oracle**            | Simulated using sample tables or optional lightweight container later |
| **PostgreSQL**        | Local/containerised source database                                   |
| **FastAPI**           | Local service                                                         |
| **NeoDash/Bloom**     | Connected to Neo4j                                                    |
| **Power BI**          | Connect later from Windows/Power BI service/API path                  |

# Before we proceed to installation

```bash
lscpu && echo "----- MEMORY -----" && free -h && echo "----- DISK -----" && df -h /
```

# Practical resource allocation for our machine

| Component / Layer | Estimated Memory Requirement |
| :--- | :--- |
| **Neo4j** | 6–8 GB RAM |
| **Spark local mode** | 8–12 GB RAM |
| **Airflow stack** | 4–6 GB RAM |
| **PostgreSQL/Redis** | 1–2 GB RAM |
| **FastAPI** | 1–2 GB RAM |
| **Local small LLM** | 1–4 GB RAM |
| **Embedding model** | 1–2 GB RAM |
| **System buffer** | Still plenty |

# Step 1 — Check whether Docker already exists

```bash
docker --version || true
docker compose version || true
containerd --version || true
```

# Step 2 — Install Docker repository prerequisites

```bash
#Refreshes your Ubuntu package index.
apt update

#Installs the two required packages from Docker’s official instructions:
# ca-certificates — needed for secure HTTPS certificate validation.
# curl — needed to download Docker’s official GPG key in the next step.
apt install -y ca-certificates curl 
```

# Step 3 — Add Docker’s official GPG key

```bash
# Creates the keyrings directory with safe permissions.
install -m 0755 -d /etc/apt/keyrings

# Downloads Docker’s official GPG key.
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

# Makes the key readable by apt.
chmod a+r /etc/apt/keyrings/docker.asc

# Verifies the key file exists and shows its permissions.
ls -l /etc/apt/keyrings/docker.asc
```

# Step 4 — Add Docker’s official apt repository
```bash
# Creates the Docker repository source file.
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

cat /etc/apt/sources.list.d/docker.sources
```

## What this does?

```bash
tee /etc/apt/sources.list.d/docker.sources
```

Creates the Docker repository source file.

```bash
URIs: https://download.docker.com/linux/ubuntu
```

Points apt to Docker’s official Ubuntu package repository.

```bash
Suites: noble
```

Should automatically resolve to your Ubuntu codename, because your system is Ubuntu 24.04 Noble.

```bash
Architectures: amd64
```

Should automatically resolve to your CPU architecture.

```bash
Signed-By: /etc/apt/keyrings/docker.asc
```

Tells apt to trust packages from this repository only when signed by the Docker GPG key we just added.

# Step 5 — Refresh apt after adding Docker repository

```bash
apt update
```

# Step 6 — Check Docker package availability

```bash
apt-cache policy docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

# Step 7 — Install Docker Engine and plugins

```bash
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

# Step 8 — Verify Docker service status

```bash
systemctl status docker --no-pager
```

# Step 9 — Verify Docker can run containers

```bash
docker run hello-world
```

# Step 10 — Verify Docker Compose plugin

```bash
docker compose version
```

# Step 11 — Create the project and Neo4j folders

```bash
mkdir -p /opt/supportgraph/neo4j/data
mkdir -p /opt/supportgraph/neo4j/logs
mkdir -p /opt/supportgraph/neo4j/import
mkdir -p /opt/supportgraph/neo4j/plugins

ls -ld /opt/supportgraph
ls -ld /opt/supportgraph/neo4j
ls -ld /opt/supportgraph/neo4j/data
ls -ld /opt/supportgraph/neo4j/logs
ls -ld /opt/supportgraph/neo4j/import
ls -ld /opt/supportgraph/neo4j/plugins
```

> This is important because the official Neo4j Docker documentation cautions that folders to be mounted must exist before starting Docker, otherwise Neo4j may fail to start due to permissions errors.

# Step 12 — Check whether Neo4j ports are free

```bash
ss -lntp | grep -E ':7474|:7687' || echo "Ports 7474 and 7687 are free"
```

>Neo4j Browser web interface port. The Neo4j documentation says you can open Neo4j Browser at http://localhost:7474/ after starting the container.
>
>Neo4j Bolt port, published in the official Docker example as --publish=7687:7687.

# Step 13 — Start Neo4j container

```bash
docker run -d \
  --name supportgraph-neo4j \
  --restart always \
  --publish=7474:7474 \
  --publish=7687:7687 \
  --env NEO4J_AUTH=neo4j/SupportGraph@123 \
  --volume=/opt/supportgraph/neo4j/data:/data \
  neo4j:2026.04.0
```

> We started Neo4j using the official Docker pattern from the Neo4j Operations Manual: publishing ports 7474 and 7687, setting NEO4J_AUTH, mounting /data, and using the neo4j:2026.04.0 Community Edition image.

# Step 14 — Check Neo4j container status

```bash
docker ps --filter "name=supportgraph-neo4j"
```

# Step 15 — Check Neo4j startup logs

```bash
docker logs supportgraph-neo4j --tail 80
```

> The official Neo4j Docker documentation says the container can be accessed through Neo4j Browser after starting, and the Docker run example exposes 7474 and 7687; checking logs before browser login is a safe operational verification step based on that startup flow.

# Step 16 — Test Neo4j Browser endpoint locally

```bash
curl -I http://localhost:7474/
```

# Step 17 — Open Neo4j Browser and log in

> Please open this URL in your browser: http://localhost:7474/
> 
> Then log in with:
> 
> Username: neo4j
> Password: SupportGraph@123

# Step 18 — Run the first Cypher sanity-check query

```bash
RETURN "SupportGraph Neo4j is ready" AS message;
```

where: -

```text
RETURN
```

asks Neo4j to return something in the result set. The official Cypher Manual says RETURN defines the parts included in the query result.

```text
"SupportGraph Neo4j is ready"
```

is a text value, also called a literal expression.

```text
AS message
```

renames the output column to message. The official Cypher Manual says returned columns can be renamed using the AS operator.

# Step 19 — Create the first Customer node

```bash
CREATE (c:Customer {
  customerId: "C001",
  name: "Asha Sharma",
  region: "North",
  segment: "Premium"
})
RETURN c;
```

# Step 20 — Verify the Customer node by customerId

```bash
MATCH (c:Customer {customerId: "C001"})
RETURN c.customerId AS customerId,
       c.name AS name,
       c.region AS region,
       c.segment AS segment;
```

# Step 21 — Create the first Ticket node

```bash
CREATE (t:Ticket {
  ticketId: "T001",
  title: "Unable to login to mobile app",
  issueType: "Login Failure",
  priority: "High",
  status: "Open",
  createdDate: "2026-05-24"
})
RETURN t;
```

# Step 22 — Create relationship between Customer and Ticket

```bash
MATCH (c:Customer {customerId: "C001"})
MATCH (t:Ticket {ticketId: "T001"})
CREATE (c)-[:RAISED]->(t)
RETURN c, t;
```

# Step 23 — Verify the Customer → Ticket path

```bash
MATCH path = (c:Customer {customerId: "C001"})-[r:RAISED]->(t:Ticket {ticketId: "T001"})
RETURN path,
       type(r) AS relationshipType,
       c.name AS customerName,
       t.ticketId AS ticketId,
       t.title AS ticketTitle;
```


# Step 24 — Create the first Product node

```bash
CREATE (p:Product {
  productId: "P001",
  name: "Mobile App",
  category: "Digital",
  platform: "Android/iOS"
})
RETURN p;
```

# Step 25 — Connect Ticket to Product

```bash
MATCH (t:Ticket {ticketId: "T001"})
MATCH (p:Product {productId: "P001"})
CREATE (t)-[:ABOUT]->(p)
RETURN t, p;
```

# Step 26 — Verify Customer → Ticket → Product path

```bash
MATCH path = (c:Customer {customerId: "C001"})-[:RAISED]->(t:Ticket {ticketId: "T001"})-[:ABOUT]->(p:Product {productId: "P001"})
RETURN path,
       c.name AS customerName,
       t.ticketId AS ticketId,
       t.title AS ticketTitle,
       p.name AS productName;
```

#  Step 27 — Create the first Issue node

```bash 
CREATE (i:Issue {
  issueId: "I001",
  name: "Login Failure",
  category: "Authentication",
  severity: "High"
})
RETURN i;
```

# Step 28 — Connect Ticket to Issue

```bash
MATCH (t:Ticket {ticketId: "T001"})
MATCH (i:Issue {issueId: "I001"})
CREATE (t)-[:HAS_ISSUE]->(i)
RETURN t, i;
```

# Step 29 — Verify Customer, Ticket, Product, and Issue together

```bash
MATCH path1 = (c:Customer {customerId: "C001"})-[:RAISED]->(t:Ticket {ticketId: "T001"})-[:ABOUT]->(p:Product {productId: "P001"})
MATCH path2 = (t)-[:HAS_ISSUE]->(i:Issue {issueId: "I001"})
RETURN path1,
       path2,
       c.name AS customerName,
       t.ticketId AS ticketId,
       p.name AS productName,
       i.name AS issueName,
       i.severity AS issueSeverity;
```

# Step 30 — Create the first Agent node

```bash
CREATE (a:Agent {
  agentId: "A001",
  name: "Rajat Support",
  team: "L1 Support",
  location: "India"
})
RETURN a;
```

#  Step 31 — Connect Ticket to Agent

```bash
MATCH (t:Ticket {ticketId: "T001"})
MATCH (a:Agent {agentId: "A001"})
CREATE (t)-[:ASSIGNED_TO]->(a)
RETURN t, a;
```

# Step 32 — Verify the full Day 1 mini graph

```bash
MATCH path1 = (c:Customer {customerId: "C001"})-[:RAISED]->(t:Ticket {ticketId: "T001"})
MATCH path2 = (t)-[:ABOUT]->(p:Product {productId: "P001"})
MATCH path3 = (t)-[:HAS_ISSUE]->(i:Issue {issueId: "I001"})
MATCH path4 = (t)-[:ASSIGNED_TO]->(a:Agent {agentId: "A001"})
RETURN path1,
       path2,
       path3,
       path4,
       c.name AS customerName,
       t.ticketId AS ticketId,
       t.title AS ticketTitle,
       p.name AS productName,
       i.name AS issueName,
       a.name AS agentName;
```

# Step 33 — Create the second Customer node

```bash
CREATE (c:Customer {
  customerId: "C002",
  name: "Ravi Mehta",
  region: "West",
  segment: "Standard"
})
RETURN c;
```

#  Step 34 — Create Ravi’s Ticket node

```bash
CREATE (t:Ticket {
  ticketId: "T002",
  title: "Payment failed during checkout",
  issueType: "Payment Failure",
  priority: "Medium",
  status: "Open",
  createdDate: "2026-05-24"
})
RETURN t;
```

# Step 35 — Connect Ravi to Ticket T002

```bash
MATCH (c:Customer {customerId: "C002"})
MATCH (t:Ticket {ticketId: "T002"})
CREATE (c)-[:RAISED]->(t)
RETURN c, t;
```

# Step 36 — Create the second Product node

```bash
CREATE (p:Product {
  productId: "P002",
  name: "Payment Gateway",
  category: "Payments",
  platform: "Web/API"
})
RETURN p;
```

# Step 37 — Connect Ticket T002 to Product P002

```bash
MATCH (t:Ticket {ticketId: "T002"})
MATCH (p:Product {productId: "P002"})
CREATE (t)-[:ABOUT]->(p)
RETURN t, p;
```

# Step 38 — Create the second Issue node

```bash
CREATE (i:Issue {
  issueId: "I002",
  name: "Payment Failure",
  category: "Transaction",
  severity: "Medium"
})
RETURN i;
```

# Step 39 — Connect Ticket T002 to Issue I002

```bash
MATCH (t:Ticket {ticketId: "T002"})
MATCH (i:Issue {issueId: "I002"})
CREATE (t)-[:HAS_ISSUE]->(i)
RETURN t, i;
```

# Step 40 — Assign Ticket T002 to Agent A001

```bash
MATCH (t:Ticket {ticketId: "T002"})
MATCH (a:Agent {agentId: "A001"})
CREATE (t)-[:ASSIGNED_TO]->(a)
RETURN t, a;
```

# Step 41 — Verify both support scenarios together

```bash
MATCH (c:Customer)-[:RAISED]->(t:Ticket)
MATCH (t)-[:ABOUT]->(p:Product)
MATCH (t)-[:HAS_ISSUE]->(i:Issue)
MATCH (t)-[:ASSIGNED_TO]->(a:Agent)
RETURN c.name AS customerName,
       t.ticketId AS ticketId,
       t.title AS ticketTitle,
       p.name AS productName,
       i.name AS issueName,
       i.severity AS issueSeverity,
       a.name AS agentName
ORDER BY t.ticketId;
```

# Step 42 — Count tickets by issue

```bash
MATCH (t:Ticket)-[:HAS_ISSUE]->(i:Issue)
RETURN i.name AS issueName,
       i.severity AS severity,
       count(t) AS ticketCount
ORDER BY ticketCount
```

# Step 43 — Count tickets by product

```bash
MATCH (t:Ticket)-[:ABOUT]->(p:Product)
RETURN p.name AS productName,
       p.category AS productCategory,
       count(t) AS ticketCount
ORDER BY ticketCount DESC;
```

# Step 44 — Count open tickets by assigned agent

```bash
MATCH (t:Ticket)-[:ASSIGNED_TO]->(a:Agent)
WHERE t.status = "Open"
RETURN a.name AS agentName,
       a.team AS team,
       count(t) AS openTicketCount
ORDER BY openTicketCount DESC;
```

# Step 45 — Show each agent’s open ticket list

```bash
MATCH (t:Ticket)-[:ASSIGNED_TO]->(a:Agent)
WHERE t.status = "Open"
RETURN a.name AS agentName,
       a.team AS team,
       count(t) AS openTicketCount,
       collect(t.ticketId) AS openTicketIds,
       collect(t.title) AS openTicketTitles
ORDER BY openTicketCount DESC;
```

# Step 46 — Find customers handled by Rajat Support

```bash
MATCH (c:Customer)-[:RAISED]->(t:Ticket)-[:ASSIGNED_TO]->(a:Agent {agentId: "A001"})
RETURN c.customerId AS customerId,
       c.name AS customerName,
       t.ticketId AS ticketId,
       t.title AS ticketTitle,
       a.name AS agentName
ORDER BY c.customerId;
```

# Step 47 — Find products handled by Rajat Support’s tickets

```bash
MATCH (a:Agent {agentId: "A001"})<-[:ASSIGNED_TO]-(t:Ticket)-[:ABOUT]->(p:Product)
RETURN a.name AS agentName,
       t.ticketId AS ticketId,
       t.title AS ticketTitle,
       p.productId AS productId,
       p.name AS productName,
       p.category AS productCategory
ORDER BY t.ticketId;
```

# Step 48 — Group products handled by Rajat Support

```bash
MATCH (a:Agent {agentId: "A001"})<-[:ASSIGNED_TO]-(t:Ticket)-[:ABOUT]->(p:Product)
RETURN a.name AS agentName,
       count(t) AS ticketCount,
       collect(t.ticketId) AS ticketIds,
       collect(p.name) AS affectedProducts,
       collect(p.category) AS productCategories;
```

# Step 49 — Final Day 1 support case summary query

```bash
MATCH (c:Customer)-[:RAISED]->(t:Ticket)
MATCH (t)-[:ABOUT]->(p:Product)
MATCH (t)-[:HAS_ISSUE]->(i:Issue)
MATCH (t)-[:ASSIGNED_TO]->(a:Agent)
RETURN c.customerId AS customerId,
       c.name AS customerName,
       t.ticketId AS ticketId,
       t.title AS ticketTitle,
       t.priority AS priority,
       t.status AS status,
       p.name AS productName,
       i.name AS issueName,
       i.severity AS issueSeverity,
       a.name AS agentName
ORDER BY t.ticketId;
```

# Step 50 — Count nodes by label

```bash
MATCH (n)
RETURN labels(n) AS nodeLabels,
       count(*) AS nodeCount
ORDER BY nodeLabels;
```

# Step 51 — Count relationships by type

```bash
MATCH ()-[r]->()
RETURN type(r) AS relationshipType,
       count(*) AS relationshipCount
ORDER BY relationshipType;
```

# Step 52 — Create uniqueness constraint for Customer.customerId

```bash
CREATE CONSTRAINT customer_customerId_unique
FOR (c:Customer)
REQUIRE c.customerId IS UNIQUE;
```

# Step 53 — Verify the Customer.customerId constraint

```bash
SHOW CONSTRAINTS
YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex
WHERE name = "customer_customerId_unique"
RETURN name, type, entityType, labelsOrTypes, properties, ownedIndex;
```

# Step 54 — Create uniqueness constraint for Ticket.ticketId

```bash
CREATE CONSTRAINT ticket_ticketId_unique
FOR (t:Ticket)
REQUIRE t.ticketId IS UNIQUE;
```

# Step 55 — Verify the Ticket.ticketId constraint

```bash
SHOW CONSTRAINTS
YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex
WHERE name = "ticket_ticketId_unique"
RETURN name, type, entityType, labelsOrTypes, properties, ownedIndex;
```

# Step 56 — Create uniqueness constraint for Product.productId

```bash
CREATE CONSTRAINT product_productId_unique
FOR (p:Product)
REQUIRE p.productId IS UNIQUE;
```

# Step 57 — Verify the Product.productId constraint

```bash
SHOW CONSTRAINTS
YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex
WHERE name = "product_productId_unique"
RETURN name, type, entityType, labelsOrTypes, properties, ownedIndex;
```

# Step 58 — Create uniqueness constraint for Issue.issueId

```bash
CREATE CONSTRAINT issue_issueId_unique
FOR (i:Issue)
REQUIRE i.issueId IS UNIQUE;
```

# Step 59 — Verify Issue.issueId uniqueness constraint

```bash
SHOW CONSTRAINTS
YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex
WHERE name = "issue_issueId_unique"
RETURN name, type, entityType, labelsOrTypes, properties, ownedIndex;
```

# Step 60 — Create uniqueness constraint for Agent.agentId

```bash
CREATE CONSTRAINT agent_agentId_unique
FOR (a:Agent)
REQUIRE a.agentId IS UNIQUE;
```

# Step 61 — Verify Agent.agentId uniqueness constraint

```bash
SHOW CONSTRAINTS
YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex
WHERE name = "agent_agentId_unique"
RETURN name, type, entityType, labelsOrTypes, properties, ownedIndex;
```

# Step 62 — Final summary check for all Day 1 ID constraints

```bash
SHOW CONSTRAINTS
YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex
WHERE name IN [
  "customer_customerId_unique",
  "ticket_ticketId_unique",
  "product_productId_unique",
  "issue_issueId_unique",
  "agent_agentId_unique"
]
RETURN name,
       type,
       entityType,
       labelsOrTypes,
       properties,
       ownedIndex
ORDER BY name;
```

# Step 63 — Find nearby support entities from one customer

```bash
MATCH path = (c:Customer {customerId: "C001"})--{1,3}(connected)
RETURN path
LIMIT 25;
```

# Step 64 — Return readable details from the variable-length paths

```bash
MATCH path = (c:Customer {customerId: "C001"})--{1,3}(connected)
RETURN
  c.name AS customer,
  labels(connected) AS connectedNodeLabels,
  connected AS connectedNode,
  length(path) AS hops
ORDER BY hops, connectedNodeLabels
LIMIT 25;
```

# Step 65 — Safer variable-length path using support relationship types

```bash
MATCH path =
  (c:Customer {customerId: "C001"})
  ((start)-[:RAISED|ABOUT|HAS_ISSUE|ASSIGNED_TO]-(next)){1,3}
  (connected)
RETURN
  c.name AS customer,
  labels(connected) AS connectedNodeLabels,
  connected AS connectedNode,
  length(path) AS hops
ORDER BY hops, connectedNodeLabels
LIMIT 25;
```

# Step 66 — Basic UNWIND practice with support issue names

```bash
UNWIND ["Login Failure", "Payment Failure", "App Crash"] AS issueName
RETURN issueName
ORDER BY issueName;
```

# Step 67 — UNWIND a list of maps

```bash
UNWIND [
  {issueId: "I001", name: "Login Failure", severity: "High"},
  {issueId: "I002", name: "Payment Failure", severity: "Medium"},
  {issueId: "I003", name: "App Crash", severity: "High"}
] AS issue
RETURN
  issue.issueId AS issueId,
  issue.name AS issueName,
  issue.severity AS severity
ORDER BY issueId;
```

# Step 68 — Use UNWIND + MERGE to add or match App Crash

```bash
UNWIND [
  {issueId: "I003", name: "App Crash", severity: "High"}
] AS issue
MERGE (i:Issue {issueId: issue.issueId})
SET
  i.name = issue.name,
  i.severity = issue.severity
RETURN
  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS severity;
```

# Step 69 — Verify all Issue nodes

```bash
MATCH (i:Issue)
RETURN
  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS severity
ORDER BY issueId;
```

# Step 70 — Basic WITH chaining from customer to ticket to product

```bash
MATCH (c:Customer)-[:RAISED]->(t:Ticket)
WITH c, t
MATCH (t)-[:ABOUT]->(p:Product)
RETURN
  c.name AS customer,
  t.ticketId AS ticketId,
  t.title AS ticketTitle,
  p.name AS product
ORDER BY ticketId;
```

# Step 71 — Use WITH to aggregate ticket count per product

```bash
MATCH (t:Ticket)-[:ABOUT]->(p:Product)
WITH
  p,
  count(t) AS ticketCount
WHERE ticketCount >= 1
RETURN
  p.productId AS productId,
  p.name AS productName,
  ticketCount
ORDER BY ticketCount DESC, productName;
```

# Step 72 — Use WITH and collect() to list tickets per product

```bash
MATCH (t:Ticket)-[:ABOUT]->(p:Product)
WITH
  p,
  collect(t.ticketId) AS ticketIds,
  count(t) AS ticketCount
RETURN
  p.productId AS productId,
  p.name AS productName,
  ticketCount,
  ticketIds
ORDER BY ticketCount DESC, productName;
```

# Step 73 — Use WITH ... WHERE to filter high-priority tickets

```bash
MATCH (c:Customer)-[:RAISED]->(t:Ticket)-[:ABOUT]->(p:Product)
WITH
  c,
  t,
  p,
  t.priority AS priority
WHERE priority = "High"
RETURN
  c.name AS customer,
  t.ticketId AS ticketId,
  t.title AS ticketTitle,
  priority,
  p.name AS product
ORDER BY ticketId;
```

# Step 74 — Check whether APOC procedures are available

```bash
SHOW PROCEDURES
YIELD name
WHERE name STARTS WITH "apoc."
RETURN name
ORDER BY name
LIMIT 20;
```

> it means that the APOC is not enabled/installed.

# Step 75 — Identify your running Neo4j Docker container

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```

# Step 76 — Check whether NEO4J_PLUGINS is set on the running container

```bash
docker inspect supportgraph-neo4j --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -i -E 'NEO4J_PLUGINS|NEO4JLABS_PLUGINS|apoc'
```

# Step 77 — Check APOC jar location inside the container

```bash
docker exec supportgraph-neo4j sh -c 'ls -la /var/lib/neo4j/labs 2>/dev/null; echo "--- plugins ---"; ls -la /var/lib/neo4j/plugins 2>/dev/null'
```

> APOC is packaged with Neo4j, but it is not installed/enabled yet.

# Step 78 — Copy APOC jar into the plugins directory

```bash
docker exec supportgraph-neo4j sh -c 'cp /var/lib/neo4j/labs/apoc-2026.04.0-core.jar /var/lib/neo4j/plugins/apoc.jar && ls -la /var/lib/neo4j/plugins'
```

# Step 79 — Fix APOC jar ownership inside the container

```bash
docker exec supportgraph-neo4j sh -c 'chown neo4j:neo4j /var/lib/neo4j/plugins/apoc.jar && ls -la /var/lib/neo4j/plugins'
```

# Step 80 — Verify the exact APOC plugin filename

```bash
docker exec supportgraph-neo4j sh -c 'find /var/lib/neo4j/plugins -maxdepth 1 -type f -printf "%f\n"'
```

# Step 81 — Restart the Neo4j Docker container

```bash
docker restart supportgraph-neo4j
```

> Now, you will need to re-sign in using the password that was last seen on step 17.
# Step 82 — Verify APOC is now visible in Neo4j

```bash
SHOW PROCEDURES
YIELD name
WHERE name STARTS WITH "apoc."
RETURN name
ORDER BY name
LIMIT 20;
```

# Step 83 — Check APOC version

```bash
RETURN apoc.version() AS apocVersion;
```

# Step 84 — Use APOC built-in help

```bash
CALL apoc.help("text")
YIELD name, type, text, signature
RETURN
  name,
  type,
  text,
  signature
ORDER BY name
LIMIT 10;
```

# Step 86 — Run first EXPLAIN query

```bash
EXPLAIN
MATCH (c:Customer)-[:RAISED]->(t:Ticket)
RETURN
  c.name AS customer,
  t.ticketId AS ticketId;
```

# Step 87 — Run first safe PROFILE query

```bash
PROFILE
MATCH (c:Customer)-[:RAISED]->(t:Ticket)
RETURN
  c.name AS customer,
  t.ticketId AS ticketId;
```

# Day 1 agenda finishes here!!!!! Congratulations to you if you have come this far!!

# Day 2 starts now

# Step 1: Check whether GDS is already installed

```bash
RETURN gds.version() AS gdsVersion;
```

> it is not present.

# Step 2: Check whether Docker was configured with GDS

```bash
docker inspect supportgraph-neo4j --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -i -E 'NEO4J_PLUGINS|NEO4JLABS_PLUGINS|gds|graph'
```

> it is not present as well.

# Step 3: Check whether a GDS plugin file exists inside the container

```bash
docker exec supportgraph-neo4j sh -c 'echo "--- plugins ---"; ls -la /var/lib/neo4j/plugins 2>/dev/null | grep -i -E "gds|graph-data-science|neo4j-graph|README|apoc" || true; echo "--- labs ---"; ls -la /var/lib/neo4j/labs 2>/dev/null | grep -i -E "gds|graph-data-science|neo4j-graph|README|apoc" || true'
```

# Step 4: Inspect data and plugin mounts for the current container

```bash
docker inspect supportgraph-neo4j --format '{{json .Mounts}}'
```

> This means your Neo4j database data is stored on the host path, not only inside the container. So if we recreate the container carefully and reuse the same /data mount, the graph data should remain available.

# Step 5: Inspect current Docker runtime settings

```bash
docker inspect supportgraph-neo4j --format '
Name: {{.Name}}
Image: {{.Config.Image}}
NetworkMode: {{.HostConfig.NetworkMode}}
RestartPolicy: {{json .HostConfig.RestartPolicy}}
PortBindings: {{json .HostConfig.PortBindings}}
'
```

# Step 6: Stop and rename the current container as a rollback copy

```bash
docker stop supportgraph-neo4j && docker rename supportgraph-neo4j supportgraph-neo4j-before-gds
```

# Step 7: Create the new Neo4j container with APOC + GDS enabled

```bash
docker run -d \
  --name supportgraph-neo4j \
  --restart always \
  -p 7474:7474 \
  -p 7687:7687 \
  -v /opt/supportgraph/neo4j/data:/data \
  -e NEO4J_AUTH='neo4j/SupportGraph@123' \
  -e NEO4J_PLUGINS='["apoc","graph-data-science"]' \
  neo4j:2026.04.0
```

# Step 8: Check container logs for plugin installation

```bash
docker logs supportgraph-neo4j 2>&1 | grep -i -E "plugin|apoc|graph-data-science|gds|started|error|warn" | tail -n 80
```

# Step 9: Verify GDS from Neo4j Browser

```bash
RETURN gds.version() AS gdsVersion;
```

# Step 10: List available GDS operations

```bash
CALL gds.list()
YIELD name, type, description
RETURN
  name,
  type,
  description
ORDER BY name
LIMIT 20;
```

# Step 11: Find available GDS graph catalogue operations

```bash
CALL gds.list()
YIELD name, type, description
WHERE name STARTS WITH "gds.graph."
RETURN
  name,
  type,
  description
ORDER BY name
LIMIT 30;
```

# Step 12: List current GDS graph projections

```bash
CALL gds.graph.list();
```

# Step 13: Inspect the exact gds.graph.project signature

```bash
CALL gds.list()
YIELD name, type, signature, description
WHERE name IN ["gds.graph.project", "gds.graph.project.estimate"]
RETURN
  name,
  type,
  signature,
  description
ORDER BY name;
```

# Step 14: Check whether supportGraph already exists

```bash
RETURN gds.graph.exists("supportGraph") AS graphExists;
```

# Step 15: Create the first supportGraph GDS projection

```bash
CALL gds.graph.project(
  'supportGraph',
  ['Customer', 'Ticket', 'Product'],
  ['RAISED', 'ABOUT']
)
YIELD
  graphName,
  nodeCount,
  relationshipCount,
  projectMillis
RETURN
  graphName,
  nodeCount,
  relationshipCount,
  projectMillis;
```

# Step 16: Confirm supportGraph in the GDS graph catalogue

```bash
CALL gds.graph.list()
YIELD graphName, nodeCount, relationshipCount
WHERE graphName = "supportGraph"
RETURN
  graphName,
  nodeCount,
  relationshipCount;
```

# Step 17: Inspect Degree Centrality procedure signatures

```bash
CALL gds.list()
YIELD name, type, signature, description
WHERE name STARTS WITH "gds.degree."
RETURN
  name,
  type,
  signature,
  description
ORDER BY name;
```

# Step 18: Run Degree Centrality in stream mode

```bash
CALL gds.degree.stream(
  'supportGraph',
  { orientation: 'UNDIRECTED' }
)
YIELD nodeId, score
RETURN
  labels(gds.util.asNode(nodeId)) AS labels,
  coalesce(
    gds.util.asNode(nodeId).name,
    gds.util.asNode(nodeId).ticketId,
    gds.util.asNode(nodeId).productId
  ) AS nodeNameOrId,
  score AS degreeScore
ORDER BY degreeScore DESC, nodeNameOrId;
```

# Step 19: Run Degree Centrality in stats mode

```bash
CALL gds.degree.stats(
  'supportGraph',
  { orientation: 'UNDIRECTED' }
)
YIELD centralityDistribution, computeMillis
RETURN
  centralityDistribution.min AS minDegree,
  centralityDistribution.mean AS meanDegree,
  centralityDistribution.max AS maxDegree,
  computeMillis;
```

# Step 20: Check whether degreeScore already exists

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product
RETURN
  labels(n) AS labels,
  count(n) AS totalNodes,
  count(n.degreeScore) AS nodesWithDegreeScore
ORDER BY labels;
```

# Step 21: Write Degree Centrality back to Neo4j

```bash
CALL gds.degree.write(
  'supportGraph',
  {
    orientation: 'UNDIRECTED',
    writeProperty: 'degreeScore'
  }
)
YIELD
  centralityDistribution,
  nodePropertiesWritten,
  computeMillis,
  writeMillis
RETURN
  centralityDistribution.min AS minDegree,
  centralityDistribution.mean AS meanDegree,
  centralityDistribution.max AS maxDegree,
  nodePropertiesWritten,
  computeMillis,
  writeMillis;
```

# Step 22: Verify degreeScore on Neo4j nodes

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product
RETURN
  labels(n) AS labels,
  coalesce(n.name, n.ticketId, n.productId) AS nodeNameOrId,
  n.degreeScore AS degreeScore
ORDER BY degreeScore DESC, nodeNameOrId;
```

# Step 23: Inspect actual Ticket/Issue/Agent relationship types

```bash
MATCH (a)-[r]->(b)
WHERE
  a:Ticket OR b:Ticket OR
  a:Issue OR b:Issue OR
  a:Agent OR b:Agent
RETURN
  labels(a) AS fromLabels,
  type(r) AS relationshipType,
  labels(b) AS toLabels,
  count(*) AS relationshipCount
ORDER BY
  relationshipType,
  fromLabels,
  toLabels;
```

# Step 24: Check whether supportGraphFull already exists

```bash
RETURN gds.graph.exists("supportGraphFull") AS graphExists;
```

# Step 25: Create the richer supportGraphFull projection

```bash
CALL gds.graph.project(
  'supportGraphFull',
  ['Customer', 'Ticket', 'Product', 'Agent', 'Issue'],
  ['RAISED', 'ABOUT', 'ASSIGNED_TO', 'HAS_ISSUE']
)
YIELD
  graphName,
  nodeCount,
  relationshipCount,
  projectMillis
RETURN
  graphName,
  nodeCount,
  relationshipCount,
  projectMillis;
```

# Step 26: Confirm supportGraphFull in the GDS graph catalogue

```bash
CALL gds.graph.list()
YIELD graphName, nodeCount, relationshipCount
WHERE graphName = "supportGraphFull"
RETURN
  graphName,
  nodeCount,
  relationshipCount;
```

# Step 27: Run Degree Centrality stream mode on supportGraphFull

```bash
CALL gds.degree.stream(
  'supportGraphFull',
  { orientation: 'UNDIRECTED' }
)
YIELD nodeId, score
RETURN
  labels(gds.util.asNode(nodeId)) AS labels,
  coalesce(
    gds.util.asNode(nodeId).name,
    gds.util.asNode(nodeId).ticketId,
    gds.util.asNode(nodeId).productId,
    gds.util.asNode(nodeId).issueId,
    gds.util.asNode(nodeId).agentId
  ) AS nodeNameOrId,
  score AS degreeScore
ORDER BY degreeScore DESC, nodeNameOrId;
```

# Step 28: Run Degree Centrality stats mode on supportGraphFull

```bash
CALL gds.degree.stats(
  'supportGraphFull',
  { orientation: 'UNDIRECTED' }
)
YIELD centralityDistribution, computeMillis
RETURN
  centralityDistribution.min AS minDegree,
  centralityDistribution.mean AS meanDegree,
  centralityDistribution.max AS maxDegree,
  computeMillis;
```

# Step 29: Check whether fullDegreeScore already exists

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  count(n) AS totalNodes,
  count(n.fullDegreeScore) AS nodesWithFullDegreeScore
ORDER BY labels;
```

# Step 30: Write richer Degree Centrality back to Neo4j

```bash
CALL gds.degree.write(
  'supportGraphFull',
  {
    orientation: 'UNDIRECTED',
    writeProperty: 'fullDegreeScore'
  }
)
YIELD
  centralityDistribution,
  nodePropertiesWritten,
  computeMillis,
  writeMillis
RETURN
  centralityDistribution.min AS minDegree,
  centralityDistribution.mean AS meanDegree,
  centralityDistribution.max AS maxDegree,
  nodePropertiesWritten,
  computeMillis,
  writeMillis;
```

# Step 31: Verify fullDegreeScore on Neo4j nodes

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  )
```

# Step 31 retry: Verify fullDegreeScore explicitly

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  properties(n) AS allProperties,
  n.fullDegreeScore AS fullDegreeScore
ORDER BY fullDegreeScore DESC, nodeNameOrId;
```

# Step 32: Compare degreeScore vs fullDegreeScore

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.degreeScore AS degreeScore,
  n.fullDegreeScore AS fullDegreeScore
ORDER BY fullDegreeScore DESC, nodeNameOrId;
```

# Step 33: Inspect PageRank procedure signatures

```bash
CALL gds.list()
YIELD name, type AS operationType, signature, description
WHERE name STARTS WITH "gds.pageRank."
RETURN
  name,
  operationType,
  signature,
  description
```

# Step 34: Run PageRank stream mode on supportGraphFull

```bash
CALL gds.pageRank.stream(
  'supportGraphFull',
  {
    maxIterations: 20,
    dampingFactor: 0.85
  }
)
YIELD nodeId, score
RETURN
  labels(gds.util.asNode(nodeId)) AS labels,
  coalesce(
    gds.util.asNode(nodeId).name,
    gds.util.asNode(nodeId).ticketId,
    gds.util.asNode(nodeId).productId,
    gds.util.asNode(nodeId).issueId,
    gds.util.asNode(nodeId).agentId
  ) AS nodeNameOrId,
  score AS pageRankScore
ORDER BY pageRankScore DESC, nodeNameOrId;
```

# Step 35: Run PageRank stats mode on supportGraphFull

```bash
CALL gds.pageRank.stats(
  'supportGraphFull',
  {
    maxIterations: 20,
    dampingFactor: 0.85
  }
)
YIELD
  ranIterations,
  didConverge,
  centralityDistribution,
  computeMillis
RETURN
  ranIterations,
  didConverge,
  centralityDistribution.min AS minPageRank,
  centralityDistribution.mean AS meanPageRank,
  centralityDistribution.max AS maxPageRank,
  computeMillis;
```

# Step 36: Check whether pageRankScore already exists

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  count(n) AS totalNodes,
  count(n.pageRankScore) AS nodesWithPageRankScore
ORDER BY labels;
```

# Step 37: Write PageRank back to Neo4j

```bash
CALL gds.pageRank.write(
  'supportGraphFull',
  {
    maxIterations: 20,
    dampingFactor: 0.85,
    writeProperty: 'pageRankScore'
  }
)
YIELD
  ranIterations,
  didConverge,
  centralityDistribution,
  nodePropertiesWritten,
  computeMillis,
  writeMillis
RETURN
  ranIterations,
  didConverge,
  centralityDistribution.min AS minPageRank,
  centralityDistribution.mean AS meanPageRank,
  centralityDistribution.max AS maxPageRank,
  nodePropertiesWritten,
  computeMillis,
  writeMillis;
```

# Step 38: Verify pageRankScore on Neo4j nodes

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.pageRankScore AS pageRankScore
ORDER BY pageRankScore DESC, nodeNameOrId;
```

# Step 39: Compare fullDegreeScore vs pageRankScore

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
WITH
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.fullDegreeScore AS fullDegreeScore,
  n.pageRankScore AS pageRankScore
RETURN
  labels,
  nodeNameOrId,
  fullDegreeScore,
  pageRankScore
ORDER BY pageRankScore DESC;
```

# Step 40: Inspect Betweenness Centrality procedure signatures

```bash
CALL gds.list()
YIELD name, type AS operationType, signature, description
WHERE name STARTS WITH "gds.betweenness."
RETURN
  name,
  operationType,
  signature,
  description
ORDER BY name;
```

# Step 41: Run Betweenness Centrality stream mode

```bash
CALL gds.betweenness.stream('supportGraphFull')
YIELD nodeId, score
RETURN
  labels(gds.util.asNode(nodeId)) AS labels,
  coalesce(
    gds.util.asNode(nodeId).name,
    gds.util.asNode(nodeId).ticketId,
    gds.util.asNode(nodeId).productId,
    gds.util.asNode(nodeId).issueId,
    gds.util.asNode(nodeId).agentId
  ) AS nodeNameOrId,
  score AS betweennessScore
ORDER BY betweennessScore DESC, nodeNameOrId;
```

# Step 42: Run Betweenness Centrality stats mode

```bash
CALL gds.betweenness.stats('supportGraphFull')
YIELD centralityDistribution, computeMillis
RETURN
  centralityDistribution.min AS minBetweenness,
  centralityDistribution.mean AS meanBetweenness,
  centralityDistribution.max AS maxBetweenness,
  computeMillis;
```

# Step 43: Check whether betweennessScore already exists

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  count(n) AS totalNodes,
  count(n.betweennessScore) AS nodesWithBetweennessScore
ORDER BY labels;
```

# Step 44: Write Betweenness Centrality back to Neo4j

```bash
CALL gds.betweenness.write(
  'supportGraphFull',
  {
    writeProperty: 'betweennessScore'
  }
)
YIELD
  centralityDistribution,
  nodePropertiesWritten,
  computeMillis,
  writeMillis
RETURN
  centralityDistribution.min AS minBetweenness,
  centralityDistribution.mean AS meanBetweenness,
  centralityDistribution.max AS maxBetweenness,
  nodePropertiesWritten,
  computeMillis,
  writeMillis;
```

# Step 45: Verify betweennessScore on Neo4j nodes

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.betweennessScore AS betweennessScore
ORDER BY betweennessScore DESC, nodeNameOrId;
```

# Step 46: Compare centrality scores together

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
WITH
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.fullDegreeScore AS fullDegreeScore,
  n.pageRankScore AS pageRankScore,
  n.betweennessScore AS betweennessScore
RETURN
  labels,
  nodeNameOrId,
  fullDegreeScore,
  pageRankScore,
  betweennessScore
ORDER BY
  betweennessScore DESC,
  pageRankScore DESC,
  fullDegreeScore DESC,
  nodeNameOrId;
```

> Degree Centrality      → Who is most connected?
> PageRank               → Who is most influential?
> Betweenness Centrality → Who acts as a bridge?
> Louvain.                  → Which nodes naturally group together?

# Step 47: Inspect Louvain procedure signatures

```bash
CALL gds.list()
YIELD name, type AS operationType, signature, description
WHERE name STARTS WITH "gds.louvain."
RETURN
  name,
  operationType,
  signature,
  description
ORDER BY name;
```

# Step 48: Inspect supportGraphFull projection schema before Louvain

```bash
CALL gds.graph.list()
YIELD graphName, nodeCount, relationshipCount, schema
WHERE graphName = 'supportGraphFull'
RETURN
  graphName,
  nodeCount,
  relationshipCount,
  schema;
```

# Step 49: Check whether supportGraphLouvain already exists

```bash
CALL gds.graph.list()
YIELD graphName, nodeCount, relationshipCount, schema
WHERE graphName = 'supportGraphLouvain'
RETURN
  graphName,
  nodeCount,
  relationshipCount,
  schema;
```

# Step 50: Create supportGraphLouvain as an undirected projection

```bash
MATCH (source)
WHERE source:Customer
   OR source:Ticket
   OR source:Product
   OR source:Agent
   OR source:Issue

OPTIONAL MATCH (source)-[r]->(target)
WHERE r IS NULL
   OR (
        type(r) IN ['RAISED', 'ABOUT', 'ASSIGNED_TO', 'HAS_ISSUE']
        AND (
          target:Customer
          OR target:Ticket
          OR target:Product
          OR target:Agent
          OR target:Issue
        )
      )

RETURN gds.graph.project(
  'supportGraphLouvain',
  source,
  target,
  {},
  {
    undirectedRelationshipTypes: ['*']
  }
)
```

# Step 51: Confirm supportGraphLouvain in graph catalogue

```bash
CALL gds.graph.list()
YIELD graphName, nodeCount, relationshipCount
WHERE graphName = 'supportGraphLouvain'
RETURN
  graphName,
  nodeCount,
  relationshipCount;
```

# Step 52: Run Louvain stream mode on supportGraphLouvain

```bash
CALL gds.louvain.stream('supportGraphLouvain')
YIELD nodeId, communityId, intermediateCommunityIds
RETURN
  labels(gds.util.asNode(nodeId)) AS labels,
  coalesce(
    gds.util.asNode(nodeId).name,
    gds.util.asNode(nodeId).ticketId,
    gds.util.asNode(nodeId).productId,
    gds.util.asNode(nodeId).issueId,
    gds.util.asNode(nodeId).agentId
  ) AS nodeNameOrId,
  communityId,
  intermediateCommunityIds
ORDER BY communityId, nodeNameOrId;
```

# Step 53: Run Louvain stats mode on supportGraphLouvain

```bash
CALL gds.louvain.stats('supportGraphLouvain')
YIELD
  communityCount,
  ranLevels,
  modularity,
  modularities,
  communityDistribution,
  computeMillis
RETURN
  communityCount,
  ranLevels,
  modularity,
  modularities,
  communityDistribution.min AS minCommunitySize,
  communityDistribution.mean AS meanCommunitySize,
  communityDistribution.max AS maxCommunitySize,
  computeMillis;
```

# Step 54: Check whether louvainCommunityId already exists

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  count(n) AS totalNodes,
  count(n.louvainCommunityId) AS nodesWithLouvainCommunityId
ORDER BY labels;
```

# Step 55: Write Louvain community IDs back to Neo4j

```bash
CALL gds.louvain.write(
  'supportGraphLouvain',
  {
    writeProperty: 'louvainCommunityId'
  }
)
YIELD
  communityCount,
  ranLevels,
  modularity,
  modularities,
  communityDistribution,
  nodePropertiesWritten,
  computeMillis,
  writeMillis
RETURN
  communityCount,
  ranLevels,
  modularity,
  modularities,
  communityDistribution.min AS minCommunitySize,
  communityDistribution.mean AS meanCommunitySize,
  communityDistribution.max AS maxCommunitySize,
  nodePropertiesWritten,
  computeMillis,
  writeMillis;
```

# Step 56: Verify louvainCommunityId on Neo4j nodes

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.louvainCommunityId AS louvainCommunityId
ORDER BY louvainCommunityId, nodeNameOrId;
```

# Step 57: Compare centrality scores with Louvain community

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
WITH
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.fullDegreeScore AS fullDegreeScore,
  n.pageRankScore AS pageRankScore,
  n.betweennessScore AS betweennessScore,
  n.louvainCommunityId AS louvainCommunityId
RETURN
  labels,
  nodeNameOrId,
  fullDegreeScore,
  pageRankScore,
  betweennessScore,
  louvainCommunityId
ORDER BY
  louvainCommunityId,
  betweennessScore DESC,
  pageRankScore DESC,
  fullDegreeScore DESC,
  nodeNameOrId;
```

# Step 58: Inspect Label Propagation procedure signatures

```bash
CALL gds.list()
YIELD name, type AS operationType, signature, description
WHERE name STARTS WITH "gds.labelPropagation."
RETURN
  name,
  operationType,
  signature,
  description
ORDER BY name;
```

# Step 59: Run Label Propagation stream mode

```bash
CALL gds.labelPropagation.stream('supportGraphLouvain')
YIELD nodeId, communityId
RETURN
  labels(gds.util.asNode(nodeId)) AS labels,
  coalesce(
    gds.util.asNode(nodeId).name,
    gds.util.asNode(nodeId).ticketId,
    gds.util.asNode(nodeId).productId,
    gds.util.asNode(nodeId).issueId,
    gds.util.asNode(nodeId).agentId
  ) AS nodeNameOrId,
  communityId AS labelPropagationCommunityId
ORDER BY labelPropagationCommunityId, nodeNameOrId;
```

# Step 61: Run Label Propagation stats mode

```bash
CALL gds.labelPropagation.stats('supportGraphLouvain')
YIELD
  communityCount,
  ranIterations,
  didConverge,
  communityDistribution,
  computeMillis
RETURN
  communityCount,
  ranIterations,
  didConverge,
  communityDistribution.min AS minCommunitySize,
  communityDistribution.mean AS meanCommunitySize,
  communityDistribution.max AS maxCommunitySize,
  computeMillis;
```

# Step 62: Check whether labelPropagationCommunityId already exists

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  count(n) AS totalNodes,
  count(n.labelPropagationCommunityId) AS nodesWithLabelPropagationCommunityId
ORDER BY labels;
```

# Step 63: Write Label Propagation communities back to Neo4j

```bash
CALL gds.labelPropagation.write(
  'supportGraphLouvain',
  {
    writeProperty: 'labelPropagationCommunityId'
  }
)
YIELD
  communityCount,
  ranIterations,
  didConverge,
  communityDistribution,
  nodePropertiesWritten,
  computeMillis,
  writeMillis
RETURN
  communityCount,
  ranIterations,
  didConverge,
  communityDistribution.min AS minCommunitySize,
  communityDistribution.mean AS meanCommunitySize,
  communityDistribution.max AS maxCommunitySize,
  nodePropertiesWritten,
  computeMillis,
  writeMillis;
```

# Step 64: Verify labelPropagationCommunityId on Neo4j nodes

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.labelPropagationCommunityId AS labelPropagationCommunityId
ORDER BY labelPropagationCommunityId, nodeNameOrId;
```

# Step 65: Compare Louvain vs Label Propagation communities

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.louvainCommunityId AS louvainCommunityId,
  n.labelPropagationCommunityId AS labelPropagationCommunityId
ORDER BY
  louvainCommunityId,
  labelPropagationCommunityId,
  nodeNameOrId;
```

# Step 66: Final combined analytics view

```bash
MATCH (n)
WHERE n:Customer OR n:Ticket OR n:Product OR n:Agent OR n:Issue
RETURN
  labels(n) AS labels,
  coalesce(
    n.name,
    n.ticketId,
    n.productId,
    n.issueId,
    n.agentId
  ) AS nodeNameOrId,
  n.fullDegreeScore AS fullDegreeScore,
  n.pageRankScore AS pageRankScore,
  n.betweennessScore AS betweennessScore,
  n.louvainCommunityId AS louvainCommunityId,
  n.labelPropagationCommunityId AS labelPropagationCommunityId
ORDER BY
  louvainCommunityId,
  labelPropagationCommunityId,
  betweennessScore DESC,
  pageRankScore DESC,
  fullDegreeScore DESC,
  nodeNameOrId;
```

# This finishes day 2. Now, onto the day 3.

# Before we start our day 3, we need to verify the current graph state before adding Day 3 data.

# Step 1 — Verify current node counts before adding knowledge data

```cypher
MATCH (n)
RETURN labels(n) AS nodeLabels,
       count(*) AS nodeCount
ORDER BY nodeLabels;
```

# Step 2: Check current relationship types

```cypher
MATCH ()-[r]->()
RETURN type(r) AS relationshipType,
       count(*) AS relationshipCount
ORDER BY relationshipType;
```

# Step 3 — Verify existing constraints

```cypher
SHOW CONSTRAINTS
YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex
RETURN name,
       type,
       entityType,
       labelsOrTypes,
       properties,
       ownedIndex
ORDER BY name;
```

# Step 4 — Create uniqueness constraint for KnowledgeArticle.articleId

```cypher
CREATE CONSTRAINT knowledgeArticle_articleId_unique IF NOT EXISTS
FOR (ka:KnowledgeArticle)
REQUIRE ka.articleId IS UNIQUE;
```

# Step 5 — Verify KnowledgeArticle.articleId uniqueness constraint

```cypher
SHOW CONSTRAINTS
YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex
WHERE name = "knowledgeArticle_articleId_unique"
RETURN name,
       type,
       entityType,
       labelsOrTypes,
       properties,
       ownedIndex;
```

# Step 6 — Create uniqueness constraint for DocumentChunk.chunkId

```cypher
CREATE CONSTRAINT documentChunk_chunkId_unique IF NOT EXISTS
FOR (dc:DocumentChunk)
REQUIRE dc.chunkId IS UNIQUE;
```

# Step 7 — Verify DocumentChunk.chunkId uniqueness constraint

```cypher
SHOW CONSTRAINTS
YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex
WHERE name = "documentChunk_chunkId_unique"
RETURN name,
       type,
       entityType,
       labelsOrTypes,
       properties,
       ownedIndex;
```

# Step 8 — Load sample KnowledgeArticle nodes

```cypher
UNWIND [
  {
    articleId: "K001",
    title: "Fix login failure",
    issueType: "Login Failure",
    content: "If a customer cannot sign in, ask them to reset the password, verify OTP delivery, clear app cache, and retry login."
  },
  {
    articleId: "K002",
    title: "Resolve payment failure",
    issueType: "Payment Failure",
    content: "If payment fails during checkout, check card status, verify available balance, confirm payment gateway response, and retry the transaction."
  },
  {
    articleId: "K003",
    title: "Fix app crash",
    issueType: "App Crash",
    content: "If the mobile app crashes, ask the customer to update the app, clear cache, restart the device, and check device compatibility."
  }
] AS article
MERGE (ka:KnowledgeArticle {articleId: article.articleId})
SET
  ka.title = article.title,
  ka.issueType = article.issueType,
  ka.content = article.content,
  ka.source = "Day 3 sample knowledge base"
RETURN
  ka.articleId AS articleId,
  ka.title AS title,
  ka.issueType AS issueType
ORDER BY articleId;
```

# Step 9 — Verify loaded KnowledgeArticle nodes

```cypher
MATCH (ka:KnowledgeArticle)
RETURN
  ka.articleId AS articleId,
  ka.title AS title,
  ka.issueType AS issueType,
  ka.source AS source
ORDER BY articleId;
```

# Step 10 — Create SOLVES relationships

```cypher
MATCH (ka:KnowledgeArticle)
MATCH (i:Issue {name: ka.issueType})
MERGE (ka)-[:SOLVES]->(i)
RETURN
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  ka.issueType AS articleIssueType,
  i.issueId AS issueId,
  i.name AS issueName
ORDER BY articleId;
```

# Step 11 — Verify SOLVES relationship count

```cypher
MATCH (:KnowledgeArticle)-[r:SOLVES]->(:Issue)
RETURN
  type(r) AS relationshipType,
  count(*) AS relationshipCount;
```

# Step 12 — Verify readable KnowledgeArticle -> SOLVES -> Issue paths

```cypher
MATCH path = (ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
RETURN
  path,
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS issueSeverity
ORDER BY articleId;
```

# Step 13 — Create DocumentChunk nodes and connect them to articles

```cypher
UNWIND [
  {
    chunkId: "C-K001-001",
    articleId: "K001",
    chunkOrder: 1,
    text: "If a customer cannot sign in, ask them to reset the password and verify OTP delivery."
  },
  {
    chunkId: "C-K001-002",
    articleId: "K001",
    chunkOrder: 2,
    text: "For login failure, clear the mobile app cache and retry login."
  },
  {
    chunkId: "C-K002-001",
    articleId: "K002",
    chunkOrder: 1,
    text: "If payment fails during checkout, check card status and verify available balance."
  },
  {
    chunkId: "C-K002-002",
    articleId: "K002",
    chunkOrder: 2,
    text: "For payment failure, confirm the payment gateway response and retry the transaction."
  },
  {
    chunkId: "C-K003-001",
    articleId: "K003",
    chunkOrder: 1,
    text: "If the mobile app crashes, ask the customer to update the app and clear cache."
  },
  {
    chunkId: "C-K003-002",
    articleId: "K003",
    chunkOrder: 2,
    text: "For app crash issues, restart the device and check device compatibility."
  }
] AS chunk
MATCH (ka:KnowledgeArticle {articleId: chunk.articleId})
MERGE (dc:DocumentChunk {chunkId: chunk.chunkId})
SET
  dc.text = chunk.text,
  dc.chunkOrder = chunk.chunkOrder,
  dc.articleId = chunk.articleId,
  dc.source = "Day 3 sample document chunks"
MERGE (dc)-[:PART_OF]->(ka)
RETURN
  dc.chunkId AS chunkId,
  dc.articleId AS articleId,
  dc.chunkOrder AS chunkOrder,
  ka.title AS articleTitle
ORDER BY articleId, chunkOrder;
```

# Step 14 — Verify PART_OF relationship count

```cypher
MATCH (:DocumentChunk)-[r:PART_OF]->(:KnowledgeArticle)
RETURN
  type(r) AS relationshipType,
  count(*) AS relationshipCount;
```

# Step 15 — Verify readable DocumentChunk -> PART_OF -> KnowledgeArticle paths

```cypher
MATCH path = (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)
RETURN
  path,
  dc.chunkId AS chunkId,
  dc.chunkOrder AS chunkOrder,
  dc.text AS chunkText,
  ka.articleId AS articleId,
  ka.title AS articleTitle
ORDER BY articleId, chunkOrder;
```

# Step 16 — Check existing vector indexes

```cypher
SHOW VECTOR INDEXES
YIELD name, state, populationPercent, type, entityType, labelsOrTypes, properties, indexProvider
RETURN
  name,
  state,
  populationPercent,
  type,
  entityType,
  labelsOrTypes,
  properties,
  indexProvider
ORDER BY name;
```

# Step 17 — Add simple learning embeddings to DocumentChunk nodes

```cypher
UNWIND [
  {
    chunkId: "C-K001-001",
    embedding: [0.95, 0.10, 0.05]
  },
  {
    chunkId: "C-K001-002",
    embedding: [0.90, 0.15, 0.05]
  },
  {
    chunkId: "C-K002-001",
    embedding: [0.05, 0.95, 0.10]
  },
  {
    chunkId: "C-K002-002",
    embedding: [0.10, 0.90, 0.15]
  },
  {
    chunkId: "C-K003-001",
    embedding: [0.10, 0.10, 0.95]
  },
  {
    chunkId: "C-K003-002",
    embedding: [0.15, 0.10, 0.90]
  }
] AS row
MATCH (dc:DocumentChunk {chunkId: row.chunkId})
SET dc.embedding = row.embedding
RETURN
  dc.chunkId AS chunkId,
  dc.text AS chunkText,
  dc.embedding AS embedding,
  size(dc.embedding) AS embeddingDimension
ORDER BY chunkId;
```

# Step 18 — Verify embedding coverage and dimensions

```cypher
MATCH (dc:DocumentChunk)
RETURN
  count(dc) AS totalChunks,
  count(dc.embedding) AS chunksWithEmbedding,
  collect(DISTINCT size(dc.embedding)) AS embeddingDimensions;
```

# Step 19 — Create vector index on DocumentChunk.embedding

```cypher
CREATE VECTOR INDEX documentChunk_embedding_vector IF NOT EXISTS
FOR (dc:DocumentChunk)
ON dc.embedding
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 3,
    `vector.similarity_function`: 'cosine'
  }
};
```

# Step 20 — Verify vector index status

```cypher
SHOW VECTOR INDEXES
YIELD name, state, populationPercent, type, entityType, labelsOrTypes, properties, indexProvider
WHERE name = "documentChunk_embedding_vector"
RETURN
  name,
  state,
  populationPercent,
  type,
  entityType,
  labelsOrTypes,
  properties,
  indexProvider;
```

# Step 21 — Verify vector index configuration

```cypher
SHOW VECTOR INDEXES
YIELD name, state, options, createStatement
WHERE name = "documentChunk_embedding_vector"
RETURN
  name,
  state,
  options,
  createStatement;
```

# Step 22 — Run first vector similarity search

```cypher
CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  3,
  [0.92, 0.12, 0.05]
)
YIELD node AS dc, score
RETURN
  dc.chunkId AS chunkId,
  dc.text AS chunkText,
  dc.embedding AS embedding,
  score
ORDER BY score DESC;
```

# Step 23 — Hybrid vector search with graph traversal

```cypher
CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  3,
  [0.92, 0.12, 0.05]
)
YIELD node AS dc, score
MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
RETURN
  dc.chunkId AS chunkId,
  dc.text AS chunkText,
  score,
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS issueSeverity
ORDER BY score DESC;
```

# Step 24A — Final Day 3 validation summary

```cypher
MATCH (ka:KnowledgeArticle)
WITH count(ka) AS knowledgeArticleCount

MATCH (dc:DocumentChunk)
WITH
  knowledgeArticleCount,
  count(dc) AS documentChunkCount,
  count(dc.embedding) AS chunksWithEmbedding,
  collect(DISTINCT size(dc.embedding)) AS embeddingDimensions

MATCH (:KnowledgeArticle)-[s:SOLVES]->(:Issue)
WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  count(s) AS solvesRelationshipCount

MATCH (:DocumentChunk)-[p:PART_OF]->(:KnowledgeArticle)
WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  solvesRelationshipCount,
  count(p) AS partOfRelationshipCount

CALL () {
  SHOW VECTOR INDEXES
  YIELD name, state, populationPercent, labelsOrTypes, properties
  WHERE name = "documentChunk_embedding_vector"
  RETURN
    state AS vectorIndexState,
    populationPercent AS vectorIndexPopulation,
    labelsOrTypes AS vectorIndexLabels,
    properties AS vectorIndexProperties
}

RETURN
  knowledgeArticleCount,
  documentChunkCount,
  solvesRelationshipCount,
  partOfRelationshipCount,
  chunksWithEmbedding,
  embeddingDimensions,
  vectorIndexState,
  vectorIndexPopulation,
  vectorIndexLabels,
  vectorIndexProperties;
```

# Step 24B — Final vector index validation summary

```cypher
SHOW VECTOR INDEXES
YIELD name, state, populationPercent, labelsOrTypes, properties
WHERE name = "documentChunk_embedding_vector"
RETURN
  state AS vectorIndexState,
  populationPercent AS vectorIndexPopulation,
  labelsOrTypes AS vectorIndexLabels,
  properties AS vectorIndexProperties;
```

# Addendum A — Step A1: Current knowledge graph audit

```cypher
MATCH (ka:KnowledgeArticle)
WITH count(ka) AS knowledgeArticleCount

MATCH (dc:DocumentChunk)
WITH
  knowledgeArticleCount,
  count(dc) AS documentChunkCount,
  count(dc.embedding) AS chunksWithEmbedding,
  collect(DISTINCT size(dc.embedding)) AS embeddingDimensions

MATCH (:KnowledgeArticle)-[s:SOLVES]->(:Issue)
WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  count(s) AS solvesRelationshipCount

MATCH (:DocumentChunk)-[p:PART_OF]->(:KnowledgeArticle)
RETURN
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  solvesRelationshipCount,
  count(p) AS partOfRelationshipCount;
```

# Addendum A — Step A2: Detect orphan knowledge graph records

```cypher
MATCH (ka:KnowledgeArticle)
WHERE NOT EXISTS {
  MATCH (ka)-[:SOLVES]->(:Issue)
}
RETURN
  "KnowledgeArticle without SOLVES relationship" AS issueType,
  ka.articleId AS recordId,
  ka.title AS recordTitleOrText

UNION ALL

MATCH (dc:DocumentChunk)
WHERE NOT EXISTS {
  MATCH (dc)-[:PART_OF]->(:KnowledgeArticle)
}
RETURN
  "DocumentChunk without PART_OF relationship" AS issueType,
  dc.chunkId AS recordId,
  dc.text AS recordTitleOrText

UNION ALL

MATCH (dc:DocumentChunk)
WHERE dc.embedding IS NULL
RETURN
  "DocumentChunk without embedding" AS issueType,
  dc.chunkId AS recordId,
  dc.text AS recordTitleOrText;
```

# Addendum A — Step A3: Validate Issue-to-KnowledgeArticle coverage

```cypher
MATCH (i:Issue)
OPTIONAL MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)
RETURN
  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS severity,
  count(ka) AS knowledgeArticleCount,
  collect(ka.articleId) AS knowledgeArticleIds,
  collect(ka.title) AS knowledgeArticleTitles
ORDER BY issueId;
```

# Addendum A — Step A4: Validate chunk coverage per article

```cypher
MATCH (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)
RETURN
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  count(dc) AS chunkCount,
  count(dc.embedding) AS chunksWithEmbedding,
  collect(dc.chunkId) AS chunkIds,
  collect(dc.chunkOrder) AS chunkOrders,
  collect(DISTINCT size(dc.embedding)) AS embedding
```

# Addendum A — Step A5: Validate vector index readiness

```cypher
SHOW VECTOR INDEXES
YIELD
  name,
  state,
  populationPercent,
  type,
  entityType,
  labelsOrTypes,
  properties,
  indexProvider,
  options
WHERE name = "documentChunk_embedding_vector"
RETURN
  name,
  state,
  populationPercent,
  type,
  entityType,
  labelsOrTypes,
  properties,
  indexProvider,
  options;
```

# Addendum A — Step A6: Controlled vector retrieval quality test

```cypher
CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  3,
  [0.92, 0.12, 0.05]
)
YIELD node AS dc, score
MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
RETURN
  "Login-style test vector" AS testName,
  dc.chunkId AS chunkId,
  dc.text AS chunkText,
  score,
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  i.issueId AS issueId,
  i.name AS issueName
ORDER BY score DESC;
```

# Addendum A — Step A7: Controlled retrieval tests for Payment Failure and App Crash

```cypher
CALL {
  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    3,
    [0.08, 0.92, 0.12]
  )
  YIELD node AS dc, score
  MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
  RETURN
    "Payment-style test vector" AS testName,
    dc.chunkId AS chunkId,
    dc.text AS chunkText,
    score,
    ka.articleId AS articleId,
    ka.title AS articleTitle,
    i.issueId AS issueId,
    i.name AS issueName

  UNION ALL

  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    3,
    [0.12, 0.10, 0.92]
  )
  YIELD node AS dc, score
  MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
  RETURN
    "App-crash-style test vector" AS testName,
    dc.chunkId AS chunkId,
    dc.text AS chunkText,
    score,
    ka.articleId AS articleId,
    ka.title AS articleTitle,
    i.issueId AS issueId,
    i.name AS issueName
}
RETURN
  testName,
  chunkId,
  chunkText,
  score,
  articleId,
  articleTitle,
  issueId,
  issueName
ORDER BY
  testName,
  score DESC;
```

## Day 3 Addendum A — Production-readiness checks for the knowledge and vector layer

In the main Day 3 lab, we created a basic GraphRAG-ready knowledge layer using:

- `KnowledgeArticle`
- `DocumentChunk`
- `SOLVES`
- `PART_OF`
- `DocumentChunk.embedding`
- `documentChunk_embedding_vector`

This addendum validates that the knowledge and vector retrieval layer is healthy before moving to Day 4.

A production-ready retrieval graph should not only create nodes, relationships, embeddings, and indexes. It should also verify that:

- every article is connected to an issue,
- every chunk belongs to an article,
- every chunk has an embedding,
- all embeddings have the expected dimension,
- the vector index is online,
- the vector index is fully populated,
- controlled test vectors retrieve the expected chunks.

These checks help students move beyond “the command ran successfully” into production-style validation.

### Note about the learning embeddings

The embeddings used in this Day 3 lab are intentionally simple 3-dimensional learning vectors.

They are used so that students can understand vector similarity behaviour clearly:

- login-related chunks are close to vectors near `[1, 0, 0]`
- payment-related chunks are close to vectors near `[0, 1, 0]`
- app-crash-related chunks are close to vectors near `[0, 0, 1]`

In production, embeddings should be generated using a real embedding model. Production text embeddings commonly use dimensions such as 256, 768, 1536, or 3072 depending on the model.

### Note about vector query syntax

This lab uses `db.index.vector.queryNodes()` because it is easy to understand for first-time learners.

In newer Neo4j versions, the Cypher `SEARCH` clause is the preferred production-style way to query vector indexes. A later day in this programme can introduce the `SEARCH` clause version of the same retrieval workflow.

# Addendum B — Step B1: Explainable GraphRAG context for a login-style query

```cypher
CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  2,
  [0.92, 0.12, 0.05]
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

RETURN
  "Login-style explainable retrieval" AS testName,
  dc.chunkId AS chunkId,
  dc.text AS retrievedChunk,
  score AS vectorScore,

  ka.articleId AS articleId,
  ka.title AS articleTitle,

  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS issueSeverity,

  collect(DISTINCT t.ticketId) AS relatedTicketIds,
  collect(DISTINCT c.name) AS relatedCustomers,
  collect(DISTINCT p.name) AS relatedProducts,
  collect(DISTINCT a.name) AS assignedAgents,

  collect(DISTINCT {
    ticketId: t.ticketId,
    priority: t.priority,
    status: t.status,
    fullDegreeScore: t.fullDegreeScore,
    pageRankScore: t.pageRankScore,
    betweennessScore: t.betweennessScore,
    louvainCommunityId: t.louvainCommunityId,
    labelPropagationCommunityId: t.labelPropagationCommunityId
  }) AS ticketAnalyticsContext

ORDER BY vectorScore DESC;
```

# Addendum B — Step B2: Explainable GraphRAG context for Payment Failure and App Crash

```cypher
CALL {
  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    2,
    [0.08, 0.92, 0.12]
  )
  YIELD node AS dc, score

  MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
  OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
  OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
  OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
  OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

  RETURN
    "Payment-style explainable retrieval" AS testName,
    dc.chunkId AS chunkId,
    dc.text AS retrievedChunk,
    score AS vectorScore,
    ka.articleId AS articleId,
    ka.title AS articleTitle,
    i.issueId AS issueId,
    i.name AS issueName,
    i.severity AS issueSeverity,
    collect(DISTINCT t.ticketId) AS relatedTicketIds,
    collect(DISTINCT c.name) AS relatedCustomers,
    collect(DISTINCT p.name) AS relatedProducts,
    collect(DISTINCT a.name) AS assignedAgents,
    collect(DISTINCT {
      ticketId: t.ticketId,
      priority: t.priority,
      status: t.status,
      fullDegreeScore: t.fullDegreeScore,
      pageRankScore: t.pageRankScore,
      betweennessScore: t.betweennessScore,
      louvainCommunityId: t.louvainCommunityId,
      labelPropagationCommunityId: t.labelPropagationCommunityId
    }) AS ticketAnalyticsContext

  UNION ALL

  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    2,
    [0.12, 0.10, 0.92]
  )
  YIELD node AS dc, score

  MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
  OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
  OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
  OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
  OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

  RETURN
    "App-crash-style explainable retrieval" AS testName,
    dc.chunkId AS chunkId,
    dc.text AS retrievedChunk,
    score AS vectorScore,
    ka.articleId AS articleId,
    ka.title AS articleTitle,
    i.issueId AS issueId,
    i.name AS issueName,
    i.severity AS issueSeverity,
    collect(DISTINCT t.ticketId) AS relatedTicketIds,
    collect(DISTINCT c.name) AS relatedCustomers,
    collect(DISTINCT p.name) AS relatedProducts,
    collect(DISTINCT a.name) AS assignedAgents,
    collect(DISTINCT {
      ticketId: t.ticketId,
      priority: t.priority,
      status: t.status,
      fullDegreeScore: t.fullDegreeScore,
      pageRankScore: t.pageRankScore,
      betweennessScore: t.betweennessScore,
      louvainCommunityId: t.louvainCommunityId,
      labelPropagationCommunityId: t.labelPropagationCommunityId
    }) AS ticketAnalyticsContext
}
RETURN
  testName,
  chunkId,
  retrievedChunk,
  vectorScore,
  articleId,
  articleTitle,
  issueId,
  issueName,
  issueSeverity,
  relatedTicketIds,
  relatedCustomers,
  relatedProducts,
  assignedAgents,
  ticketAnalyticsContext
ORDER BY
  testName,
  vectorScore DESC;
```

# Addendum B — Step B3: Clean App Crash explainable output

```cypher
CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  2,
  [0.12, 0.10, 0.92]
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

RETURN
  "App-crash-style cleaned explainable retrieval" AS testName,
  dc.chunkId AS chunkId,
  dc.text AS retrievedChunk,
  score AS vectorScore,

  ka.articleId AS articleId,
  ka.title AS articleTitle,

  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS issueSeverity,

  collect(DISTINCT t.ticketId) AS relatedTicketIds,
  collect(DISTINCT c.name) AS relatedCustomers,
  collect(DISTINCT p.name) AS relatedProducts,
  collect(DISTINCT a.name) AS assignedAgents,

  collect(DISTINCT
    CASE
      WHEN t IS NULL THEN null
      ELSE {
        ticketId: t.ticketId,
        priority: t.priority,
        status: t.status,
        fullDegreeScore: t.fullDegreeScore,
        pageRankScore: t.pageRankScore,
        betweennessScore: t.betweennessScore,
        louvainCommunityId: t.louvainCommunityId,
        labelPropagationCommunityId: t.labelPropagationCommunityId
      }
    END
  ) AS ticketAnalyticsContext,

  CASE
    WHEN count(t) = 0 THEN "No related ticket found for this issue"
    ELSE "Related ticket context found"
  END AS operationalContextStatus

ORDER BY vectorScore DESC;
```

### Why we clean null analytics context

When an `OPTIONAL MATCH` does not find a related ticket, the ticket variable becomes `null`.

If we directly build a map from that null ticket variable, Cypher may return a map where every ticket field is null. This is technically valid, but confusing for students and for downstream applications.

To make the output production-friendly, we use a generic `CASE` expression:

- if `t IS NULL`, return `null`
- otherwise, return the ticket analytics map

When collected, this gives a cleaner result:

- issues with related tickets return ticket analytics objects
- issues without related tickets return an empty analytics list

This makes the output easier to interpret and safer for APIs, dashboards, and GraphRAG responses.

# Addendum B — Step B4: Clean explainable GraphRAG query for all three test vectors

```cypher
CALL {
  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    2,
    [0.92, 0.12, 0.05]
  )
  YIELD node AS dc, score
  RETURN
    "Login-style cleaned explainable retrieval" AS testName,
    dc,
    score

  UNION ALL

  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    2,
    [0.08, 0.92, 0.12]
  )
  YIELD node AS dc, score
  RETURN
    "Payment-style cleaned explainable retrieval" AS testName,
    dc,
    score

  UNION ALL

  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    2,
    [0.12, 0.10, 0.92]
  )
  YIELD node AS dc, score
  RETURN
    "App-crash-style cleaned explainable retrieval" AS testName,
    dc,
    score
}

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

RETURN
  testName,
  dc.chunkId AS chunkId,
  dc.text AS retrievedChunk,
  score AS vectorScore,

  ka.articleId AS articleId,
  ka.title AS articleTitle,

  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS issueSeverity,

  collect(DISTINCT t.ticketId) AS relatedTicketIds,
  collect(DISTINCT c.name) AS relatedCustomers,
  collect(DISTINCT p.name) AS relatedProducts,
  collect(DISTINCT a.name) AS assignedAgents,

  collect(DISTINCT
    CASE
      WHEN t IS NULL THEN null
      ELSE {
        ticketId: t.ticketId,
        priority: t.priority,
        status: t.status,
        fullDegreeScore: t.fullDegreeScore,
        pageRankScore: t.pageRankScore,
        betweennessScore: t.betweennessScore,
        louvainCommunityId: t.louvainCommunityId,
        labelPropagationCommunityId: t.labelPropagationCommunityId
      }
    END
  ) AS ticketAnalyticsContext,

  CASE
    WHEN count(t) = 0 THEN "No related ticket found for this issue"
    ELSE "Related ticket context found"
  END AS operationalContextStatus

ORDER BY
  testName,
  vectorScore DESC;
```

## Day 3 Addendum B — Explainable GraphRAG context query

In this addendum, we combine vector search with graph traversal to create an explainable retrieval workflow.

A basic vector search returns relevant chunks. An explainable GraphRAG query goes further:

1. It retrieves relevant `DocumentChunk` nodes using the vector index.
2. It identifies the parent `KnowledgeArticle`.
3. It identifies the `Issue` solved by that article.
4. It expands into related operational support data:
   - tickets
   - customers
   - products
   - assigned agents
5. It includes Day 2 graph analytics properties on related tickets:
   - `fullDegreeScore`
   - `pageRankScore`
   - `betweennessScore`
   - `louvainCommunityId`
   - `labelPropagationCommunityId`
6. It uses conditional logic to cleanly handle issues that have no related ticket.

This teaches students that GraphRAG is not just semantic search. It is semantic search plus graph-grounded explanation.

### Final insight from the cleaned explainable retrieval query

The cleaned explainable GraphRAG query successfully validates all three knowledge areas:

- Login Failure retrieves login chunks and expands to ticket T001, Asha Sharma, Mobile App, and Rajat Support.
- Payment Failure retrieves payment chunks and expands to ticket T002, Ravi Mehta, Payment Gateway, and Rajat Support.
- App Crash retrieves the correct app-crash knowledge chunks, but does not expand to any ticket, customer, product, or agent.

This confirms that App Crash has knowledge coverage but no operational ticket coverage. The same issue was already detected during Day 2 using graph analytics, and is now confirmed again through the retrieval layer.

# Addendum C — Step C1: Final Day 3 knowledge and operational coverage summary

```cypher
MATCH (ka:KnowledgeArticle)
WITH count(ka) AS knowledgeArticleCount

MATCH (dc:DocumentChunk)
WITH
  knowledgeArticleCount,
  count(dc) AS documentChunkCount,
  count(dc.embedding) AS chunksWithEmbedding,
  collect(DISTINCT size(dc.embedding)) AS embeddingDimensions

MATCH (:KnowledgeArticle)-[s:SOLVES]->(:Issue)
WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  count(s) AS solvesRelationshipCount

MATCH (:DocumentChunk)-[p:PART_OF]->(:KnowledgeArticle)
WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  solvesRelationshipCount,
  count(p) AS partOfRelationshipCount

MATCH (i:Issue)
OPTIONAL MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  solvesRelationshipCount,
  partOfRelationshipCount,
  i,
  count(DISTINCT ka) AS articlesForIssue,
  count(DISTINCT t) AS ticketsForIssue

WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  solvesRelationshipCount,
  partOfRelationshipCount,
  count(i) AS totalIssueCount,
  sum(CASE WHEN articlesForIssue > 0 THEN 1 ELSE 0 END) AS issuesWithKnowledgeCoverage,
  sum(CASE WHEN ticketsForIssue > 0 THEN 1 ELSE 0 END) AS issuesWithOperationalTicketCoverage,
  collect(
    CASE
      WHEN articlesForIssue > 0 AND ticketsForIssue = 0 THEN i.name
      ELSE null
    END
  ) AS knowledgeCoveredButNoTicketIssues

RETURN
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  solvesRelationshipCount,
  partOfRelationshipCount,
  totalIssueCount,
  issuesWithKnowledgeCoverage,
  issuesWithOperationalTicketCoverage,
  [issueName IN knowledgeCoveredButNoTicketIssues WHERE issueName IS NOT NULL] AS knowledgeCoveredButNoTicketIssues;
```

## Final Day 3 completion summary

Day 3 extended the SupportGraph Intelligence Platform from graph analytics into knowledge-aware retrieval.

We added:

- `KnowledgeArticle` nodes
- `DocumentChunk` nodes
- `SOLVES` relationships from articles to issues
- `PART_OF` relationships from chunks to articles
- simple learning embeddings on document chunks
- a vector index on `DocumentChunk.embedding`
- controlled vector retrieval tests
- explainable GraphRAG-style queries
- production-readiness checks

The final validation confirmed:

- 3 knowledge articles
- 6 document chunks
- 6 chunks with embeddings
- consistent embedding dimension `[3]`
- 3 `SOLVES` relationships
- 6 `PART_OF` relationships
- all 3 issues have knowledge coverage
- only 2 issues have operational ticket coverage
- `App Crash` has knowledge coverage but no related ticket

This means the platform can retrieve relevant support knowledge and explain how that knowledge connects to tickets, customers, products, agents, and graph analytics context.

The key Day 3 production insight is:

`App Crash` is not a knowledge gap. It is an operational graph coverage gap.

---
# And this finishes day 3. Now, onto day 4th.
---
## Day 4 story — Build an API-ready GraphRAG Retrieval Service

Day 4 converts the Day 3 explainable GraphRAG query into an API-ready backend workflow.

By the end of Day 3, the graph could retrieve relevant `DocumentChunk` nodes using vector search and expand them into `KnowledgeArticle`, `Issue`, `Ticket`, `Customer`, `Product`, `Agent`, and graph analytics context.

Day 4 takes that working retrieval pattern and prepares it for application use.

The goal is to create a small service layer that can:

- verify service health,
- verify Neo4j connectivity,
- accept controlled retrieval requests,
- run vector search,
- expand results through the graph,
- return clean JSON responses,
- include Day 2 analytics context,
- clearly warn when an issue has knowledge coverage but no operational ticket context.

Day 4 is complete when the API can return clean explainable retrieval results for:

- Login Failure,
- Payment Failure,
- App Crash.

The key production lesson of Day 4 is:

A GraphRAG system is not complete when a Cypher query works in the database. It becomes application-ready when the retrieval logic is exposed through a stable, validated, explainable API response.

## Day 4 — Build an API-ready GraphRAG Retrieval Service

### Step 1 — Define the API response contract before coding

In Day 3, we successfully built an explainable GraphRAG query inside Neo4j. That query was able to:

- retrieve relevant `DocumentChunk` nodes using vector search,
- connect each chunk to its parent `KnowledgeArticle`,
- connect the article to the `Issue` it solves,
- expand into operational support context such as `Ticket`, `Customer`, `Product`, and `Agent`,
- include Day 2 graph analytics properties,
- handle missing ticket context cleanly for `App Crash`.

In Day 4, we will convert that database-level retrieval workflow into an API-ready backend service.

Before writing code, we first define the API response contract.

This is a production-level habit. In real projects, API development should not begin by randomly writing endpoint code. The team should first agree on:

- what the endpoint will be called,
- what input it accepts,
- what output it returns,
- how success is represented,
- how warnings are represented,
- how missing graph context is represented,
- how downstream systems such as dashboards, frontends, or LLM prompts will consume the response.

The official FastAPI documentation explains that a FastAPI application is created from a `FastAPI` app instance, that route decorators such as `@app.get("/")` define path operations, and that a Python dictionary returned from a path operation function is automatically converted to JSON. It also explains that FastAPI automatically generates OpenAPI schema and interactive API documentation for the application. This makes response design an important part of API development, not an afterthought.

Reference: FastAPI First Steps — https://fastapi.tiangolo.com/tutorial/first-steps/

---

### Proposed endpoint for controlled retrieval testing

For Day 4, we will start with a controlled test endpoint:

```text
GET /retrieval/test/{query_type}
```

### Supported query_type values:

login
payment
app_crash

### These values map to the controlled learning vectors used during Day 3:

login     -> [0.92, 0.12, 0.05]
payment   -> [0.08, 0.92, 0.12]
app_crash -> [0.12, 0.10, 0.92]

This keeps Day 4 deterministic and easy to validate.

In a later step, we can add support for raw query vectors or real embedding-model-generated vectors.

### Expected response structure

The API should return a clean JSON response with the following top-level structure:

```json
{
  "queryType": "login",
  "retrievalMode": "controlled_test_vector",
  "topK": 2,
  "retrievalStatus": "success",
  "summary": {
    "resultCount": 2,
    "hasOperationalContext": true,
    "warnings": []
  },
  "results": []
}
```

The response should separate the result into clear sections:

- `chunk`
- `article`
- `issue`
- `operationalContext`
- `analyticsContext`

This structure makes the response easier to consume by dashboards, applications, and future LLM prompts.

### Example response for Login Failure

```json
{
  "queryType": "login",
  "retrievalMode": "controlled_test_vector",
  "topK": 2,
  "retrievalStatus": "success",
  "summary": {
    "resultCount": 2,
    "hasOperationalContext": true,
    "warnings": []
  },
  "results": [
    {
      "chunk": {
        "chunkId": "C-K001-001",
        "text": "If a customer cannot sign in, ask them to reset the password and verify OTP delivery.",
        "vectorScore": 0.9998319745063782
      },
      "article": {
        "articleId": "K001",
        "title": "Fix login failure"
      },
      "issue": {
        "issueId": "I001",
        "name": "Login Failure",
        "severity": "High"
      },
      "operationalContext": {
        "status": "Related ticket context found",
        "ticketIds": ["T001"],
        "customers": ["Asha Sharma"],
        "products": ["Mobile App"],
        "assignedAgents": ["Rajat Support"]
      },
      "analyticsContext": [
        {
          "ticketId": "T001",
          "priority": "High",
          "status": "Open",
          "fullDegreeScore": 4.0,
          "pageRankScore": 0.2775,
          "betweennessScore": 3.0,
          "louvainCommunityId": 1,
          "labelPropagationCommunityId": 1
        }
      ]
    }
  ]
}
```

### Example response for Payment Failure

```json
{
  "queryType": "payment",
  "retrievalMode": "controlled_test_vector",
  "topK": 2,
  "retrievalStatus": "success",
  "summary": {
    "resultCount": 2,
    "hasOperationalContext": true,
    "warnings": []
  },
  "results": [
    {
      "chunk": {
        "chunkId": "C-K002-001",
        "text": "If payment fails during checkout, check card status and verify available balance.",
        "vectorScore": 0.9995620250701904
      },
      "article": {
        "articleId": "K002",
        "title": "Resolve payment failure"
      },
      "issue": {
        "issueId": "I002",
        "name": "Payment Failure",
        "severity": "Medium"
      },
      "operationalContext": {
        "status": "Related ticket context found",
        "ticketIds": ["T002"],
        "customers": ["Ravi Mehta"],
        "products": ["Payment Gateway"],
        "assignedAgents": ["Rajat Support"]
      },
      "analyticsContext": [
        {
          "ticketId": "T002",
          "priority": "Medium",
          "status": "Open",
          "fullDegreeScore": 4.0,
          "pageRankScore": 0.2775,
          "betweennessScore": 3.0,
          "louvainCommunityId": 6,
          "labelPropagationCommunityId": 1
        }
      ]
    }
  ]
}
```

### Example response for App Crash

```json
{
  "queryType": "app_crash",
  "retrievalMode": "controlled_test_vector",
  "topK": 2,
  "retrievalStatus": "success",
  "summary": {
    "resultCount": 2,
    "hasOperationalContext": false,
    "warnings": [
      "No related ticket found for this issue"
    ]
  },
  "results": [
    {
      "chunk": {
        "chunkId": "C-K003-001",
        "text": "If the mobile app crashes, ask the customer to update the app and clear cache.",
        "vectorScore": 0.9998478293418884
      },
      "article": {
        "articleId": "K003",
        "title": "Fix app crash"
      },
      "issue": {
        "issueId": "I003",
        "name": "App Crash",
        "severity": "High"
      },
      "operationalContext": {
        "status": "No related ticket found for this issue",
        "ticketIds": [],
        "customers": [],
        "products": [],
        "assignedAgents": []
      },
      "analyticsContext": [
```
# Step 2 — Verify local Python and FastAPI environment

```bash
echo "----- CURRENT USER -----"
whoami

echo "----- CURRENT DIRECTORY -----"
pwd

echo "----- PYTHON VERSION -----"
python3 --version || true

echo "----- PIP VERSION -----"
python3 -m pip --version || true

echo "----- FASTAPI CLI VERSION -----"
fastapi --version || true

echo "----- PYTHON LOCATION -----"
which python3 || true

echo "----- FASTAPI LOCATION -----"
which fastapi || true
```

## Step 3 — Prepare the Graph RAG API module directory

The existing SupportGraph project root is `/opt/supportgraph`, which was created earlier for the Neo4j project infrastructure.

The project plan defines `06_graph_rag_api` as the module for the Graph RAG API layer.

The official FastAPI virtual environments documentation recommends working inside a project directory and creating the virtual environment inside that project.

Reference:
- Project root from lab guide: `/opt/supportgraph`
- Project plan module: `06_graph_rag_api`
- FastAPI Virtual Environments: https://fastapi.tiangolo.com/virtual-environments/

Run:

```bash
echo "----- ENTER SUPPORTGRAPH ROOT -----"
cd /opt/supportgraph

echo "----- CREATE API MODULE IF NEEDED -----"
mkdir -p 06_graph_rag_api

echo "----- ENTER API MODULE -----"
cd /opt/supportgraph/06_graph_rag_api

echo "----- CONFIRM CURRENT LOCATION -----"
pwd

echo "----- LIST API MODULE CONTENTS -----"
ls -la
```

# Step 3.2 — Create the virtual environment

```bash
echo "----- CURRENT LOCATION BEFORE VENV -----"
pwd

echo "----- CREATE PYTHON VIRTUAL ENVIRONMENT -----"
python3 -m venv .venv

echo "----- VERIFY VENV DIRECTORY -----"
ls -la .venv

echo "----- VERIFY VENV PYTHON EXISTS -----"
ls -la .venv/bin/python .venv/bin/python3 2>/dev/null || true
```