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

## Day 4 — Graph RAG Retrieval Pipeline

# Step 1 — Re-run the final Day 3 explainable retrieval query as the Day 4 baseline

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

# Step 2 — Add a controlled natural-language question for Login Failure

```cypher
WITH
  "Why can't customers log in?" AS userQuestion,
  "login" AS queryType,
  [0.92, 0.12, 0.05] AS queryVector,
  2 AS topK

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  topK,
  queryVector
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

RETURN
  userQuestion,
  queryType,
  queryVector,
  topK,

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
  vectorScore DESC;
```

# Step 3 — Add controlled questions for all three retrieval scenarios

```cypher
UNWIND [
  {
    queryType: "login",
    userQuestion: "Why can't customers log in?",
    queryVector: [0.92, 0.12, 0.05],
    topK: 2
  },
  {
    queryType: "payment",
    userQuestion: "Why are payments failing?",
    queryVector: [0.08, 0.92, 0.12],
    topK: 2
  },
  {
    queryType: "app_crash",
    userQuestion: "Why does the app crash?",
    queryVector: [0.12, 0.10, 0.92],
    topK: 2
  }
] AS request

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

RETURN
  request.queryType AS queryType,
  request.userQuestion AS userQuestion,
  request.queryVector AS queryVector,
  request.topK AS topK,

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
  queryType,
  vectorScore DESC;
```

# Step 4 — Add a GDS-aware retrieval ranking score

```cypher
UNWIND [
  {
    queryType: "login",
    userQuestion: "Why can't customers log in?",
    queryVector: [0.92, 0.12, 0.05],
    topK: 2
  },
  {
    queryType: "payment",
    userQuestion: "Why are payments failing?",
    queryVector: [0.08, 0.92, 0.12],
    topK: 2
  },
  {
    queryType: "app_crash",
    userQuestion: "Why does the app crash?",
    queryVector: [0.12, 0.10, 0.92],
    topK: 2
  }
] AS request

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

WITH
  request,
  dc,
  score,
  ka,
  i,
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
  count(t) AS ticketCount,
  max(t.pageRankScore) AS maxPageRankScore,
  max(t.fullDegreeScore) AS maxFullDegreeScore,
  max(t.betweennessScore) AS maxBetweennessScore

WITH
  request,
  dc,
  score,
  ka,
  i,
  relatedTicketIds,
  relatedCustomers,
  relatedProducts,
  assignedAgents,
  ticketAnalyticsContext,
  ticketCount,
  coalesce(maxPageRankScore, 0.0) AS pageRankBoost,
  coalesce(maxFullDegreeScore, 0.0) * 0.01 AS degreeBoost,
  coalesce(maxBetweennessScore, 0.0) * 0.01 AS betweennessBoost

RETURN
  request.queryType AS queryType,
  request.userQuestion AS userQuestion,

  dc.chunkId AS chunkId,
  dc.text AS retrievedChunk,
  score AS vectorScore,

  pageRankBoost,
  degreeBoost,
  betweennessBoost,

  score + pageRankBoost + degreeBoost + betweennessBoost AS retrievalRankScore,

  ka.articleId AS articleId,
  ka.title AS articleTitle,

  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS issueSeverity,

  relatedTicketIds,
  relatedCustomers,
  relatedProducts,
  assignedAgents,
  ticketAnalyticsContext,

  CASE
    WHEN ticketCount = 0 THEN "No related ticket found for this issue"
    ELSE "Related ticket context found"
  END AS operationalContextStatus

ORDER BY
  queryType,
  retrievalRankScore DESC,
  vectorScore DESC;
```

# Step 5 — Assemble retrieved results into grouped subgraph context

```cypher
UNWIND [
  {
    queryType: "login",
    userQuestion: "Why can't customers log in?",
    queryVector: [0.92, 0.12, 0.05],
    topK: 2
  },
  {
    queryType: "payment",
    userQuestion: "Why are payments failing?",
    queryVector: [0.08, 0.92, 0.12],
    topK: 2
  },
  {
    queryType: "app_crash",
    userQuestion: "Why does the app crash?",
    queryVector: [0.12, 0.10, 0.92],
    topK: 2
  }
] AS request

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

WITH
  request,
  dc,
  score,
  ka,
  i,
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
  count(t) AS ticketCount,
  coalesce(max(t.pageRankScore), 0.0) AS pageRankBoost,
  coalesce(max(t.fullDegreeScore), 0.0) * 0.01 AS degreeBoost,
  coalesce(max(t.betweennessScore), 0.0) * 0.01 AS betweennessBoost

WITH
  request,
  dc,
  score,
  ka,
  i,
  relatedTicketIds,
  relatedCustomers,
  relatedProducts,
  assignedAgents,
  ticketAnalyticsContext,
  ticketCount,
  pageRankBoost,
  degreeBoost,
  betweennessBoost,
  score + pageRankBoost + degreeBoost + betweennessBoost AS retrievalRankScore

WITH
  request,
  collect({
    chunkId: dc.chunkId,
    text: dc.text,
    vectorScore: score,
    pageRankBoost: pageRankBoost,
    degreeBoost: degreeBoost,
    betweennessBoost: betweennessBoost,
    retrievalRankScore: retrievalRankScore,
    article: {
      articleId: ka.articleId,
      title: ka.title
    },
    issue: {
      issueId: i.issueId,
      name: i.name,
      severity: i.severity
    },
    operationalContext: {
      status: CASE
        WHEN ticketCount = 0 THEN "No related ticket found for this issue"
        ELSE "Related ticket context found"
      END,
      ticketIds: relatedTicketIds,
      customers: relatedCustomers,
      products: relatedProducts,
      assignedAgents: assignedAgents
    },
    analyticsContext: ticketAnalyticsContext
  }) AS retrievedContext

RETURN
  request.queryType AS queryType,
  request.userQuestion AS userQuestion,
  request.topK AS top
```

> this output means Step 5 is not validated yet.


> You got 3 rows, which is the right number of grouped questions, but the output only shows:
> 
> queryType
> userQuestion
> top


> Expected Step 5 output should include at least:
> 
> queryType
> userQuestion
> topK
> retrievedContext
> retrievedContextCount

> So we need to run a smaller, diagnostic version of Step 5. This is production-grade troubleshooting: instead of assuming the large nested output worked, we validate the assembled context with a compact result.

# Step 5 Retry — Validate grouped subgraph context assembly

```cypher
UNWIND [
  {
    queryType: "login",
    userQuestion: "Why can't customers log in?",
    queryVector: [0.92, 0.12, 0.05],
    topK: 2
  },
  {
    queryType: "payment",
    userQuestion: "Why are payments failing?",
    queryVector: [0.08, 0.92, 0.12],
    topK: 2
  },
  {
    queryType: "app_crash",
    userQuestion: "Why does the app crash?",
    queryVector: [0.12, 0.10, 0.92],
    topK: 2
  }
] AS request

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

WITH
  request,
  dc,
  score,
  ka,
  i,
  collect(DISTINCT t.ticketId) AS relatedTicketIds,
  collect(DISTINCT c.name) AS relatedCustomers,
  collect(DISTINCT p.name) AS relatedProducts,
  collect(DISTINCT a.name) AS assignedAgents,
  count(t) AS ticketCount,
  coalesce(max(t.pageRankScore), 0.0) AS pageRankBoost,
  coalesce(max(t.fullDegreeScore), 0.0) * 0.01 AS degreeBoost,
  coalesce(max(t.betweennessScore), 0.0) * 0.01 AS betweennessBoost

WITH
  request,
  {
    chunkId: dc.chunkId,
    articleId: ka.articleId,
    issueName: i.name,
    vectorScore: score,
    retrievalRankScore: score + pageRankBoost + degreeBoost + betweennessBoost,
    ticketIds: relatedTicketIds,
    customers: relatedCustomers,
    products: relatedProducts,
    assignedAgents: assignedAgents,
    operationalStatus: CASE
      WHEN ticketCount = 0 THEN "No related ticket found for this issue"
      ELSE "Related ticket context found"
    END
  } AS contextItem

WITH
  request,
  collect(contextItem) AS retrievedContext

RETURN
  request.queryType AS queryType,
  request.userQuestion AS userQuestion,
  request.topK AS topK,
  size(retrievedContext) AS retrievedContextCount,
  [item IN retrievedContext | item.chunkId] AS retrievedChunkIds,
  [item IN retrievedContext | item.issueName] AS retrievedIssueNames,
  [item IN retrievedContext | item.ticketIds] AS retrievedTicketIds,
  [item IN retrievedContext | item.operationalStatus] AS operationalStatuses,
  [item IN retrievedContext | item.retrievalRankScore] AS retrievalRankScores

ORDER BY
  queryType;
```

# Step 6 — Format grouped context as LLM-ready context

```cypher
UNWIND [
  {
    queryType: "login",
    userQuestion: "Why can't customers log in?",
    queryVector: [0.92, 0.12, 0.05],
    topK: 2
  },
  {
    queryType: "payment",
    userQuestion: "Why are payments failing?",
    queryVector: [0.08, 0.92, 0.12],
    topK: 2
  },
  {
    queryType: "app_crash",
    userQuestion: "Why does the app crash?",
    queryVector: [0.12, 0.10, 0.92],
    topK: 2
  }
] AS request

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

WITH
  request,
  dc,
  score,
  ka,
  i,
  collect(DISTINCT t.ticketId) AS relatedTicketIds,
  collect(DISTINCT c.name) AS relatedCustomers,
  collect(DISTINCT p.name) AS relatedProducts,
  collect(DISTINCT a.name) AS assignedAgents,
  count(t) AS ticketCount,
  coalesce(max(t.pageRankScore), 0.0) AS pageRankBoost,
  coalesce(max(t.fullDegreeScore), 0.0) * 0.01 AS degreeBoost,
  coalesce(max(t.betweennessScore), 0.0) * 0.01 AS betweennessBoost

WITH
  request,
  {
    chunkId: dc.chunkId,
    text: dc.text,
    vectorScore: score,
    retrievalRankScore: score + pageRankBoost + degreeBoost + betweennessBoost,
    article: {
      articleId: ka.articleId,
      title: ka.title
    },
    issue: {
      issueId: i.issueId,
      name: i.name,
      severity: i.severity
    },
    operationalContext: {
      status: CASE
        WHEN ticketCount = 0 THEN "No related ticket found for this issue"
        ELSE "Related ticket context found"
      END,
      ticketIds: relatedTicketIds,
      customers: relatedCustomers,
      products: relatedProducts,
      assignedAgents: assignedAgents
    },
    rankingSignals: {
      vectorScore: score,
      pageRankBoost: pageRankBoost,
      degreeBoost: degreeBoost,
      betweennessBoost: betweennessBoost,
      retrievalRankScore: score + pageRankBoost + degreeBoost + betweennessBoost
    }
  } AS contextItem

WITH
  request,
  collect(contextItem) AS retrievedContext

WITH
  request,
  retrievedContext,
  [item IN retrievedContext | {
    chunkId: item.chunkId,
    text: item.text,
    vectorScore: item.vectorScore,
    retrievalRankScore: item.retrievalRankScore
  }] AS retrievedEvidence,
  [item IN retrievedContext | item.issue] AS issueContext,
  [item IN retrievedContext | item.operationalContext] AS operationalContext,
  [item IN retrievedContext | item.rankingSignals] AS rankingSignals,
  [item IN retrievedContext
    WHERE item.operationalContext.status = "No related ticket found for this issue"
    | item.operationalContext.status
  ] AS warnings

RETURN
  request.queryType AS queryType,
  request.userQuestion AS userQuestion,
  {
    question: request.userQuestion,
    retrievedEvidence: retrievedEvidence,
    issueContext: issueContext,
    operationalContext: operationalContext,
    rankingSignals: rankingSignals,
    warnings: warnings,
    instructionForDay5LLM: "Use only the provided retrieved evidence and graph context. If warnings exist, mention the data-quality limitation."
  } AS llmReadyContext,
  size(retrievedContext) AS retrievedContextCount,
  size(warnings) AS warningCount

ORDER BY
  queryType;
```

# Step 6B — Deduplicate repeated LLM-ready context fields

