
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