```cypher
UNWIND [
  {
    queryType: "login",
    userQuestion: "Why can't customers log in?",
    queryVector: [0.92, 0.12, 0.05],
    topK: 2
  },
  {
    queryType: "payment",
    userQuestion: "Why are payments failing?",
    queryVector: [0.08, 0.92, 0.12],
    topK: 2
  },
  {
    queryType: "app_crash",
    userQuestion: "Why does the app crash?",
    queryVector: [0.12, 0.10, 0.92],
    topK: 2
  }
] AS request

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

WITH
  request,
  dc,
  score,
  ka,
  i,
  collect(DISTINCT t.ticketId) AS relatedTicketIds,
  collect(DISTINCT c.name) AS relatedCustomers,
  collect(DISTINCT p.name) AS relatedProducts,
  collect(DISTINCT a.name) AS assignedAgents,
  count(t) AS ticketCount,
  coalesce(max(t.pageRankScore), 0.0) AS pageRankBoost,
  coalesce(max(t.fullDegreeScore), 0.0) * 0.01 AS degreeBoost,
  coalesce(max(t.betweennessScore), 0.0) * 0.01 AS betweennessBoost

WITH
  request,
  {
    chunkId: dc.chunkId,
    text: dc.text,
    vectorScore: score,
    retrievalRankScore: score + pageRankBoost + degreeBoost + betweennessBoost,
    article: {
      articleId: ka.articleId,
      title: ka.title
    },
    issue: {
      issueId: i.issueId,
      name: i.name,
      severity: i.severity
    },
    operationalContext: {
      status: CASE
        WHEN ticketCount = 0 THEN "No related ticket found for this issue"
        ELSE "Related ticket context found"
      END,
      ticketIds: relatedTicketIds,
      customers: relatedCustomers,
      products: relatedProducts,
      assignedAgents: assignedAgents
    },
    rankingSignals: {
      vectorScore: score,
      pageRankBoost: pageRankBoost,
      degreeBoost: degreeBoost,
      betweennessBoost: betweennessBoost,
      retrievalRankScore: score + pageRankBoost + degreeBoost + betweennessBoost
    }
  } AS contextItem

WITH
  request,
  collect(contextItem) AS retrievedContext,
  collect(DISTINCT contextItem.issue) AS dedupedIssueContext,
  collect(DISTINCT contextItem.operationalContext) AS dedupedOperationalContext,
  collect(DISTINCT
    CASE
      WHEN contextItem.operationalContext.status = "No related ticket found for this issue"
      THEN contextItem.operationalContext.status
      ELSE null
    END
  ) AS dedupedWarnings

RETURN
  request.queryType AS queryType,
  request.userQuestion AS userQuestion,
  {
    question: request.userQuestion,

    retrievedEvidence: [
      item IN retrievedContext | {
        chunkId: item.chunkId,
        text: item.text,
        vectorScore: item.vectorScore,
        retrievalRankScore: item.retrievalRankScore
      }
    ],

    issueContext: dedupedIssueContext,

    operationalContext: dedupedOperationalContext,

    rankingSignals: [
      item IN retrievedContext | item.rankingSignals
    ],

    warnings: dedupedWarnings,

    instructionForDay5LLM: "Use only the provided retrieved evidence and graph context. If warnings exist, mention the data-quality limitation."
  } AS llmReadyContext,

  size(retrievedContext) AS retrievedEvidenceCount,
  size(dedupedIssueContext) AS issueContextCount,
  size(dedupedOperationalContext) AS operationalContextCount,
  size(dedupedWarnings) AS warningCount

ORDER BY
  queryType;
```

# Step 7 — Final Day 4 retrieval validation summary

```cypher
UNWIND [
  {
    queryType: "login",
    userQuestion: "Why can't customers log in?",
    queryVector: [0.92, 0.12, 0.05],
    topK: 2
  },
  {
    queryType: "payment",
    userQuestion: "Why are payments failing?",
    queryVector: [0.08, 0.92, 0.12],
    topK: 2
  },
  {
    queryType: "app_crash",
    userQuestion: "Why does the app crash?",
    queryVector: [0.12, 0.10, 0.92],
    topK: 2
  }
] AS request

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)
YIELD node AS dc, score

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

WITH
  request,
  dc,
  score,
  ka,
  i,
  collect(DISTINCT t.ticketId) AS relatedTicketIds,
  collect(DISTINCT c.name) AS relatedCustomers,
  collect(DISTINCT p.name) AS relatedProducts,
  collect(DISTINCT a.name) AS assignedAgents,
  count(t) AS ticketCount,
  coalesce(max(t.pageRankScore), 0.0) AS pageRankBoost,
  coalesce(max(t.fullDegreeScore), 0.0) * 0.01 AS degreeBoost,
  coalesce(max(t.betweennessScore), 0.0) * 0.01 AS betweennessBoost

WITH
  request,
  {
    chunkId: dc.chunkId,
    text: dc.text,
    vectorScore: score,
    retrievalRankScore: score + pageRankBoost + degreeBoost + betweennessBoost,
    issue: {
      issueId: i.issueId,
      name: i.name,
      severity: i.severity
    },
    operationalContext: {
      status: CASE
        WHEN ticketCount = 0 THEN "No related ticket found for this issue"
        ELSE "Related ticket context found"
      END,
      ticketIds: relatedTicketIds,
      customers: relatedCustomers,
      products: relatedProducts,
      assignedAgents: assignedAgents
    },
    rankingSignals: {
      vectorScore: score,
      pageRankBoost: pageRankBoost,
      degreeBoost: degreeBoost,
      betweennessBoost: betweennessBoost,
      retrievalRankScore: score + pageRankBoost + degreeBoost + betweennessBoost
    }
  } AS contextItem

WITH
  request,
  collect(contextItem) AS retrievedContext,
  collect(DISTINCT contextItem.issue) AS dedupedIssueContext,
  collect(DISTINCT contextItem.operationalContext) AS dedupedOperationalContext,
  collect(DISTINCT
    CASE
      WHEN contextItem.operationalContext.status = "No related ticket found for this issue"
      THEN contextItem.operationalContext.status
      ELSE null
    END
  ) AS dedupedWarnings

WITH
  request,
  retrievedContext,
  dedupedIssueContext,
  dedupedOperationalContext,
  dedupedWarnings,
  CASE
    WHEN size(dedupedWarnings) > 0 THEN false
    ELSE true
  END AS hasOperationalContext

WITH
  collect({
    queryType: request.queryType,
    userQuestion: request.userQuestion,
    retrievedEvidenceCount: size(retrievedContext),
    issueContextCount: size(dedupedIssueContext),
    operationalContextCount: size(dedupedOperationalContext),
    warningCount: size(dedupedWarnings),
    hasOperationalContext: hasOperationalContext,
    warnings: dedupedWarnings,
    maxRetrievalRankScore: reduce(
      maxScore = 0.0,
      item IN retrievedContext |
      CASE
        WHEN item.retrievalRankScore > maxScore THEN item.retrievalRankScore
        ELSE maxScore
      END
    )
  }) AS questionSummaries

RETURN
  size(questionSummaries) AS totalQuestionsTested,

  size([
    q IN questionSummaries
    WHERE q.retrievedEvidenceCount > 0
  ]) AS questionsWithRetrievedEvidence,

  size([
    q IN questionSummaries
    WHERE q.issueContextCount > 0
  ]) AS questionsWithIssueContext,

  size([
    q IN questionSummaries
    WHERE q.hasOperationalContext = true
  ]) AS questionsWithOperationalContext,

  size([
    q IN questionSummaries
    WHERE q.warningCount > 0
  ]) AS questionsWithWarnings,

  [q IN questionSummaries WHERE q.warningCount > 0 | q.queryType] AS queryTypesWithWarnings,

  questionSummaries AS day4RetrievalValidationSummary;
```

# This finishes Day 4. Now, onto Day 5

# Step 1: LMStudio install

You can download LMStudio using https://lmstudio.ai/download

The file will get downloaded by default in /home/student/Downloads/LM-Studio-0.4.15-2-x64.deb if default firefox settings are being used. If not, then check where the firefox is downloading.

# Step 2 — Verify the downloaded .deb installer file

```shell
ls -lh /home/student/Downloads/LM-Studio-0.4.15-2-x64.deb && dpkg-deb -I /home/student/Downloads/LM-Studio-0.4.15-2-x64.deb | head -40
```

# Step 3 — Verify Ubuntu machine architecture

```shell
dpkg --print-architecture
```

# Step 4 — Install LM Studio using APT

```shell
apt install /home/student/Downloads/LM-Studio-0.4.15-2-x64.deb
```

# Step 5 — Verify LM Studio installation and available commands

```shell
dpkg -l | grep lm-studio
```

# Step 6 — Check LM Studio local server status

```shell
lms server status
```

# Step 7 — Inspect installed LM Studio files and look for CLI binaries

```shell
dpkg -L lm-studio | grep -E '/bin/|/opt/|lms|lm-studio' | head -120
```

# Step 8 — Search the installed LM Studio package for actual executable names

```shell
dpkg -L lm-studio | grep -E '(^/usr/bin/|^/opt/LM-Studio/(lm-studio|lms)$|/lms$|/lm-studio$)'
```

# Step 9 — Search the installed package for lms and llmster

```shell
dpkg -L lm-studio | grep -Ei '(^/usr/bin/|/lms$|/lms[^/]*$|/llmster$|/llmster[^/]*$|cli|bin|lm-studio$)'
```

# Step 11 — Open LM Studio from the graphical desktop

```shell
lm-studio
```

# Step 12 — Verify that lms is now available from the CLI path

Open a normal terminal as the GUI user, not a root terminal.

```shell
which lms && lms ls
```

# Step 13 — Load the local LLM into memory

```shell
lms load nvidia/nemotron-3-nano-4b
```

# Step 14 — Start the LM Studio local HTTP server

```bash
lms server start
```

# Step 15 — Confirm LM Studio server status

```bash
lms server status
```

# Step 16 — Verify that the LM Studio HTTP server is reachable

```bash
curl -i http://localhost:1234
```

# Step 17 — Verify OpenAI-compatible model listing endpoint

```bash
curl http://localhost:1234/v1/models
```

# Step 18 — Test a minimal OpenAI-compatible chat completion

```bash
curl http://localhost:1234/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/nemotron-3-nano-4b",
    "messages": [
      {
        "role": "system",
        "content": "You are a concise assistant for a classroom lab validation."
      },
      {
        "role": "user",
        "content": "Reply with exactly one sentence confirming that the local LM Studio model is working."
      }
    ],
    "temperature": 0.2,
    "max_tokens": 80
  }'
```

# Step 19 — Check whether a Python virtual environment already exists

```bash
cd /opt/supportgraph/06_graph_rag_api && pwd && ls -la && find . -maxdepth 2 -type f -path "*/bin/activate" -print
```

# Step 19A — Check whether Python and pip are available

```bash
python3 --version && python3 -m pip --version
```

# Step 19B — Install Python runtime and pip prerequisites

```bash
# If running as root
apt install -y python3-full python3-pip python3-pip-whl

# If running as student
sudo apt install -y python3-full python3-pip python3-pip-whl
```

# Step 19C — Verify Python and pip after installation

```shell
python3 --version
python3 -m pip --version
```

# Step 20 — Create the Day 5 GraphRAG API workspace

```shell
sudo mkdir -p /opt/supportgraph/06_graph_rag_api && \
sudo chown -R student:student /opt/supportgraph/06_graph_rag_api && \
ls -ld /opt/supportgraph/06_graph_rag_api
```

# Step 21 — Create a fresh Python virtual environment

```shell
cd /opt/supportgraph/06_graph_rag_api && \
python3 -m venv .venv && \
ls -ld .venv && \
ls -l .venv/bin/activate
```

# Step 22 — Activate the fresh virtual environment and verify Python/pip

```shell
cd /opt/supportgraph/06_graph_rag_api && \
source .venv/bin/activate && \
which python && \
python --version && \
python -m pip --version
```

# Step 23 — Check whether the OpenAI Python package is already installed

Please run this while your prompt starts with (.venv):

```shell
python -m pip show openai
```

# Step 24 — Install the OpenAI Python SDK inside .venv

```shell
python -m pip install openai
```

# Step 25 — Verify the OpenAI Python SDK installation

```shell
python -m pip show openai
```

# Step 26 — Create a Python smoke-test script for LM Studio

```shell
cd /opt/supportgraph/06_graph_rag_api && \
tee test_lmstudio_chat.py
```

After running it, paste the following Python code into the terminal:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

response = client.chat.completions.create(
    model="nvidia/nemotron-3-nano-4b",
    messages=[
        {
            "role": "system",
            "content": "You are a concise assistant validating a local GraphRAG classroom lab."
        },
        {
            "role": "user",
            "content": "Reply with one sentence confirming that Python can call the local LM Studio model."
        }
    ],
    temperature=0.2,
    max_tokens=80
)

print(response.choices[0].message.content.strip())
```

Then finish the input with Ctrl+D.

# Step 27 — Run the Python LM Studio smoke test

```shell
python test_lmstudio_chat.py
```

# Step 28 — Capture Python dependencies in requirements.txt

```shell
cd /opt/supportgraph/06_graph_rag_api && \
python -m pip freeze > requirements.txt && \
cat requirements.txt
```

# Step 29 — Check whether the Neo4j Python Driver is installed

```shell
python -m pip show neo4j
```

# Step 30 — Install the Neo4j Python Driver inside .venv

```shell
python -m pip install neo4j
```

# Step 31 — Verify the Neo4j Python Driver installation

```shell
python -m pip show neo4j
```

# Step 32 — Create a Neo4j Python connection smoke-test script

```shell
cd /opt/supportgraph/06_graph_rag_api && \
tee test_neo4j_connection.py
```

Now paste this Python code:

```python
from neo4j import GraphDatabase

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
    records, summary, keys = driver.execute_query(
        "RETURN 'SupportGraph Neo4j Python connection is working' AS message",
        database_=NEO4J_DATABASE
    )

    for record in records:
        print(record["message"])
```

Then finish input with: Ctrl+D

# Step 33 — Run the Neo4j Python connection smoke test

```shell
python test_neo4j_connection.py
```

# Step 34 — Update requirements.txt after installing Neo4j Driver

```shell
cd /opt/supportgraph/06_graph_rag_api && \
python -m pip freeze > requirements.txt && \
cat requirements.txt
```

# Step 35 — Create a SupportGraph data read smoke-test script

```shell
cd /opt/supportgraph/06_graph_rag_api && \
tee test_supportgraph_read.py
```

Now paste this Python code:

```python
from neo4j import GraphDatabase

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

QUERY = """
MATCH (n)
RETURN labels(n) AS labels, count(n) AS nodeCount
ORDER BY labels
"""

with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
    records, summary, keys = driver.execute_query(
        QUERY,
        database_=NEO4J_DATABASE
    )

    print("SupportGraph node counts:")
    for record in records:
        print(f"{record['labels']}: {record['nodeCount']}")
```

Then press: Ctrl+D

# Step 36 — Run the SupportGraph data read smoke test

```shell
python test_supportgraph_read.py
```

# Step 37 — Create a GraphRAG readiness read script

```shell
cd /opt/supportgraph/06_graph_rag_api && \
tee test_graphrag_readiness.py
```

Now paste this Python code:

```python
from neo4j import GraphDatabase

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

QUERY = """
MATCH (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)
RETURN
    dc.chunkId AS chunkId,
    dc.text AS chunkText,
    ka.articleId AS articleId,
    ka.title AS articleTitle,
    i.issueId AS issueId,
    i.name AS issueName,
    i.severity AS issueSeverity,
    collect(DISTINCT t.ticketId) AS relatedTicketIds,
    collect(DISTINCT c.name) AS relatedCustomers,
    collect(DISTINCT p.name) AS relatedProducts,
    collect(DISTINCT a.name) AS assignedAgents
ORDER BY
    issueName,
    chunkId
"""

with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
    records, summary, keys = driver.execute_query(
        QUERY,
        database_=NEO4J_DATABASE
    )

    print("GraphRAG readiness context:")
    for record in records:
        print("-" * 80)
        print(f"Chunk: {record['chunkId']}")
        print(f"Text: {record['chunkText']}")
        print(f"Article: {record['articleId']} - {record['articleTitle']}")
        print(f"Issue: {record['issueId']} - {record['issueName']} ({record['issueSeverity']})")
        print(f"Tickets: {record['relatedTicketIds']}")
        print(f"Customers: {record['relatedCustomers']}")
        print(f"Products: {record['relatedProducts']}")
        print(f"Assigned agents: {record['assignedAgents']}")
```

Then press: Ctrl+D

# Step 38 — Run the GraphRAG readiness read script

```shell
python test_graphrag_readiness.py
```

# Step 39 — Create the first dynamic retrieval planner script

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee retrieval_planner.py
```

Now paste this Python code exactly:

```python
def classify_question(question: str) -> str:
    q = question.lower()

    if any(word in q for word in ["customer", "customer id", "summary", "customer summary"]):
        return "customer_summary"

    if any(word in q for word in ["product", "mobile app", "payment gateway", "issues affecting", "top product issues"]):
        return "product_issue_analysis"

    if any(word in q for word in ["agent", "assigned", "workload", "handled by", "support engineer"]):
        return "agent_workload"

    if any(word in q for word in ["recommend", "what should we do", "what should agents do", "knowledge article", "fix", "resolve"]):
        return "knowledge_recommendation"

    if any(word in q for word in ["login", "payment", "app crash", "issue", "failure", "problem", "error", "why are"]):
        return "issue_troubleshooting"

    return "generic_support_question"


if __name__ == "__main__":
    examples = [
        "Why are customers facing login failure?",
        "Summarise customer Asha Sharma's support situation.",
        "What are the main issues affecting the Mobile App?",
        "Which issues is Rajat Support handling most often?",
        "What should we recommend for app crash issues?",
        "Help me understand the current support situation."
    ]

    print("Dynamic retrieval planner classification:")
    for question in examples:
        category = classify_question(question)
        print(f"{question} -> {category}")
```

Then press: Ctrl+D

# Step 40 — Run the dynamic retrieval planner

```bash
python retrieval_planner.py
```

# Step 41 — Refine the dynamic retrieval planner rules

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee retrieval_planner.py
```

Now paste this updated Python code exactly:

```python
def classify_question(question: str) -> str:
    q = question.lower()

    # Agent/workload questions should be checked early
    if any(phrase in q for phrase in [
        "handled by",
        "handling most often",
        "which issues is",
        "agent",
        "assigned",
        "workload",
        "support engineer",
        "support agent"
    ]):
        return "agent_workload"

    # Knowledge recommendation questions should be checked before generic issue checks
    if any(phrase in q for phrase in [
        "what should we recommend",
        "what should we do",
        "what should agents do",
        "knowledge article",
        "fix",
        "resolve",
        "recommendation"
    ]):
        return "knowledge_recommendation"

    # Customer summary should require clear summary-style intent
    if (
        "customer id" in q
        or (
            "customer" in q
            and any(word in q for word in [
                "summary",
                "summarise",
                "summarize",
                "situation",
                "history",
                "profile"
            ])
        )
    ):
        return "customer_summary"

    # Product issue questions should be checked before generic issue troubleshooting
    if any(phrase in q for phrase in [
        "product",
        "mobile app",
        "payment gateway",
        "issues affecting",
        "top product issues"
    ]):
        return "product_issue_analysis"

    # Issue troubleshooting is the generic issue-focused path
    if any(phrase in q for phrase in [
        "login",
        "payment",
        "app crash",
        "issue",
        "failure",
        "problem",
        "error",
        "why are",
        "why is",
        "why do"
    ]):
        return "issue_troubleshooting"

    return "generic_support_question"


if __name__ == "__main__":
    examples = [
        "Why are customers facing login failure?",
        "Summarise customer Asha Sharma's support situation.",
        "What are the main issues affecting the Mobile App?",
        "Which issues is Rajat Support handling most often?",
        "What should we recommend for app crash issues?",
        "Help me understand the current support situation."
    ]

    print("Dynamic retrieval planner classification:")
    for question in examples:
        category = classify_question(question)
        print(f"{question} -> {category}")
```

Then press: Ctrl+D

# Step 41A — Verify the current retrieval planner file

```bash
cd /opt/supportgraph/06_graph_rag_api && \
cat -n retrieval_planner.py
```

# Step 41B — Overwrite retrieval_planner.py with the refined rule order

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee retrieval_planner.py
```

Now paste this exact content:

```python
def classify_question(question: str) -> str:
    q = question.lower()

    if any(phrase in q for phrase in [
        "handled by",
        "handling most often",
        "which issues is",
        "agent",
        "assigned",
        "workload",
        "support engineer",
        "support agent",
        "rajat support"
    ]):
        return "agent_workload"

    elif any(phrase in q for phrase in [
        "what should we recommend",
        "what should we do",
        "what should agents do",
        "knowledge article",
        "fix",
        "resolve",
        "recommendation"
    ]):
        return "knowledge_recommendation"

    elif (
        "customer id" in q
        or (
            "customer" in q
            and any(word in q for word in [
                "summary",
                "summarise",
                "summarize",
                "situation",
                "history",
                "profile"
            ])
        )
    ):
        return "customer_summary"

    elif any(phrase in q for phrase in [
        "product",
        "mobile app",
        "payment gateway",
        "issues affecting",
        "top product issues"
    ]):
        return "product_issue_analysis"

    elif any(phrase in q for phrase in [
        "login",
        "payment",
        "app crash",
        "issue",
        "failure",
        "problem",
        "error",
        "why are",
        "why is",
        "why do"
    ]):
        return "issue_troubleshooting"

    else:
        return "generic_support_question"


if __name__ == "__main__":
    examples = [
        "Why are customers facing login failure?",
        "Summarise customer Asha Sharma's support situation.",
        "What are the main issues affecting the Mobile App?",
        "Which issues is Rajat Support handling most often?",
        "What should we recommend for app crash issues?",
        "Help me understand the current support situation."
    ]

    print("Dynamic retrieval planner classification:")
    for question in examples:
        category = classify_question(question)
        print(f"{question} -> {category}")
```

Then press: Ctrl+D

# Step 42 — Verify the updated retrieval planner file

```bash
cd /opt/supportgraph/06_graph_rag_api && \
cat -n retrieval_planner.py
```

# Step 43 — Run the refined dynamic retrieval planner

```bash
python retrieval_planner.py
```

# Step 44 — Create the first planner-driven issue retrieval query shape

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee issue_retrieval_query.py
```

Now paste this exact code:

```python
from retrieval_planner import classify_question


def build_issue_troubleshooting_query():
    return """
MATCH (i:Issue)
OPTIONAL MATCH (i)<-[:ABOUT]-(t:SupportTicket)
OPTIONAL MATCH (t)-[:RAISED_BY]->(c:Customer)
OPTIONAL MATCH (t)-[:AFFECTS]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:SupportAgent)
OPTIONAL MATCH (ka:KnowledgeArticle)-[:RESOLVES]->(i)
RETURN
    i.title AS issue_title,
    i.description AS issue_description,
    collect(t.ticketId) AS ticket_ids,
    collect(c.name) AS customer_names,
    collect(p.name) AS product_names,
    collect(a.name) AS assigned_agents,
    collect(ka.title) AS knowledge_articles
"""


if __name__ == "__main__":
    question = "Why are customers facing login failure?"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category == "issue_troubleshooting":
        print("Cypher query shape for issue troubleshooting:")
        print(build_issue_troubleshooting_query())
    else:
        print("This script currently only defines the issue_troubleshooting query shape.")
```

Then press: Ctrl+D

# Step 45 — Run the planner-driven issue retrieval query-shape script

```bash
python issue_retrieval_query.py
```

# Step 46 — Correct issue_retrieval_query.py to match the real SupportGraph schema

> the real SupportGraph schema uses Ticket, Customer, Product, Agent, Issue, and KnowledgeArticle with relationships such as (:Ticket)-[:HAS_ISSUE]->(:Issue), (:Customer)-[:RAISED]->(:Ticket), (:Ticket)-[:ABOUT]->(:Product), (:Ticket)-[:ASSIGNED_TO]->(:Agent), and (:KnowledgeArticle)-[:SOLVES]->(:Issue). It does not use SupportTicket, SupportAgent, RAISED_BY, AFFECTS, or RESOLVES. It also defines issue properties as issueId, name, category, and severity, not title / description.


```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee issue_retrieval_query.py
```

Now paste this exact code:

```python
from retrieval_planner import classify_question


def build_issue_troubleshooting_query():
    return """
MATCH (i:Issue)
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)
OPTIONAL MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)
RETURN
    i.issueId AS issue_id,
    i.name AS issue_name,
    i.severity AS issue_severity,
    collect(t.ticketId) AS ticket_ids,
    collect(c.name) AS customer_names,
    collect(p.name) AS product_names,
    collect(a.name) AS assigned_agents,
    collect(ka.title) AS knowledge_articles
"""


if __name__ == "__main__":
    question = "Why are customers facing login failure?"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category == "issue_troubleshooting":
        print("Cypher query shape for issue troubleshooting:")
        print(build_issue_troubleshooting_query())
    else:
        print("This script currently only defines the issue_troubleshooting query shape.")
```

Then press: Ctrl+D

# Step 47 — Run the corrected issue retrieval query-shape script

```bash
python issue_retrieval_query.py
```

# Step 48 — Create the live issue retrieval script

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee issue_retrieval_live.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"


if __name__ == "__main__":
    question = "Why are customers facing login failure?"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category != "issue_troubleshooting":
        print("This script currently runs only the issue_troubleshooting retrieval path.")
    else:
        query = build_issue_troubleshooting_query()

        with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
            records, summary, keys = driver.execute_query(
                query,
                database_=NEO4J_DATABASE
            )

            print("Live issue retrieval results:")
            for record in records:
                print("-" * 80)
                print(f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})")
                print(f"Tickets: {record['ticket_ids']}")
                print(f"Customers: {record['customer_names']}")
                print(f"Products: {record['product_names']}")
                print(f"Assigned agents: {record['assigned_agents']}")
                print(f"Knowledge articles: {record['knowledge_articles']}")
```

Then press: Ctrl+D

# Step 49 — Run the live issue retrieval script

```shell
python issue_retrieval_live.py
```

# Step 50 — Make the issue query parameter-ready

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee issue_retrieval_query.py
```

Now paste this exact code:

```python
from retrieval_planner import classify_question


def build_issue_troubleshooting_query():
    return """
MATCH (i:Issue)
WHERE i.name = $issue_name
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)
OPTIONAL MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)
RETURN
    i.issueId AS issue_id,
    i.name AS issue_name,
    i.severity AS issue_severity,
    collect(t.ticketId) AS ticket_ids,
    collect(c.name) AS customer_names,
    collect(p.name) AS product_names,
    collect(a.name) AS assigned_agents,
    collect(ka.title) AS knowledge_articles
"""


if __name__ == "__main__":
    question = "Why are customers facing login failure?"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category == "issue_troubleshooting":
        print("Cypher query shape for issue troubleshooting:")
        print(build_issue_troubleshooting_query())
        print()
        print("Expected parameter:")
        print({"issue_name": "Login Failure"})
    else:
        print("This script currently only defines the issue_troubleshooting query shape.")
```

Then press: Ctrl+D

# Step 51 — Run the parameter-ready query-shape script

```bash
python issue_retrieval_query.py
```

# Step 52 — Update the live retrieval script to pass the issue parameter

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee issue_retrieval_live.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"


if __name__ == "__main__":
    question = "Why are customers facing login failure?"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category != "issue_troubleshooting":
        print("This script currently runs only the issue_troubleshooting retrieval path.")
    else:
        query = build_issue_troubleshooting_query()
        params = {"issue_name": "Login Failure"}

        with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
            records, summary, keys = driver.execute_query(
                query,
                params,
                database_=NEO4J_DATABASE
            )

            print("Live issue retrieval results:")
            for record in records:
                print("-" * 80)
                print(f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})")
                print(f"Tickets: {record['ticket_ids']}")
                print(f"Customers: {record['customer_names']}")
                print(f"Products: {record['product_names']}")
                print(f"Assigned agents: {record['assigned_agents']}")
                print(f"Knowledge articles: {record['knowledge_articles']}")
```

Then press: Ctrl+D

# Step 53 — Run the parameterised live issue retrieval script

```bash
python issue_retrieval_live.py
```

# Step 54 — Update the live retrieval script to infer issue_name from the question

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee issue_retrieval_live.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"


def infer_issue_name(question):
    q = question.lower()

    if any(phrase in q for phrase in [
        "login",
        "log in",
        "sign in",
        "otp"
    ]):
        return "Login Failure"

    elif any(phrase in q for phrase in [
        "payment",
        "checkout",
        "transaction",
        "card"
    ]):
        return "Payment Failure"

    elif any(phrase in q for phrase in [
        "app crash",
        "crash",
        "compatibility"
    ]):
        return "App Crash"

    else:
        return None


if __name__ == "__main__":
    question = "Why are customers facing login failure?"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category != "issue_troubleshooting":
        print("This script currently runs only the issue_troubleshooting retrieval path.")
    else:
        issue_name = infer_issue_name(question)

        print("Inferred issue name:")
        print(issue_name)
        print()

        if issue_name is None:
            print("Could not infer a supported issue name from the question.")
        else:
            query = build_issue_troubleshooting_query()
            params = {"issue_name": issue_name}

            with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
                records, summary, keys = driver.execute_query(
                    query,
                    params,
                    database_=NEO4J_DATABASE
                )

                print("Live issue retrieval results:")
                for record in records:
                    print("-" * 80)
                    print(f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})")
                    print(f"Tickets: {record['ticket_ids']}")
                    print(f"Customers: {record['customer_names']}")
                    print(f"Products: {record['product_names']}")
                    print(f"Assigned agents: {record['assigned_agents']}")
                    print(f"Knowledge articles: {record['knowledge_articles']}")
```

Then press: Ctrl+D

# Step 55 — Run the updated live retrieval script with inferred issue mapping

```bash
python issue_retrieval_live.py
```

# Step 56 — Update the test question to Payment Failure

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee issue_retrieval_live.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"


def infer_issue_name(question):
    q = question.lower()

    if any(phrase in q for phrase in [
        "login",
        "log in",
        "sign in",
        "otp"
    ]):
        return "Login Failure"

    elif any(phrase in q for phrase in [
        "payment",
        "checkout",
        "transaction",
        "card"
    ]):
        return "Payment Failure"

    elif any(phrase in q for phrase in [
        "app crash",
        "crash",
        "compatibility"
    ]):
        return "App Crash"

    else:
        return None


if __name__ == "__main__":
    question = "Why are payments failing?"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category != "issue_troubleshooting":
        print("This script currently runs only the issue_troubleshooting retrieval path.")
    else:
        issue_name = infer_issue_name(question)

        print("Inferred issue name:")
        print(issue_name)
        print()

        if issue_name is None:
            print("Could not infer a supported issue name from the question.")
        else:
            query = build_issue_troubleshooting_query()
            params = {"issue_name": issue_name}

            with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
                records, summary, keys = driver.execute_query(
                    query,
                    params,
                    database_=NEO4J_DATABASE
                )

                print("Live issue retrieval results:")
                for record in records:
                    print("-" * 80)
                    print(f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})")
                    print(f"Tickets: {record['ticket_ids']}")
                    print(f"Customers: {record['customer_names']}")
                    print(f"Products: {record['product_names']}")
                    print(f"Assigned agents: {record['assigned_agents']}")
                    print(f"Knowledge articles: {record['knowledge_articles']}")
```

Then press: Ctrl+D

# Step 57 — Run the Payment Failure test

```bash
python issue_retrieval_live.py
```

# Step 58 — Change the test question to App Crash

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee issue_retrieval_live.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"


def infer_issue_name(question):
    q = question.lower()

    if any(phrase in q for phrase in [
        "login",
        "log in",
        "sign in",
        "otp"
    ]):
        return "Login Failure"

    elif any(phrase in q for phrase in [
        "payment",
        "checkout",
        "transaction",
        "card"
    ]):
        return "Payment Failure"

    elif any(phrase in q for phrase in [
        "app crash",
        "crash",
        "compatibility"
    ]):
        return "App Crash"

    else:
        return None


if __name__ == "__main__":
    question = "Why does the app crash?"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category != "issue_troubleshooting":
        print("This script currently runs only the issue_troubleshooting retrieval path.")
    else:
        issue_name = infer_issue_name(question)

        print("Inferred issue name:")
        print(issue_name)
        print()

        if issue_name is None:
            print("Could not infer a supported issue name from the question.")
        else:
            query = build_issue_troubleshooting_query()
            params = {"issue_name": issue_name}

            with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
                records, summary, keys = driver.execute_query(
                    query,
                    params,
                    database_=NEO4J_DATABASE
                )

                print("Live issue retrieval results:")
                for record in records:
                    print("-" * 80)
                    print(f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})")
                    print(f"Tickets: {record['ticket_ids']}")
                    print(f"Customers: {record['customer_names']}")
                    print(f"Products: {record['product_names']}")
                    print(f"Assigned agents: {record['assigned_agents']}")
                    print(f"Knowledge articles: {record['knowledge_articles']}")
```

Then press: Ctrl+D


# Step 59 — Run the App Crash test

```bash
python issue_retrieval_live.py
```

# Step 60 — Create the first retrieval + LM Studio answer script

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee ask_supportgraph_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query
from issue_retrieval_live import infer_issue_name

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


def fetch_issue_context(question):
    category = classify_question(question)

    if category != "issue_troubleshooting":
        return {
            "category": category,
            "issue_name": None,
            "records": []
        }

    issue_name = infer_issue_name(question)

    if issue_name is None:
        return {
            "category": category,
            "issue_name": None,
            "records": []
        }

    query = build_issue_troubleshooting_query()
    params = {"issue_name": issue_name}

    with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
        records, summary, keys = driver.execute_query(
            query,
            params,
            database_=NEO4J_DATABASE
        )

    return {
        "category": category,
        "issue_name": issue_name,
        "records": records
    }


def build_context_text(question, retrieval_result):
    category = retrieval_result["category"]
    issue_name = retrieval_result["issue_name"]
    records = retrieval_result["records"]

    lines = []
    lines.append(f"User question: {question}")
    lines.append(f"Planner category: {category}")
    lines.append(f"Inferred issue name: {issue_name}")
    lines.append("")

    if not records:
        lines.append("No graph retrieval context was found.")
        return "\n".join(lines)

    lines.append("Retrieved SupportGraph context:")

    for record in records:
        lines.append("-" * 60)
        lines.append(
            f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})"
        )
        lines.append(f"Tickets: {record['ticket_ids']}")
        lines.append(f"Customers: {record['customer_names']}")
        lines.append(f"Products: {record['product_names']}")
        lines.append(f"Assigned agents: {record['assigned_agents']}")
        lines.append(f"Knowledge articles: {record['knowledge_articles']}")

        if not record["ticket_ids"]:
            lines.append("Operational note: knowledge exists, but no related ticket context was found.")

    return "\n".join(lines)


if __name__ == "__main__":
    question = "Why are payments failing?"

    retrieval_result = fetch_issue_context(question)
    context_text = build_context_text(question, retrieval_result)

    print("Retrieved context sent to LM Studio:")
    print(context_text)
    print()
    print("=" * 80)
    print()

    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a support assistant for the SupportGraph Intelligence Platform. "
                    "Answer only from the retrieved graph context provided. "
                    "If operational context is missing, say so clearly."
                )
            },
            {
                "role": "user",
                "content": context_text
            }
        ],
        temperature=0.2,
        max_tokens=300
    )

    print("LM Studio answer:")
    print(response.choices[0].message.content.strip())
```

Then press: Ctrl+D

# Step 61 — Run the first retrieval + LM Studio answer script

```bash
python ask_supportgraph_once.py
```

# Step 62 — Update ask_supportgraph_once.py to prompt for a question

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee ask_supportgraph_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query
from issue_retrieval_live import infer_issue_name

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


def fetch_issue_context(question):
    category = classify_question(question)

    if category != "issue_troubleshooting":
        return {
            "category": category,
            "issue_name": None,
            "records": []
        }

    issue_name = infer_issue_name(question)

    if issue_name is None:
        return {
            "category": category,
            "issue_name": None,
            "records": []
        }

    query = build_issue_troubleshooting_query()
    params = {"issue_name": issue_name}

    with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
        records, summary, keys = driver.execute_query(
            query,
            params,
            database_=NEO4J_DATABASE
        )

    return {
        "category": category,
        "issue_name": issue_name,
        "records": records
    }


def build_context_text(question, retrieval_result):
    category = retrieval_result["category"]
    issue_name = retrieval_result["issue_name"]
    records = retrieval_result["records"]

    lines = []
    lines.append(f"User question: {question}")
    lines.append(f"Planner category: {category}")
    lines.append(f"Inferred issue name: {issue_name}")
    lines.append("")

    if not records:
        lines.append("No graph retrieval context was found.")
        return "\n".join(lines)

    lines.append("Retrieved SupportGraph context:")

    for record in records:
        lines.append("-" * 60)
        lines.append(
            f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})"
        )
        lines.append(f"Tickets: {record['ticket_ids']}")
        lines.append(f"Customers: {record['customer_names']}")
        lines.append(f"Products: {record['product_names']}")
        lines.append(f"Assigned agents: {record['assigned_agents']}")
        lines.append(f"Knowledge articles: {record['knowledge_articles']}")

        if not record["ticket_ids"]:
            lines.append("Operational note: knowledge exists, but no related ticket context was found.")

    return "\n".join(lines)


if __name__ == "__main__":
    question = input("Enter your support question: ").strip()

    retrieval_result = fetch_issue_context(question)
    context_text = build_context_text(question, retrieval_result)

    print()
    print("Retrieved context sent to LM Studio:")
    print(context_text)
    print()
    print("=" * 80)
    print()

    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a support assistant for the SupportGraph Intelligence Platform. "
                    "Answer only from the retrieved graph context provided. "
                    "If operational context is missing, say so clearly."
                )
            },
            {
                "role": "user",
                "content": context_text
            }
        ],
        temperature=0.2,
        max_tokens=300
    )

    print("LM Studio answer:")
    print(response.choices[0].message.content.strip())
```

Then press: Ctrl+D

# Step 63 — Run the reusable interactive script

```bash
python ask_supportgraph_once.py
```

# Step 64 — Return knowledge article content from the retrieval query

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee issue_retrieval_query.py
```

Now paste this exact code:

```python
from retrieval_planner import classify_question


def build_issue_troubleshooting_query():
    return """
MATCH (i:Issue)
WHERE i.name = $issue_name
OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)
OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)
OPTIONAL MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)
RETURN
    i.issueId AS issue_id,
    i.name AS issue_name,
    i.severity AS issue_severity,
    collect(t.ticketId) AS ticket_ids,
    collect(c.name) AS customer_names,
    collect(p.name) AS product_names,
    collect(a.name) AS assigned_agents,
    collect(ka.title) AS knowledge_articles,
    collect(ka.content) AS knowledge_article_contents
"""


if __name__ == "__main__":
    question = "Why are customers facing login failure?"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category == "issue_troubleshooting":
        print("Cypher query shape for issue troubleshooting:")
        print(build_issue_troubleshooting_query())
        print()
        print("Expected parameter:")
        print({"issue_name": "Login Failure"})
    else:
        print("This script currently only defines the issue_troubleshooting query shape.")
```

Then press: Ctrl+D

# Step 65 — Update ask_supportgraph_once.py to send article content to LM Studio

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee ask_supportgraph_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query
from issue_retrieval_live import infer_issue_name

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


def fetch_issue_context(question):
    category = classify_question(question)

    if category != "issue_troubleshooting":
        return {
            "category": category,
            "issue_name": None,
            "records": []
        }

    issue_name = infer_issue_name(question)

    if issue_name is None:
        return {
            "category": category,
            "issue_name": None,
            "records": []
        }

    query = build_issue_troubleshooting_query()
    params = {"issue_name": issue_name}

    with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
        records, summary, keys = driver.execute_query(
            query,
            params,
            database_=NEO4J_DATABASE
        )

    return {
        "category": category,
        "issue_name": issue_name,
        "records": records
    }


def build_context_text(question, retrieval_result):
    category = retrieval_result["category"]
    issue_name = retrieval_result["issue_name"]
    records = retrieval_result["records"]

    lines = []
    lines.append(f"User question: {question}")
    lines.append(f"Planner category: {category}")
    lines.append(f"Inferred issue name: {issue_name}")
    lines.append("")

    if not records:
        lines.append("No graph retrieval context was found.")
        return "\n".join(lines)

    lines.append("Retrieved SupportGraph context:")

    for record in records:
        lines.append("-" * 60)
        lines.append(
            f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})"
        )
        lines.append(f"Tickets: {record['ticket_ids']}")
        lines.append(f"Customers: {record['customer_names']}")
        lines.append(f"Products: {record['product_names']}")
        lines.append(f"Assigned agents: {record['assigned_agents']}")
        lines.append(f"Knowledge articles: {record['knowledge_articles']}")
        lines.append(f"Knowledge article contents: {record['knowledge_article_contents']}")

        if not record["ticket_ids"]:
            lines.append("Operational note: knowledge exists, but no related ticket context was found.")

    return "\n".join(lines)


if __name__ == "__main__":
    question = input("Enter your support question: ").strip()

    retrieval_result = fetch_issue_context(question)
    context_text = build_context_text(question, retrieval_result)

    print()
    print("Retrieved context sent to LM Studio:")
    print(context_text)
    print()
    print("=" * 80)
    print()

    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a support assistant for the SupportGraph Intelligence Platform. "
                    "Answer only from the retrieved graph context provided. "
                    "If operational context is missing, say so clearly. "
                    "If knowledge article contents are provided, use them to explain causes or recommended actions."
                )
            },
            {
                "role": "user",
                "content": context_text
            }
        ],
        temperature=0.2,
        max_tokens=300
    )

    print("LM Studio answer:")
    print(response.choices[0].message.content.strip())
```

Then press: Ctrl+D

# Step 66 — Rerun the interactive script with the payment question

```bash
python ask_supportgraph_once.py
```

type

```text
why are payments failing?
```
# Step 67 — Run the interactive script and test App Crash

```bash
python ask_supportgraph_once.py
```

When it prompts: Enter your support question:

please type exactly:

```text
why does the app crash?
```

# Step 68 — Add clean unsupported-question handling to ask_supportgraph_once.py

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee ask_supportgraph_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query
from issue_retrieval_live import infer_issue_name

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"

SUPPORTED_ISSUES = [
    "Login Failure",
    "Payment Failure",
    "App Crash"
]


def fetch_issue_context(question):
    category = classify_question(question)

    if category != "issue_troubleshooting":
        return {
            "category": category,
            "issue_name": None,
            "records": []
        }

    issue_name = infer_issue_name(question)

    if issue_name is None:
        return {
            "category": category,
            "issue_name": None,
            "records": []
        }

    query = build_issue_troubleshooting_query()
    params = {"issue_name": issue_name}

    with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
        records, summary, keys = driver.execute_query(
            query,
            params,
            database_=NEO4J_DATABASE
        )

    return {
        "category": category,
        "issue_name": issue_name,
        "records": records
    }


def build_context_text(question, retrieval_result):
    category = retrieval_result["category"]
    issue_name = retrieval_result["issue_name"]
    records = retrieval_result["records"]

    lines = []
    lines.append(f"User question: {question}")
    lines.append(f"Planner category: {category}")
    lines.append(f"Inferred issue name: {issue_name}")
    lines.append("")

    if not records:
        lines.append("No graph retrieval context was found.")
        return "\n".join(lines)

    lines.append("Retrieved SupportGraph context:")

    for record in records:
        lines.append("-" * 60)
        lines.append(
            f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})"
        )
        lines.append(f"Tickets: {record['ticket_ids']}")
        lines.append(f"Customers: {record['customer_names']}")
        lines.append(f"Products: {record['product_names']}")
        lines.append(f"Assigned agents: {record['assigned_agents']}")
        lines.append(f"Knowledge articles: {record['knowledge_articles']}")
        lines.append(f"Knowledge article contents: {record['knowledge_article_contents']}")

        if not record["ticket_ids"]:
            lines.append("Operational note: knowledge exists, but no related ticket context was found.")

    return "\n".join(lines)


if __name__ == "__main__":
    question = input("Enter your support question: ").strip()

    retrieval_result = fetch_issue_context(question)
    category = retrieval_result["category"]
    issue_name = retrieval_result["issue_name"]
    records = retrieval_result["records"]

    if category != "issue_troubleshooting":
        print()
        print("This interactive script currently supports issue-troubleshooting questions only.")
        print("Supported issue examples:")
        for item in SUPPORTED_ISSUES:
            print("- " + item)

    elif issue_name is None:
        print()
        print("I could not map your question to a currently supported issue.")
        print("Supported issue examples:")
        for item in SUPPORTED_ISSUES:
            print("- " + item)

    elif not records:
        print()
        print("The question mapped to a supported issue, but no graph context was found.")
        print("Issue name:", issue_name)

    else:
        context_text = build_context_text(question, retrieval_result)

        print()
        print("Retrieved context sent to LM Studio:")
        print(context_text)
        print()
        print("=" * 80)
        print()

        response = LMSTUDIO_CLIENT.chat.completions.create(
            model=LMSTUDIO_MODEL,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "You are a support assistant for the SupportGraph Intelligence Platform. "
                        "Answer only from the retrieved graph context provided. "
                        "If operational context is missing, say so clearly. "
                        "If knowledge article contents are provided, use them to explain causes or recommended actions."
                    )
                },
                {
                    "role": "user",
                    "content": context_text
                }
            ],
            temperature=0.2,
            max_tokens=300
        )

        print("LM Studio answer:")
        print(response.choices[0].message.content.strip())
```

Then press: Ctrl+D

# Step 69 — Run the script with an unsupported question

```bash
python ask_supportgraph_once.py
```

type

```text
summarise customer Asha Sharma's support situation
```

# Step 70 — Create the first customer_summary query-shape script

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee customer_summary_query.py
```

Now paste this exact code:

```python
from retrieval_planner import classify_question


def build_customer_summary_query():
    return """
MATCH (c:Customer)
WHERE c.name = $customer_name
OPTIONAL MATCH (c)-[:RAISED]->(t:Ticket)
OPTIONAL MATCH (t)-[:HAS_ISSUE]->(i:Issue)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)
OPTIONAL MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)
RETURN
    c.customerId AS customer_id,
    c.name AS customer_name,
    collect(t.ticketId) AS ticket_ids,
    collect(i.name) AS issue_names,
    collect(i.severity) AS issue_severities,
    collect(p.name) AS product_names,
    collect(a.name) AS assigned_agents,
    collect(ka.title) AS knowledge_articles
"""


if __name__ == "__main__":
    question = "summarise customer Asha Sharma's support situation"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category == "customer_summary":
        print("Cypher query shape for customer summary:")
        print(build_customer_summary_query())
        print()
        print("Expected parameter:")
        print({"customer_name": "Asha Sharma"})
    else:
        print("This script currently only defines the customer_summary query shape.")
```

Then press: Ctrl+D

# Step 71 — Run the customer_summary query-shape script

```bash
python customer_summary_query.py
```

# Step 72 — Create the live customer_summary retrieval script

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee customer_summary_live.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from retrieval_planner import classify_question
from customer_summary_query import build_customer_summary_query

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"


if __name__ == "__main__":
    question = "summarise customer Asha Sharma's support situation"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category != "customer_summary":
        print("This script currently runs only the customer_summary retrieval path.")
    else:
        query = build_customer_summary_query()
        params = {"customer_name": "Asha Sharma"}

        with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
            records, summary, keys = driver.execute_query(
                query,
                params,
                database_=NEO4J_DATABASE
            )

            print("Live customer summary retrieval results:")
            for record in records:
                print("-" * 80)
                print(f"Customer: {record['customer_id']} - {record['customer_name']}")
                print(f"Tickets: {record['ticket_ids']}")
                print(f"Issues: {record['issue_names']}")
                print(f"Issue severities: {record['issue_severities']}")
                print(f"Products: {record['product_names']}")
                print(f"Assigned agents: {record['assigned_agents']}")
                print(f"Knowledge articles: {record['knowledge_articles']}")
```

Then press: Ctrl+D

# Step 73 — Run the live customer_summary retrieval script

```python
python customer_summary_live.py
```

# Step 74 — Add knowledge article content to customer_summary_query.py

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee customer_summary_query.py
```

Now paste this exact code:

```python
from retrieval_planner import classify_question


def build_customer_summary_query():
    return """
MATCH (c:Customer)
WHERE c.name = $customer_name
OPTIONAL MATCH (c)-[:RAISED]->(t:Ticket)
OPTIONAL MATCH (t)-[:HAS_ISSUE]->(i:Issue)
OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)
OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)
OPTIONAL MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)
RETURN
    c.customerId AS customer_id,
    c.name AS customer_name,
    collect(t.ticketId) AS ticket_ids,
    collect(i.name) AS issue_names,
    collect(i.severity) AS issue_severities,
    collect(p.name) AS product_names,
    collect(a.name) AS assigned_agents,
    collect(ka.title) AS knowledge_articles,
    collect(ka.content) AS knowledge_article_contents
"""


if __name__ == "__main__":
    question = "summarise customer Asha Sharma's support situation"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category == "customer_summary":
        print("Cypher query shape for customer summary:")
        print(build_customer_summary_query())
        print()
        print("Expected parameter:")
        print({"customer_name": "Asha Sharma"})
    else:
        print("This script currently only defines the customer_summary query shape.")
```

# Step 75 — Update customer_summary_live.py to print article content

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee customer_summary_live.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from retrieval_planner import classify_question
from customer_summary_query import build_customer_summary_query

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"


if __name__ == "__main__":
    question = "summarise customer Asha Sharma's support situation"
    category = classify_question(question)

    print("Question:")
    print(question)
    print()
    print("Category:")
    print(category)
    print()

    if category != "customer_summary":
        print("This script currently runs only the customer_summary retrieval path.")
    else:
        query = build_customer_summary_query()
        params = {"customer_name": "Asha Sharma"}

        with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
            records, summary, keys = driver.execute_query(
                query,
                params,
                database_=NEO4J_DATABASE
            )

            print("Live customer summary retrieval results:")
            for record in records:
                print("-" * 80)
                print(f"Customer: {record['customer_id']} - {record['customer_name']}")
                print(f"Tickets: {record['ticket_ids']}")
                print(f"Issues: {record['issue_names']}")
                print(f"Issue severities: {record['issue_severities']}")
                print(f"Products: {record['product_names']}")
                print(f"Assigned agents: {record['assigned_agents']}")
                print(f"Knowledge articles: {record['knowledge_articles']}")
                print(f"Knowledge article contents: {record['knowledge_article_contents']}")
```

Then press: Ctrl+D

# Step 76 — Run the updated live customer_summary retrieval script

```bash
python customer_summary_live.py
```


# Step 77 — Create the first customer_summary + LM Studio answer script

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee ask_customer_summary_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from retrieval_planner import classify_question
from customer_summary_query import build_customer_summary_query

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


def fetch_customer_summary_context(question):
    category = classify_question(question)

    if category != "customer_summary":
        return {
            "category": category,
            "customer_name": None,
            "records": []
        }

    customer_name = "Asha Sharma"

    query = build_customer_summary_query()
    params = {"customer_name": customer_name}

    with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
        records, summary, keys = driver.execute_query(
            query,
            params,
            database_=NEO4J_DATABASE
        )

    return {
        "category": category,
        "customer_name": customer_name,
        "records": records
    }


def build_context_text(question, retrieval_result):
    category = retrieval_result["category"]
    customer_name = retrieval_result["customer_name"]
    records = retrieval_result["records"]

    lines = []
    lines.append(f"User question: {question}")
    lines.append(f"Planner category: {category}")
    lines.append(f"Requested customer name: {customer_name}")
    lines.append("")

    if not records:
        lines.append("No customer summary graph retrieval context was found.")
        return "\n".join(lines)

    lines.append("Retrieved SupportGraph context:")

    for record in records:
        lines.append("-" * 60)
        lines.append(f"Customer: {record['customer_id']} - {record['customer_name']}")
        lines.append(f"Tickets: {record['ticket_ids']}")
        lines.append(f"Issues: {record['issue_names']}")
        lines.append(f"Issue severities: {record['issue_severities']}")
        lines.append(f"Products: {record['product_names']}")
        lines.append(f"Assigned agents: {record['assigned_agents']}")
        lines.append(f"Knowledge articles: {record['knowledge_articles']}")
        lines.append(f"Knowledge article contents: {record['knowledge_article_contents']}")

    return "\n".join(lines)


if __name__ == "__main__":
    question = "summarise customer Asha Sharma's support situation"

    retrieval_result = fetch_customer_summary_context(question)
    context_text = build_context_text(question, retrieval_result)

    print("Retrieved context sent to LM Studio:")
    print(context_text)
    print()
    print("=" * 80)
    print()

    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a support assistant for the SupportGraph Intelligence Platform. "
                    "Answer only from the retrieved graph context provided. "
                    "Summarise the customer's current support situation clearly and concisely."
                )
            },
            {
                "role": "user",
                "content": context_text
            }
        ],
        temperature=0.2,
        max_tokens=300
    )

    print("LM Studio answer:")
    print(response.choices[0].message.content.strip())
```

Then press: Ctrl+D

# Step 78 — Run the first customer_summary + LM Studio answer script

```bash
python ask_customer_summary_once.py
```

# Step 79 — Overwrite ask_customer_summary_once.py with the dynamic version

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee ask_customer_summary_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from retrieval_planner import classify_question
from customer_summary_query import build_customer_summary_query

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"

SUPPORTED_CUSTOMERS = [
    "Asha Sharma",
    "Ravi Mehta"
]


def infer_customer_name(question):
    q = question.lower()

    if "asha sharma" in q:
        return "Asha Sharma"
    elif "ravi mehta" in q:
        return "Ravi Mehta"
    else:
        return None


def fetch_customer_summary_context(question):
    category = classify_question(question)

    if category != "customer_summary":
        return {
            "category": category,
            "customer_name": None,
            "records": []
        }

    customer_name = infer_customer_name(question)

    if customer_name is None:
        return {
            "category": category,
            "customer_name": None,
            "records": []
        }

    query = build_customer_summary_query()
    params = {"customer_name": customer_name}

    with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
        records, summary, keys = driver.execute_query(
            query,
            params,
            database_=NEO4J_DATABASE
        )

    return {
        "category": category,
        "customer_name": customer_name,
        "records": records
    }


def build_context_text(question, retrieval_result):
    category = retrieval_result["category"]
    customer_name = retrieval_result["customer_name"]
    records = retrieval_result["records"]

    lines = []
    lines.append(f"User question: {question}")
    lines.append(f"Planner category: {category}")
    lines.append(f"Requested customer name: {customer_name}")
    lines.append("")

    if not records:
        lines.append("No customer summary graph retrieval context was found.")
        return "\n".join(lines)

    lines.append("Retrieved SupportGraph context:")

    for record in records:
        lines.append("-" * 60)
        lines.append(f"Customer: {record['customer_id']} - {record['customer_name']}")
        lines.append(f"Tickets: {record['ticket_ids']}")
        lines.append(f"Issues: {record['issue_names']}")
        lines.append(f"Issue severities: {record['issue_severities']}")
        lines.append(f"Products: {record['product_names']}")
        lines.append(f"Assigned agents: {record['assigned_agents']}")
        lines.append(f"Knowledge articles: {record['knowledge_articles']}")
        lines.append(f"Knowledge article contents: {record['knowledge_article_contents']}")

    return "\n".join(lines)


if __name__ == "__main__":
    question = input("Enter your customer summary question: ").strip()

    retrieval_result = fetch_customer_summary_context(question)
    category = retrieval_result["category"]
    customer_name = retrieval_result["customer_name"]
    records = retrieval_result["records"]

    if category != "customer_summary":
        print()
        print("This script currently supports customer-summary questions only.")
        print("Supported customers:")
        for item in SUPPORTED_CUSTOMERS:
            print("- " + item)

    elif customer_name is None:
        print()
        print("I could not map your question to a currently supported customer.")
        print("Supported customers:")
        for item in SUPPORTED_CUSTOMERS:
            print("- " + item)

    elif not records:
        print()
        print("The question mapped to a supported customer, but no graph context was found.")
        print("Customer name:", customer_name)

    else:
        context_text = build_context_text(question, retrieval_result)

        print()
        print("Retrieved context sent to LM Studio:")
        print(context_text)
        print()
        print("=" * 80)
```

Then press: Ctrl+D

# Step 80 — Run the dynamic customer-summary script with Ravi Mehta

```bash
python ask_customer_summary_once.py
```

When it prompts: Enter your customer summary question:

please type exactly:

```text
summarise customer Ravi Mehta's support situation
```


# Step 81 — Create the unified interactive Ask SupportGraph script

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee ask_supportgraph_unified.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from retrieval_planner import classify_question
from issue_retrieval_query import build_issue_troubleshooting_query
from customer_summary_query import build_customer_summary_query
from issue_retrieval_live import infer_issue_name

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"

SUPPORTED_ISSUES = [
    "Login Failure",
    "Payment Failure",
    "App Crash"
]

SUPPORTED_CUSTOMERS = [
    "Asha Sharma",
    "Ravi Mehta"
]


def infer_customer_name(question):
    q = question.lower()

    if "asha sharma" in q:
        return "Asha Sharma"
    elif "ravi mehta" in q:
        return "Ravi Mehta"
    else:
        return None


def fetch_issue_context(question):
    issue_name = infer_issue_name(question)

    if issue_name is None:
        return {
            "kind": "issue_troubleshooting",
            "target_name": None,
            "records": []
        }

    query = build_issue_troubleshooting_query()
    params = {"issue_name": issue_name}

    with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
        records, summary, keys = driver.execute_query(
            query,
            params,
            database_=NEO4J_DATABASE
        )

    return {
        "kind": "issue_troubleshooting",
        "target_name": issue_name,
        "records": records
    }


def fetch_customer_context(question):
    customer_name = infer_customer_name(question)

    if customer_name is None:
        return {
            "kind": "customer_summary",
            "target_name": None,
            "records": []
        }

    query = build_customer_summary_query()
    params = {"customer_name": customer_name}

    with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
        records, summary, keys = driver.execute_query(
            query,
            params,
            database_=NEO4J_DATABASE
        )

    return {
        "kind": "customer_summary",
        "target_name": customer_name,
        "records": records
    }


def build_issue_context_text(question, retrieval_result):
    issue_name = retrieval_result["target_name"]
    records = retrieval_result["records"]

    lines = []
    lines.append(f"User question: {question}")
    lines.append("Planner category: issue_troubleshooting")
    lines.append(f"Inferred issue name: {issue_name}")
    lines.append("")

    if not records:
        lines.append("No graph retrieval context was found.")
        return "\n".join(lines)

    lines.append("Retrieved SupportGraph context:")

    for record in records:
        lines.append("-" * 60)
        lines.append(
            f"Issue: {record['issue_id']} - {record['issue_name']} ({record['issue_severity']})"
        )
        lines.append(f"Tickets: {record['ticket_ids']}")
        lines.append(f"Customers: {record['customer_names']}")
        lines.append(f"Products: {record['product_names']}")
        lines.append(f"Assigned agents: {record['assigned_agents']}")
        lines.append(f"Knowledge articles: {record['knowledge_articles']}")
        lines.append(f"Knowledge article contents: {record['knowledge_article_contents']}")

        if not record["ticket_ids"]:
            lines.append("Operational note: knowledge exists, but no related ticket context was found.")

    return "\n".join(lines)


def build_customer_context_text(question, retrieval_result):
    customer_name = retrieval_result["target_name"]
    records = retrieval_result["records"]

    lines = []
    lines.append(f"User question: {question}")
    lines.append("Planner category: customer_summary")
    lines.append(f"Requested customer name: {customer_name}")
    lines.append("")

    if not records:
        lines.append("No customer summary graph retrieval context was found.")
        return "\n".join(lines)

    lines.append("Retrieved SupportGraph context:")

    for record in records:
        lines.append("-" * 60)
        lines.append(f"Customer: {record['customer_id']} - {record['customer_name']}")
        lines.append(f"Tickets: {record['ticket_ids']}")
        lines.append(f"Issues: {record['issue_names']}")
        lines.append(f"Issue severities: {record['issue_severities']}")
        lines.append(f"Products: {record['product_names']}")
        lines.append(f"Assigned agents: {record['assigned_agents']}")
        lines.append(f"Knowledge articles: {record['knowledge_articles']}")
        lines.append(f"Knowledge article contents: {record['knowledge_article_contents']}")

    return "\n".join(lines)


if __name__ == "__main__":
    question = input("Enter your Ask SupportGraph question: ").strip()
    category = classify_question(question)

    if category == "issue_troubleshooting":
        retrieval_result = fetch_issue_context(question)

        if retrieval_result["target_name"] is None:
            print()
            print("I could not map your question to a currently supported issue.")
            print("Supported issues:")
            for item in SUPPORTED_ISSUES:
                print("- " + item)

        elif not retrieval_result["records"]:
            print()
            print("The question mapped to a supported issue, but no graph context was found.")
            print("Issue name:", retrieval_result["target_name"])

        else:
            context_text = build_issue_context_text(question, retrieval_result)

            print()
            print("Retrieved context sent to LM Studio:")
            print(context_text)
            print()
            print("=" * 80)
            print()

            response = LMSTUDIO_CLIENT.chat.completions.create(
                model=LMSTUDIO_MODEL,
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "You are a support assistant for the SupportGraph Intelligence Platform. "
                            "Answer only from the retrieved graph context provided. "
                            "If operational context is missing, say so clearly. "
                            "If knowledge article contents are provided, use them to explain causes or recommended actions."
                        )
                    },
                    {
                        "role": "user",
                        "content": context_text
                    }
                ],
                temperature=0.2,
                max_tokens=300
            )

            print("LM Studio answer:")
            print(response.choices[0].message.content.strip())

    elif category == "customer_summary":
        retrieval_result = fetch_customer_context(question)

        if retrieval_result["target_name"] is None:
            print()
            print("I could not map your question to a currently supported customer.")
            print("Supported customers:")
            for item in SUPPORTED_CUSTOMERS:
                print("- " + item)

        elif not retrieval_result["records"]:
            print()
            print("The question mapped to a supported customer, but no graph context was found.")
            print("Customer name:", retrieval_result["target_name"])

        else:
            context_text = build_customer_context_text(question, retrieval_result)

            print()
            print("Retrieved context sent to LM Studio:")
            print(context_text)
            print()
            print("=" * 80)
            print()

            response = LMSTUDIO_CLIENT.chat.completions.create(
                model=LMSTUDIO_MODEL,
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "You are a support assistant for the SupportGraph Intelligence Platform. "
                            "Answer only from the retrieved graph context provided. "
                            "Summarise the customer's current support situation clearly and concisely."
                        )
                    },
                    {
                        "role": "user",
                        "content": context_text
                    }
                ],
                temperature=0.2,
                max_tokens=300
            )

            print("LM Studio answer:")
            print(response.choices[0].message.content.strip())

    else:
        print()
        print("This unified script currently supports two planner categories only:")
        print("- issue_troubleshooting")
        print("- customer_summary")
```

Then press: Ctrl+D

# Step 82 — Run the unified Ask SupportGraph script with an issue question

```bash
python ask_supportgraph_unified.py
```

When it prompts: Enter your Ask SupportGraph question:

Please type exactly:

```text
why are payments failing?
```

# Step 83 — Run the unified script with a customer-summary question

```bash
python ask_supportgraph_unified.py
```

When it prompts: Enter your Ask SupportGraph question:

Please type exactly:

```text
summarise customer Ravi Mehta's support situation
```

# Step 84 — Check the current Neo4j edition from your environment

Before we write the text-to-Cypher code, we need to confirm one thing from your actual environment:

> Are you running Neo4j Community Edition or Neo4j Enterprise Edition?

```bash
docker exec supportgraph-neo4j cypher-shell -u neo4j -p 'SupportGraph@123' "CALL dbms.components() YIELD name, versions, edition RETURN name, versions, edition;"
```

We already consulted the official Neo4j user-management documentation and confirmed that Community Edition cannot provide the restricted-role safety layer we wanted.

# Step 85 — Create the first Cypher safety validator

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee cypher_safety.py
```

Now paste this exact code:

```python
BLOCKED_KEYWORDS = [
    "CREATE",
    "MERGE",
    "DELETE",
    "DETACH",
    "SET",
    "REMOVE",
    "DROP",
    "LOAD CSV",
    "ALTER",
    "GRANT",
    "DENY",
    "REVOKE",
    "CREATE USER",
    "DROP USER",
    "SHOW USERS",
    "SHOW USER",
    "SHOW PRIVILEGES",
    "SHOW ROLES",
    "CALL DBMS",
    "CALL GDS",
    "CALL APOC"
]


def normalise_cypher(query):
    return " ".join(query.upper().split())


def has_blocked_keyword(query):
    q = normalise_cypher(query)

    for keyword in BLOCKED_KEYWORDS:
        if keyword in q:
            return True, keyword

    return False, None


def has_limit(query):
    q = normalise_cypher(query)
    return " LIMIT " in (" " + q + " ")


def looks_like_read_query(query):
    q = normalise_cypher(query)

    allowed_start = (
        q.startswith("MATCH ")
        or q.startswith("OPTIONAL MATCH ")
        or q.startswith("WITH ")
    )

    return allowed_start and " RETURN " in (" " + q + " ")


def validate_cypher(query):
    blocked, keyword = has_blocked_keyword(query)

    if blocked:
        return {
            "safe": False,
            "reason": "Blocked keyword found: " + keyword
        }

    if not looks_like_read_query(query):
        return {
            "safe": False,
            "reason": "Query does not look like a read-only MATCH/WITH/RETURN query."
        }

    if not has_limit(query):
        return {
            "safe": False,
            "reason": "Query is missing LIMIT."
        }

    return {
        "safe": True,
        "reason": "Query passed basic read-only safety checks."
    }


if __name__ == "__main__":
    test_queries = [
        "MATCH (a:Agent) RETURN a.name LIMIT 5",
        "MATCH (n) RETURN n",
        "CREATE (n) RETURN n LIMIT 0",
        "MATCH (n) SET n.x = 1 RETURN n LIMIT 1",
        "SHOW USERS",
        "MATCH (a:Agent) WHERE toLower(a.name) CONTAINS 'rajat' RETURN a.name, a.team LIMIT 5"
    ]

    print("Cypher safety validator test:")
    for query in test_queries:
        result = validate_cypher(query)
        print("-" * 80)
        print("Query:")
        print(query)
        print("Validation result:")
        print(result)
```

Then press: Ctrl+D

Run cypher_safety.py and check that it accepts safe read queries and rejects unsafe ones.

# Step 86 — Run the Cypher safety validator

```bash
python cypher_safety.py
```

Now, create the first LLM-generated Cypher prototype

# Step 87 — Create the first LLM-generated Cypher prototype

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee llm_generate_cypher_once.py
```

Now paste this exact code:

```python
from openai import OpenAI

from cypher_safety import validate_cypher

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


GRAPH_SCHEMA = """
You are working with a Neo4j customer support graph.

Node labels and useful properties:
- Customer: customerId, name, region, segment
- Ticket: ticketId, title, issueType, priority, status, createdDate
- Product: productId, name, category, platform
- Agent: agentId, name, team, location
- Issue: issueId, name, category, severity
- KnowledgeArticle: articleId, title, issueType, content, source
- DocumentChunk: chunkId, text, chunkOrder, articleId, source

Relationships:
- (:Customer)-[:RAISED]->(:Ticket)
- (:Ticket)-[:ABOUT]->(:Product)
- (:Ticket)-[:ASSIGNED_TO]->(:Agent)
- (:Ticket)-[:HAS_ISSUE]->(:Issue)
- (:KnowledgeArticle)-[:SOLVES]->(:Issue)
- (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)

Important rules:
- Generate only one Cypher query.
- The query must be read-only.
- Use MATCH / OPTIONAL MATCH / WHERE / RETURN / ORDER BY / LIMIT only.
- Do not use CREATE, MERGE, DELETE, DETACH, SET, REMOVE, DROP, ALTER, GRANT, DENY, REVOKE, LOAD CSV, SHOW, CALL, or admin commands.
- Always include LIMIT 10 or smaller.
- Return useful properties, not full nodes.
- Do not wrap the query in markdown code fences.
"""


def clean_generated_cypher(text):
    cleaned = text.strip()

    if cleaned.startswith("```"):
        cleaned = cleaned.replace("```cypher", "")
        cleaned = cleaned.replace("```", "")
        cleaned = cleaned.strip()

    return cleaned


if __name__ == "__main__":
    question = "who is rajat?"

    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You generate safe read-only Neo4j Cypher queries. "
                    "Return only the Cypher query text and nothing else."
                )
            },
            {
                "role": "user",
                "content": GRAPH_SCHEMA + "\n\nUser question:\n" + question
            }
        ],
        temperature=0.1,
        max_tokens=250
    )

    generated_cypher = clean_generated_cypher(
        response.choices[0].message.content
    )

    print("User question:")
    print(question)
    print()
    print("Generated Cypher:")
    print(generated_cypher)
    print()
    print("Safety validation:")
    print(validate_cypher(generated_cypher))
```

Then press: Ctrl+D

# Step 88 — Run the LLM-generated Cypher prototype

```bash
python llm_generate_cypher_once.py
```

Create a script that generates Cypher, validates it, and executes it only if it passes validation.

# Step 89 — Create the first validated LLM-Cypher execution script

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee llm_cypher_execute_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from cypher_safety import validate_cypher

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


GRAPH_SCHEMA = """
You are working with a Neo4j customer support graph.

Node labels and useful properties:
- Customer: customerId, name, region, segment
- Ticket: ticketId, title, issueType, priority, status, createdDate
- Product: productId, name, category, platform
- Agent: agentId, name, team, location
- Issue: issueId, name, category, severity
- KnowledgeArticle: articleId, title, issueType, content, source
- DocumentChunk: chunkId, text, chunkOrder, articleId, source

Relationships:
- (:Customer)-[:RAISED]->(:Ticket)
- (:Ticket)-[:ABOUT]->(:Product)
- (:Ticket)-[:ASSIGNED_TO]->(:Agent)
- (:Ticket)-[:HAS_ISSUE]->(:Issue)
- (:KnowledgeArticle)-[:SOLVES]->(:Issue)
- (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)

Important rules:
- Generate only one Cypher query.
- The query must be read-only.
- Use MATCH / OPTIONAL MATCH / WHERE / RETURN / ORDER BY / LIMIT only.
- Do not use CREATE, MERGE, DELETE, DETACH, SET, REMOVE, DROP, ALTER, GRANT, DENY, REVOKE, LOAD CSV, SHOW, CALL, or admin commands.
- Always include LIMIT 10 or smaller.
- Return useful properties, not full nodes.
- Do not wrap the query in markdown code fences.
"""


def clean_generated_cypher(text):
    cleaned = text.strip()

    if cleaned.startswith("```"):
        cleaned = cleaned.replace("```cypher", "")
        cleaned = cleaned.replace("```", "")
        cleaned = cleaned.strip()

    return cleaned


def generate_cypher(question):
    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You generate safe read-only Neo4j Cypher queries. "
                    "Return only the Cypher query text and nothing else."
                )
            },
            {
                "role": "user",
                "content": GRAPH_SCHEMA + "\n\nUser question:\n" + question
            }
        ],
        temperature=0.1,
        max_tokens=250
    )

    return clean_generated_cypher(response.choices[0].message.content)


if __name__ == "__main__":
    question = "who is rajat?"

    generated_cypher = generate_cypher(question)
    validation = validate_cypher(generated_cypher)

    print("User question:")
    print(question)
    print()
    print("Generated Cypher:")
    print(generated_cypher)
    print()
    print("Safety validation:")
    print(validation)
    print()

    if not validation["safe"]:
        print("Query was not executed because it failed safety validation.")
    else:
        with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
            records, summary, keys = driver.execute_query(
                generated_cypher,
                database_=NEO4J_DATABASE
            )

        print("Neo4j returned rows:")
        print(len(records))
        print()

        for record in records:
            print(record.data())
```

Then press: Ctrl+D

Run the validated LLM-Cypher execution script and inspect the Neo4j rows returned.

# Step 90 — Run the validated LLM-Cypher execution script

```bash
python llm_cypher_execute_once.py
```

Update the LLM prompt inside llm_cypher_execute_once.py so that for short person/customer/agent names, it prefers partial case-insensitive matching.

# Step 91 — Update the LLM-Cypher prompt for partial case-insensitive name matching

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee llm_cypher_execute_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from cypher_safety import validate_cypher

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


GRAPH_SCHEMA = """
You are working with a Neo4j customer support graph.

Node labels and useful properties:
- Customer: customerId, name, region, segment
- Ticket: ticketId, title, issueType, priority, status, createdDate
- Product: productId, name, category, platform
- Agent: agentId, name, team, location
- Issue: issueId, name, category, severity
- KnowledgeArticle: articleId, title, issueType, content, source
- DocumentChunk: chunkId, text, chunkOrder, articleId, source

Relationships:
- (:Customer)-[:RAISED]->(:Ticket)
- (:Ticket)-[:ABOUT]->(:Product)
- (:Ticket)-[:ASSIGNED_TO]->(:Agent)
- (:Ticket)-[:HAS_ISSUE]->(:Issue)
- (:KnowledgeArticle)-[:SOLVES]->(:Issue)
- (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)

Important rules:
- Generate only one Cypher query.
- The query must be read-only.
- Use MATCH / OPTIONAL MATCH / WHERE / RETURN / ORDER BY / LIMIT only.
- Do not use CREATE, MERGE, DELETE, DETACH, SET, REMOVE, DROP, ALTER, GRANT, DENY, REVOKE, LOAD CSV, SHOW, CALL, or admin commands.
- Always include LIMIT 10 or smaller.
- Return useful properties, not full nodes.
- Do not wrap the query in markdown code fences.

Name matching guidance:
- If the user asks about a short or partial person name, such as "rajat", do not use exact equality.
- Prefer partial case-insensitive matching with Cypher regular expressions.
- Example:
  MATCH (a:Agent)
  WHERE a.name =~ '(?i).*rajat.*'
  RETURN a.agentId, a.name, a.team, a.location
  LIMIT 10

Useful expansion guidance:
- If the question asks "who is <name>?", first search Agent, Customer, and related Ticket context if useful.
- For an Agent, include assigned tickets, related issues, related products, and related customers if available.
"""


def clean_generated_cypher(text):
    cleaned = text.strip()

    if cleaned.startswith("```"):
        cleaned = cleaned.replace("```cypher", "")
        cleaned = cleaned.replace("```", "")
        cleaned = cleaned.strip()

    return cleaned


def generate_cypher(question):
    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You generate safe read-only Neo4j Cypher queries. "
                    "Return only the Cypher query text and nothing else."
                )
            },
            {
                "role": "user",
                "content": GRAPH_SCHEMA + "\n\nUser question:\n" + question
            }
        ],
        temperature=0.1,
        max_tokens=300
    )

    return clean_generated_cypher(response.choices[0].message.content)


if __name__ == "__main__":
    question = "who is rajat?"

    generated_cypher = generate_cypher(question)
    validation = validate_cypher(generated_cypher)

    print("User question:")
    print(question)
    print()
    print("Generated Cypher:")
    print(generated_cypher)
    print()
    print("Safety validation:")
    print(validation)
    print()

    if not validation["safe"]:
        print("Query was not executed because it failed safety validation.")
    else:
        with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
            records, summary, keys = driver.execute_query(
                generated_cypher,
                database_=NEO4J_DATABASE
            )

        print("Neo4j returned rows:")
        print(len(records))
        print()

        for record in records:
            print(record.data())
```

Then press: Ctrl+D

# Step 92 — Run the updated LLM-Cypher execution script

```bash
python llm_cypher_execute_once.py
```

# Step 93 — Simplify the LLM-Cypher prompt for the Rajat test

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee llm_cypher_execute_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from cypher_safety import validate_cypher

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


GRAPH_SCHEMA = """
You are working with a Neo4j customer support graph.

Current relevant node label:
- Agent: agentId, name, team, location

Current relevant relationship context:
- (:Ticket)-[:ASSIGNED_TO]->(:Agent)

Important safety rules:
- Generate exactly one Cypher query.
- Return only the Cypher query text.
- Do not explain.
- Do not use markdown.
- The query must be read-only.
- The query must include RETURN.
- The query must include LIMIT 10.
- Do not use CREATE, MERGE, DELETE, DETACH, SET, REMOVE, DROP, ALTER, GRANT, DENY, REVOKE, LOAD CSV, SHOW, CALL, or admin commands.

For short person-name questions, use partial case-insensitive matching.
Use this exact style:

MATCH (a:Agent)
WHERE a.name =~ '(?i).*rajat.*'
RETURN a.agentId, a.name, a.team, a.location
LIMIT 10
"""


def clean_generated_cypher(text):
    cleaned = text.strip()

    if cleaned.startswith("```"):
        cleaned = cleaned.replace("```cypher", "")
        cleaned = cleaned.replace("```", "")
        cleaned = cleaned.strip()

    return cleaned


def generate_cypher(question):
    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You generate safe read-only Neo4j Cypher. "
                    "Return only one complete Cypher query and nothing else."
                )
            },
            {
                "role": "user",
                "content": GRAPH_SCHEMA + "\n\nUser question:\n" + question
            }
        ],
        temperature=0.0,
        max_tokens=180
    )

    return clean_generated_cypher(response.choices[0].message.content)


if __name__ == "__main__":
    question = "who is rajat?"

    generated_cypher = generate_cypher(question)
    validation = validate_cypher(generated_cypher)

    print("User question:")
    print(question)
    print()
    print("Generated Cypher:")
    print(generated_cypher)
    print()
    print("Safety validation:")
    print(validation)
    print()

    if not validation["safe"]:
        print("Query was not executed because it failed safety validation.")
    else:
        with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
            records, summary, keys = driver.execute_query(
                generated_cypher,
                database_=NEO4J_DATABASE
            )

        print("Neo4j returned rows:")
        print(len(records))
        print()

        for record in records:
            print(record.data())
```

Then press: Ctrl+D

Add the final answer-generation stage after the validated Neo4j rows are returned.

> Right now, the script only prints raw graph rows:
> ```text
> {'a.agentId': 'A001', 'a.name': 'Rajat Support', 'a.team': 'L1 Support', 'a.location': 'India'}
> ```
> Next, we want:
> ```text
> Neo4j rows
-> LM Studio final answer
> ```

So the user sees something like:

```text
Rajat refers to Rajat Support, an L1 Support agent located in India.
```

This still keeps the answer grounded in retrieved graph rows only.

Lets update llm_cypher_execute_once.py so that after Neo4j returns rows, LM Studio generates the final natural-language answer from those rows.

# Step 94 — Add final answer generation from returned Neo4j rows

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee llm_cypher_execute_once.py
```

Now paste this exact code:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from cypher_safety import validate_cypher

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


GRAPH_SCHEMA = """
You are working with a Neo4j customer support graph.

Current relevant node label:
- Agent: agentId, name, team, location

Current relevant relationship context:
- (:Ticket)-[:ASSIGNED_TO]->(:Agent)

Important safety rules:
- Generate exactly one Cypher query.
- Return only the Cypher query text.
- Do not explain.
- Do not use markdown.
- The query must be read-only.
- The query must include RETURN.
- The query must include LIMIT 10.
- Do not use CREATE, MERGE, DELETE, DETACH, SET, REMOVE, DROP, ALTER, GRANT, DENY, REVOKE, LOAD CSV, SHOW, CALL, or admin commands.

For short person-name questions, use partial case-insensitive matching.
Use this exact style:

MATCH (a:Agent)
WHERE a.name =~ '(?i).*rajat.*'
RETURN a.agentId, a.name, a.team, a.location
LIMIT 10
"""


def clean_generated_cypher(text):
    cleaned = text.strip()

    if cleaned.startswith("```"):
        cleaned = cleaned.replace("```cypher", "")
        cleaned = cleaned.replace("```", "")
        cleaned = cleaned.strip()

    return cleaned


def generate_cypher(question):
    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You generate safe read-only Neo4j Cypher. "
                    "Return only one complete Cypher query and nothing else."
                )
            },
            {
                "role": "user",
                "content": GRAPH_SCHEMA + "\n\nUser question:\n" + question
            }
        ],
        temperature=0.0,
        max_tokens=180
    )

    return clean_generated_cypher(response.choices[0].message.content)


def format_records_as_context(question, generated_cypher, records):
    lines = []
    lines.append("User question:")
    lines.append(question)
    lines.append("")
    lines.append("Validated generated Cypher:")
    lines.append(generated_cypher)
    lines.append("")
    lines.append("Neo4j returned rows:")

    if not records:
        lines.append("No rows were returned.")
        return "\n".join(lines)

    for index, record in enumerate(records, start=1):
        lines.append("-" * 60)
        lines.append("Row " + str(index) + ":")
        row = record.data()

        for key, value in row.items():
            lines.append(str(key) + ": " + str(value))

    return "\n".join(lines)


def generate_final_answer(question, context_text):
    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a support graph assistant. "
                    "Answer only from the Neo4j rows provided. "
                    "If the rows do not contain enough information, say what is missing. "
                    "Keep the answer concise and factual."
                )
            },
            {
                "role": "user",
                "content": (
                    "Original user question:\n"
                    + question
                    + "\n\nRetrieved graph context:\n"
                    + context_text
                )
            }
        ],
        temperature=0.2,
        max_tokens=220
    )

    return response.choices[0].message.content.strip()


if __name__ == "__main__":
    question = "who is rajat?"

    generated_cypher = generate_cypher(question)
    validation = validate_cypher(generated_cypher)

    print("User question:")
    print(question)
    print()
    print("Generated Cypher:")
    print(generated_cypher)
    print()
    print("Safety validation:")
    print(validation)
    print()

    if not validation["safe"]:
        print("Query was not executed because it failed safety validation.")
    else:
        with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
            records, summary, keys = driver.execute_query(
                generated_cypher,
                database_=NEO4J_DATABASE
            )

        print("Neo4j returned rows:")
        print(len(records))
        print()

        for record in records:
            print(record.data())

        print()
        print("=" * 80)
        print()

        context_text = format_records_as_context(
            question,
            generated_cypher,
            records
        )

        print("Context sent to LM Studio for final answer:")
        print(context_text)

        print()
        print("=" * 80)
        print()

        final_answer = generate_final_answer(question, context_text)

        print("Final LM Studio answer:")
        print(final_answer)
```

Then press: Ctrl+D

Lets update llm_cypher_execute_once.py to use input() so you can type different questions interactively.

> change the hard-coded question
question = "who is rajat?"
into interactive input.

So you can run:

```shell
python llm_cypher_execute_once.py
```

and type the question at runtime.

# Step 95 — Make the LLM-Cypher script interactive

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee llm_cypher_execute_once.py
```

Now paste this full clean version:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from cypher_safety import validate_cypher

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


GRAPH_SCHEMA = """
You are working with a Neo4j customer support graph.

Current relevant node label:
- Agent: agentId, name, team, location

Current relevant relationship context:
- (:Ticket)-[:ASSIGNED_TO]->(:Agent)

Important safety rules:
- Generate exactly one Cypher query.
- Return only the Cypher query text.
- Do not explain.
- Do not use markdown.
- The query must be read-only.
- The query must include RETURN.
- The query must include LIMIT 10.
- Do not use CREATE, MERGE, DELETE, DETACH, SET, REMOVE, DROP, ALTER, GRANT, DENY, REVOKE, LOAD CSV, SHOW, CALL, or admin commands.

For short person-name questions, use partial case-insensitive matching.
Use this style:

MATCH (a:Agent)
WHERE a.name =~ '(?i).*rajat.*'
RETURN a.agentId, a.name, a.team, a.location
LIMIT 10

If the user asks about a different short name, replace rajat with the lowercase name from the question.
"""


def clean_generated_cypher(text):
    cleaned = text.strip()

    if cleaned.startswith("```"):
        cleaned = cleaned.replace("```cypher", "")
        cleaned = cleaned.replace("```", "")
        cleaned = cleaned.strip()

    return cleaned


def generate_cypher(question):
    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You generate safe read-only Neo4j Cypher. "
                    "Return only one complete Cypher query and nothing else."
                )
            },
            {
                "role": "user",
                "content": GRAPH_SCHEMA + "\n\nUser question:\n" + question
            }
        ],
        temperature=0.0,
        max_tokens=180
    )

    return clean_generated_cypher(response.choices[0].message.content)


def format_records_as_context(question, generated_cypher, records):
    lines = []
    lines.append("User question:")
    lines.append(question)
    lines.append("")
    lines.append("Validated generated Cypher:")
    lines.append(generated_cypher)
    lines.append("")
    lines.append("Neo4j returned rows:")

    if not records:
        lines.append("No rows were returned.")
        return "\n".join(lines)

    for index, record in enumerate(records, start=1):
        lines.append("-" * 60)
        lines.append("Row " + str(index) + ":")
        row = record.data()

        for key, value in row.items():
            lines.append(str(key) + ": " + str(value))

    return "\n".join(lines)


def generate_final_answer(question, context_text):
    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a support graph assistant. "
                    "Answer only from the Neo4j rows provided. "
                    "If the rows do not contain enough information, say what is missing. "
                    "Keep the answer concise and factual."
                )
            },
            {
                "role": "user",
                "content": (
                    "Original user question:\n"
                    + question
                    + "\n\nRetrieved graph context:\n"
                    + context_text
                )
            }
        ],
        temperature=0.2,
        max_tokens=220
    )

    return response.choices[0].message.content.strip()


if __name__ == "__main__":
    question = input("Enter your SupportGraph question: ").strip()

    if not question:
        print("No question was entered.")
    else:
        generated_cypher = generate_cypher(question)
        validation = validate_cypher(generated_cypher)

        print()
        print("User question:")
        print(question)
        print()
        print("Generated Cypher:")
        print(generated_cypher)
        print()
        print("Safety validation:")
        print(validation)
        print()

        if not validation["safe"]:
            print("Query was not executed because it failed safety validation.")
        else:
            with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
                records, summary, keys = driver.execute_query(
                    generated_cypher,
                    database_=NEO4J_DATABASE
                )

            print("Neo4j returned rows:")
            print(len(records))
            print()

            for record in records:
                print(record.data())

            print()
            print("=" * 80)
            print()

            context_text = format_records_as_context(
                question,
                generated_cypher,
                records
            )

            print("Context sent to LM Studio for final answer:")
            print(context_text)

            print()
            print("=" * 80)
            print()

            final_answer = generate_final_answer(question, context_text)

            print("Final LM Studio answer:")
            print(final_answer)
```

Then press: Ctrl+D

## What we are going to do now

> replace the current Agent-focused prompt with a generic entity discovery prompt.

This will guide the LLM to:

1. search for a matching node first,
2. return that node’s labels and properties,
3. return connected node labels/properties and relationship types,
4. then answer from that discovered context.

# Step 97 — Replace llm_cypher_execute_once.py with generic entity discovery version

```bash
cd /opt/supportgraph/06_graph_rag_api && \
tee llm_cypher_execute_once.py
```

Now paste this full clean version:

```python
from neo4j import GraphDatabase
from openai import OpenAI

from cypher_safety import validate_cypher

NEO4J_URI = "neo4j" + "://localhost:7687"
NEO4J_AUTH = ("neo4j", "SupportGraph@123")
NEO4J_DATABASE = "neo4j"

LMSTUDIO_CLIENT = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"
)

LMSTUDIO_MODEL = "nvidia/nemotron-3-nano-4b"


GRAPH_SCHEMA = """
You are working with a Neo4j customer support graph.

Known node labels:
- Customer
- Ticket
- Product
- Agent
- Issue
- KnowledgeArticle
- DocumentChunk

Known relationships:
- (:Customer)-[:RAISED]->(:Ticket)
- (:Ticket)-[:ASSIGNED_TO]->(:Agent)
- (:Ticket)-[:ABOUT]->(:Product)
- (:Ticket)-[:HAS_ISSUE]->(:Issue)
- (:KnowledgeArticle)-[:SOLVES]->(:Issue)
- (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)

Important safety rules:
- Generate exactly one Cypher query.
- Return only the Cypher query text.
- Do not explain.
- Do not use markdown.
- The query must be read-only.
- The query must include RETURN.
- The query must include LIMIT 10.
- Do not use CREATE, MERGE, DELETE, DETACH, SET, REMOVE, DROP, ALTER, GRANT, DENY, REVOKE, LOAD CSV, SHOW, CALL, or admin commands.
- Return scalar/map/list values, not full nodes.

Generic entity discovery guidance:
- For questions like "who is asha?", "who is rajat?", "what is mobile app?", or "what is payment failure?", do not guess the label first.
- First search for a matching node by common text properties.
- Then inspect its labels, properties, relationship types, and connected node labels/properties.
- Use labels(n), properties(n), type(r), labels(m), and properties(m).
- Use partial case-insensitive regex matching with (?i).

For a question like "who is asha?", use this style:

MATCH (n)
WHERE
    (n.name IS NOT NULL AND n.name =~ '(?i).*asha.*')
    OR (n.title IS NOT NULL AND n.title =~ '(?i).*asha.*')
    OR (n.issueType IS NOT NULL AND n.issueType =~ '(?i).*asha.*')
    OR (n.content IS NOT NULL AND n.content =~ '(?i).*asha.*')
OPTIONAL MATCH (n)-[r]-(m)
RETURN
    labels(n) AS matchedNodeLabels,
    properties(n) AS matchedNodeProperties,
    type(r) AS relationshipType,
    labels(m) AS connectedNodeLabels,
    properties(m) AS connectedNodeProperties
LIMIT 10

If the user asks about a different term, replace asha with the lowercase search term from the question.
"""


def clean_generated_cypher(text):
    cleaned = text.strip()

    if cleaned.startswith("```"):
        cleaned = cleaned.replace("```cypher", "")
        cleaned = cleaned.replace("```", "")
        cleaned = cleaned.strip()

    return cleaned


def call_llm_for_cypher(question, repair_instruction=None):
    user_content = GRAPH_SCHEMA + "\n\nUser question:\n" + question

    if repair_instruction:
        user_content = (
            user_content
            + "\n\nThe previous query failed validation for this reason:\n"
            + repair_instruction
            + "\n\nGenerate a corrected complete read-only Cypher query with RETURN and LIMIT 10."
        )

    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You generate safe read-only Neo4j Cypher. "
                    "Return only one complete Cypher query and nothing else."
                )
            },
            {
                "role": "user",
                "content": user_content
            }
        ],
        temperature=0.0,
        max_tokens=500
    )

    return clean_generated_cypher(response.choices[0].message.content)


def generate_validated_cypher(question):
    generated_cypher = call_llm_for_cypher(question)
    validation = validate_cypher(generated_cypher)

    if validation["safe"]:
        return generated_cypher, validation

    repaired_cypher = call_llm_for_cypher(
        question,
        validation["reason"]
    )
    repaired_validation = validate_cypher(repaired_cypher)

    return repaired_cypher, repaired_validation


def format_records_as_context(question, generated_cypher, records):
    lines = []
    lines.append("User question:")
    lines.append(question)
    lines.append("")
    lines.append("Validated generated Cypher:")
    lines.append(generated_cypher)
    lines.append("")
    lines.append("Neo4j returned rows:")

    if not records:
        lines.append("No rows were returned.")
        return "\n".join(lines)

    for index, record in enumerate(records, start=1):
        lines.append("-" * 60)
        lines.append("Row " + str(index) + ":")
        row = record.data()

        for key, value in row.items():
            lines.append(str(key) + ": " + str(value))

    return "\n".join(lines)


def generate_final_answer(question, context_text):
    response = LMSTUDIO_CLIENT.chat.completions.create(
        model=LMSTUDIO_MODEL,
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a support graph assistant. "
                    "Answer only from the Neo4j rows provided. "
                    "Use the matched node labels, matched node properties, relationship types, "
                    "and connected node properties to infer what the entity is. "
                    "If the rows do not contain enough information, say what is missing. "
                    "Ignore fields with value None. "
                    "Keep the answer concise and factual."
                )
            },
            {
                "role": "user",
                "content": (
                    "Original user question:\n"
                    + question
                    + "\n\nRetrieved graph context:\n"
                    + context_text
                )
            }
        ],
        temperature=0.2,
        max_tokens=260
    )

    return response.choices[0].message.content.strip()


if __name__ == "__main__":
    question = input("Enter your SupportGraph question: ").strip()

    if not question:
        print("No question was entered.")
    else:
        generated_cypher, validation = generate_validated_cypher(question)

        print()
        print("User question:")
        print(question)
        print()
        print("Generated Cypher:")
        print(generated_cypher)
        print()
        print("Safety validation:")
        print(validation)
        print()

        if not validation["safe"]:
            print("Query was not executed because it failed safety validation.")
        else:
            with GraphDatabase.driver(NEO4J_URI, auth=NEO4J_AUTH) as driver:
                records, summary, keys = driver.execute_query(
                    generated_cypher,
                    database_=NEO4J_DATABASE
                )

            print("Neo4j returned rows:")
            print(len(records))
            print()

            for record in records:
                print(record.data())

            print()
            print("=" * 80)
            print()

            context_text = format_records_as_context(
                question,
                generated_cypher,
                records
            )

            print("Context sent to LM Studio for final answer:")
            print(context_text)

            print()
            print("=" * 80)
            print()

            final_answer = generate_final_answer(question, context_text)

            print("Final LM Studio answer:")
            print(final_answer)
```

Then press: Ctrl+D