Step 1 — Verify current node counts before adding knowledge data

```cypher
// =============================================================================
// GRAPH NODE INVENTORY QUERY
// =============================================================================
// This query gives us a high-level inventory of the Neo4j graph database.
//
// In simple terms, it answers this question:
//
//     "What types of nodes exist in this graph,
//      and how many nodes do we have for each type?"
//
// This is usually one of the first queries we run when exploring a new graph,
// especially after loading data, because it helps us quickly understand the
// structure of the database.
//
// For example, the graph may contain nodes such as:
// - Document
// - Chunk
// - Person
// - Company
// - Product
// - Topic
//
// Instead of manually inspecting individual nodes, this query groups nodes by
// their labels and counts how many nodes exist in each label group.
//
// This is very useful for:
// - Verifying that data was loaded successfully
// - Checking whether expected node types exist
// - Understanding the graph schema at a basic level
// - Detecting unexpected or incorrectly labelled nodes
// - Preparing for more detailed graph exploration queries

MATCH (n)

// =============================================================================
// MATCH ALL NODES
// =============================================================================
// MATCH (n) means:
// "Find every node in the graph and temporarily assign it to the variable n."
//
// In Cypher:
// - Parentheses () represent a node.
// - The variable name n is just an alias.
// - We use n later in the RETURN clause to inspect each matched node.
//
// Notice that we are not writing something specific like:
//
//     MATCH (n:Document)
//
// That would only match nodes with the Document label.
//
// Instead, we are writing:
//
//     MATCH (n)
//
// This means Neo4j will match all nodes, regardless of their label.
//
// This is ideal for a database overview query because we do not yet want to
// limit ourselves to one specific node type.
//
// Production note:
// In very large databases, unrestricted MATCH queries can scan many nodes.
// That is acceptable for exploration and validation, but in performance-sensitive
// production workloads, we usually prefer more targeted queries whenever possible.

RETURN labels(n) AS nodeLabels,
       count(*) AS nodeCount

// =============================================================================
// RETURN NODE LABELS AND COUNTS
// =============================================================================
// The RETURN clause defines what information we want Neo4j to show in the final
// result table.
//
// Here we are returning two columns:
//
// 1. labels(n) AS nodeLabels
// 2. count(*) AS nodeCount
//
// -----------------------------------------------------------------------------
// labels(n) AS nodeLabels
// -----------------------------------------------------------------------------
// labels(n) returns the list of labels attached to each node.
//
// In Neo4j, labels are used to classify nodes into meaningful categories.
//
// For example:
// - A document node may have the label :Document
// - A text chunk node may have the label :Chunk
// - A person node may have the label :Person
// - A company node may have the label :Company
//
// A node can also have more than one label.
//
// For example:
//
//     (:Person:Employee)
//
// In that case, labels(n) would return something like:
//
//     ["Person", "Employee"]
//
// This is important because this query groups nodes by their exact label
// combination.
//
// That means:
//
//     ["Person"]
//
// and
//
//     ["Person", "Employee"]
//
// are treated as two different label groups.
//
// We rename labels(n) to nodeLabels using AS so that the output column has a
// clear and readable name.
//
// Instead of showing:
//
//     labels(n)
//
// the result will show:
//
//     nodeLabels
//
// This makes the result easier to understand, especially when sharing it with
// students, teammates, or during demos.
//
// -----------------------------------------------------------------------------
// count(*) AS nodeCount
// -----------------------------------------------------------------------------
// count(*) counts how many nodes exist in each group.
//
// Because we are returning labels(n) along with count(*), Neo4j automatically
// groups the results by labels(n).
//
// In simple terms:
//
//     "For each unique label combination, count how many nodes have that
//      combination."
//
// We rename count(*) to nodeCount so the result clearly tells us that this
// column represents the number of nodes.
//
// Example output may look like:
//
//     nodeLabels        nodeCount
//     ---------------------------
//     ["Chunk"]         120
//     ["Document"]      10
//     ["Entity"]        55
//
// This result tells us that the graph contains:
// - 120 Chunk nodes
// - 10 Document nodes
// - 55 Entity nodes
//
// This kind of summary is very helpful before running more advanced graph
// queries because it confirms what kinds of data are available.

ORDER BY nodeLabels;

// =============================================================================
// SORT THE RESULT
// =============================================================================
// ORDER BY nodeLabels sorts the final result by the nodeLabels column.
//
// Sorting is not required for correctness, but it makes the output easier to
// read and compare.
//
// Without ORDER BY, Neo4j may return the grouped rows in an order that is not
// visually organized.
//
// By sorting the labels, we make the result more predictable and easier to scan.
//
// This is especially useful when:
// - The graph contains many node types
// - We are comparing results before and after data loading
// - We are documenting database state for a lab guide
// - We are validating whether expected labels are present
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a graph inventory query.
//
// It does not inspect relationships or properties.
// It only focuses on node labels and node counts.
//
// The main purpose is to quickly understand:
//
//     "What kinds of nodes exist in my Neo4j database,
//      and how many of each kind are present?"
//
// This is a simple but very useful first step in graph exploration, data
// validation, and troubleshooting.
```

# Step 2: Check current relationship types

```cypher
// =============================================================================
// GRAPH RELATIONSHIP INVENTORY QUERY
// =============================================================================
// This query gives us a high-level inventory of the relationships present in
// the Neo4j graph database.
//
// In simple terms, it answers this question:
//
//     "What types of relationships exist in this graph,
//      and how many relationships do we have for each type?"
//
// This is usually one of the first relationship-focused queries we run when
// exploring or validating a graph.
//
// After checking what node labels exist, the next natural question is:
//
//     "How are those nodes connected?"
//
// In a graph database, relationships are just as important as nodes because
// relationships describe how entities are linked together.
//
// For example, the graph may contain relationships such as:
// - (:Document)-[:HAS_CHUNK]->(:Chunk)
// - (:Chunk)-[:MENTIONS]->(:Entity)
// - (:Person)-[:WORKS_FOR]->(:Company)
// - (:Customer)-[:PLACED]->(:Order)
//
// Instead of manually inspecting relationships one by one, this query groups
// relationships by their type and counts how many relationships exist for each
// type.
//
// This is very useful for:
// - Verifying that relationships were loaded successfully
// - Checking whether expected relationship types exist
// - Detecting missing or incorrectly named relationship types
// - Understanding the graph structure at a connection level
// - Preparing for more detailed graph traversal queries
// - Documenting the current database state in a lab guide or demo

MATCH ()-[r]->()

// =============================================================================
// MATCH ALL DIRECTED RELATIONSHIPS
// =============================================================================
// MATCH ()-[r]->() means:
// "Find every directed relationship in the graph and temporarily assign each
// relationship to the variable r."
//
// In Cypher:
// - Parentheses () represent nodes.
// - Square brackets [] represent relationships.
// - The variable name r is an alias for each matched relationship.
// - The arrow -> indicates the direction of the relationship.
//
// Let us break it down:
//
//     ()-[r]->()
//
// Left side:
//     ()
//     This represents the starting node of the relationship.
//     We do not give it a variable name because we do not need to return or
//     inspect the node itself in this query.
//
// Middle:
//     [r]
//     This represents the relationship.
//     We name it r because we want to inspect the relationship type later using
//     type(r).
//
// Right side:
//     ()
//     This represents the ending node of the relationship.
//     Again, we do not give it a variable name because this query is focused
//     only on relationship types and counts.
//
// Arrow:
//     ->
//     This means we are matching relationships in their stored direction.
//
// Neo4j relationships always have a direction when they are created. Even if
// an application treats a relationship as conceptually bidirectional, Neo4j
// still stores it with a start node and an end node.
//
// Why we do not specify node labels:
// We are not writing something specific like:
//
//     MATCH (:Document)-[r]->(:Chunk)
//
// That would only match relationships from Document nodes to Chunk nodes.
//
// Instead, we write:
//
//     MATCH ()-[r]->()
//
// This means Neo4j will match all directed relationships, regardless of the
// labels of the start and end nodes.
//
// Why we do not specify relationship type:
// We are not writing something specific like:
//
//     MATCH ()-[r:HAS_CHUNK]->()
//
// That would only match HAS_CHUNK relationships.
//
// Instead, we leave the relationship type unspecified so Neo4j can inspect all
// relationship types in the database.
//
// Production note:
// In very large graphs, unrestricted relationship scans can be expensive because
// Neo4j may need to inspect many relationships. This is fine for exploration,
// schema validation, and lab demos, but production application queries should
// usually be more targeted.

RETURN type(r) AS relationshipType,
       count(*) AS relationshipCount

// =============================================================================
// RETURN RELATIONSHIP TYPES AND COUNTS
// =============================================================================
// The RETURN clause defines what information Neo4j should display in the final
// result table.
//
// Here we return two columns:
//
// 1. type(r) AS relationshipType
// 2. count(*) AS relationshipCount
//
// -----------------------------------------------------------------------------
// type(r) AS relationshipType
// -----------------------------------------------------------------------------
// type(r) returns the type/name of the relationship stored in variable r.
//
// In Neo4j, relationship types describe the meaning of the connection between
// two nodes.
//
// For example:
// - HAS_CHUNK may mean a Document contains a Chunk
// - MENTIONS may mean a Chunk mentions an Entity
// - WORKS_FOR may mean a Person works for a Company
// - BOUGHT may mean a Customer bought a Product
//
// Relationship types are important because they define the semantics of the
// graph. Without relationship types, we would only know that two nodes are
// connected, but not why they are connected.
//
// We rename type(r) to relationshipType using AS so that the output column has
// a clear and readable name.
//
// Instead of showing:
//
//     type(r)
//
// the result will show:
//
//     relationshipType
//
// This makes the result easier to understand during teaching, debugging,
// validation, or documentation.
//
// -----------------------------------------------------------------------------
// count(*) AS relationshipCount
// -----------------------------------------------------------------------------
// count(*) counts how many relationships exist for each relationship type.
//
// Because we are returning type(r) together with count(*), Neo4j automatically
// groups the result by type(r).
//
// In simple terms, Neo4j reads this as:
//
//     "For each unique relationship type, count how many relationships have
//      that type."
//
// We rename count(*) to relationshipCount so the result clearly tells us that
// this column represents the number of relationships of that type.
//
// Example output may look like:
//
//     relationshipType    relationshipCount
//     -------------------------------------
//     HAS_CHUNK           10
//     MENTIONS            250
//     SIMILAR_TO          45
//
// This result tells us that the graph contains:
// - 10 HAS_CHUNK relationships
// - 250 MENTIONS relationships
// - 45 SIMILAR_TO relationships
//
// This kind of summary is very helpful after loading data because it confirms
// whether the graph connections were created as expected.

ORDER BY relationshipType;

// =============================================================================
// SORT THE RESULT
// =============================================================================
// ORDER BY relationshipType sorts the final result alphabetically by the
// relationshipType column.
//
// Sorting is not required for correctness, but it makes the output easier to
// read and compare.
//
// Without ORDER BY, Neo4j may return grouped rows in an order that is not
// visually organized.
//
// By sorting the relationship types, we make the result:
// - More predictable
// - Easier to scan
// - Easier to compare between multiple runs
// - Better suited for screenshots, demos, and lab documentation
//
// This is especially useful when:
// - The graph contains many relationship types
// - We are validating data after an import process
// - We are comparing the database state before and after a load
// - We are documenting the graph schema for students or teammates
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a graph relationship inventory query.
//
// It does not inspect node labels.
// It does not inspect relationship properties.
// It does not traverse a specific business path.
//
// It focuses only on:
//
//     "What relationship types exist in my Neo4j database,
//      and how many of each type are present?"
//
// This is a simple but very useful query for graph exploration, validation,
// troubleshooting, and documentation.
//
// A good practical workflow is:
//
//     1. First inspect node labels and node counts.
//     2. Then inspect relationship types and relationship counts.
//     3. Then write targeted traversal queries using the labels and relationship
//        types discovered from the first two inventory queries.
//
// Together, these inventory queries help us understand both:
// - What entities exist in the graph
// - How those entities are connected
```

# Step 3 — Verify existing constraints

```cypher
// =============================================================================
// DATABASE CONSTRAINT INVENTORY QUERY
// =============================================================================
// This query gives us a high-level inventory of the constraints defined in the
// Neo4j database.
//
// In simple terms, it answers this question:
//
//     "What constraints exist in this database,
//      what kind of constraints are they,
//      which labels or relationship types do they apply to,
//      which properties do they protect,
//      and what backing index is owned by each constraint?"
//
// Constraints are important because they protect the correctness and consistency
// of the graph.
//
// For example, a constraint can ensure that:
// - Every Document has a unique documentId
// - Every Chunk has a required chunkId
// - No two User nodes share the same email
// - A required property exists on a node or relationship
//
// This query is especially useful after schema setup because it helps confirm
// whether the expected database rules were created successfully.
//
// It is useful for:
// - Verifying schema setup
// - Auditing database constraints
// - Troubleshooting data load failures
// - Understanding which properties are protected
// - Checking whether a constraint has an associated backing index
// - Documenting the database state for a lab guide or demo

SHOW CONSTRAINTS

// =============================================================================
// SHOW ALL CONSTRAINTS
// =============================================================================
// SHOW CONSTRAINTS is an administrative Cypher command that lists the constraints
// currently defined in the database.
//
// Think of constraints as "rules enforced by the database."
//
// Application code can make mistakes. Import scripts can load bad data.
// Different teams can write data in different ways.
//
// Constraints help protect the database by enforcing important rules at the
// database level, rather than relying only on application logic.
//
// Common examples include:
// - Uniqueness constraints
// - Property existence constraints
// - Node key constraints
// - Relationship key constraints
//
// This command does not create or modify constraints.
// It only reads and displays metadata about existing constraints.

YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex

// =============================================================================
// SELECT CONSTRAINT METADATA FIELDS
// =============================================================================
// YIELD chooses which columns from SHOW CONSTRAINTS we want to work with.
//
// SHOW CONSTRAINTS can expose several metadata fields, but this query focuses
// on the most useful ones for understanding and documenting the schema.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// name is the constraint name.
//
// A good constraint name usually describes:
// - The entity it applies to
// - The property it protects
// - The rule being enforced
//
// For example:
// - document_id_unique
// - chunk_id_unique
// - user_email_unique
//
// Naming constraints clearly is important because error messages, troubleshooting
// steps, and schema reviews become much easier when names are meaningful.
//
// -----------------------------------------------------------------------------
// type
// -----------------------------------------------------------------------------
// type tells us what kind of constraint this is.
//
// Examples may include uniqueness constraints, existence constraints, key
// constraints, or other constraint types supported by the Neo4j version.
//
// This matters because different constraint types protect the database in
// different ways.
//
// For example:
// - A uniqueness constraint prevents duplicate values.
// - An existence constraint ensures a property must be present.
// - A key constraint can combine existence and uniqueness semantics.
//
// -----------------------------------------------------------------------------
// entityType
// -----------------------------------------------------------------------------
// entityType tells us whether the constraint applies to nodes or relationships.
//
// In Neo4j, both nodes and relationships can have properties, and modern graph
// schemas may enforce rules on either one.
//
// For example:
// - A constraint on (:Document) applies to nodes.
// - A constraint on -[:MENTIONS]- applies to relationships.
//
// This field helps us immediately understand which graph element the rule is
// protecting.
//
// -----------------------------------------------------------------------------
// labelsOrTypes
// -----------------------------------------------------------------------------
// labelsOrTypes tells us which node labels or relationship types the constraint
// applies to.
//
// For node constraints, this usually contains labels such as:
// - Document
// - Chunk
// - Entity
// - User
//
// For relationship constraints, this may contain relationship types such as:
// - HAS_CHUNK
// - MENTIONS
// - SIMILAR_TO
//
// This is important because a constraint is not applied globally to every node
// or every relationship unless it is explicitly defined that way. It applies to
// a specific label or relationship type.
//
// -----------------------------------------------------------------------------
// properties
// -----------------------------------------------------------------------------
// properties lists the property or properties involved in the constraint.
//
// For example:
// - documentId
// - chunkId
// - email
// - sourceId
//
// Some constraints involve a single property.
// Others involve multiple properties together as a composite rule.
//
// This field is critical because it tells us exactly which data values are being
// protected by the constraint.
//
// -----------------------------------------------------------------------------
// ownedIndex
// -----------------------------------------------------------------------------
// ownedIndex shows the backing index owned by the constraint, if one exists.
//
// Some constraints, especially uniqueness and key-style constraints, rely on an
// index internally so Neo4j can efficiently check whether values are unique or
// quickly locate matching records.
//
// Think of the owned index as the performance structure that helps the database
// enforce the rule efficiently.
//
// This is useful when auditing schema because it shows the connection between
// logical rules and physical/index structures.

RETURN name,
       type,
       entityType,
       labelsOrTypes,
       properties,
       ownedIndex

// =============================================================================
// RETURN THE SELECTED FIELDS
// =============================================================================
// The RETURN clause defines the final output table shown to the user.
//
// We return the same fields selected in the YIELD clause:
//
// - name
// - type
// - entityType
// - labelsOrTypes
// - properties
// - ownedIndex
//
// This makes the output focused and easy to read.
//
// Instead of showing every possible column from SHOW CONSTRAINTS, we only show
// the columns that are most useful for understanding constraint definitions.
//
// Example output may look conceptually like this:
//
//     name                  type        entityType  labelsOrTypes  properties    ownedIndex
//     --------------------------------------------------------------------------------------
//     chunk_id_unique       UNIQUENESS  NODE        ["Chunk"]      ["chunkId"]   chunk_id_unique
//     document_id_unique    UNIQUENESS  NODE        ["Document"]   ["docId"]     document_id_unique
//
// This result tells us:
// - Which constraints exist
// - What type of rule each one enforces
// - Whether it applies to nodes or relationships
// - Which labels or relationship types are involved
// - Which properties are protected
// - Which backing index belongs to the constraint

ORDER BY name;

// =============================================================================
// SORT THE RESULT
// =============================================================================
// ORDER BY name sorts the final result alphabetically by constraint name.
//
// Sorting is not required for correctness, but it makes the output easier to
// inspect, compare, and document.
//
// This is especially useful when:
// - The database has many constraints
// - We are validating schema setup after running a script
// - We are comparing environments such as dev, test, and production
// - We are preparing screenshots or notes for a lab guide
//
// Without ORDER BY, Neo4j may return constraints in an order that is not ideal
// for manual review.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a database constraint inventory query.
//
// It does not inspect actual graph data.
// Instead, it inspects the schema rules that Neo4j enforces.
//
// The main purpose is to quickly understand:
//
//     "What constraints are defined,
//      what do they protect,
//      and what backing index supports them?"
//
// This is a very useful query for schema validation, troubleshooting, production
// readiness checks, and documentation.
```

# Step 4 — Create uniqueness constraint for KnowledgeArticle.articleId

```cypher
// =============================================================================
// CREATE UNIQUE CONSTRAINT ON KNOWLEDGE ARTICLE ARTICLE ID
// =============================================================================
// This query creates a uniqueness constraint for KnowledgeArticle nodes.
//
// In simple terms, it tells Neo4j:
//
//     "For every node with the label KnowledgeArticle,
//      the articleId property must be unique."
//
// This means two KnowledgeArticle nodes cannot have the same articleId value.
//
// This is important because articleId usually represents the business identifier
// or stable reference ID for a knowledge article.
//
// For example, if we have articles like:
//
//     (:KnowledgeArticle {articleId: "KA-1001"})
//     (:KnowledgeArticle {articleId: "KA-1002"})
//
// then Neo4j will allow them because both articleId values are different.
//
// But if another node is inserted like:
//
//     (:KnowledgeArticle {articleId: "KA-1001"})
//
// Neo4j will reject it because "KA-1001" already exists for another
// KnowledgeArticle node.
//
// This constraint is useful for:
// - Preventing duplicate knowledge articles
// - Protecting data quality during imports
// - Supporting reliable lookups by articleId
// - Making articleId behave like a natural/business key
// - Improving query performance for articleId-based searches
// - Making the graph safer for production workloads

CREATE CONSTRAINT knowledgeArticle_articleId_unique IF NOT EXISTS

// =============================================================================
// CREATE CONSTRAINT STATEMENT
// =============================================================================
// CREATE CONSTRAINT tells Neo4j that we want to define a new schema rule.
//
// A constraint is not normal data.
// It is a database-level rule that Neo4j enforces automatically.
//
// This is important because application code, import scripts, or manual queries
// can accidentally create bad data. A constraint protects the database even if
// the mistake comes from outside the main application.
//
// -----------------------------------------------------------------------------
// knowledgeArticle_articleId_unique
// -----------------------------------------------------------------------------
// This is the name of the constraint.
//
// Naming constraints clearly is a best practice because it makes schema
// management, troubleshooting, and error messages easier to understand.
//
// This name tells us three things:
//
//     knowledgeArticle_articleId_unique
//
// - knowledgeArticle  -> the label/entity this constraint applies to
// - articleId         -> the property being protected
// - unique            -> the rule being enforced
//
// A meaningful name is much better than an auto-generated name because when an
// error occurs, Neo4j can show this constraint name and we immediately know what
// rule was violated.
//
// -----------------------------------------------------------------------------
// IF NOT EXISTS
// -----------------------------------------------------------------------------
// IF NOT EXISTS makes the query idempotent.
//
// Idempotent means we can run the same query multiple times without creating
// duplicate constraints or causing an error just because the constraint already
// exists.
//
// This is very useful in:
// - Lab guides
// - Setup scripts
// - CI/CD deployments
// - Terraform or automation workflows
// - Repeatable database initialization scripts
//
// Without IF NOT EXISTS, running this query again after the constraint already
// exists could fail with an error saying that the constraint already exists.
//
// With IF NOT EXISTS, Neo4j checks first:
//
//     "Does this constraint already exist?"
//
// If yes, Neo4j safely skips creating it again.
// If no, Neo4j creates it.

FOR (ka:KnowledgeArticle)

// =============================================================================
// TARGET LABEL AND VARIABLE
// =============================================================================
// FOR (ka:KnowledgeArticle) defines which nodes this constraint applies to.
//
// Let us break this down:
//
//     (ka:KnowledgeArticle)
//
// - Parentheses () represent a node.
// - ka is a variable name used inside this constraint definition.
// - KnowledgeArticle is the node label.
//
// The label KnowledgeArticle identifies the category of nodes protected by this
// constraint.
//
// This constraint does NOT apply to every node in the graph.
// It only applies to nodes that have the label:
//
//     :KnowledgeArticle
//
// For example, the constraint applies to:
//
//     (:KnowledgeArticle {articleId: "KA-1001"})
//
// But it does not apply to:
//
//     (:Document {articleId: "KA-1001"})
//     (:Chunk {articleId: "KA-1001"})
//     (:User {articleId: "KA-1001"})
//
// This label-specific behavior is important because different node types can
// have different identity rules.
//
// -----------------------------------------------------------------------------
// Why use the variable ka?
// -----------------------------------------------------------------------------
// The variable ka acts as a placeholder for each KnowledgeArticle node.
//
// We need this variable so that the REQUIRE clause can refer to the articleId
// property like this:
//
//     ka.articleId
//
// Think of ka as saying:
//
//     "For each KnowledgeArticle node, inspect its articleId property."

REQUIRE ka.articleId IS UNIQUE;

// =============================================================================
// UNIQUE PROPERTY REQUIREMENT
// =============================================================================
// REQUIRE ka.articleId IS UNIQUE defines the actual rule.
//
// It means:
//
//     "Among all nodes labelled KnowledgeArticle,
//      no two nodes are allowed to have the same articleId value."
//
// This turns articleId into a unique identifier within the KnowledgeArticle
// label.
//
// -----------------------------------------------------------------------------
// Why uniqueness matters
// -----------------------------------------------------------------------------
// Uniqueness is important when a property represents identity.
//
// If articleId is supposed to identify one knowledge article, then duplicates
// would create confusion.
//
// For example, this would be invalid after the constraint is created:
//
//     CREATE (:KnowledgeArticle {articleId: "KA-1001", title: "Reset Password"})
//     CREATE (:KnowledgeArticle {articleId: "KA-1001", title: "VPN Troubleshooting"})
//
// Why is this bad?
//
// Because now the graph has two different articles with the same identity.
// A lookup like this:
//
//     MATCH (ka:KnowledgeArticle {articleId: "KA-1001"})
//     RETURN ka
//
// would return multiple nodes, even though the application probably expects
// exactly one article.
//
// The uniqueness constraint prevents this kind of data corruption.
//
// -----------------------------------------------------------------------------
// Performance benefit
// -----------------------------------------------------------------------------
// A uniqueness constraint also gives Neo4j an index-backed way to find nodes by
// articleId efficiently.
//
// This means queries like:
//
//     MATCH (ka:KnowledgeArticle {articleId: "KA-1001"})
//     RETURN ka
//
// can become faster because Neo4j can use the schema/index structure created
// for the constraint instead of scanning all KnowledgeArticle nodes.
//
// -----------------------------------------------------------------------------
// Important behavior to remember
// -----------------------------------------------------------------------------
// This constraint does not mean every KnowledgeArticle must have articleId.
//
// It only means that if articleId exists on KnowledgeArticle nodes, its value
// must be unique.
//
// If the requirement is:
//
//     "Every KnowledgeArticle must have articleId,
//      and articleId must be unique"
//
// then we would need a different or additional constraint depending on the
// Neo4j version and schema design approach.
//
// -----------------------------------------------------------------------------
// Production note
// -----------------------------------------------------------------------------
// Before creating a uniqueness constraint on existing data, always make sure
// there are no duplicate articleId values already present.
//
// If duplicates already exist, Neo4j cannot safely create the constraint because
// the existing data violates the rule.
//
// A typical pre-check query would be:
//
//     MATCH (ka:KnowledgeArticle)
//     WHERE ka.articleId IS NOT NULL
//     RETURN ka.articleId AS articleId, count(*) AS count
//     ORDER BY count DESC
//
// In production, schema changes should usually be applied through a controlled
// migration process so that dev, test, staging, and production remain consistent.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query creates a database-enforced uniqueness rule.
//
// The main purpose is:
//
//     "Make sure every KnowledgeArticle articleId is unique
//      across all KnowledgeArticle nodes."
//
// This improves:
// - Data correctness
// - Duplicate prevention
// - Reliable lookups
// - Query performance
// - Production readiness
//
// After this constraint exists, Neo4j becomes responsible for protecting the
// uniqueness of KnowledgeArticle.articleId, instead of depending only on
// application logic or manual discipline.
```

# Step 5 — Verify KnowledgeArticle.articleId uniqueness constraint

```cypher
// =============================================================================
// VERIFY SPECIFIC CONSTRAINT BY NAME
// =============================================================================
// This query checks whether one specific Neo4j constraint exists in the database.
//
// In simple terms, it answers this question:
//
//     "Does the constraint named knowledgeArticle_articleId_unique exist,
//      and if it exists, what are its details?"
//
// This is commonly run immediately after creating a constraint to confirm that
// Neo4j registered it correctly.
//
// In this case, we are verifying the constraint:
//
//     knowledgeArticle_articleId_unique
//
// That constraint was intended to enforce uniqueness on:
//
//     (:KnowledgeArticle).articleId
//
// This verification query is useful for:
// - Confirming schema setup completed successfully
// - Checking whether an automation script created the expected constraint
// - Validating that the constraint applies to the correct label and property
// - Inspecting the constraint type, entity type, and owned backing index
// - Documenting the database schema state in a lab guide or demo
// - Troubleshooting cases where duplicate data protection is not working as expected

SHOW CONSTRAINTS

// =============================================================================
// SHOW ALL CONSTRAINTS
// =============================================================================
// SHOW CONSTRAINTS asks Neo4j to list the constraints currently defined in the
// active database.
//
// Think of constraints as database-enforced rules.
//
// They are different from normal graph data:
// - Normal graph data contains nodes, relationships, labels, and properties.
// - Constraints describe rules that Neo4j must enforce on that data.
//
// For example, a uniqueness constraint can prevent two nodes with the same label
// from having the same value for a specific property.
//
// This command does not create, change, or delete anything.
// It only reads schema metadata so we can inspect the current database rules.

YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex

// =============================================================================
// CHOOSE WHICH CONSTRAINT COLUMNS TO INSPECT
// =============================================================================
// YIELD selects specific metadata fields from the output of SHOW CONSTRAINTS.
//
// SHOW CONSTRAINTS can expose multiple columns depending on the Neo4j version,
// but this query focuses only on the fields that are most useful for verifying
// one specific constraint.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// name is the constraint name.
//
// This is the human-readable identifier assigned when the constraint was created.
//
// In our case, we expect the name to be:
//
//     knowledgeArticle_articleId_unique
//
// A clear constraint name is very helpful because when Neo4j reports a schema
// error, the name immediately tells us which rule is involved.
//
// -----------------------------------------------------------------------------
// type
// -----------------------------------------------------------------------------
// type tells us what kind of constraint Neo4j created.
//
// For this specific constraint, we expect a uniqueness-related type because the
// original statement used:
//
//     REQUIRE ka.articleId IS UNIQUE
//
// The exact value displayed can vary depending on Neo4j version, but conceptually
// it should represent a uniqueness constraint.
//
// -----------------------------------------------------------------------------
// entityType
// -----------------------------------------------------------------------------
// entityType tells us whether the constraint applies to nodes or relationships.
//
// Since the original constraint was created with:
//
//     FOR (ka:KnowledgeArticle)
//
// we expect this field to indicate that the constraint applies to nodes.
//
// This matters because Neo4j can support rules on different graph entities, and
// this field confirms which kind of entity the rule protects.
//
// -----------------------------------------------------------------------------
// labelsOrTypes
// -----------------------------------------------------------------------------
// labelsOrTypes tells us which node label or relationship type the constraint
// applies to.
//
// For this query, we expect it to include:
//
//     KnowledgeArticle
//
// That confirms the rule is attached to KnowledgeArticle nodes, not to another
// label such as Document, Chunk, User, or Entity.
//
// -----------------------------------------------------------------------------
// properties
// -----------------------------------------------------------------------------
// properties tells us which property or properties are involved in the constraint.
//
// For this query, we expect it to include:
//
//     articleId
//
// That confirms Neo4j is enforcing uniqueness on the correct property.
//
// -----------------------------------------------------------------------------
// ownedIndex
// -----------------------------------------------------------------------------
// ownedIndex shows the backing index owned by the constraint, if Neo4j created
// one for it.
//
// Uniqueness constraints usually have an index behind the scenes so Neo4j can
// efficiently check whether a value already exists.
//
// This matters for two reasons:
// 1. It helps Neo4j enforce the uniqueness rule efficiently.
// 2. It can improve lookup performance for queries filtering by the constrained
//    property, such as:
//
//        MATCH (ka:KnowledgeArticle {articleId: "KA-1001"})
//        RETURN ka

WHERE name = "knowledgeArticle_articleId_unique"

// =============================================================================
// FILTER TO ONE SPECIFIC CONSTRAINT
// =============================================================================
// WHERE filters the list of constraints returned by SHOW CONSTRAINTS.
//
// Without this WHERE clause, Neo4j would return all constraints in the database.
//
// Here, we only want to inspect the constraint whose name exactly matches:
//
//     knowledgeArticle_articleId_unique
//
// This makes the result focused and easy to read.
//
// If the constraint exists:
// - Neo4j will return one row showing its metadata.
//
// If the constraint does not exist:
// - Neo4j will return no rows.
//
// That "no rows" result is also useful because it tells us the expected schema
// rule is missing and may need to be created again.
//
// Important:
// Constraint names are matched exactly. If the name has different capitalization,
// spelling, underscores, or suffixes, this filter will not match it.

RETURN name,
       type,
       entityType,
       labelsOrTypes,
       properties,
       ownedIndex;

// =============================================================================
// RETURN THE CONSTRAINT DETAILS
// =============================================================================
// RETURN defines the final output table shown to the user.
//
// We return:
//
// - name
// - type
// - entityType
// - labelsOrTypes
// - properties
// - ownedIndex
//
// These fields are enough to verify whether the constraint is correct.
//
// A successful result should conceptually confirm:
//
//     name          -> knowledgeArticle_articleId_unique
//     type          -> uniqueness constraint
//     entityType    -> NODE
//     labelsOrTypes -> ["KnowledgeArticle"]
//     properties    -> ["articleId"]
//     ownedIndex    -> backing index used by Neo4j, if applicable
//
// Example conceptual output:
//
//     name                                type        entityType  labelsOrTypes          properties     ownedIndex
//     --------------------------------------------------------------------------------------------------------------
//     knowledgeArticle_articleId_unique   UNIQUENESS  NODE        ["KnowledgeArticle"]   ["articleId"]  <index-name>
//
// If this row appears, it means the expected uniqueness rule exists.
//
// If no row appears, it means one of the following may be true:
// - The constraint was not created.
// - The constraint was created with a different name.
// - The query is running against a different database.
// - The schema creation command failed earlier.
// - The user does not have permission to view constraints.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a targeted constraint verification query.
//
// It does not create or modify the constraint.
// It only checks whether the specific constraint named:
//
//     knowledgeArticle_articleId_unique
//
// exists and shows its metadata.
//
// The main purpose is:
//
//     "Confirm that KnowledgeArticle.articleId is protected by the expected
//      uniqueness constraint."
//
// This is a good production-readiness habit because after any schema creation
// step, we should verify the result instead of assuming it worked.
```

# Step 6 — Create uniqueness constraint for DocumentChunk.chunkId

```cypher
// =============================================================================
// CREATE UNIQUE CONSTRAINT ON DOCUMENT CHUNK CHUNK ID
// =============================================================================
// This query creates a uniqueness constraint for DocumentChunk nodes.
//
// In simple terms, it tells Neo4j:
//
//     "For every node with the label DocumentChunk,
//      the chunkId property must be unique."
//
// This means two DocumentChunk nodes cannot have the same chunkId value.
//
// This is important because chunkId usually represents the unique identifier
// for a specific chunk of a larger document.
//
// In document-processing or RAG-style systems, one large document is often split
// into smaller pieces called chunks. Each chunk needs a stable identifier so it
// can be tracked, embedded, searched, linked back to the source document, and
// updated safely later.
//
// For example, if we have chunks like:
//
//     (:DocumentChunk {chunkId: "DOC-1001-CHUNK-001"})
//     (:DocumentChunk {chunkId: "DOC-1001-CHUNK-002"})
//
// then Neo4j will allow them because both chunkId values are different.
//
// But if another node is inserted like:
//
//     (:DocumentChunk {chunkId: "DOC-1001-CHUNK-001"})
//
// Neo4j will reject it because that chunkId already exists for another
// DocumentChunk node.
//
// This constraint is useful for:
// - Preventing duplicate document chunks
// - Protecting data quality during document ingestion
// - Supporting reliable lookups by chunkId
// - Making chunkId behave like a stable business key
// - Improving query performance for chunkId-based searches
// - Making the graph safer for production workloads

CREATE CONSTRAINT documentChunk_chunkId_unique IF NOT EXISTS

// =============================================================================
// CREATE CONSTRAINT STATEMENT
// =============================================================================
// CREATE CONSTRAINT tells Neo4j that we want to define a new schema rule.
//
// A constraint is not normal graph data.
// It is a database-level rule that Neo4j enforces automatically.
//
// This matters because import scripts, application services, batch jobs, or
// manual queries can accidentally create duplicate or inconsistent data.
// A constraint protects the database even if bad data is attempted from outside
// the main application.
//
// -----------------------------------------------------------------------------
// documentChunk_chunkId_unique
// -----------------------------------------------------------------------------
// This is the name of the constraint.
//
// Naming constraints clearly is a production-grade best practice because it
// makes schema management, troubleshooting, automation, and error messages much
// easier to understand.
//
// This name tells us three things:
//
//     documentChunk_chunkId_unique
//
// - documentChunk  -> the label/entity this constraint applies to
// - chunkId        -> the property being protected
// - unique         -> the rule being enforced
//
// A meaningful name is much better than relying on an auto-generated name.
// If Neo4j later reports a constraint violation, the name itself immediately
// tells us which rule was broken.
//
// -----------------------------------------------------------------------------
// IF NOT EXISTS
// -----------------------------------------------------------------------------
// IF NOT EXISTS makes this schema command idempotent.
//
// Idempotent means we can safely run the same command multiple times without
// creating duplicate constraints or failing just because the constraint already
// exists.
//
// This is very useful in:
// - Lab guides
// - Repeatable setup scripts
// - CI/CD pipelines
// - Database migration scripts
// - Terraform or automation workflows
// - Development, test, staging, and production environment bootstrapping
//
// Without IF NOT EXISTS, running this command again after the constraint already
// exists could fail with an error.
//
// With IF NOT EXISTS, Neo4j checks first:
//
//     "Does this constraint already exist?"
//
// If yes, Neo4j safely skips creating it again.
// If no, Neo4j creates it.

FOR (dc:DocumentChunk)

// =============================================================================
// TARGET LABEL AND VARIABLE
// =============================================================================
// FOR (dc:DocumentChunk) defines which nodes this constraint applies to.
//
// Let us break this down:
//
//     (dc:DocumentChunk)
//
// - Parentheses () represent a node.
// - dc is a variable name used inside this constraint definition.
// - DocumentChunk is the node label.
//
// The label DocumentChunk identifies the category of nodes protected by this
// constraint.
//
// This constraint does NOT apply to every node in the graph.
// It only applies to nodes that have the label:
//
//     :DocumentChunk
//
// For example, the constraint applies to:
//
//     (:DocumentChunk {chunkId: "DOC-1001-CHUNK-001"})
//
// But it does not apply to:
//
//     (:Document {chunkId: "DOC-1001-CHUNK-001"})
//     (:KnowledgeArticle {chunkId: "DOC-1001-CHUNK-001"})
//     (:Entity {chunkId: "DOC-1001-CHUNK-001"})
//
// This label-specific behavior is important because different node types may
// use similarly named properties for different purposes.
//
// -----------------------------------------------------------------------------
// Why use the variable dc?
// -----------------------------------------------------------------------------
// The variable dc acts as a placeholder for each DocumentChunk node.
//
// We need this variable so that the REQUIRE clause can refer to the chunkId
// property like this:
//
//     dc.chunkId
//
// Think of dc as saying:
//
//     "For each DocumentChunk node, inspect its chunkId property."

REQUIRE dc.chunkId IS UNIQUE;

// =============================================================================
// UNIQUE PROPERTY REQUIREMENT
// =============================================================================
// REQUIRE dc.chunkId IS UNIQUE defines the actual rule.
//
// It means:
//
//     "Among all nodes labelled DocumentChunk,
//      no two nodes are allowed to have the same chunkId value."
//
// This turns chunkId into a unique identifier within the DocumentChunk label.
//
// -----------------------------------------------------------------------------
// Why uniqueness matters
// -----------------------------------------------------------------------------
// Uniqueness is important when a property represents identity.
//
// In document-processing pipelines, a chunk is usually one specific segment of
// a larger source document. If two different DocumentChunk nodes share the same
// chunkId, the graph can no longer reliably identify which chunk is correct.
//
// For example, this would be invalid after the constraint is created:
//
//     CREATE (:DocumentChunk {
//       chunkId: "DOC-1001-CHUNK-001",
//       text: "Password reset instructions..."
//     })
//
//     CREATE (:DocumentChunk {
//       chunkId: "DOC-1001-CHUNK-001",
//       text: "VPN troubleshooting steps..."
//     })
//
// Why is this bad?
//
// Because now the graph has two different chunks with the same identity.
// A lookup like this:
//
//     MATCH (dc:DocumentChunk {chunkId: "DOC-1001-CHUNK-001"})
//     RETURN dc
//
// would return multiple nodes, even though the application probably expects
// exactly one chunk.
//
// The uniqueness constraint prevents this kind of data corruption.
//
// -----------------------------------------------------------------------------
// Why this matters in RAG / vector-search systems
// -----------------------------------------------------------------------------
// DocumentChunk nodes are often used in retrieval-augmented generation systems.
//
// A typical flow may look like:
//
//     Source document
//         -> split into chunks
//         -> generate embeddings for each chunk
//         -> store chunks and embeddings in Neo4j
//         -> retrieve relevant chunks during question answering
//
// In that kind of system, chunkId becomes extremely important because it helps
// connect together:
// - The original source document
// - The specific chunk text
// - The embedding/vector representation
// - The search result returned to the user
// - Any relationships from the chunk to entities, topics, or citations
//
// If chunkId is duplicated, retrieval results may become confusing or incorrect.
// The application might show the wrong text, update the wrong chunk, or attach
// relationships to the wrong node.
//
// -----------------------------------------------------------------------------
// Performance benefit
// -----------------------------------------------------------------------------
// A uniqueness constraint also gives Neo4j an index-backed way to find nodes by
// chunkId efficiently.
//
// This means queries like:
//
//     MATCH (dc:DocumentChunk {chunkId: "DOC-1001-CHUNK-001"})
//     RETURN dc
//
// can become faster because Neo4j can use the schema/index structure created
// for the constraint instead of scanning all DocumentChunk nodes.
//
// -----------------------------------------------------------------------------
// Important behavior to remember
// -----------------------------------------------------------------------------
// This constraint does not necessarily mean every DocumentChunk must have a
// chunkId property.
//
// It means that when chunkId exists on DocumentChunk nodes, its value must be
// unique.
//
// If the requirement is:
//
//     "Every DocumentChunk must have chunkId,
//      and chunkId must be unique"
//
// then the schema should also ensure property existence, or use the appropriate
// key-style constraint depending on the Neo4j version and database design.
//
// -----------------------------------------------------------------------------
// Production note
// -----------------------------------------------------------------------------
// Before creating a uniqueness constraint on existing data, always check whether
// duplicate chunkId values already exist.
//
// If duplicate values are already present, Neo4j cannot safely create the
// uniqueness constraint because the current data violates the rule.
//
// A typical pre-check query would be:
//
//     MATCH (dc:DocumentChunk)
//     WHERE dc.chunkId IS NOT NULL
//     WITH dc.chunkId AS chunkId, count(*) AS count
//     WHERE count > 1
//     RETURN chunkId, count
//     ORDER BY count DESC
//
// If this pre-check returns rows, those duplicates should be cleaned up before
// applying the constraint.
//
// In production environments, schema changes should usually be applied through
// a controlled migration process so that development, test, staging, and
// production remain consistent.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query creates a database-enforced uniqueness rule.
//
// The main purpose is:
//
//     "Make sure every DocumentChunk chunkId is unique
//      across all DocumentChunk nodes."
//
// This improves:
// - Data correctness
// - Duplicate prevention
// - Reliable chunk lookups
// - Safer document ingestion
// - Better vector-search/RAG traceability
// - Query performance
// - Production readiness
//
// After this constraint exists, Neo4j becomes responsible for protecting the
// uniqueness of DocumentChunk.chunkId instead of depending only on application
// logic, import scripts, or manual discipline.
```

# Step 7 — Verify DocumentChunk.chunkId uniqueness constraint

```cypher
// =============================================================================
// VERIFY DOCUMENT CHUNK UNIQUE CONSTRAINT BY NAME
// =============================================================================
// This query checks whether one specific Neo4j constraint exists in the database.
//
// In simple terms, it answers this question:
//
//     "Does the constraint named documentChunk_chunkId_unique exist,
//      and if it exists, what are its details?"
//
// This is commonly run immediately after creating a constraint to confirm that
// Neo4j registered the schema rule correctly.
//
// In this case, we are verifying the constraint:
//
//     documentChunk_chunkId_unique
//
// That constraint was intended to enforce uniqueness on:
//
//     (:DocumentChunk).chunkId
//
// This verification query is useful for:
// - Confirming schema setup completed successfully
// - Checking whether an automation script created the expected constraint
// - Validating that the constraint applies to the correct label and property
// - Inspecting the constraint type, entity type, and owned backing index
// - Documenting the database schema state in a lab guide or demo
// - Troubleshooting cases where duplicate DocumentChunk records are still possible

SHOW CONSTRAINTS

// =============================================================================
// SHOW ALL CONSTRAINTS
// =============================================================================
// SHOW CONSTRAINTS asks Neo4j to list the constraints currently defined in the
// active database.
//
// Think of constraints as database-enforced rules.
//
// They are different from normal graph data:
//
// - Normal graph data contains nodes, relationships, labels, and properties.
// - Constraints describe rules that Neo4j must enforce on that data.
//
// For example, a uniqueness constraint can prevent two nodes with the same label
// from having the same value for a specific property.
//
// In this case, we are not creating or modifying anything.
// We are only asking Neo4j:
//
//     "Show me the schema rules that currently exist."
//
// This is a safe read-only schema inspection command.

YIELD name, type, entityType, labelsOrTypes, properties, ownedIndex

// =============================================================================
// CHOOSE WHICH CONSTRAINT COLUMNS TO INSPECT
// =============================================================================
// YIELD selects specific metadata fields from the output of SHOW CONSTRAINTS.
//
// SHOW CONSTRAINTS may expose several columns depending on the Neo4j version,
// but this query focuses only on the fields that are most useful for verifying
// one specific constraint.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// name is the constraint name.
//
// This is the human-readable identifier assigned when the constraint was created.
//
// In our case, we expect the name to be:
//
//     documentChunk_chunkId_unique
//
// A clear constraint name is very helpful because when Neo4j reports a schema
// error, the name immediately tells us which rule is involved.
//
// -----------------------------------------------------------------------------
// type
// -----------------------------------------------------------------------------
// type tells us what kind of constraint Neo4j created.
//
// For this specific constraint, we expect a uniqueness-related type because the
// original statement used:
//
//     REQUIRE dc.chunkId IS UNIQUE
//
// Conceptually, this means Neo4j should enforce:
//
//     "No two DocumentChunk nodes can share the same chunkId value."
//
// The exact text shown in the type column can vary slightly by Neo4j version,
// but the important point is that it should represent a uniqueness constraint.
//
// -----------------------------------------------------------------------------
// entityType
// -----------------------------------------------------------------------------
// entityType tells us whether the constraint applies to nodes or relationships.
//
// Since the original constraint was created with:
//
//     FOR (dc:DocumentChunk)
//
// we expect this field to indicate that the constraint applies to nodes.
//
// This matters because Neo4j can define schema rules for different graph
// entities. This field confirms that the rule protects nodes, not relationships.
//
// -----------------------------------------------------------------------------
// labelsOrTypes
// -----------------------------------------------------------------------------
// labelsOrTypes tells us which node label or relationship type the constraint
// applies to.
//
// For this query, we expect it to include:
//
//     DocumentChunk
//
// That confirms the rule is attached to DocumentChunk nodes, not to another
// label such as Document, KnowledgeArticle, Entity, Chunk, or User.
//
// -----------------------------------------------------------------------------
// properties
// -----------------------------------------------------------------------------
// properties tells us which property or properties are involved in the constraint.
//
// For this query, we expect it to include:
//
//     chunkId
//
// That confirms Neo4j is enforcing uniqueness on the correct property.
//
// This is very important in document-processing and RAG-style systems because
// chunkId usually identifies one specific piece of source content.
//
// If the wrong property is constrained, duplicate chunks may still be created
// even though the schema appears to contain a constraint.
//
// -----------------------------------------------------------------------------
// ownedIndex
// -----------------------------------------------------------------------------
// ownedIndex shows the backing index owned by the constraint, if Neo4j created
// one for it.
//
// Uniqueness constraints usually rely on an index behind the scenes so Neo4j can
// efficiently check whether a value already exists.
//
// This matters for two reasons:
//
// 1. It helps Neo4j enforce the uniqueness rule efficiently.
// 2. It can improve lookup performance for queries filtering by the constrained
//    property, such as:
//
//        MATCH (dc:DocumentChunk {chunkId: "DOC-1001-CHUNK-001"})
//        RETURN dc
//
// Think of the owned index as the performance structure that supports the
// logical rule.

WHERE name = "documentChunk_chunkId_unique"

// =============================================================================
// FILTER TO ONE SPECIFIC CONSTRAINT
// =============================================================================
// WHERE filters the list of constraints returned by SHOW CONSTRAINTS.
//
// Without this WHERE clause, Neo4j would return all constraints in the database.
//
// Here, we only want to inspect the constraint whose name exactly matches:
//
//     documentChunk_chunkId_unique
//
// This makes the result focused and easy to read.
//
// If the constraint exists:
// - Neo4j will return one row showing its metadata.
//
// If the constraint does not exist:
// - Neo4j will return no rows.
//
// That "no rows" result is also useful because it tells us the expected schema
// rule is missing and may need to be created again.
//
// Important:
// Constraint names are matched exactly. If the name has different capitalization,
// spelling, underscores, or suffixes, this filter will not match it.
//
// For example, these names would not match:
//
//     DocumentChunk_chunkId_unique
//     documentchunk_chunkId_unique
//     documentChunk_chunkID_unique
//     documentChunk_chunkId_uniqueness
//
// This exact-name filtering is useful during verification because we want to
// confirm the specific constraint created by our setup script.

RETURN name,
       type,
       entityType,
       labelsOrTypes,
       properties,
       ownedIndex;

// =============================================================================
// RETURN THE CONSTRAINT DETAILS
// =============================================================================
// RETURN defines the final output table shown to the user.
//
// We return:
//
// - name
// - type
// - entityType
// - labelsOrTypes
// - properties
// - ownedIndex
//
// These fields are enough to verify whether the constraint is correct.
//
// A successful result should conceptually confirm:
//
//     name          -> documentChunk_chunkId_unique
//     type          -> uniqueness constraint
//     entityType    -> NODE
//     labelsOrTypes -> ["DocumentChunk"]
//     properties    -> ["chunkId"]
//     ownedIndex    -> backing index used by Neo4j, if applicable
//
// Example conceptual output:
//
//     name                          type        entityType  labelsOrTypes       properties   ownedIndex
//     ---------------------------------------------------------------------------------------------------
//     documentChunk_chunkId_unique   UNIQUENESS  NODE        ["DocumentChunk"]   ["chunkId"]  <index-name>
//
// If this row appears, it means the expected uniqueness rule exists.
//
// If no row appears, it means one of the following may be true:
//
// - The constraint was not created.
// - The constraint was created with a different name.
// - The query is running against a different database.
// - The schema creation command failed earlier.
// - The connected user does not have permission to view constraints.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a targeted constraint verification query.
//
// It does not create, modify, or delete the constraint.
// It only checks whether the specific constraint named:
//
//     documentChunk_chunkId_unique
//
// exists and shows its metadata.
//
// The main purpose is:
//
//     "Confirm that DocumentChunk.chunkId is protected by the expected
//      uniqueness constraint."
//
// This is a good production-readiness habit because after any schema creation
// step, we should verify the result instead of assuming it worked.
//
// In a document-ingestion or RAG pipeline, this verification is especially
// important because duplicate chunkId values can lead to confusing retrieval,
// incorrect updates, wrong relationships, or poor traceability back to the
// original source document.
```

# Step 8 — Load sample KnowledgeArticle nodes

```cypher
// =============================================================================
// LOAD SAMPLE KNOWLEDGE ARTICLES INTO NEO4J
// =============================================================================
// This query creates or updates a small sample knowledge base inside Neo4j.
//
// In simple terms, it loads three KnowledgeArticle nodes:
//
//     1. K001 - Fix login failure
//     2. K002 - Resolve payment failure
//     3. K003 - Fix app crash
//
// Each article contains:
// - articleId  -> unique identifier for the article
// - title      -> human-readable article title
// - issueType  -> category of customer issue
// - content    -> troubleshooting guidance text
// - source     -> where this sample data came from
//
// This kind of query is very useful in a lab or demo because it gives us
// realistic-looking knowledge base content that we can later use for:
// - Graph search
// - RAG pipelines
// - Vector embeddings
// - Similarity search
// - Customer support automation
// - Connecting issues, articles, chunks, and entities
//
// The important thing to understand is that this query is not only inserting
// raw data. It is establishing the first layer of a knowledge graph where each
// article becomes a node that can later be linked to chunks, topics, issues,
// entities, customers, or support tickets.

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

// =============================================================================
// UNWIND SAMPLE DATA INTO ROWS
// =============================================================================
// UNWIND takes a list and expands it into individual rows.
//
// Think of the list above like a small in-memory table.
//
// Conceptually, before UNWIND, the data looks like one list:
//
//     [
//       {articleId: "K001", ...},
//       {articleId: "K002", ...},
//       {articleId: "K003", ...}
//     ]
//
// After UNWIND, Neo4j processes it as three separate rows:
//
//     Row 1 -> article = {articleId: "K001", title: "Fix login failure", ...}
//     Row 2 -> article = {articleId: "K002", title: "Resolve payment failure", ...}
//     Row 3 -> article = {articleId: "K003", title: "Fix app crash", ...}
//
// This is useful because Cypher operations such as MERGE and SET work row by row.
// So each article map becomes one input record for creating or updating one
// KnowledgeArticle node.
//
// -----------------------------------------------------------------------------
// Why use inline sample data?
// -----------------------------------------------------------------------------
// For a lab or demo, inline data is very convenient because everything needed to
// create the sample graph is contained in one Cypher query.
//
// In production, this data would usually come from:
// - CSV files
// - JSON files
// - APIs
// - Databases
// - Kafka topics
// - ETL/ELT pipelines
// - Document ingestion workflows
//
// But for learning, UNWIND with inline maps is a clean way to demonstrate how
// structured data becomes graph data.
//
// -----------------------------------------------------------------------------
// What each map represents
// -----------------------------------------------------------------------------
// Each object inside the list is a map.
//
// A map is a key-value structure.
//
// For example:
//
//     {
//       articleId: "K001",
//       title: "Fix login failure",
//       issueType: "Login Failure",
//       content: "..."
//     }
//
// This means:
// - articleId is the unique article identifier
// - title is the article heading
// - issueType tells us what customer problem this article solves
// - content contains the actual troubleshooting guidance
//
// The alias:
//
//     AS article
//
// means that each row will expose one map through the variable article.
//
// So later in the query, we can access values like:
//
//     article.articleId
//     article.title
//     article.issueType
//     article.content

MERGE (ka:KnowledgeArticle {articleId: article.articleId})

// =============================================================================
// CREATE OR MATCH KNOWLEDGE ARTICLE NODE
// =============================================================================
// MERGE is used when we want "create if missing, reuse if already present"
// behavior.
//
// In simple terms, this line tells Neo4j:
//
//     "Find a KnowledgeArticle node with this articleId.
//      If it already exists, use it.
//      If it does not exist, create it."
//
// This is different from CREATE.
//
// CREATE always creates a new node, even if another node with the same articleId
// already exists.
//
// MERGE is safer here because articleId represents the identity of the article.
// We do not want duplicate KnowledgeArticle nodes for the same article.
//
// -----------------------------------------------------------------------------
// Breaking down the pattern
// -----------------------------------------------------------------------------
//
//     (ka:KnowledgeArticle {articleId: article.articleId})
//
// - Parentheses () represent a node.
// - ka is the variable name assigned to the node.
// - KnowledgeArticle is the label of the node.
// - articleId is the property used to identify the node.
// - article.articleId comes from the current UNWIND row.
//
// So when the first row is processed, Neo4j effectively sees:
//
//     MERGE (ka:KnowledgeArticle {articleId: "K001"})
//
// For the second row:
//
//     MERGE (ka:KnowledgeArticle {articleId: "K002"})
//
// For the third row:
//
//     MERGE (ka:KnowledgeArticle {articleId: "K003"})
//
// -----------------------------------------------------------------------------
// Why MERGE uses only articleId here
// -----------------------------------------------------------------------------
// Notice that MERGE only uses articleId, not title, issueType, or content.
//
// This is intentional.
//
// The articleId is the stable identity of the article.
// The title, issueType, and content are descriptive attributes that may change
// over time.
//
// For example, the troubleshooting steps for "Fix login failure" may be updated
// later. We still want it to remain the same article K001, not create a brand-new
// node just because the content changed.
//
// This is a very important modeling principle:
//
//     Use MERGE with stable identity fields.
//     Use SET for fields that may need to be updated.
//
// -----------------------------------------------------------------------------
// Relationship with uniqueness constraint
// -----------------------------------------------------------------------------
// Earlier, we created a uniqueness constraint on:
//
//     (:KnowledgeArticle).articleId
//
// That constraint supports this MERGE pattern because it ensures that there
// cannot be two KnowledgeArticle nodes with the same articleId.
//
// This gives us both:
// - Data correctness through the constraint
// - Idempotent loading through MERGE
//
// Together, they make the query safer and more production-ready.

SET
  ka.title = article.title,
  ka.issueType = article.issueType,
  ka.content = article.content,
  ka.source = "Day 3 sample knowledge base"

// =============================================================================
// SET ARTICLE PROPERTIES
// =============================================================================
// SET assigns or updates properties on the matched or created KnowledgeArticle
// node.
//
// At this point, the variable ka refers to the KnowledgeArticle node found or
// created by MERGE.
//
// We then copy values from the current article map onto the node.
//
// -----------------------------------------------------------------------------
// ka.title = article.title
// -----------------------------------------------------------------------------
// This stores the article title on the node.
//
// Example:
//
//     ka.title = "Fix login failure"
//
// The title is useful for humans because it gives a short readable summary of
// what the knowledge article is about.
//
// -----------------------------------------------------------------------------
// ka.issueType = article.issueType
// -----------------------------------------------------------------------------
// This stores the type of issue that the article helps solve.
//
// Example:
//
//     ka.issueType = "Login Failure"
//
// This is useful because we can later search, group, or connect articles by
// issue category.
//
// For example, we may later write:
//
//     MATCH (ka:KnowledgeArticle)
//     WHERE ka.issueType = "Login Failure"
//     RETURN ka
//
// In a more advanced graph model, issueType might become its own node, such as:
//
//     (:IssueType {name: "Login Failure"})
//
// and the article could be connected to it using a relationship like:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:IssueType)
//
// But for this beginner-friendly stage, keeping issueType as a property is
// simple and sufficient.
//
// -----------------------------------------------------------------------------
// ka.content = article.content
// -----------------------------------------------------------------------------
// This stores the full troubleshooting guidance text.
//
// This is the most important text field for later search and RAG-style use cases.
//
// The content can later be:
// - Split into smaller chunks
// - Converted into embeddings
// - Stored in vector indexes
// - Retrieved as context for an LLM
// - Connected to entities such as OTP, payment gateway, card, app cache, etc.
//
// In other words, this property is the knowledge payload.
//
// -----------------------------------------------------------------------------
// ka.source = "Day 3 sample knowledge base"
// -----------------------------------------------------------------------------
// This stores a fixed source value on every article created by this query.
//
// The source property tells us where the data came from.
//
// This is useful for traceability.
//
// In real-world systems, source metadata is extremely important because teams
// need to know:
// - Which file/API/system produced the data
// - Which ingestion pipeline loaded it
// - Whether the data is sample, test, or production data
// - Whether it can be refreshed or deleted later
//
// Here, the value:
//
//     "Day 3 sample knowledge base"
//
// clearly tells us that these articles belong to the Day 3 lab/demo dataset.
//
// -----------------------------------------------------------------------------
// Why use SET instead of ON CREATE SET only?
// -----------------------------------------------------------------------------
// This query uses SET for all properties, which means the properties are updated
// every time the query runs.
//
// That makes the query refresh-friendly.
//
// If you change the title, issueType, or content in the inline list and run the
// query again, Neo4j will update the existing article node instead of leaving
// old values behind.
//
// This is useful for lab development because you can rerun the query as many
// times as needed and keep the sample data in sync.
//
// Production note:
// In some systems, you may want different behavior:
//
//     ON CREATE SET ...
//     ON MATCH SET ...
//
// That allows you to treat newly created records and already-existing records
// differently.
//
// For example:
// - ON CREATE SET createdAt = datetime()
// - ON MATCH SET updatedAt = datetime()
//
// But for this sample load, simple SET keeps the query easy to understand.

RETURN
  ka.articleId AS articleId,
  ka.title AS title,
  ka.issueType AS issueType

// =============================================================================
// RETURN LOADED ARTICLE SUMMARY
// =============================================================================
// RETURN defines what Neo4j should show after the query finishes.
//
// Here we return a small summary of the KnowledgeArticle nodes that were created
// or updated.
//
// We return:
//
// - ka.articleId AS articleId
// - ka.title AS title
// - ka.issueType AS issueType
//
// -----------------------------------------------------------------------------
// Why not return content?
// -----------------------------------------------------------------------------
// The content field may be long, so we do not return it in this summary.
//
// For verification, articleId, title, and issueType are usually enough to confirm
// that the correct articles were loaded.
//
// This keeps the output clean and readable.
//
// If we wanted to inspect the full content, we could also return:
//
//     ka.content AS content
//
// But for a lab/demo validation step, a compact summary is better.
//
// -----------------------------------------------------------------------------
// Why use aliases?
// -----------------------------------------------------------------------------
// The aliases make the output table easier to read.
//
// Instead of column names like:
//
//     ka.articleId
//     ka.title
//     ka.issueType
//
// Neo4j will show:
//
//     articleId
//     title
//     issueType
//
// This is cleaner for screenshots, documentation, and student understanding.

ORDER BY articleId;

// =============================================================================
// SORT THE OUTPUT
// =============================================================================
// ORDER BY articleId sorts the returned rows by the articleId column.
//
// Sorting is not required for the data load itself.
// The nodes would still be created or updated correctly without ORDER BY.
//
// But sorting makes the output easier to read and verify.
//
// Instead of appearing in an arbitrary or implementation-dependent order, the
// result will appear like:
//
//     articleId   title                    issueType
//     -----------------------------------------------------
//     K001        Fix login failure        Login Failure
//     K002        Resolve payment failure  Payment Failure
//     K003        Fix app crash            App Crash
//
// This is especially useful when:
// - Taking screenshots for a lab guide
// - Comparing expected vs actual output
// - Teaching students how to verify loaded data
// - Running the same query multiple times during practice
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query loads sample knowledge article data into Neo4j in an idempotent way.
//
// The main flow is:
//
//     1. UNWIND turns the inline list into individual rows.
//     2. MERGE finds or creates one KnowledgeArticle node per articleId.
//     3. SET updates the article properties.
//     4. RETURN shows a clean summary of what was loaded.
//     5. ORDER BY makes the verification output predictable.
//
// The most important production-style idea here is:
//
//     Use MERGE with a stable unique identifier,
//     and use SET to update descriptive properties.
//
// Because articleId is protected by a uniqueness constraint, this query becomes
// safe to rerun and helps prevent duplicate KnowledgeArticle nodes.
//
// This is a strong foundation for the next stages of the graph project, where
// these articles can be chunked, embedded, connected to issue types, and used in
// semantic search or RAG-style workflows.
```

# Step 9 — Verify loaded KnowledgeArticle nodes

```cypher
// =============================================================================
// VERIFY LOADED KNOWLEDGE ARTICLES
// =============================================================================
// This query reads the KnowledgeArticle nodes from the Neo4j database and shows
// a clean summary of the articles that were loaded earlier.
//
// In simple terms, it answers this question:
//
//     "Which KnowledgeArticle records currently exist in the graph,
//      and what are their key readable properties?"
//
// This query is usually run after an insert/load query to confirm that the data
// was created or updated successfully.
//
// Earlier, we loaded sample knowledge articles with properties such as:
//
// - articleId
// - title
// - issueType
// - content
// - source
//
// This verification query does not return the full article content because that
// can be long and noisy. Instead, it returns only the most useful fields for a
// quick validation check.
//
// This is useful for:
// - Confirming that KnowledgeArticle nodes exist
// - Checking that article IDs were stored correctly
// - Verifying titles and issue types
// - Confirming the source value
// - Producing clean output for screenshots or lab documentation
// - Making sure the data load query behaved as expected

MATCH (ka:KnowledgeArticle)

// =============================================================================
// MATCH KNOWLEDGE ARTICLE NODES
// =============================================================================
// MATCH tells Neo4j what pattern we want to find in the graph.
//
// Here the pattern is:
//
//     (ka:KnowledgeArticle)
//
// Let us break that down:
//
// - Parentheses () represent a node.
// - ka is the variable name assigned to each matched node.
// - KnowledgeArticle is the node label.
//
// So this line means:
//
//     "Find every node that has the label KnowledgeArticle,
//      and temporarily call each one ka."
//
// This is different from:
//
//     MATCH (ka)
//
// which would match every node in the database, regardless of label.
//
// By using:
//
//     MATCH (ka:KnowledgeArticle)
//
// we are being more precise. We only want knowledge article records, not chunks,
// entities, documents, users, tickets, or any other node type.
//
// This is a good habit because targeted queries are easier to understand and
// usually more efficient than unrestricted graph scans.
//
// -----------------------------------------------------------------------------
// Why this matters after loading data
// -----------------------------------------------------------------------------
// After running a data load query, we should not simply assume it worked.
//
// We should verify it.
//
// This MATCH query gives us direct confirmation that KnowledgeArticle nodes are
// present in the graph and available for later steps such as chunking, embedding,
// search, or relationship creation.

RETURN
  ka.articleId AS articleId,
  ka.title AS title,
  ka.issueType AS issueType,
  ka.source AS source

// =============================================================================
// RETURN A CLEAN ARTICLE SUMMARY
// =============================================================================
// RETURN defines what Neo4j should show in the final result table.
//
// Here we return four properties from each KnowledgeArticle node:
//
// 1. ka.articleId AS articleId
// 2. ka.title AS title
// 3. ka.issueType AS issueType
// 4. ka.source AS source
//
// -----------------------------------------------------------------------------
// ka.articleId AS articleId
// -----------------------------------------------------------------------------
// articleId is the unique business identifier for each knowledge article.
//
// For example:
//
//     K001
//     K002
//     K003
//
// This field is important because it identifies each article in a stable way.
//
// The title or content may change over time, but the articleId should remain the
// same so the application can reliably find and update the correct article.
//
// Earlier, we also created a uniqueness constraint for:
//
//     (:KnowledgeArticle).articleId
//
// That makes articleId a safe lookup key and helps prevent duplicate article
// records.
//
// -----------------------------------------------------------------------------
// ka.title AS title
// -----------------------------------------------------------------------------
// title is the short human-readable name of the article.
//
// For example:
//
//     Fix login failure
//     Resolve payment failure
//     Fix app crash
//
// This field helps users, developers, and support teams quickly understand what
// the article is about without reading the full content.
//
// -----------------------------------------------------------------------------
// ka.issueType AS issueType
// -----------------------------------------------------------------------------
// issueType tells us the category of customer problem that this article helps
// solve.
//
// For example:
//
//     Login Failure
//     Payment Failure
//     App Crash
//
// This field is useful for grouping, filtering, and later graph modeling.
//
// In a more advanced graph design, issueType could become its own node, such as:
//
//     (:IssueType {name: "Login Failure"})
//
// and the article could be connected to it with a relationship such as:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:IssueType)
//
// But at this stage, storing issueType as a property keeps the model simple and
// easy to verify.
//
// -----------------------------------------------------------------------------
// ka.source AS source
// -----------------------------------------------------------------------------
// source tells us where this knowledge article data came from.
//
// In our sample load, the source value was:
//
//     Day 3 sample knowledge base
//
// This is useful because source tracking helps us understand data lineage.
//
// Data lineage means knowing where a piece of data originated.
//
// In production systems, source may point to:
// - A support knowledge base
// - A document management system
// - A CRM system
// - A ticketing platform
// - A file name
// - An API feed
// - A batch ingestion job
//
// Keeping source information is a good production-grade habit because it helps
// with auditing, debugging, reprocessing, and explaining search results.
//
// -----------------------------------------------------------------------------
// Why use aliases?
// -----------------------------------------------------------------------------
// The AS keyword renames each returned expression into a clean column name.
//
// Without aliases, Neo4j may show column names like:
//
//     ka.articleId
//     ka.title
//     ka.issueType
//     ka.source
//
// With aliases, the output becomes:
//
//     articleId
//     title
//     issueType
//     source
//
// This is cleaner for demos, lab guides, screenshots, and student readability.
//
// -----------------------------------------------------------------------------
// Why not return content here?
// -----------------------------------------------------------------------------
// The content property contains the full troubleshooting text.
//
// That text is useful, but it can make the result table wide and harder to read.
//
// For a quick verification query, we usually return only identifying and summary
// fields.
//
// If we specifically wanted to inspect the full article content, we could add:
//
//     ka.content AS content
//
// But for this validation step, articleId, title, issueType, and source are
// enough.

ORDER BY articleId;

// =============================================================================
// SORT ARTICLES BY ARTICLE ID
// =============================================================================
// ORDER BY articleId sorts the final result by the articleId column.
//
// Sorting does not change the data stored in the database.
// It only controls how the result is displayed.
//
// This is important because without ORDER BY, Neo4j does not guarantee that rows
// will appear in a predictable order.
//
// By sorting on articleId, the output becomes stable and easy to compare.
//
// Expected output may look like:
//
//     articleId   title                    issueType          source
//     -------------------------------------------------------------------------
//     K001        Fix login failure        Login Failure      Day 3 sample knowledge base
//     K002        Resolve payment failure  Payment Failure    Day 3 sample knowledge base
//     K003        Fix app crash            App Crash          Day 3 sample knowledge base
//
// This predictable ordering is useful when:
// - Taking screenshots
// - Writing lab documentation
// - Comparing results before and after changes
// - Verifying that all expected sample records exist
// - Teaching students how to validate graph data step by step
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a data verification query.
//
// It does not create, update, or delete anything.
// It only reads existing KnowledgeArticle nodes and displays a clean summary.
//
// The main purpose is:
//
//     "Confirm that the KnowledgeArticle sample data exists
//      and that the key fields were stored correctly."
//
// The query follows a simple read pattern:
//
//     1. MATCH all KnowledgeArticle nodes.
//     2. RETURN the important summary fields.
//     3. ORDER the result by articleId for easy verification.
//
// This is a good habit in graph projects because after every data load step,
// we should run a validation query to prove that the graph now contains what we
// expected.
```

# Step 10 — Create SOLVES relationships

```cypher
// =============================================================================
// CONNECT KNOWLEDGE ARTICLES TO ISSUE NODES
// =============================================================================
// This query creates relationships between existing KnowledgeArticle nodes and
// existing Issue nodes.
//
// In simple terms, it answers this business question:
//
//     "Which issue does each knowledge article solve?"
//
// Earlier, each KnowledgeArticle had an issueType property, such as:
//
//     "Login Failure"
//     "Payment Failure"
//     "App Crash"
//
// Separately, the graph may also contain Issue nodes such as:
//
//     (:Issue {name: "Login Failure"})
//     (:Issue {name: "Payment Failure"})
//     (:Issue {name: "App Crash"})
//
// This query connects those two parts of the graph by creating a SOLVES
// relationship:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// That means:
//
//     "This knowledge article solves this issue."
//
// This is an important graph-modeling step because we are moving from simple
// property-based data to connected graph data.
//
// Instead of only storing issueType as text on the article, we now create an
// explicit relationship between the article and the issue it solves.
//
// This is useful for:
// - Traversing from issues to recommended knowledge articles
// - Finding which articles solve a given customer problem
// - Building customer support recommendation flows
// - Creating a more meaningful knowledge graph
// - Supporting graph-based search and reasoning
// - Preparing the data for RAG-style retrieval workflows

MATCH (ka:KnowledgeArticle)

// =============================================================================
// MATCH KNOWLEDGE ARTICLE NODES
// =============================================================================
// This first MATCH finds all nodes labelled KnowledgeArticle.
//
// Let us break down the pattern:
//
//     (ka:KnowledgeArticle)
//
// - Parentheses () represent a node.
// - ka is the variable name assigned to each matched node.
// - KnowledgeArticle is the label we are searching for.
//
// So this line means:
//
//     "Find every KnowledgeArticle node and temporarily call each one ka."
//
// Each matched article should already have properties such as:
//
// - ka.articleId
// - ka.title
// - ka.issueType
// - ka.content
// - ka.source
//
// The important property for this query is:
//
//     ka.issueType
//
// That value will be used to find the matching Issue node.
//
// For example, if an article has:
//
//     ka.issueType = "Login Failure"
//
// then the query will later try to find:
//
//     (:Issue {name: "Login Failure"})
//
// This is how the text property on the article becomes the bridge to a real
// graph node.

MATCH (i:Issue {name: ka.issueType})

// =============================================================================
// MATCH THE ISSUE NODE THAT CORRESPONDS TO THE ARTICLE ISSUE TYPE
// =============================================================================
// This second MATCH finds an Issue node whose name matches the issueType stored
// on the current KnowledgeArticle node.
//
// The pattern is:
//
//     (i:Issue {name: ka.issueType})
//
// Let us break it down:
//
// - i is the variable name assigned to the matched Issue node.
// - Issue is the node label.
// - name is the Issue node property being compared.
// - ka.issueType is the value from the current KnowledgeArticle node.
//
// In plain English, this means:
//
//     "For the current knowledge article, find the Issue node where
//      Issue.name is equal to KnowledgeArticle.issueType."
//
// Example:
//
// If the current article is:
//
//     (:KnowledgeArticle {
//       articleId: "K001",
//       title: "Fix login failure",
//       issueType: "Login Failure"
//     })
//
// then Neo4j will look for:
//
//     (:Issue {name: "Login Failure"})
//
// If that Issue node exists, the query continues and creates the relationship.
//
// If that Issue node does not exist, this MATCH will fail for that article row,
// and no SOLVES relationship will be created for that article.
//
// -----------------------------------------------------------------------------
// Important behavior of MATCH
// -----------------------------------------------------------------------------
// MATCH behaves like an inner join in relational database terms.
//
// That means both sides must exist:
//
// - The KnowledgeArticle must exist.
// - The matching Issue must exist.
//
// If a KnowledgeArticle has an issueType value that does not match any Issue.name,
// that article will be skipped by this query.
//
// This is useful when we only want to connect articles to valid, pre-existing
// Issue nodes.
//
// If we wanted to create the Issue node automatically when it is missing, we
// would use MERGE for the Issue node instead of MATCH.
//
// -----------------------------------------------------------------------------
// Why match by name?
// -----------------------------------------------------------------------------
// In this lab model, the KnowledgeArticle stores issueType as text, and the
// Issue node stores the same text in its name property.
//
// That gives us a simple mapping rule:
//
//     KnowledgeArticle.issueType = Issue.name
//
// In a production-grade model, we may prefer matching by a stable identifier,
// such as issueId, because names can change over time.
//
// For example:
//
//     "Login Failure"
//
// could later be renamed to:
//
//     "Customer Login Failure"
//
// If the relationship depends only on name matching, renaming can break the
// connection logic.
//
// But for this sample lab/demo dataset, matching issueType to Issue.name is
// simple, readable, and easy for students to understand.

MERGE (ka)-[:SOLVES]->(i)

// =============================================================================
// CREATE OR REUSE THE SOLVES RELATIONSHIP
// =============================================================================
// MERGE creates the relationship if it does not already exist.
//
// In simple terms, this line tells Neo4j:
//
//     "Make sure this KnowledgeArticle is connected to this Issue
//      with a SOLVES relationship."
//
// The pattern is:
//
//     (ka)-[:SOLVES]->(i)
//
// Let us break it down:
//
// - ka is the KnowledgeArticle node matched earlier.
// - i is the Issue node matched earlier.
// - [:SOLVES] is the relationship type.
// - -> shows the relationship direction.
//
// The direction means:
//
//     KnowledgeArticle SOLVES Issue
//
// Conceptually:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// This reads naturally:
//
//     "This article solves this issue."
//
// -----------------------------------------------------------------------------
// Why use MERGE instead of CREATE?
// -----------------------------------------------------------------------------
// CREATE would blindly create a new SOLVES relationship every time the query runs.
//
// That could produce duplicate relationships like:
//
//     (K001)-[:SOLVES]->(Login Failure)
//     (K001)-[:SOLVES]->(Login Failure)
//     (K001)-[:SOLVES]->(Login Failure)
//
// MERGE avoids that problem.
//
// MERGE means:
//
//     "If this exact relationship already exists, reuse it.
//      If it does not exist, create it."
//
// This makes the query idempotent.
//
// Idempotent means we can safely run the query multiple times without creating
// duplicate SOLVES relationships.
//
// This is very important for:
// - Lab guides
// - Repeatable setup scripts
// - CI/CD database initialization
// - Data refresh jobs
// - Production-safe graph loading
//
// -----------------------------------------------------------------------------
// Why relationship direction matters
// -----------------------------------------------------------------------------
// Neo4j relationships are stored with direction.
//
// Here, the direction is:
//
//     KnowledgeArticle -> Issue
//
// This direction is chosen because the sentence reads naturally:
//
//     "KnowledgeArticle SOLVES Issue"
//
// Later, we can query in either direction if needed.
//
// For example, from article to issue:
//
//     MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//     RETURN ka, i
//
// Or from issue to articles:
//
//     MATCH (i:Issue)<-[:SOLVES]-(ka:KnowledgeArticle)
//     RETURN i, collect(ka)
//
// Even though Neo4j stores direction, Cypher allows us to traverse relationships
// in either direction when the query asks for it.
//
// -----------------------------------------------------------------------------
// Why convert a property into a relationship?
// -----------------------------------------------------------------------------
// Before this query, the article only had:
//
//     ka.issueType = "Login Failure"
//
// That is useful, but it is just text.
//
// After this query, the graph has an explicit relationship:
//
//     (:KnowledgeArticle {articleId: "K001"})-[:SOLVES]->
//     (:Issue {name: "Login Failure"})
//
// This is more powerful because relationships can be traversed, counted,
// visualized, and used in graph algorithms or recommendation logic.
//
// A graph database becomes valuable when important business meaning is modeled
// as relationships, not only as isolated properties.

RETURN
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  ka.issueType AS articleIssueType,
  i.issueId AS issueId,
  i.name AS issueName

// =============================================================================
// RETURN ARTICLE-TO-ISSUE CONNECTION SUMMARY
// =============================================================================
// RETURN defines the output table shown after the relationships are created or
// reused.
//
// Here we return fields from both sides of the connection:
//
// From the KnowledgeArticle node:
// - ka.articleId AS articleId
// - ka.title AS articleTitle
// - ka.issueType AS articleIssueType
//
// From the Issue node:
// - i.issueId AS issueId
// - i.name AS issueName
//
// This output lets us verify that each article was connected to the correct
// issue.
//
// -----------------------------------------------------------------------------
// ka.articleId AS articleId
// -----------------------------------------------------------------------------
// articleId identifies the knowledge article.
//
// Example:
//
//     K001
//     K002
//     K003
//
// This helps us confirm which article row is being reported.
//
// -----------------------------------------------------------------------------
// ka.title AS articleTitle
// -----------------------------------------------------------------------------
// articleTitle shows the human-readable title of the article.
//
// Example:
//
//     Fix login failure
//
// This makes the output easier to understand than looking only at IDs.
//
// -----------------------------------------------------------------------------
// ka.issueType AS articleIssueType
// -----------------------------------------------------------------------------
// articleIssueType shows the original issueType property stored on the article.
//
// This is useful because it lets us compare the article's issueType value with
// the matched Issue node's name.
//
// Ideally:
//
//     articleIssueType = issueName
//
// For example:
//
//     Login Failure = Login Failure
//
// If these values do not match, the relationship logic should be reviewed.
//
// -----------------------------------------------------------------------------
// i.issueId AS issueId
// -----------------------------------------------------------------------------
// issueId identifies the Issue node.
//
// This is useful if Issue nodes have their own stable identifiers.
//
// For example:
//
//     I001
//     I002
//     I003
//
// In a production graph, stable IDs are usually safer than relying only on names
// because names can be edited or standardized later.
//
// -----------------------------------------------------------------------------
// i.name AS issueName
// -----------------------------------------------------------------------------
// issueName shows the name of the matched Issue node.
//
// This confirms which issue the article was connected to.
//
// Example:
//
//     Login Failure
//     Payment Failure
//     App Crash
//
// Together, articleIssueType and issueName prove that the property-to-node
// mapping worked correctly.

ORDER BY articleId;

// =============================================================================
// SORT THE RESULT BY ARTICLE ID
// =============================================================================
// ORDER BY articleId sorts the final output by the articleId column.
//
// Sorting does not affect the relationships created in the database.
// It only controls the display order of the returned rows.
//
// This makes the verification output predictable and easy to read.
//
// Expected output may look conceptually like:
//
//     articleId   articleTitle             articleIssueType   issueId   issueName
//     ------------------------------------------------------------------------------
//     K001        Fix login failure        Login Failure      I001      Login Failure
//     K002        Resolve payment failure  Payment Failure    I002      Payment Failure
//     K003        Fix app crash            App Crash          I003      App Crash
//
// This stable ordering is useful for:
// - Screenshots
// - Lab documentation
// - Comparing expected vs actual results
// - Teaching students how to verify graph relationships
// - Re-running the same query during demos without confusing output order
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query connects KnowledgeArticle nodes to Issue nodes using a SOLVES
// relationship.
//
// The main flow is:
//
//     1. MATCH all KnowledgeArticle nodes.
//     2. MATCH the Issue node whose name matches ka.issueType.
//     3. MERGE a SOLVES relationship from the article to the issue.
//     4. RETURN a clean summary of the created or existing connections.
//     5. ORDER the result by articleId for easy validation.
//
// The most important graph-modeling idea here is:
//
//     A property tells us something about a node,
//     but a relationship connects that node to another meaningful entity.
//
// Before this query:
//
//     (:KnowledgeArticle {issueType: "Login Failure"})
//
// After this query:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue {name: "Login Failure"})
//
// That relationship makes the graph more useful for traversal, recommendations,
// support workflows, and future RAG-style retrieval.
//
// Production note:
// The pasted query used -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the actual Cypher arrow:
//
//     MERGE (ka)-[:SOLVES]->(i)

```

# Step 11 — Verify SOLVES relationship count

```cypher
// =============================================================================
// COUNT SOLVES RELATIONSHIPS BETWEEN KNOWLEDGE ARTICLES AND ISSUES
// =============================================================================
// This query verifies how many SOLVES relationships currently exist between
// KnowledgeArticle nodes and Issue nodes.
//
// In simple terms, it answers this question:
//
//     "How many KnowledgeArticle-to-Issue SOLVES connections exist in the graph?"
//
// This is usually run after creating relationships like:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// The purpose is to confirm that the relationship creation step worked
// successfully.
//
// For example, if we loaded three knowledge articles and connected each one to
// one matching issue, we would expect the count to be:
//
//     SOLVES    3
//
// This query is useful for:
// - Verifying that SOLVES relationships were created
// - Checking whether the number of relationships matches expectations
// - Confirming that articles are actually connected to issues
// - Validating relationship creation in a lab guide or demo
// - Detecting missing links between knowledge articles and issue nodes
// - Preparing for more advanced traversal queries

MATCH (:KnowledgeArticle)-[r:SOLVES]->(:Issue)

// =============================================================================
// MATCH KNOWLEDGE ARTICLE TO ISSUE SOLVES RELATIONSHIPS
// =============================================================================
// MATCH defines the graph pattern we want Neo4j to find.
//
// The pattern is:
//
//     (:KnowledgeArticle)-[r:SOLVES]->(:Issue)
//
// Let us break it down piece by piece.
//
// -----------------------------------------------------------------------------
// (:KnowledgeArticle)
// -----------------------------------------------------------------------------
// This represents the starting node of the relationship.
//
// The label KnowledgeArticle means:
//
//     "Only match nodes that are knowledge articles."
//
// We do not assign this node to a variable such as ka because this query does
// not need to return article-level details like articleId or title.
//
// We only care about counting relationships.
//
// If we needed article details, we could write:
//
//     MATCH (ka:KnowledgeArticle)-[r:SOLVES]->(i:Issue)
//
// But for a simple count, anonymous nodes keep the query clean.
//
// -----------------------------------------------------------------------------
// [r:SOLVES]
// -----------------------------------------------------------------------------
// Square brackets represent a relationship in Cypher.
//
// The relationship is assigned to the variable r so that we can inspect it later
// using:
//
//     type(r)
//
// The relationship type is SOLVES.
//
// This means Neo4j will only match relationships whose type is exactly:
//
//     SOLVES
//
// It will not match other relationships such as:
//
//     HAS_CHUNK
//     MENTIONS
//     RELATED_TO
//     SIMILAR_TO
//
// This is important because we are specifically validating the article-to-issue
// relationship created in the previous step.
//
// -----------------------------------------------------------------------------
// -> 
// -----------------------------------------------------------------------------
// The arrow shows relationship direction.
//
// Here, the direction is:
//
//     KnowledgeArticle -> Issue
//
// This reads naturally as:
//
//     "A knowledge article solves an issue."
//
// Neo4j stores relationships with direction, so this pattern only matches SOLVES
// relationships going from KnowledgeArticle nodes to Issue nodes.
//
// If the relationship was accidentally created in the opposite direction:
//
//     (:Issue)-[:SOLVES]->(:KnowledgeArticle)
//
// then this query would not count it.
//
// That makes this query useful not only for checking whether relationships exist,
// but also for verifying that they were created in the intended direction.
//
// -----------------------------------------------------------------------------
// (:Issue)
// -----------------------------------------------------------------------------
// This represents the ending node of the relationship.
//
// The label Issue means:
//
//     "Only count SOLVES relationships that point to Issue nodes."
//
// This protects the query from counting a SOLVES relationship that may point to
// some other node type by mistake.
//
// For example, this query would count:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// But it would not count:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Ticket)
//     (:KnowledgeArticle)-[:SOLVES]->(:ProblemCategory)
//
// That makes the validation more precise.

RETURN
  type(r) AS relationshipType,
  count(*) AS relationshipCount;

// =============================================================================
// RETURN RELATIONSHIP TYPE AND COUNT
// =============================================================================
// RETURN defines what Neo4j should show in the final result table.
//
// Here we return two values:
//
//     1. type(r) AS relationshipType
//     2. count(*) AS relationshipCount
//
// -----------------------------------------------------------------------------
// type(r) AS relationshipType
// -----------------------------------------------------------------------------
// type(r) returns the relationship type of r.
//
// Since the MATCH pattern already restricts r to:
//
//     [r:SOLVES]
//
// the value of type(r) should be:
//
//     SOLVES
//
// Returning type(r) may seem redundant because we already know the relationship
// type, but it is useful for validation output.
//
// Instead of showing only a number like:
//
//     3
//
// the result clearly shows:
//
//     relationshipType    relationshipCount
//     -------------------------------------
//     SOLVES              3
//
// That makes the output easier to understand in screenshots, documentation, and
// lab guides.
//
// -----------------------------------------------------------------------------
// count(*) AS relationshipCount
// -----------------------------------------------------------------------------
// count(*) counts how many matched rows exist.
//
// In this query, each matched row represents one relationship pattern:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// So count(*) gives us the total number of SOLVES relationships from
// KnowledgeArticle nodes to Issue nodes.
//
// Because we also return type(r), Neo4j groups the results by relationship type.
//
// Since the query only matches SOLVES relationships, the output should normally
// contain one row.
//
// Example expected output:
//
//     relationshipType    relationshipCount
//     -------------------------------------
//     SOLVES              3
//
// If the count is 3, and we expected three knowledge articles to be connected
// to three issues, then the relationship creation step likely worked correctly.
//
// If the count is 0, it may mean:
// - The SOLVES relationships were not created.
// - The relationship direction is reversed.
// - The Issue nodes do not exist.
// - The KnowledgeArticle issueType values did not match Issue.name values.
// - The query is being run against a different database.
// - The pasted query used an escaped arrow instead of the real Cypher arrow.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a relationship validation query.
//
// It does not create, update, or delete anything.
// It only counts existing SOLVES relationships between KnowledgeArticle and
// Issue nodes.
//
// The main purpose is:
//
//     "Confirm how many KnowledgeArticle nodes are connected to Issue nodes
//      through SOLVES relationships."
//
// This is a good follow-up query after running:
//
//     MERGE (ka)-[:SOLVES]->(i)
//
// A healthy validation flow is:
//
//     1. Load KnowledgeArticle nodes.
//     2. Verify KnowledgeArticle nodes exist.
//     3. Match KnowledgeArticle.issueType to Issue.name.
//     4. Create SOLVES relationships.
//     5. Count SOLVES relationships to confirm the graph connections exist.
//
// Production note:
// The pasted query used -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the actual Cypher arrow:
//
//     MATCH (:KnowledgeArticle)-[r:SOLVES]->(:Issue)
```

# Step 12 — Verify readable KnowledgeArticle -> SOLVES -> Issue paths

```cypher
// =============================================================================
// VISUALIZE KNOWLEDGE ARTICLE TO ISSUE SOLVES PATHS
// =============================================================================
// This query retrieves the full graph path between KnowledgeArticle nodes and
// Issue nodes connected by the SOLVES relationship.
//
// In simple terms, it answers this question:
//
//     "Which knowledge articles solve which issues,
//      and can we visually inspect those connections as graph paths?"
//
// Earlier, we created relationships like:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// This query now reads those relationships back and returns both:
// - the full path, so Neo4j Browser can visualize the graph connection
// - selected properties, so we can inspect the article and issue details in a
//   readable table format
//
// This is useful for:
// - Visually confirming that SOLVES relationships exist
// - Checking that each article is connected to the correct issue
// - Reviewing article-to-issue mappings after relationship creation
// - Producing graph screenshots for lab documentation
// - Teaching how graph paths work in Neo4j
// - Preparing for more advanced traversals and recommendation queries

MATCH path = (ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// MATCH AND STORE THE ARTICLE-TO-ISSUE PATH
// =============================================================================
// MATCH tells Neo4j what graph pattern we want to find.
//
// Here the pattern is:
//
//     (ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// This means:
//
//     "Find KnowledgeArticle nodes that have a SOLVES relationship pointing
//      to Issue nodes."
//
// The important part is:
//
//     path = ...
//
// This assigns the entire matched pattern to a variable named path.
//
// So instead of only storing the individual nodes, Neo4j stores the full graph
// structure:
//
//     KnowledgeArticle node
//         -> SOLVES relationship
//         -> Issue node
//
// This is very useful in Neo4j Browser because returning a path allows Neo4j to
// display the result as a graph visualization, not only as table rows.
//
// -----------------------------------------------------------------------------
// Breaking down the pattern
// -----------------------------------------------------------------------------
//
//     (ka:KnowledgeArticle)
//
// This is the starting node.
//
// - Parentheses () represent a node.
// - ka is the variable name assigned to the node.
// - KnowledgeArticle is the node label.
//
// In plain English:
//
//     "Find a node that represents a knowledge article."
//
// Each matched article may have properties such as:
//
// - articleId
// - title
// - issueType
// - content
// - source
//
// -----------------------------------------------------------------------------
//
//     -[:SOLVES]->
//
// This is the relationship pattern.
//
// - Square brackets [] represent a relationship.
// - SOLVES is the relationship type.
// - The arrow -> shows the direction.
//
// In plain English:
//
//     "Follow a SOLVES relationship from the article to the issue."
//
// The direction matters because the business meaning is:
//
//     KnowledgeArticle SOLVES Issue
//
// This reads naturally as:
//
//     "This article solves this issue."
//
// If the relationship was accidentally created in the opposite direction:
//
//     (:Issue)-[:SOLVES]->(:KnowledgeArticle)
//
// then this query would not match it.
//
// That makes this query useful for verifying both:
// - the relationship type
// - the relationship direction
//
// -----------------------------------------------------------------------------
//
//     (i:Issue)
//
// This is the ending node.
//
// - i is the variable name assigned to the issue node.
// - Issue is the node label.
//
// In plain English:
//
//     "Find the issue that is solved by the knowledge article."
//
// Each matched issue may have properties such as:
//
// - issueId
// - name
// - severity
//
// -----------------------------------------------------------------------------
// Why return a path?
// -----------------------------------------------------------------------------
// Returning a path is different from returning only nodes or properties.
//
// If we return only:
//
//     ka, i
//
// Neo4j can show the two nodes, but the relationship may not always be as clear
// in the output table.
//
// If we return:
//
//     path
//
// Neo4j returns the complete connected structure:
//
//     (ka)-[:SOLVES]->(i)
//
// This is especially useful when teaching graph concepts because students can
// visually see the connection instead of only reading rows in a table.
//
// -----------------------------------------------------------------------------
// Why this is a validation query
// -----------------------------------------------------------------------------
// This query is normally run after relationship creation.
//
// It helps confirm:
//
// - KnowledgeArticle nodes exist.
// - Issue nodes exist.
// - SOLVES relationships exist.
// - The SOLVES relationships point in the correct direction.
// - The article is connected to the expected issue.
//
// If the query returns no rows, possible reasons include:
//
// - No KnowledgeArticle nodes exist.
// - No Issue nodes exist.
// - SOLVES relationships were not created.
// - SOLVES relationships were created in the opposite direction.
// - The query is running against the wrong database.
// - The pasted query used an escaped arrow instead of the real Cypher arrow.

RETURN
  path,
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS issueSeverity

// =============================================================================
// RETURN PATH AND READABLE DETAILS
// =============================================================================
// RETURN defines what Neo4j should show after matching the article-to-issue
// paths.
//
// Here we return both:
//
// 1. The full graph path
// 2. Selected article and issue properties
//
// This gives us the best of both worlds:
//
// - path gives us a visual graph representation.
// - the property columns give us a clean table for verification.
//
// -----------------------------------------------------------------------------
// path
// -----------------------------------------------------------------------------
// path returns the complete matched graph structure:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// In Neo4j Browser, this can be displayed visually as connected nodes and
// relationships.
//
// This is helpful because graph databases are not only about rows and columns.
// Their main strength is showing how entities are connected.
//
// Returning path helps us visually verify the model.
//
// -----------------------------------------------------------------------------
// ka.articleId AS articleId
// -----------------------------------------------------------------------------
// ka.articleId returns the unique identifier of the KnowledgeArticle node.
//
// Example:
//
//     K001
//     K002
//     K003
//
// This tells us which article is part of the matched path.
//
// We alias it as articleId so the output column is clean and easy to read.
//
// Instead of showing:
//
//     ka.articleId
//
// the result table shows:
//
//     articleId
//
// -----------------------------------------------------------------------------
// ka.title AS articleTitle
// -----------------------------------------------------------------------------
// ka.title returns the readable title of the knowledge article.
//
// Example:
//
//     Fix login failure
//     Resolve payment failure
//     Fix app crash
//
// This is useful because article IDs are precise, but titles are easier for
// humans to understand during validation and demos.
//
// We alias it as articleTitle to make it clear that this title belongs to the
// KnowledgeArticle side of the path.
//
// -----------------------------------------------------------------------------
// i.issueId AS issueId
// -----------------------------------------------------------------------------
// i.issueId returns the unique identifier of the Issue node.
//
// Example:
//
//     I001
//     I002
//     I003
//
// This helps confirm which issue node is connected to the article.
//
// In production-style graph models, stable IDs are important because display
// names can change over time, but IDs should remain consistent.
//
// -----------------------------------------------------------------------------
// i.name AS issueName
// -----------------------------------------------------------------------------
// i.name returns the readable name of the issue.
//
// Example:
//
//     Login Failure
//     Payment Failure
//     App Crash
//
// This helps us verify that the article is connected to the correct business
// issue.
//
// For example, we expect:
//
//     K001 / Fix login failure
//
// to connect to:
//
//     Login Failure
//
// If the article title and issue name do not logically match, then the earlier
// relationship creation logic should be reviewed.
//
// -----------------------------------------------------------------------------
// i.severity AS issueSeverity
// -----------------------------------------------------------------------------
// i.severity returns the severity level stored on the Issue node.
//
// Example values might be:
//
//     High
//     Medium
//     Low
//     Critical
//
// Severity is useful because it gives business priority to the issue.
//
// For example, if an issue has high severity, a support system may recommend
// its related knowledge article more urgently.
//
// Including severity in the output helps us inspect not only the article-to-issue
// connection, but also the operational importance of that issue.
//
// -----------------------------------------------------------------------------
// Why return both graph and table data?
// -----------------------------------------------------------------------------
// Returning only path is good for visualization.
//
// Returning only properties is good for tabular validation.
//
// Returning both is ideal for a lab/demo because we can:
//
// - See the graph connection visually.
// - Read the important values clearly.
// - Take useful screenshots.
// - Explain both graph structure and business meaning together.

ORDER BY articleId;

// =============================================================================
// SORT THE RESULT BY ARTICLE ID
// =============================================================================
// ORDER BY articleId sorts the final output by the articleId column.
//
// Sorting does not change the graph data.
// It only controls the display order of the returned rows.
//
// Without ORDER BY, Neo4j may return rows in an order that is not predictable.
//
// By sorting on articleId, the output becomes easier to verify.
//
// Expected output may look conceptually like:
//
//     articleId   articleTitle             issueId   issueName          issueSeverity
//     -------------------------------------------------------------------------------
//     K001        Fix login failure        I001      Login Failure      High
//     K002        Resolve payment failure  I002      Payment Failure    Medium
//     K003        Fix app crash            I003      App Crash          High
//
// This stable ordering is useful for:
//
// - Lab guide screenshots
// - Comparing expected and actual output
// - Teaching students step by step
// - Re-running the query during demos
// - Confirming all expected article-to-issue paths exist
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a graph path verification query.
//
// It does not create, update, or delete anything.
// It only reads existing paths between KnowledgeArticle and Issue nodes.
//
// The main purpose is:
//
//     "Show the actual graph connections where knowledge articles solve issues."
//
// The query follows this flow:
//
//     1. MATCH a path from KnowledgeArticle to Issue through SOLVES.
//     2. Store the complete matched structure in the variable path.
//     3. RETURN the visual path plus useful article and issue fields.
//     4. ORDER the output by articleId for clean validation.
//
// The key graph concept here is:
//
//     A path is not just a node.
//     A path includes the node, the relationship, and the connected node.
//
// So this query helps students understand that Neo4j is designed to answer
// connected-data questions, not just return isolated records.
//
// Production note:
// The pasted query used -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the actual Cypher arrow:
//
//     MATCH path = (ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
```

# Step 13 — Create DocumentChunk nodes and connect them to articles

```cypher
// =============================================================================
// LOAD DOCUMENT CHUNKS AND CONNECT THEM TO KNOWLEDGE ARTICLES
// =============================================================================
// This query loads sample document chunks into Neo4j and connects each chunk
// back to the KnowledgeArticle it belongs to.
//
// In simple terms, it answers this data-modeling requirement:
//
//     "Take each knowledge article, split it into smaller searchable chunks,
//      store those chunks as DocumentChunk nodes, and connect each chunk back
//      to its parent KnowledgeArticle."
//
// This is an important step in document-based graph and RAG-style systems.
//
// A full knowledge article can be long. Instead of storing and searching only
// the complete article text, we often split the article into smaller pieces
// called chunks.
//
// These chunks are useful because they can later be:
// - embedded into vector representations
// - searched semantically
// - retrieved as focused context for an LLM
// - connected to entities, topics, issues, or source documents
// - traced back to the original article
//
// The graph model created by this query looks like:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)
//
// This relationship means:
//
//     "This chunk is part of this knowledge article."
//
// This query is useful for:
// - Creating chunk-level search units
// - Preserving traceability from chunk to article
// - Preparing data for vector indexes
// - Supporting RAG-style retrieval
// - Validating document ingestion logic
// - Demonstrating how text content becomes graph-connected data

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

// =============================================================================
// UNWIND CHUNK DATA INTO INDIVIDUAL ROWS
// =============================================================================
// UNWIND takes the list of chunk maps above and expands it into one row per
// chunk.
//
// Think of the list as a small in-memory table.
//
// Before UNWIND, Neo4j sees one list containing six maps:
//
//     [
//       {chunkId: "C-K001-001", articleId: "K001", ...},
//       {chunkId: "C-K001-002", articleId: "K001", ...},
//       ...
//     ]
//
// After UNWIND, Neo4j processes the data row by row:
//
//     Row 1 -> chunk = {chunkId: "C-K001-001", articleId: "K001", ...}
//     Row 2 -> chunk = {chunkId: "C-K001-002", articleId: "K001", ...}
//     Row 3 -> chunk = {chunkId: "C-K002-001", articleId: "K002", ...}
//     Row 4 -> chunk = {chunkId: "C-K002-002", articleId: "K002", ...}
//     Row 5 -> chunk = {chunkId: "C-K003-001", articleId: "K003", ...}
//     Row 6 -> chunk = {chunkId: "C-K003-002", articleId: "K003", ...}
//
// This row-by-row behavior is important because the MATCH, MERGE, SET, and
// relationship creation steps that follow will run once for each chunk.
//
// -----------------------------------------------------------------------------
// What each chunk map represents
// -----------------------------------------------------------------------------
// Each map represents one smaller piece of a larger knowledge article.
//
// Each chunk contains:
//
// - chunkId:
//   A unique identifier for the chunk.
//
// - articleId:
//   The ID of the parent KnowledgeArticle this chunk belongs to.
//
// - chunkOrder:
//   The position of this chunk inside the article.
//
// - text:
//   The actual text content of the chunk.
//
// For example:
//
//     chunkId: "C-K001-001"
//
// can be read as:
//
//     "Chunk 001 belonging to article K001."
//
// This naming convention is useful because it makes the chunk identity readable
// and traceable even before looking at relationships.
//
// -----------------------------------------------------------------------------
// Why chunking matters
// -----------------------------------------------------------------------------
// In RAG and semantic search systems, we usually do not embed or retrieve one
// very large document as a single unit.
//
// Instead, we split the document into smaller chunks so that retrieval can find
// the most relevant portion of text.
//
// For example, if the user asks about OTP delivery, the system should retrieve
// the chunk that talks about OTP delivery, not necessarily the entire login
// failure article.
//
// Chunking improves:
// - search precision
// - retrieval relevance
// - LLM context quality
// - traceability
// - update flexibility
//
// In this query, each chunk becomes its own DocumentChunk node.

MATCH (ka:KnowledgeArticle {articleId: chunk.articleId})

// =============================================================================
// MATCH THE PARENT KNOWLEDGE ARTICLE
// =============================================================================
// This MATCH finds the KnowledgeArticle node that each chunk belongs to.
//
// The pattern is:
//
//     (ka:KnowledgeArticle {articleId: chunk.articleId})
//
// Let us break it down:
//
// - ka is the variable name assigned to the matched KnowledgeArticle node.
// - KnowledgeArticle is the label we are searching for.
// - articleId is the property used to identify the article.
// - chunk.articleId is the articleId value from the current chunk row.
//
// In plain English:
//
//     "For the current chunk, find the KnowledgeArticle whose articleId matches
//      the chunk's articleId."
//
// Example:
//
// For this chunk:
//
//     {
//       chunkId: "C-K001-001",
//       articleId: "K001",
//       ...
//     }
//
// Neo4j tries to find:
//
//     (:KnowledgeArticle {articleId: "K001"})
//
// If that KnowledgeArticle exists, the query continues and creates or updates
// the DocumentChunk node.
//
// If that KnowledgeArticle does not exist, this MATCH fails for that chunk row,
// and Neo4j will not create the chunk for that row.
//
// -----------------------------------------------------------------------------
// Important behavior of MATCH
// -----------------------------------------------------------------------------
// MATCH works like an inner join.
//
// That means the parent KnowledgeArticle must already exist.
//
// This is intentional because a chunk should not be created as an orphan if its
// parent article does not exist.
//
// An orphan chunk would be a DocumentChunk node that has an articleId property
// but no actual relationship to a KnowledgeArticle node.
//
// By matching the parent article first, this query protects graph quality.
//
// -----------------------------------------------------------------------------
// Why match by articleId?
// -----------------------------------------------------------------------------
// articleId is the stable identifier of the KnowledgeArticle.
//
// Earlier, we created or expected a uniqueness constraint on:
//
//     (:KnowledgeArticle).articleId
//
// That makes articleId a safe lookup key.
//
// Matching by articleId is better than matching by title because titles can
// change over time.
//
// For example:
//
//     "Fix login failure"
//
// may later become:
//
//     "Troubleshoot customer login failure"
//
// But the articleId:
//
//     K001
//
// should remain stable.
//
// This is a production-style modeling habit:
// use stable IDs for matching, and use display fields only for readability.

MERGE (dc:DocumentChunk {chunkId: chunk.chunkId})

// =============================================================================
// CREATE OR REUSE THE DOCUMENT CHUNK NODE
// =============================================================================
// MERGE creates the DocumentChunk node if it does not already exist, or reuses it
// if it already exists.
//
// In simple terms, this line tells Neo4j:
//
//     "Find a DocumentChunk with this chunkId.
//      If it exists, use it.
//      If it does not exist, create it."
//
// The pattern is:
//
//     (dc:DocumentChunk {chunkId: chunk.chunkId})
//
// Let us break it down:
//
// - dc is the variable name assigned to the DocumentChunk node.
// - DocumentChunk is the label.
// - chunkId is the identity property for the chunk.
// - chunk.chunkId comes from the current UNWIND row.
//
// Example:
//
// For the first row, Neo4j effectively runs:
//
//     MERGE (dc:DocumentChunk {chunkId: "C-K001-001"})
//
// -----------------------------------------------------------------------------
// Why use MERGE instead of CREATE?
// -----------------------------------------------------------------------------
// CREATE would create a new DocumentChunk node every time the query is run.
//
// That would produce duplicates if the script is executed multiple times.
//
// MERGE prevents duplicates by making the load idempotent.
//
// Idempotent means:
//
//     "You can safely run the same query multiple times,
//      and the final graph will remain logically correct."
//
// This is especially important for:
// - lab guides
// - demos
// - CI/CD setup scripts
// - repeatable data loads
// - ingestion pipelines
// - production refresh jobs
//
// -----------------------------------------------------------------------------
// Relationship with chunkId uniqueness constraint
// -----------------------------------------------------------------------------
// Earlier, we created or expected a uniqueness constraint on:
//
//     (:DocumentChunk).chunkId
//
// That constraint supports this MERGE pattern.
//
// Together, MERGE and the uniqueness constraint give us:
//
// - safe chunk creation
// - duplicate prevention
// - reliable chunk lookup
// - faster lookup by chunkId
// - production-grade data consistency

SET
  dc.text = chunk.text,
  dc.chunkOrder = chunk.chunkOrder,
  dc.articleId = chunk.articleId,
  dc.source = "Day 3 sample document chunks"

// =============================================================================
// SET OR UPDATE DOCUMENT CHUNK PROPERTIES
// =============================================================================
// SET assigns properties to the DocumentChunk node.
//
// At this point, dc refers to the DocumentChunk node found or created by MERGE.
//
// The SET clause copies values from the current chunk map onto the node.
//
// -----------------------------------------------------------------------------
// dc.text = chunk.text
// -----------------------------------------------------------------------------
// This stores the actual chunk text.
//
// This is the most important content field on the DocumentChunk node.
//
// In a RAG-style system, this text is the unit that may later be:
// - embedded into a vector
// - stored in a vector index
// - retrieved during semantic search
// - passed to an LLM as context
// - displayed as supporting evidence or citation
//
// For example:
//
//     "If a customer cannot sign in, ask them to reset the password..."
//
// becomes searchable chunk-level knowledge.
//
// -----------------------------------------------------------------------------
// dc.chunkOrder = chunk.chunkOrder
// -----------------------------------------------------------------------------
// This stores the chunk's position inside the parent article.
//
// For example:
//
//     chunkOrder = 1
//     chunkOrder = 2
//
// This is important because when chunks are retrieved or displayed, we may need
// to reconstruct the original article order.
//
// Without chunkOrder, we may know that multiple chunks belong to the same
// article, but not the sequence in which they appeared.
//
// This matters for readability and context reconstruction.
//
// -----------------------------------------------------------------------------
// dc.articleId = chunk.articleId
// -----------------------------------------------------------------------------
// This stores the parent articleId directly on the chunk.
//
// Even though we also create a PART_OF relationship to the KnowledgeArticle,
// keeping articleId as a property can be useful for:
//
// - quick filtering
// - simple debugging
// - export jobs
// - joining with external systems
// - validating chunk lineage
//
// However, the relationship is still the richer graph-modeling structure.
//
// The property tells us the parent article ID.
// The relationship lets us traverse to the actual parent article node.
//
// -----------------------------------------------------------------------------
// dc.source = "Day 3 sample document chunks"
// -----------------------------------------------------------------------------
// This stores source metadata on every chunk created by this query.
//
// Source metadata helps us understand where the chunk came from.
//
// In production systems, source may include:
// - file name
// - document URL
// - ingestion batch ID
// - API source
// - pipeline name
// - environment name
// - timestamp or version
//
// Here, the value:
//
//     "Day 3 sample document chunks"
//
// clearly tells us that these chunks are part of the Day 3 lab/demo dataset.
//
// -----------------------------------------------------------------------------
// Why use SET instead of ON CREATE SET only?
// -----------------------------------------------------------------------------
// This query uses SET so that the chunk properties are updated every time the
// query runs.
//
// This is useful during labs because if we change the text, chunk order, or
// source value, rerunning the query refreshes the existing nodes.
//
// In production, we may use a more detailed pattern:
//
//     ON CREATE SET dc.createdAt = datetime(), ...
//     ON MATCH SET  dc.updatedAt = datetime(), ...
//
// That allows us to track when a chunk was first created and when it was last
// updated.
//
// For this lab, simple SET keeps the query clean and easy to understand.

MERGE (dc)-[:PART_OF]->(ka)

// =============================================================================
// CREATE OR REUSE THE PART_OF RELATIONSHIP
// =============================================================================
// MERGE creates the relationship between the DocumentChunk and its parent
// KnowledgeArticle if it does not already exist.
//
// The relationship pattern is:
//
//     (dc)-[:PART_OF]->(ka)
//
// This means:
//
//     "This DocumentChunk is part of this KnowledgeArticle."
//
// The direction reads naturally:
//
//     DocumentChunk PART_OF KnowledgeArticle
//
// Example:
//
//     (:DocumentChunk {chunkId: "C-K001-001"})
//         -[:PART_OF]->
//     (:KnowledgeArticle {articleId: "K001"})
//
// -----------------------------------------------------------------------------
// Why create this relationship?
// -----------------------------------------------------------------------------
// The relationship is what turns separate pieces of data into a graph.
//
// Before this relationship, the chunk only has a property:
//
//     dc.articleId = "K001"
//
// That is useful, but it is just text.
//
// After this relationship, Neo4j knows the actual graph connection:
//
//     chunk -> parent article
//
// This allows traversal queries such as:
//
//     MATCH (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)
//     RETURN dc.text, ka.title
//
// Or from article to all chunks:
//
//     MATCH (ka:KnowledgeArticle)<-[:PART_OF]-(dc:DocumentChunk)
//     RETURN ka.title, dc.text
//     ORDER BY dc.chunkOrder
//
// This is the power of graph modeling:
// important business meaning is represented as relationships, not only as
// properties.
//
// -----------------------------------------------------------------------------
// Why use MERGE instead of CREATE for the relationship?
// -----------------------------------------------------------------------------
// CREATE would create a new PART_OF relationship every time the query runs.
//
// That could result in duplicate relationships like:
//
//     (C-K001-001)-[:PART_OF]->(K001)
//     (C-K001-001)-[:PART_OF]->(K001)
//     (C-K001-001)-[:PART_OF]->(K001)
//
// MERGE prevents duplicates by reusing the relationship if it already exists.
//
// This makes the query safe to rerun.
//
// -----------------------------------------------------------------------------
// Why relationship direction matters
// -----------------------------------------------------------------------------
// The direction chosen here is:
//
//     DocumentChunk -> KnowledgeArticle
//
// This reads as:
//
//     "The chunk is part of the article."
//
// Later, we can still traverse in the reverse direction when needed:
//
//     MATCH (ka:KnowledgeArticle)<-[:PART_OF]-(dc:DocumentChunk)
//
// That query reads as:
//
//     "Find chunks that are part of this article."
//
// Neo4j stores direction, but Cypher allows us to query in either direction if
// we specify the pattern correctly.

RETURN
  dc.chunkId AS chunkId,
  dc.articleId AS articleId,
  dc.chunkOrder AS chunkOrder,
  ka.title AS articleTitle

// =============================================================================
// RETURN LOADED CHUNK SUMMARY
// =============================================================================
// RETURN defines what Neo4j should show after the chunk load and relationship
// creation finishes.
//
// Here we return a compact validation summary:
//
// - dc.chunkId AS chunkId
// - dc.articleId AS articleId
// - dc.chunkOrder AS chunkOrder
// - ka.title AS articleTitle
//
// This output confirms both:
// - the chunk was created or updated
// - the chunk was matched to the correct KnowledgeArticle
//
// -----------------------------------------------------------------------------
// dc.chunkId AS chunkId
// -----------------------------------------------------------------------------
// chunkId identifies the individual DocumentChunk.
//
// Example:
//
//     C-K001-001
//     C-K001-002
//
// This helps verify that each expected chunk exists.
//
// -----------------------------------------------------------------------------
// dc.articleId AS articleId
// -----------------------------------------------------------------------------
// articleId shows which article the chunk belongs to.
//
// Example:
//
//     K001
//
// This is useful for checking chunk lineage.
//
// If a chunk appears under the wrong articleId, the input data or MATCH logic
// should be reviewed.
//
// -----------------------------------------------------------------------------
// dc.chunkOrder AS chunkOrder
// -----------------------------------------------------------------------------
// chunkOrder shows the sequence of the chunk within the parent article.
//
// Example:
//
//     1
//     2
//
// This helps confirm that the chunks are ordered correctly.
//
// Correct ordering matters when reconstructing the article or showing retrieved
// chunks in a meaningful sequence.
//
// -----------------------------------------------------------------------------
// ka.title AS articleTitle
// -----------------------------------------------------------------------------
// articleTitle comes from the matched KnowledgeArticle node.
//
// This is important because it proves that the chunk was not only assigned an
// articleId property, but was also connected to an actual KnowledgeArticle node.
//
// For example:
//
//     C-K001-001 -> K001 -> Fix login failure
//
// That gives us a human-readable validation result.

ORDER BY articleId, chunkOrder;

// =============================================================================
// SORT OUTPUT BY ARTICLE AND CHUNK ORDER
// =============================================================================
// ORDER BY controls the display order of the returned rows.
//
// Here we sort by:
//
//     articleId, chunkOrder
//
// This means:
//
//     1. Group chunks by their parent article.
//     2. Within each article, show chunks in their original sequence.
//
// Expected output may look conceptually like:
//
//     chunkId       articleId   chunkOrder   articleTitle
//     -----------------------------------------------------------
//     C-K001-001    K001        1            Fix login failure
//     C-K001-002    K001        2            Fix login failure
//     C-K002-001    K002        1            Resolve payment failure
//     C-K002-002    K002        2            Resolve payment failure
//     C-K003-001    K003        1            Fix app crash
//     C-K003-002    K003        2            Fix app crash
//
// Sorting is especially useful for:
// - lab screenshots
// - validation output
// - comparing expected vs actual results
// - teaching how chunks belong to articles
// - checking whether chunkOrder was stored correctly
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query loads chunk-level text data and connects each chunk to its parent
// KnowledgeArticle.
//
// The main flow is:
//
//     1. UNWIND turns the inline chunk list into individual rows.
//     2. MATCH finds the parent KnowledgeArticle for each chunk.
//     3. MERGE creates or reuses the DocumentChunk node by chunkId.
//     4. SET updates the chunk properties.
//     5. MERGE creates or reuses the PART_OF relationship.
//     6. RETURN shows a clean validation summary.
//     7. ORDER BY groups chunks by article and displays them in sequence.
//
// The most important graph-modeling idea here is:
//
//     The chunk text is stored as a node,
//     and its relationship to the source article is stored explicitly.
//
// Before this query, the graph had article-level knowledge.
//
// After this query, the graph has chunk-level knowledge connected back to the
// article-level source.
//
// This is a strong foundation for vector search and RAG because the system can
// retrieve a specific relevant chunk while still tracing it back to the complete
// knowledge article.
//
// Production note:
// The pasted query used -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the actual Cypher arrow:
//
//     MERGE (dc)-[:PART_OF]->(ka)
```

# Step 14 — Verify PART_OF relationship count

```cypher
// =============================================================================
// COUNT PART_OF RELATIONSHIPS BETWEEN DOCUMENT CHUNKS AND KNOWLEDGE ARTICLES
// =============================================================================
// This query verifies how many PART_OF relationships currently exist between
// DocumentChunk nodes and KnowledgeArticle nodes.
//
// In simple terms, it answers this question:
//
//     "How many document chunks are connected back to their parent knowledge
//      articles?"
//
// This is usually run after creating relationships like:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)
//
// The purpose is to confirm that the chunk-to-article relationship creation step
// worked successfully.
//
// For example, if we loaded six document chunks and each chunk was connected to
// one parent knowledge article, we would expect the count to be:
//
//     PART_OF    6
//
// This query is useful for:
// - Verifying that PART_OF relationships were created
// - Checking whether every chunk was linked to a parent article
// - Confirming that chunk lineage is preserved
// - Detecting orphan chunks that may not be connected to articles
// - Validating document ingestion logic
// - Preparing the graph for vector search and RAG-style retrieval
// - Documenting the database state in a lab guide or demo

MATCH (:DocumentChunk)-[r:PART_OF]->(:KnowledgeArticle)

// =============================================================================
// MATCH DOCUMENT CHUNK TO KNOWLEDGE ARTICLE RELATIONSHIPS
// =============================================================================
// MATCH defines the graph pattern we want Neo4j to find.
//
// The pattern is:
//
//     (:DocumentChunk)-[r:PART_OF]->(:KnowledgeArticle)
//
// Let us break it down piece by piece.
//
// -----------------------------------------------------------------------------
// (:DocumentChunk)
// -----------------------------------------------------------------------------
// This represents the starting node of the relationship.
//
// The label DocumentChunk means:
//
//     "Only match nodes that represent document chunks."
//
// We do not assign this node to a variable such as dc because this query does
// not need to return chunk-level details like chunkId, text, or chunkOrder.
//
// We only care about counting relationships.
//
// If we wanted to inspect chunk details, we could write:
//
//     MATCH (dc:DocumentChunk)-[r:PART_OF]->(ka:KnowledgeArticle)
//
// But for a simple relationship count, anonymous nodes keep the query clean.
//
// -----------------------------------------------------------------------------
// [r:PART_OF]
// -----------------------------------------------------------------------------
// Square brackets represent a relationship in Cypher.
//
// The relationship is assigned to the variable r so that we can inspect it later
// using:
//
//     type(r)
//
// The relationship type is PART_OF.
//
// This means Neo4j will only match relationships whose type is exactly:
//
//     PART_OF
//
// It will not match other relationships such as:
//
//     SOLVES
//     HAS_CHUNK
//     MENTIONS
//     RELATED_TO
//     SIMILAR_TO
//
// This is important because we are specifically validating the relationship
// created between chunks and their parent articles.
//
// -----------------------------------------------------------------------------
// ->
// -----------------------------------------------------------------------------
// The arrow shows relationship direction.
//
// Here, the direction is:
//
//     DocumentChunk -> KnowledgeArticle
//
// This reads naturally as:
//
//     "A document chunk is part of a knowledge article."
//
// Neo4j stores relationships with direction, so this pattern only matches
// PART_OF relationships going from DocumentChunk nodes to KnowledgeArticle nodes.
//
// If the relationship was accidentally created in the opposite direction:
//
//     (:KnowledgeArticle)-[:PART_OF]->(:DocumentChunk)
//
// then this query would not count it.
//
// That makes this query useful not only for checking whether relationships exist,
// but also for verifying that they were created in the intended direction.
//
// -----------------------------------------------------------------------------
// (:KnowledgeArticle)
// -----------------------------------------------------------------------------
// This represents the ending node of the relationship.
//
// The label KnowledgeArticle means:
//
//     "Only count PART_OF relationships that point to KnowledgeArticle nodes."
//
// This protects the query from counting a PART_OF relationship that may point to
// some other node type by mistake.
//
// For example, this query would count:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)
//
// But it would not count:
//
//     (:DocumentChunk)-[:PART_OF]->(:Document)
//     (:DocumentChunk)-[:PART_OF]->(:Issue)
//     (:DocumentChunk)-[:PART_OF]->(:Topic)
//
// That makes the validation more precise.

RETURN
  type(r) AS relationshipType,
  count(*) AS relationshipCount;

// =============================================================================
// RETURN RELATIONSHIP TYPE AND COUNT
// =============================================================================
// RETURN defines what Neo4j should show in the final result table.
//
// Here we return two values:
//
//     1. type(r) AS relationshipType
//     2. count(*) AS relationshipCount
//
// -----------------------------------------------------------------------------
// type(r) AS relationshipType
// -----------------------------------------------------------------------------
// type(r) returns the relationship type of r.
//
// Since the MATCH pattern already restricts r to:
//
//     [r:PART_OF]
//
// the value of type(r) should be:
//
//     PART_OF
//
// Returning type(r) may seem obvious because we already know the relationship
// type, but it makes the validation output much clearer.
//
// Instead of showing only a number like:
//
//     6
//
// the result clearly shows:
//
//     relationshipType    relationshipCount
//     -------------------------------------
//     PART_OF             6
//
// This is much better for screenshots, lab documentation, and student
// understanding because the result explains what is being counted.
//
// -----------------------------------------------------------------------------
// count(*) AS relationshipCount
// -----------------------------------------------------------------------------
// count(*) counts how many matched rows exist.
//
// In this query, each matched row represents one relationship pattern:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)
//
// So count(*) gives us the total number of PART_OF relationships from
// DocumentChunk nodes to KnowledgeArticle nodes.
//
// Because we also return type(r), Neo4j groups the results by relationship type.
//
// Since the query only matches PART_OF relationships, the output should normally
// contain one row.
//
// Example expected output:
//
//     relationshipType    relationshipCount
//     -------------------------------------
//     PART_OF             6
//
// If the count is 6, and we expected six chunks to be connected to knowledge
// articles, then the chunk relationship creation step likely worked correctly.
//
// If the count is 0, it may mean:
// - The PART_OF relationships were not created.
// - The relationship direction is reversed.
// - The DocumentChunk nodes do not exist.
// - The KnowledgeArticle nodes do not exist.
// - The chunk.articleId values did not match KnowledgeArticle.articleId values.
// - The query is being run against a different database.
// - The pasted query used an escaped arrow instead of the real Cypher arrow.
//
// -----------------------------------------------------------------------------
// Why this count matters in RAG-style systems
// -----------------------------------------------------------------------------
// In a RAG or vector-search workflow, DocumentChunk nodes often become the main
// retrieval unit.
//
// When a chunk is retrieved, the system usually needs to trace it back to:
// - the parent knowledge article
// - the article title
// - the original source
// - the issue or business topic it supports
//
// The PART_OF relationship enables that traceability.
//
// Without this relationship, chunks may still exist as standalone text nodes,
// but the graph loses the ability to easily answer:
//
//     "Which article did this chunk come from?"
//
// That is why this validation query is important before moving into embedding,
// vector indexing, or semantic search steps.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a relationship validation query.
//
// It does not create, update, or delete anything.
// It only counts existing PART_OF relationships between DocumentChunk and
// KnowledgeArticle nodes.
//
// The main purpose is:
//
//     "Confirm how many document chunks are connected to their parent knowledge
//      articles through PART_OF relationships."
//
// This is a good follow-up query after running:
//
//     MERGE (dc)-[:PART_OF]->(ka)
//
// A healthy validation flow is:
//
//     1. Load KnowledgeArticle nodes.
//     2. Load DocumentChunk nodes.
//     3. Connect each DocumentChunk to its parent KnowledgeArticle.
//     4. Count PART_OF relationships to confirm the graph connections exist.
//     5. Later, use these relationships for retrieval, traceability, and RAG.
//
// Production note:
// The pasted query used -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the actual Cypher arrow:
//
//     MATCH (:DocumentChunk)-[r:PART_OF]->(:KnowledgeArticle)
``
```

# Step 15 — Verify readable DocumentChunk -> PART_OF -> KnowledgeArticle paths

```cypher
// =============================================================================
// VISUALIZE DOCUMENT CHUNKS CONNECTED TO KNOWLEDGE ARTICLES
// =============================================================================
// This query retrieves the full graph path between DocumentChunk nodes and their
// parent KnowledgeArticle nodes.
//
// In simple terms, it answers this question:
//
//     "Which chunks belong to which knowledge articles,
//      and can we visually inspect those chunk-to-article connections?"
//
// Earlier, we created relationships like:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)
//
// This query now reads those relationships back and returns both:
//
// - the full path, so Neo4j Browser can visualize the graph connection
// - selected properties, so we can inspect chunk and article details in a
//   readable table format
//
// This is useful for:
// - Visually confirming that PART_OF relationships exist
// - Checking that each chunk is connected to the correct article
// - Reviewing chunk order within each article
// - Inspecting the chunk text that will later be embedded or searched
// - Producing graph screenshots for lab documentation
// - Preparing the graph for vector search and RAG-style retrieval

MATCH path = (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)

// =============================================================================
// MATCH AND STORE THE CHUNK-TO-ARTICLE PATH
// =============================================================================
// MATCH tells Neo4j what graph pattern we want to find.
//
// Here the pattern is:
//
//     (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)
//
// This means:
//
//     "Find DocumentChunk nodes that have a PART_OF relationship pointing
//      to KnowledgeArticle nodes."
//
// The important part is:
//
//     path = ...
//
// This assigns the entire matched pattern to a variable named path.
//
// So instead of only storing the individual nodes, Neo4j stores the full graph
// structure:
//
//     DocumentChunk node
//         -> PART_OF relationship
//         -> KnowledgeArticle node
//
// Returning this full path is especially useful in Neo4j Browser because it can
// display the result visually as connected graph objects.
//
// -----------------------------------------------------------------------------
// Breaking down the pattern
// -----------------------------------------------------------------------------
//
//     (dc:DocumentChunk)
//
// This is the starting node.
//
// - Parentheses () represent a node.
// - dc is the variable name assigned to the node.
// - DocumentChunk is the node label.
//
// In plain English:
//
//     "Find a node that represents a document chunk."
//
// Each matched chunk may have properties such as:
//
// - chunkId
// - articleId
// - chunkOrder
// - text
// - source
//
// The chunk is the smaller unit of text that can later be embedded, indexed,
// searched, or retrieved for LLM context.
//
// -----------------------------------------------------------------------------
//
//     -[:PART_OF]->
//
// This is the relationship pattern.
//
// - Square brackets [] represent a relationship.
// - PART_OF is the relationship type.
// - The arrow -> shows the relationship direction.
//
// In plain English:
//
//     "Follow a PART_OF relationship from the chunk to the article."
//
// The direction reads naturally:
//
//     DocumentChunk PART_OF KnowledgeArticle
//
// or:
//
//     "This chunk is part of this knowledge article."
//
// If the relationship was accidentally created in the opposite direction:
//
//     (:KnowledgeArticle)-[:PART_OF]->(:DocumentChunk)
//
// then this query would not match it.
//
// That makes this query useful for verifying both:
//
// - the relationship type
// - the relationship direction
//
// -----------------------------------------------------------------------------
//
//     (ka:KnowledgeArticle)
//
// This is the ending node.
//
// - ka is the variable name assigned to the article node.
// - KnowledgeArticle is the node label.
//
// In plain English:
//
//     "Find the knowledge article that this chunk belongs to."
//
// Each matched article may have properties such as:
//
// - articleId
// - title
// - issueType
// - content
// - source
//
// -----------------------------------------------------------------------------
// Why return a path?
// -----------------------------------------------------------------------------
// Returning a path is different from returning only nodes or properties.
//
// If we return only:
//
//     dc, ka
//
// Neo4j may show the two nodes and their properties, but returning:
//
//     path
//
// makes the intended graph connection explicit:
//
//     (dc)-[:PART_OF]->(ka)
//
// This is especially helpful during demos because students can visually see the
// graph structure instead of only reading rows in a table.
//
// -----------------------------------------------------------------------------
// Why this is a validation query
// -----------------------------------------------------------------------------
// This query is normally run after loading chunks and creating PART_OF
// relationships.
//
// It helps confirm:
//
// - DocumentChunk nodes exist.
// - KnowledgeArticle nodes exist.
// - PART_OF relationships exist.
// - The relationships point in the correct direction.
// - Each chunk is connected to the expected parent article.
//
// If the query returns no rows, possible reasons include:
//
// - No DocumentChunk nodes exist.
// - No KnowledgeArticle nodes exist.
// - PART_OF relationships were not created.
// - PART_OF relationships were created in the opposite direction.
// - The chunk.articleId values did not match KnowledgeArticle.articleId values.
// - The query is running against the wrong database.
// - The pasted query used an escaped arrow instead of the real Cypher arrow.

RETURN
  path,
  dc.chunkId AS chunkId,
  dc.chunkOrder AS chunkOrder,
  dc.text AS chunkText,
  ka.articleId AS articleId,
  ka.title AS articleTitle

// =============================================================================
// RETURN PATH, CHUNK DETAILS, AND ARTICLE DETAILS
// =============================================================================
// RETURN defines what Neo4j should show after matching the chunk-to-article
// paths.
//
// Here we return both:
//
// 1. The full graph path
// 2. Selected chunk and article properties
//
// This gives us the best of both worlds:
//
// - path gives us a visual graph representation.
// - the property columns give us a clean table for verification.
//
// -----------------------------------------------------------------------------
// path
// -----------------------------------------------------------------------------
// path returns the complete matched graph structure:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)
//
// In Neo4j Browser, this can be displayed visually as connected nodes and
// relationships.
//
// This is helpful because graph databases are not only about isolated records.
// Their real strength is showing how pieces of information are connected.
//
// -----------------------------------------------------------------------------
// dc.chunkId AS chunkId
// -----------------------------------------------------------------------------
// dc.chunkId returns the unique identifier of the DocumentChunk node.
//
// Example:
//
//     C-K001-001
//     C-K001-002
//     C-K002-001
//
// This tells us exactly which chunk is part of the matched path.
//
// We alias it as chunkId so the output column is clean and easy to read.
//
// -----------------------------------------------------------------------------
// dc.chunkOrder AS chunkOrder
// -----------------------------------------------------------------------------
// dc.chunkOrder returns the position of the chunk inside its parent article.
//
// Example:
//
//     1
//     2
//
// This is important because an article may be split into multiple chunks, and
// the original sequence matters when reconstructing or displaying the content.
//
// For example:
//
//     C-K001-001  -> chunkOrder 1
//     C-K001-002  -> chunkOrder 2
//
// That tells us which part should be read first.
//
// Without chunkOrder, the system may know that chunks belong to an article, but
// it would not know their correct reading order.
//
// -----------------------------------------------------------------------------
// dc.text AS chunkText
// -----------------------------------------------------------------------------
// dc.text returns the actual text stored in the chunk.
//
// This is the most important payload field for future search and RAG workflows.
//
// In a vector-search or RAG pipeline, this text may later be:
//
// - converted into an embedding
// - stored in a vector index
// - retrieved for a user question
// - passed to an LLM as supporting context
// - shown as a citation or evidence snippet
//
// Returning chunkText here helps us verify that the correct text was stored on
// the correct chunk node.
//
// It also helps confirm that the chunking process produced meaningful,
// readable units of knowledge.
//
// -----------------------------------------------------------------------------
// ka.articleId AS articleId
// -----------------------------------------------------------------------------
// ka.articleId returns the unique identifier of the parent KnowledgeArticle.
//
// Example:
//
//     K001
//     K002
//     K003
//
// This confirms which article the chunk is connected to.
//
// It is especially important because the chunk also stores articleId as a
// property, but the relationship proves the actual graph connection.
//
// The property:
//
//     dc.articleId = "K001"
//
// tells us the intended parent.
//
// The relationship:
//
//     (dc)-[:PART_OF]->(ka)
//
// proves the actual connected parent.
//
// -----------------------------------------------------------------------------
// ka.title AS articleTitle
// -----------------------------------------------------------------------------
// ka.title returns the human-readable title of the parent KnowledgeArticle.
//
// Example:
//
//     Fix login failure
//     Resolve payment failure
//     Fix app crash
//
// This makes the output easier to understand than looking only at IDs.
//
// For validation, articleTitle helps us quickly confirm that the chunk belongs
// to the expected article topic.
//
// For example:
//
//     C-K001-001
//
// should connect to:
//
//     Fix login failure
//
// If the chunk text and article title do not logically match, the input data or
// relationship creation logic should be reviewed.
//
// -----------------------------------------------------------------------------
// Why return both graph and table data?
// -----------------------------------------------------------------------------
// Returning only path is good for graph visualization.
//
// Returning only properties is good for tabular validation.
//
// Returning both is ideal for a lab/demo because we can:
//
// - See the graph connection visually.
// - Read the important values clearly.
// - Take useful screenshots.
// - Explain both graph structure and business meaning together.

ORDER BY articleId, chunkOrder;

// =============================================================================
// SORT THE RESULT BY ARTICLE AND CHUNK ORDER
// =============================================================================
// ORDER BY controls the display order of the returned rows.
//
// Here we sort by:
//
//     articleId, chunkOrder
//
// This means:
//
//     1. Group chunks by their parent article.
//     2. Within each article, show chunks in their original sequence.
//
// Sorting does not change the graph data.
// It only changes how the result is displayed.
//
// Expected output may look conceptually like:
//
//     chunkId       chunkOrder   chunkText                                      articleId   articleTitle
//     ------------------------------------------------------------------------------------------------------
//     C-K001-001    1            If a customer cannot sign in...                K001        Fix login failure
//     C-K001-002    2            For login failure, clear the mobile app cache   K001        Fix login failure
//     C-K002-001    1            If payment fails during checkout...            K002        Resolve payment failure
//     C-K002-002    2            For payment failure, confirm the gateway...     K002        Resolve payment failure
//     C-K003-001    1            If the mobile app crashes...                   K003        Fix app crash
//     C-K003-002    2            For app crash issues, restart the device...     K003        Fix app crash
//
// This stable ordering is useful for:
//
// - Lab guide screenshots
// - Comparing expected and actual results
// - Teaching students how chunks map to articles
// - Verifying that chunkOrder was stored correctly
// - Reviewing the data before creating embeddings or vector indexes
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a graph path verification query for chunk lineage.
//
// It does not create, update, or delete anything.
// It only reads existing paths between DocumentChunk and KnowledgeArticle nodes.
//
// The main purpose is:
//
//     "Show the actual graph connections where document chunks are part of
//      knowledge articles."
//
// The query follows this flow:
//
//     1. MATCH a path from DocumentChunk to KnowledgeArticle through PART_OF.
//     2. Store the complete matched structure in the variable path.
//     3. RETURN the visual path plus useful chunk and article fields.
//     4. ORDER the output by articleId and chunkOrder for clean validation.
//
// The key graph concept here is:
//
//     A chunk is not isolated text.
//     It is connected knowledge that belongs to a larger article.
//
// This connection is very important for RAG-style systems because when a chunk
// is retrieved, the system can trace it back to the original article, title,
// source, issue type, and other surrounding context.
//
// Production note:
// The pasted query used -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the actual Cypher arrow:
//
//     MATCH path = (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)
```

# Step 16 — Check existing vector indexes

```cypher
// =============================================================================
// INSPECT VECTOR INDEXES IN THE NEO4J DATABASE
// =============================================================================
// This query lists all vector indexes currently defined in the Neo4j database.
//
// In simple terms, it answers this question:
//
//     "Which vector indexes exist in my graph database,
//      what state are they in,
//      which labels and properties do they apply to,
//      and are they fully populated and ready for use?"
//
// Vector indexes are especially important in semantic search and RAG-style
// systems.
//
// In a normal text or property index, we usually search for exact or structured
// values such as:
//
//     articleId = "K001"
//     chunkId = "C-K001-001"
//     issueType = "Login Failure"
//
// But in a vector index, Neo4j indexes high-dimensional embedding vectors.
//
// These vectors usually represent the semantic meaning of text.
//
// For example, a DocumentChunk may have:
// - dc.text       -> human-readable troubleshooting text
// - dc.embedding  -> numeric vector representation of that text
//
// A vector index allows Neo4j to quickly find chunks whose embeddings are
// semantically similar to a query embedding.
//
// This is useful for:
// - semantic search
// - RAG retrieval
// - similarity search
// - recommendation systems
// - finding related support articles
// - retrieving the most relevant document chunks for an LLM
//
// This query is a database inspection query.
// It does not create, update, or delete anything.
// It only shows metadata about existing vector indexes.

SHOW VECTOR INDEXES

// =============================================================================
// SHOW ALL VECTOR INDEXES
// =============================================================================
// SHOW VECTOR INDEXES asks Neo4j to display all vector indexes defined in the
// current database.
//
// Think of a vector index as a special search structure built for embeddings.
//
// Instead of helping Neo4j find exact property values, a vector index helps
// Neo4j find "nearby" vectors.
//
// In semantic search terms, "nearby" usually means:
//
//     "These pieces of text have similar meaning."
//
// For example, a user may ask:
//
//     "Customer cannot login because OTP is not coming."
//
// Even if no chunk contains that exact sentence, a vector search can still find
// related chunks such as:
//
//     "If a customer cannot sign in, ask them to reset the password and verify
//      OTP delivery."
//
// This is why vector indexes are important for RAG workflows.
// They help retrieve relevant context even when the user's wording is different
// from the stored document text.
//
// This command is safe to run because it only reads index metadata.

YIELD name, state, populationPercent, type, entityType, labelsOrTypes, properties, indexProvider

// =============================================================================
// CHOOSE VECTOR INDEX METADATA FIELDS
// =============================================================================
// YIELD selects which columns from SHOW VECTOR INDEXES we want to inspect.
//
// SHOW VECTOR INDEXES can expose several metadata fields, but this query focuses
// on the most useful fields for understanding whether vector indexes exist and
// whether they are ready to support similarity search.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// name is the vector index name.
//
// A good vector index name usually tells us:
// - which label/entity it indexes
// - which property contains the vector
// - that it is a vector index
//
// For example:
//
//     documentChunk_embedding_vector
//
// This name would suggest:
//
// - DocumentChunk is the node label
// - embedding is the vector property
// - vector indicates this is a vector index
//
// Clear index names are important because vector search queries usually call the
// index by name.
//
// For example, a vector query may later refer to:
//
//     "documentChunk_embedding_vector"
//
// If the index name is unclear or inconsistent, troubleshooting becomes harder.
//
// -----------------------------------------------------------------------------
// state
// -----------------------------------------------------------------------------
// state tells us the current lifecycle state of the vector index.
//
// The most important expected state is usually:
//
//     ONLINE
//
// ONLINE means the index is ready to be used by queries.
//
// If the state is still building, populating, failed, or otherwise not ready,
// vector search queries may not work correctly or may produce errors.
//
// This field is especially important immediately after creating an index because
// Neo4j may need time to populate it from existing data.
//
// -----------------------------------------------------------------------------
// populationPercent
// -----------------------------------------------------------------------------
// populationPercent tells us how much of the index population process has
// completed.
//
// For example:
//
//     100.0
//
// means the index is fully populated.
//
// This matters because creating an index is not always instantaneous.
// If nodes already contain embedding values, Neo4j needs to scan those nodes and
// add the vectors into the index structure.
//
// If populationPercent is less than 100, the index may still be building.
//
// In a lab or production validation step, we usually want:
//
//     state = ONLINE
//     populationPercent = 100.0
//
// That combination tells us the index is ready for reliable vector search.
//
// -----------------------------------------------------------------------------
// type
// -----------------------------------------------------------------------------
// type tells us what kind of index this is.
//
// Since we are running SHOW VECTOR INDEXES, the type should indicate that the
// index is a vector index.
//
// This helps distinguish vector indexes from other schema indexes such as:
// - range indexes
// - text indexes
// - full-text indexes
// - lookup indexes
//
// This is useful because each index type supports a different kind of lookup.
//
// A vector index supports nearest-neighbor similarity search over embedding
// vectors.
//
// -----------------------------------------------------------------------------
// entityType
// -----------------------------------------------------------------------------
// entityType tells us whether the vector index applies to nodes or relationships.
//
// In many RAG-style examples, embeddings are stored on nodes such as:
//
//     (:DocumentChunk)
//
// In that case, the entity type would be node-related.
//
// Some graph models may also store embeddings on relationships, but chunk-based
// retrieval usually stores embeddings on nodes because each chunk is represented
// as a node.
//
// This field helps us confirm what kind of graph entity the vector index is
// indexing.
//
// -----------------------------------------------------------------------------
// labelsOrTypes
// -----------------------------------------------------------------------------
// labelsOrTypes tells us which node label or relationship type the vector index
// applies to.
//
// For example, if the index applies to:
//
//     (:DocumentChunk)
//
// then this field should include:
//
//     DocumentChunk
//
// This is important because a vector index is not automatically applied to every
// embedding property in the whole database.
//
// It is normally scoped to a specific label and property.
//
// For a chunk retrieval workflow, we usually expect the vector index to target
// DocumentChunk nodes because those are the text units we want to retrieve.
//
// -----------------------------------------------------------------------------
// properties
// -----------------------------------------------------------------------------
// properties tells us which property or properties are indexed.
//
// For a vector index, this is usually the property that stores the embedding
// vector.
//
// Example:
//
//     embedding
//
// This means Neo4j expects nodes to have a property like:
//
//     dc.embedding = [0.123, -0.456, 0.789, ...]
//
// The values are numeric arrays representing the semantic meaning of the text.
//
// This field is critical because if the vector index is built on the wrong
// property, vector search queries will not search the expected embeddings.
//
// For example, if the index is built on:
//
//     text
//
// instead of:
//
//     embedding
//
// that would be incorrect for a vector index because text is plain language,
// not a numeric vector.
//
// -----------------------------------------------------------------------------
// indexProvider
// -----------------------------------------------------------------------------
// indexProvider tells us which Neo4j index provider is backing this vector index.
//
// The provider is the internal implementation responsible for storing and
// searching the index data.
//
// This is useful for database administration, troubleshooting, and understanding
// which indexing engine Neo4j is using behind the scenes.
//
// In most day-to-day lab work, students do not need to tune the provider
// directly, but seeing it in the output helps confirm that Neo4j created the
// index using the expected vector index implementation.

RETURN
  name,
  state,
  populationPercent,
  type,
  entityType,
  labelsOrTypes,
  properties,
  indexProvider

// =============================================================================
// RETURN VECTOR INDEX DETAILS
// =============================================================================
// RETURN defines the final output table shown to the user.
//
// We return the same fields selected in the YIELD clause:
//
// - name
// - state
// - populationPercent
// - type
// - entityType
// - labelsOrTypes
// - properties
// - indexProvider
//
// This gives us a focused and readable view of vector index metadata.
//
// Example conceptual output may look like:
//
//     name                            state   populationPercent   type     entityType   labelsOrTypes       properties     indexProvider
//     ----------------------------------------------------------------------------------------------------------------------------------
//     documentChunk_embedding_vector  ONLINE  100.0               VECTOR   NODE         ["DocumentChunk"]   ["embedding"]  vector-...
//
// This output helps us verify:
//
// - the vector index exists
// - the index is online
// - the index is fully populated
// - the index applies to the expected label
// - the index uses the expected embedding property
// - the index is backed by a vector-capable provider
//
// -----------------------------------------------------------------------------
// Why return these fields before running vector search?
// -----------------------------------------------------------------------------
// Before running a semantic search query, we should verify the vector index state.
//
// If the index is missing, offline, or not fully populated, then a vector search
// query may fail or behave unexpectedly.
//
// This is similar to checking whether a road is open before sending traffic
// through it.
//
// The vector search query depends on the vector index being ready.

ORDER BY name;

// =============================================================================
// SORT VECTOR INDEXES BY NAME
// =============================================================================
// ORDER BY name sorts the final result alphabetically by index name.
//
// Sorting is not required for correctness.
// It only makes the output easier to inspect.
//
// This is helpful when the database contains multiple vector indexes, such as:
//
// - one index for DocumentChunk embeddings
// - one index for KnowledgeArticle embeddings
// - one index for Product embeddings
// - one index for Ticket embeddings
//
// Without ORDER BY, Neo4j may return the indexes in an order that is not ideal
// for manual review.
//
// By sorting by name, the output becomes stable, predictable, and easier to use
// in screenshots, demos, and lab documentation.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a vector index inventory query.
//
// It does not inspect the actual embedding values.
// It does not run a similarity search.
// It does not create or modify an index.
//
// It only answers:
//
//     "Which vector indexes exist,
//      what do they index,
//      and are they ready to use?"
//
// In a RAG-style graph project, this is an important validation step before
// running vector similarity queries.
//
// A healthy vector-search validation flow is:
//
//     1. Load DocumentChunk nodes.
//     2. Add embedding vectors to those chunks.
//     3. Create a vector index on the embedding property.
//     4. Use SHOW VECTOR INDEXES to confirm the index is ONLINE and 100% populated.
//     5. Run vector similarity search against the index.
//     6. Trace retrieved chunks back to KnowledgeArticle and Issue nodes.
//
// The most important fields to check are:
//
//     state
//     populationPercent
//     labelsOrTypes
//     properties
//
// Ideally, for a ready vector index, we want to see:
//
//     state = ONLINE
//     populationPercent = 100.0
//     labelsOrTypes includes the expected label, such as DocumentChunk
//     properties includes the expected vector property, such as embedding
```

# Step 17 — Add simple learning embeddings to DocumentChunk nodes

```cypher
// =============================================================================
// ADD SAMPLE EMBEDDING VECTORS TO DOCUMENT CHUNKS
// =============================================================================
// This query assigns sample embedding vectors to existing DocumentChunk nodes.
//
// In simple terms, it answers this requirement:
//
//     "For each document chunk, store a numeric vector that represents the
//      semantic meaning of that chunk's text."
//
// Earlier, we created DocumentChunk nodes such as:
//
//     C-K001-001
//     C-K001-002
//     C-K002-001
//     C-K002-002
//     C-K003-001
//     C-K003-002
//
// Each chunk contains human-readable text.
//
// For example:
//
//     "If a customer cannot sign in, ask them to reset the password..."
//
// But vector search does not directly compare plain English sentences.
// Instead, semantic search compares numeric vectors called embeddings.
//
// An embedding is a list of numbers that represents the meaning of text in a
// mathematical form.
//
// In this lab, we are using small 3-dimensional sample vectors so students can
// easily understand the concept.
//
// In real production systems, embeddings are usually much larger, often hundreds
// or thousands of dimensions depending on the embedding model.
//
// This query is useful for:
// - Preparing DocumentChunk nodes for vector search
// - Demonstrating how embeddings are stored as node properties
// - Verifying that each chunk has an embedding
// - Checking embedding dimensionality
// - Preparing for vector index creation or vector similarity search
// - Teaching the relationship between text, embeddings, and semantic retrieval

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

// =============================================================================
// UNWIND EMBEDDING DATA INTO INDIVIDUAL ROWS
// =============================================================================
// UNWIND takes the list of maps above and expands it into one row per embedding
// record.
//
// Think of the list as a small in-memory table.
//
// Before UNWIND, Neo4j sees one list containing six maps:
//
//     [
//       {chunkId: "C-K001-001", embedding: [0.95, 0.10, 0.05]},
//       {chunkId: "C-K001-002", embedding: [0.90, 0.15, 0.05]},
//       ...
//     ]
//
// After UNWIND, Neo4j processes the data row by row:
//
//     Row 1 -> row = {chunkId: "C-K001-001", embedding: [0.95, 0.10, 0.05]}
//     Row 2 -> row = {chunkId: "C-K001-002", embedding: [0.90, 0.15, 0.05]}
//     Row 3 -> row = {chunkId: "C-K002-001", embedding: [0.05, 0.95, 0.10]}
//     Row 4 -> row = {chunkId: "C-K002-002", embedding: [0.10, 0.90, 0.15]}
//     Row 5 -> row = {chunkId: "C-K003-001", embedding: [0.10, 0.10, 0.95]}
//     Row 6 -> row = {chunkId: "C-K003-002", embedding: [0.15, 0.10, 0.90]}
//
// This row-by-row structure is important because the MATCH and SET clauses that
// follow will run once for each row.
//
// -----------------------------------------------------------------------------
// What each map represents
// -----------------------------------------------------------------------------
// Each map contains two fields:
//
// - chunkId:
//   The unique identifier of the DocumentChunk node that should receive the
//   embedding.
//
// - embedding:
//   A numeric list representing the semantic meaning of that chunk.
//
// For example:
//
//     {
//       chunkId: "C-K001-001",
//       embedding: [0.95, 0.10, 0.05]
//     }
//
// means:
//
//     "Find chunk C-K001-001 and assign this vector to it."
//
// -----------------------------------------------------------------------------
// Why these vectors look grouped
// -----------------------------------------------------------------------------
// Notice that chunks related to login failure have vectors with high values in
// the first position:
//
//     [0.95, 0.10, 0.05]
//     [0.90, 0.15, 0.05]
//
// Chunks related to payment failure have high values in the second position:
//
//     [0.05, 0.95, 0.10]
//     [0.10, 0.90, 0.15]
//
// Chunks related to app crash have high values in the third position:
//
//     [0.10, 0.10, 0.95]
//     [0.15, 0.10, 0.90]
//
// This makes the lab easier to understand because similar topics have similar
// vectors.
//
// In a real embedding model, humans usually cannot interpret each vector
// dimension this directly. The model learns those dimensions mathematically.
// But for teaching, these small sample vectors make the idea visible.

MATCH (dc:DocumentChunk {chunkId: row.chunkId})

// =============================================================================
// MATCH THE TARGET DOCUMENT CHUNK
// =============================================================================
// MATCH finds the DocumentChunk node that should receive the embedding.
//
// The pattern is:
//
//     (dc:DocumentChunk {chunkId: row.chunkId})
//
// Let us break it down:
//
// - dc is the variable name assigned to the matched DocumentChunk node.
// - DocumentChunk is the node label.
// - chunkId is the property used to identify the chunk.
// - row.chunkId is the chunkId value from the current UNWIND row.
//
// In plain English:
//
//     "For the current embedding row, find the DocumentChunk whose chunkId
//      matches row.chunkId."
//
// Example:
//
// For this row:
//
//     {
//       chunkId: "C-K001-001",
//       embedding: [0.95, 0.10, 0.05]
//     }
//
// Neo4j tries to find:
//
//     (:DocumentChunk {chunkId: "C-K001-001"})
//
// If the chunk exists, the query continues and sets the embedding property.
//
// If the chunk does not exist, this MATCH fails for that row, and no embedding
// is assigned for that missing chunk.
//
// -----------------------------------------------------------------------------
// Why use MATCH instead of MERGE here?
// -----------------------------------------------------------------------------
// We use MATCH because the DocumentChunk nodes should already exist.
//
// This query is not responsible for creating chunks.
// Its job is only to enrich existing chunks with embeddings.
//
// That is an important modeling discipline:
//
//     First create the content nodes.
//     Then attach embeddings to those existing nodes.
//
// If we used MERGE here, the query could accidentally create a DocumentChunk
// node that has only a chunkId and embedding, but no text, no chunkOrder, and no
// PART_OF relationship to a KnowledgeArticle.
//
// That would create incomplete or orphan-like data.
//
// By using MATCH, we make sure embeddings are only assigned to chunks that were
// already loaded correctly.
//
// -----------------------------------------------------------------------------
// Relationship with chunkId uniqueness constraint
// -----------------------------------------------------------------------------
// Earlier, we created a uniqueness constraint on:
//
//     (:DocumentChunk).chunkId
//
// That makes this lookup safer and more reliable.
//
// When Neo4j matches by chunkId, the uniqueness constraint ensures that each
// chunkId identifies at most one DocumentChunk node.
//
// This is exactly what we want before assigning embeddings.

SET dc.embedding = row.embedding

// =============================================================================
// SET THE EMBEDDING PROPERTY ON THE DOCUMENT CHUNK
// =============================================================================
// SET assigns the embedding vector from the current row to the matched
// DocumentChunk node.
//
// In plain English, this line means:
//
//     "Store this numeric vector on the DocumentChunk as dc.embedding."
//
// After this line runs, a chunk node may look conceptually like:
//
//     (:DocumentChunk {
//       chunkId: "C-K001-001",
//       text: "If a customer cannot sign in...",
//       chunkOrder: 1,
//       articleId: "K001",
//       embedding: [0.95, 0.10, 0.05]
//     })
//
// -----------------------------------------------------------------------------
// Why store embeddings on DocumentChunk nodes?
// -----------------------------------------------------------------------------
// In a RAG-style system, chunks are usually the best retrieval unit.
//
// A full article may contain multiple ideas.
// A smaller chunk is more focused.
//
// By storing embeddings on DocumentChunk nodes, we can later ask Neo4j:
//
//     "Find the chunks whose embeddings are most similar to this query vector."
//
// This allows semantic search.
//
// For example, if a user asks:
//
//     "Customer cannot receive OTP while signing in"
//
// a vector search can retrieve login-related chunks even if the exact words are
// not identical.
//
// -----------------------------------------------------------------------------
// Why SET updates existing values
// -----------------------------------------------------------------------------
// SET will create the embedding property if it does not exist.
//
// If the embedding property already exists, SET replaces the old value with the
// new value.
//
// This makes the query refresh-friendly.
//
// For example, if we later regenerate embeddings using a better model, we can
// rerun a similar update query and replace the old embeddings.
//
// Production note:
// In a real system, if embeddings are regenerated, it is also useful to track
// metadata such as:
//
// - embeddingModel
// - embeddingDimension
// - embeddingCreatedAt
// - embeddingVersion
//
// That helps teams know which model produced the vectors and whether indexes
// need to be rebuilt or refreshed.

RETURN
  dc.chunkId AS chunkId,
  dc.text AS chunkText,
  dc.embedding AS embedding,
  size(dc.embedding) AS embeddingDimension

// =============================================================================
// RETURN UPDATED CHUNK AND EMBEDDING DETAILS
// =============================================================================
// RETURN defines what Neo4j should show after assigning the embeddings.
//
// Here we return four values:
//
// 1. dc.chunkId AS chunkId
// 2. dc.text AS chunkText
// 3. dc.embedding AS embedding
// 4. size(dc.embedding) AS embeddingDimension
//
// This output helps us verify that each matched chunk received the correct
// embedding vector.
//
// -----------------------------------------------------------------------------
// dc.chunkId AS chunkId
// -----------------------------------------------------------------------------
// chunkId identifies the DocumentChunk node that was updated.
//
// Example:
//
//     C-K001-001
//     C-K001-002
//     C-K002-001
//
// This confirms which chunk each returned row refers to.
//
// -----------------------------------------------------------------------------
// dc.text AS chunkText
// -----------------------------------------------------------------------------
// chunkText shows the human-readable text stored on the chunk.
//
// Returning the text is useful because it lets us compare the meaning of the
// text with the embedding assigned to it.
//
// For example, a login-related text should receive a login-like vector:
//
//     [0.95, 0.10, 0.05]
//
// A payment-related text should receive a payment-like vector:
//
//     [0.05, 0.95, 0.10]
//
// An app-crash-related text should receive an app-crash-like vector:
//
//     [0.10, 0.10, 0.95]
//
// This makes the validation easy for students to understand.
//
// -----------------------------------------------------------------------------
// dc.embedding AS embedding
// -----------------------------------------------------------------------------
// embedding returns the numeric vector stored on the chunk.
//
// This confirms that the SET operation actually wrote the vector property to
// the node.
//
// In this lab, the embedding is a small 3-number list.
//
// In production, this list is usually much longer.
//
// Example production-style embedding dimensions may be:
//
// - 384 dimensions
// - 768 dimensions
// - 1024 dimensions
// - 1536 dimensions
// - 3072 dimensions
//
// The exact size depends on the embedding model used.
//
// -----------------------------------------------------------------------------
// size(dc.embedding) AS embeddingDimension
// -----------------------------------------------------------------------------
// size(dc.embedding) returns the number of elements in the embedding list.
//
// In this lab, each embedding has three numbers, so the expected result is:
//
//     embeddingDimension = 3
//
// This is a very useful validation check.
//
// Vector indexes usually expect all vectors stored in the indexed property to
// have the same dimension.
//
// If one chunk has a 3-dimensional vector and another chunk has a 4-dimensional
// vector, vector indexing or vector search may fail or behave incorrectly.
//
// By returning embeddingDimension, we quickly confirm that every updated chunk
// has the expected vector size.
//
// This is one of the most important checks before creating or using a vector
// index.

ORDER BY chunkId;

// =============================================================================
// SORT THE RESULT BY CHUNK ID
// =============================================================================
// ORDER BY chunkId sorts the final output by the chunkId column.
//
// Sorting does not change the stored graph data.
// It only controls how the result is displayed.
//
// Without ORDER BY, Neo4j may return rows in an order that is not predictable.
//
// By sorting by chunkId, the output becomes stable and easy to verify.
//
// Expected output may look conceptually like:
//
//     chunkId       chunkText                                      embedding             embeddingDimension
//     -----------------------------------------------------------------------------------------------------
//     C-K001-001    If a customer cannot sign in...                [0.95,0.10,0.05]     3
//     C-K001-002    For login failure, clear the mobile app...     [0.90,0.15,0.05]     3
//     C-K002-001    If payment fails during checkout...            [0.05,0.95,0.10]     3
//     C-K002-002    For payment failure, confirm the gateway...     [0.10,0.90,0.15]     3
//     C-K003-001    If the mobile app crashes...                   [0.10,0.10,0.95]     3
//     C-K003-002    For app crash issues, restart the device...     [0.15,0.10,0.90]     3
//
// This stable ordering is useful for:
//
// - lab guide screenshots
// - comparing expected and actual results
// - verifying all chunks received embeddings
// - checking that embedding dimensions are consistent
// - preparing for vector index validation
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query enriches existing DocumentChunk nodes with sample embedding vectors.
//
// The main flow is:
//
//     1. UNWIND turns the inline embedding list into individual rows.
//     2. MATCH finds the existing DocumentChunk by chunkId.
//     3. SET writes the embedding vector to the chunk.
//     4. RETURN shows the chunk text, vector, and vector dimension.
//     5. ORDER BY makes the verification output predictable.
//
// The most important concept is:
//
//     Text is what humans read.
//     Embeddings are what vector search compares.
//
// After this query, each DocumentChunk has both:
//
// - dc.text       -> the readable troubleshooting text
// - dc.embedding  -> the numeric representation used for semantic search
//
// This prepares the graph for the next steps:
//
// - creating a vector index on DocumentChunk.embedding
// - verifying the vector index is online
// - running similarity search queries
// - retrieving relevant chunks for RAG-style answers
//
// Production note:
// These embeddings are simplified demo vectors.
// In a real system, embeddings should be generated by an embedding model and
// all vectors indexed together must have the same dimension.
```

# Step 18 — Verify embedding coverage and dimensions

```cypher
// =============================================================================
// VERIFY EMBEDDING COVERAGE AND EMBEDDING DIMENSIONS FOR DOCUMENT CHUNKS
// =============================================================================
// This query checks whether DocumentChunk nodes have embedding vectors assigned
// to them.
//
// In simple terms, it answers these questions:
//
//     "How many DocumentChunk nodes exist in total?"
//     "How many of those chunks have an embedding property?"
//     "What embedding dimensions are present in the database?"
//
// This is an important validation query before creating or using a vector index.
//
// In a RAG-style or semantic-search system, every searchable chunk should usually
// have an embedding vector. If some chunks are missing embeddings, those chunks
// may not participate in vector search.
//
// This query is useful for:
// - Confirming that all DocumentChunk nodes were enriched with embeddings
// - Detecting chunks that are missing embedding vectors
// - Checking whether all embeddings have the same dimension
// - Validating data readiness before vector index creation
// - Troubleshooting vector-search errors
// - Documenting embedding readiness in a lab guide or demo

MATCH (dc:DocumentChunk)

// =============================================================================
// MATCH ALL DOCUMENT CHUNK NODES
// =============================================================================
// MATCH tells Neo4j what graph pattern we want to find.
//
// Here the pattern is:
//
//     (dc:DocumentChunk)
//
// Let us break it down:
//
// - Parentheses () represent a node.
// - dc is the variable name assigned to each matched node.
// - DocumentChunk is the node label.
//
// In plain English, this means:
//
//     "Find every node labelled DocumentChunk and temporarily call each one dc."
//
// These DocumentChunk nodes represent the smaller text units that were split
// from KnowledgeArticle records.
//
// Each DocumentChunk may contain properties such as:
//
// - chunkId
// - articleId
// - chunkOrder
// - text
// - source
// - embedding
//
// The important property for this query is:
//
//     dc.embedding
//
// That property should contain a numeric list such as:
//
//     [0.95, 0.10, 0.05]
//
// In this lab, embeddings are intentionally small 3-dimensional demo vectors.
// In production systems, embeddings are usually much larger and are generated by
// an embedding model.
//
// -----------------------------------------------------------------------------
// Why this validation matters
// -----------------------------------------------------------------------------
// Before running vector search, we need to confirm that the data is ready.
//
// If chunks exist but embeddings are missing, Neo4j may not be able to index
// those chunks for vector search.
//
// If embeddings exist but have inconsistent dimensions, vector indexing or
// similarity search may fail or produce unreliable behavior.
//
// So this query acts like a readiness checklist for the embedding layer.

RETURN
  count(dc) AS totalChunks,
  count(dc.embedding) AS chunksWithEmbedding,
  collect(DISTINCT size(dc.embedding)) AS embeddingDimensions;

// =============================================================================
// RETURN EMBEDDING READINESS SUMMARY
// =============================================================================
// RETURN defines the final result table shown by Neo4j.
//
// This query returns three validation values:
//
// 1. count(dc) AS totalChunks
// 2. count(dc.embedding) AS chunksWithEmbedding
// 3. collect(DISTINCT size(dc.embedding)) AS embeddingDimensions
//
// -----------------------------------------------------------------------------
// count(dc) AS totalChunks
// -----------------------------------------------------------------------------
// count(dc) counts how many DocumentChunk nodes were matched.
//
// Because the MATCH clause matched all nodes with the DocumentChunk label, this
// value tells us the total number of chunks currently stored in the graph.
//
// For example, if we previously loaded six chunks, we expect:
//
//     totalChunks = 6
//
// This is the baseline number.
//
// Every other embedding validation value should be compared against this number.
//
// -----------------------------------------------------------------------------
// count(dc.embedding) AS chunksWithEmbedding
// -----------------------------------------------------------------------------
// count(dc.embedding) counts how many matched DocumentChunk nodes have a non-null
// embedding property.
//
// This is different from count(dc).
//
// - count(dc) counts all DocumentChunk nodes.
// - count(dc.embedding) counts only DocumentChunk nodes where dc.embedding exists
//   and is not null.
//
// For example:
//
//     totalChunks          = 6
//     chunksWithEmbedding  = 6
//
// This means all chunks have embeddings.
//
// But if the result is:
//
//     totalChunks          = 6
//     chunksWithEmbedding  = 4
//
// then two chunks are missing embeddings.
//
// That would be a problem before vector search because those missing chunks may
// not be searchable through the vector index.
//
// -----------------------------------------------------------------------------
// Why count(property) is useful
// -----------------------------------------------------------------------------
// In Cypher, count(property) ignores null values.
//
// So count(dc.embedding) is a compact way to check embedding coverage.
//
// It helps us answer:
//
//     "Out of all chunks, how many actually have embeddings?"
//
// This is especially useful after running an embedding update query, because we
// can immediately verify whether every expected chunk was updated.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT size(dc.embedding)) AS embeddingDimensions
// -----------------------------------------------------------------------------
// size(dc.embedding) returns the number of elements inside the embedding list.
//
// For example:
//
//     size([0.95, 0.10, 0.05])
//
// returns:
//
//     3
//
// Because this lab uses 3-dimensional demo embeddings, we expect the result to
// include:
//
//     [3]
//
// The collect(DISTINCT ...) part collects only unique dimension sizes.
//
// This means if all chunks have 3-dimensional embeddings, the result will be:
//
//     embeddingDimensions = [3]
//
// That is good because it tells us all embeddings have the same dimension.
//
// -----------------------------------------------------------------------------
// Why DISTINCT is important here
// -----------------------------------------------------------------------------
// Without DISTINCT, Neo4j would return one dimension value per chunk.
//
// For six chunks, we might get:
//
//     [3, 3, 3, 3, 3, 3]
//
// That is technically correct, but noisy.
//
// With DISTINCT, Neo4j returns only the unique dimension values:
//
//     [3]
//
// This is cleaner and easier to validate.
//
// -----------------------------------------------------------------------------
// What if multiple dimensions appear?
// -----------------------------------------------------------------------------
// If the result looks like:
//
//     embeddingDimensions = [3, 4]
//
// that means some chunks have 3-dimensional embeddings and some chunks have
// 4-dimensional embeddings.
//
// That is usually a serious problem.
//
// Vector indexes expect a consistent embedding dimension for the indexed
// property. Mixing dimensions can cause vector-index creation or vector-search
// problems.
//
// In production, this usually means embeddings were generated by different
// models, different configurations, or incorrect input data.
//
// -----------------------------------------------------------------------------
// What if null appears in embeddingDimensions?
// -----------------------------------------------------------------------------
// If some chunks do not have embeddings, then size(dc.embedding) may produce
// null for those rows.
//
// Depending on the data, the collected dimension list may show something like:
//
//     [3, null]
//
// This means:
//
// - Some chunks have 3-dimensional embeddings.
// - Some chunks are missing embeddings.
//
// In that case, compare:
//
//     totalChunks
//     chunksWithEmbedding
//
// If chunksWithEmbedding is less than totalChunks, then not all chunks are ready
// for vector search.
//
// -----------------------------------------------------------------------------
// Expected result for this lab
// -----------------------------------------------------------------------------
// Since we loaded six DocumentChunk nodes and assigned a 3-number embedding to
// each one, the expected result should conceptually be:
//
//     totalChunks   chunksWithEmbedding   embeddingDimensions
//     -------------------------------------------------------
//     6             6                     [3]
//
// This means:
//
// - There are 6 chunks in total.
// - All 6 chunks have embeddings.
// - All embeddings are 3-dimensional.
// - The data is ready for vector index validation and similarity search.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is an embedding readiness validation query.
//
// It does not create, update, or delete anything.
// It only checks the current state of DocumentChunk embeddings.
//
// The main purpose is:
//
//     "Confirm that all chunks have embeddings,
//      and confirm that embedding dimensions are consistent."
//
// A healthy vector-search preparation flow is:
//
//     1. Load DocumentChunk nodes.
//     2. Add embedding vectors to those chunks.
//     3. Run this query to verify embedding coverage and dimensions.
//     4. Create or inspect the vector index.
//     5. Run vector similarity search.
//
// The most important validation condition is:
//
//     totalChunks = chunksWithEmbedding
//     embeddingDimensions contains only one expected value
//
// For this lab, the ideal result is:
//
//     totalChunks = 6
//     chunksWithEmbedding = 6
//     embeddingDimensions = [3]
```

# Step 19 — Create vector index on DocumentChunk.embedding

```cypher
// =============================================================================
// CREATE VECTOR INDEX ON DOCUMENT CHUNK EMBEDDINGS
// =============================================================================
// This query creates a vector index on the embedding property of DocumentChunk
// nodes.
//
// In simple terms, it tells Neo4j:
//
//     "Create a special vector-search index for DocumentChunk.embedding,
//      where each embedding has 3 dimensions,
//      and similarity should be calculated using cosine similarity."
//
// This is a very important step in a RAG-style graph workflow.
//
// Earlier, we created DocumentChunk nodes and assigned embedding vectors like:
//
//     dc.embedding = [0.95, 0.10, 0.05]
//
// Those embeddings are numeric representations of the chunk text.
//
// But storing embeddings alone is not enough for efficient semantic search.
// Without a vector index, Neo4j would not have an optimized structure for quickly
// finding the most similar vectors.
//
// The vector index allows Neo4j to answer questions like:
//
//     "Which document chunks are most semantically similar to this query vector?"
//
// This is useful for:
// - semantic search
// - vector similarity search
// - RAG retrieval
// - finding relevant document chunks
// - recommending related knowledge articles
// - retrieving support context for an LLM
//
// The graph model involved here is:
//
//     (:DocumentChunk { embedding: [...] })
//
// The vector index is built on:
//
//     DocumentChunk.embedding
//
// Once this index is online and populated, we can use it to run similarity
// searches against the chunk embeddings.

CREATE VECTOR INDEX documentChunk_embedding_vector IF NOT EXISTS

// =============================================================================
// CREATE VECTOR INDEX STATEMENT
// =============================================================================
// CREATE VECTOR INDEX tells Neo4j that we want to create a special index designed
// for vector similarity search.
//
// This is different from normal indexes.
//
// A normal index is usually used for exact or range-based lookups, such as:
//
//     Find the chunk where chunkId = "C-K001-001"
//
// A vector index is used for similarity-based lookup, such as:
//
//     Find the chunks whose embedding vectors are closest to this query vector.
//
// -----------------------------------------------------------------------------
// documentChunk_embedding_vector
// -----------------------------------------------------------------------------
// This is the name of the vector index.
//
// A good index name should clearly describe:
//
// - Which entity the index applies to
// - Which property is indexed
// - What kind of index it is
//
// This name tells us:
//
//     documentChunk_embedding_vector
//
// - documentChunk -> the indexed node label/entity
// - embedding     -> the indexed property
// - vector        -> the index type/purpose
//
// This matters because vector search procedures usually refer to the index by
// name.
//
// For example, a later query may call:
//
//     db.index.vector.queryNodes(
//       "documentChunk_embedding_vector",
//       3,
//       [0.94, 0.12, 0.04]
//     )
//
// If the index name is unclear, inconsistent, or misspelled, vector search
// queries become harder to troubleshoot.
//
// -----------------------------------------------------------------------------
// IF NOT EXISTS
// -----------------------------------------------------------------------------
// IF NOT EXISTS makes the index creation command idempotent.
//
// Idempotent means:
//
//     "You can run the same command multiple times safely."
//
// If the vector index already exists, Neo4j will not create a duplicate index
// and will not fail simply because the index is already present.
//
// This is very useful for:
//
// - lab guides
// - repeatable setup scripts
// - CI/CD database initialization
// - development/test environment rebuilds
// - production schema deployment automation
//
// Without IF NOT EXISTS, rerunning the same command after the index already
// exists could result in an error.
//
// With IF NOT EXISTS, the command becomes safer for repeated execution.

FOR (dc:DocumentChunk)

// =============================================================================
// TARGET NODE LABEL FOR THE VECTOR INDEX
// =============================================================================
// FOR (dc:DocumentChunk) defines which nodes this vector index applies to.
//
// Let us break it down:
//
//     (dc:DocumentChunk)
//
// - Parentheses () represent a node pattern.
// - dc is a variable name used inside the index definition.
// - DocumentChunk is the node label.
//
// In plain English, this means:
//
//     "Create this vector index for nodes labelled DocumentChunk."
//
// This index does not apply to every node in the database.
// It only applies to nodes with the label:
//
//     :DocumentChunk
//
// For example, this index applies to:
//
//     (:DocumentChunk {embedding: [0.95, 0.10, 0.05]})
//
// But it does not apply to:
//
//     (:KnowledgeArticle {embedding: [...]})
//     (:Issue {embedding: [...]})
//     (:Customer {embedding: [...]})
//
// This label-specific behavior is important because different node types may
// store different embeddings for different purposes.
//
// -----------------------------------------------------------------------------
// Why index DocumentChunk nodes?
// -----------------------------------------------------------------------------
// In RAG-style systems, chunks are usually the best retrieval unit.
//
// A full article may contain multiple troubleshooting ideas, but a chunk is
// smaller and more focused.
//
// For example:
//
//     "If a customer cannot sign in, ask them to reset the password and verify
//      OTP delivery."
//
// is more focused than the full article:
//
//     "Fix login failure"
//
// By indexing DocumentChunk embeddings, vector search can retrieve the most
// relevant small text pieces instead of returning large documents blindly.
//
// This improves:
//
// - retrieval precision
// - answer relevance
// - LLM context quality
// - citation traceability
// - explainability of retrieved results

ON dc.embedding

// =============================================================================
// INDEXED PROPERTY
// =============================================================================
// ON dc.embedding tells Neo4j which property should be indexed.
//
// In this case, the indexed property is:
//
//     embedding
//
// on each DocumentChunk node.
//
// Each DocumentChunk is expected to have a property like:
//
//     dc.embedding = [0.95, 0.10, 0.05]
//
// That property must contain a numeric list/vector.
//
// -----------------------------------------------------------------------------
// Why index the embedding property?
// -----------------------------------------------------------------------------
// The text property is for humans:
//
//     dc.text = "If a customer cannot sign in..."
//
// The embedding property is for vector search:
//
//     dc.embedding = [0.95, 0.10, 0.05]
//
// The vector index does not compare text directly.
// It compares numeric vectors.
//
// That means when a user asks a question, the question is also converted into
// an embedding vector. Neo4j then compares the query vector with stored chunk
// vectors and returns the closest matches.
//
// Conceptually:
//
//     User question
//         -> convert to query embedding
//         -> compare against DocumentChunk.embedding
//         -> return most similar chunks
//
// -----------------------------------------------------------------------------
// Important requirement
// -----------------------------------------------------------------------------
// The values stored in dc.embedding should match the vector index configuration.
//
// Since this index is configured with:
//
//     vector.dimensions = 3
//
// each dc.embedding should contain exactly three numeric values.
//
// Earlier, we validated this using:
//
//     size(dc.embedding) AS embeddingDimension
//
// and expected:
//
//     embeddingDimension = 3
//
// If vectors have inconsistent dimensions, vector indexing or querying can fail
// or behave incorrectly.

OPTIONS {
  indexConfig: {
    `vector.dimensions`: 3,
    `vector.similarity_function`: 'cosine'
  }
};

// =============================================================================
// VECTOR INDEX CONFIGURATION OPTIONS
// =============================================================================
// OPTIONS provides configuration settings for the vector index.
//
// The indexConfig map tells Neo4j how this vector index should behave.
//
// In this query, we configure two important settings:
//
//     1. vector.dimensions
//     2. vector.similarity_function
//
// -----------------------------------------------------------------------------
// `vector.dimensions`: 3
// -----------------------------------------------------------------------------
// vector.dimensions defines the required size of each embedding vector.
//
// Here we set:
//
//     `vector.dimensions`: 3
//
// This means every embedding indexed by this vector index is expected to contain
// exactly 3 numeric values.
//
// Example valid vector:
//
//     [0.95, 0.10, 0.05]
//
// This has 3 dimensions.
//
// Example invalid vector for this index:
//
//     [0.95, 0.10]
//
// This has only 2 dimensions.
//
// Another invalid vector:
//
//     [0.95, 0.10, 0.05, 0.01]
//
// This has 4 dimensions.
//
// -----------------------------------------------------------------------------
// Why dimension consistency matters
// -----------------------------------------------------------------------------
// Vector similarity comparison only works when vectors have the same length.
//
// We cannot correctly compare a 3-dimensional vector with a 4-dimensional vector
// because they do not describe points in the same vector space.
//
// Think of it like comparing locations:
//
// - A 2D location may have x and y.
// - A 3D location has x, y, and z.
//
// If one point has no z value, it is not directly comparable in the same way.
//
// In embedding systems, all vectors for one index should come from the same
// embedding model and configuration.
//
// In this lab, we intentionally use 3-dimensional demo embeddings so the concept
// remains easy to see.
//
// In production, the dimension may be much larger, such as:
//
// - 384
// - 768
// - 1024
// - 1536
// - 3072
//
// The exact dimension depends on the embedding model.
//
// -----------------------------------------------------------------------------
// `vector.similarity_function`: 'cosine'
// -----------------------------------------------------------------------------
// vector.similarity_function defines how Neo4j compares vectors.
//
// Here we use:
//
//     `vector.similarity_function`: 'cosine'
//
// Cosine similarity measures how similar two vectors are based on their
// direction, rather than only their raw magnitude.
//
// In simple terms:
//
//     Cosine similarity asks:
//     "Are these two vectors pointing in a similar direction?"
//
// This is commonly used in text embedding and semantic search workflows because
// the direction of the vector often represents semantic meaning.
//
// For example, two login-related chunks may have vectors like:
//
//     [0.95, 0.10, 0.05]
//     [0.90, 0.15, 0.05]
//
// These vectors point in a similar direction, so cosine similarity should treat
// them as semantically close.
//
// A payment-related chunk may look like:
//
//     [0.05, 0.95, 0.10]
//
// That points in a different direction, so it should be considered less similar
// to the login-related query vector.
//
// -----------------------------------------------------------------------------
// Why cosine similarity is a good teaching choice
// -----------------------------------------------------------------------------
// Cosine similarity is commonly used for text embeddings because it focuses on
// semantic direction.
//
// For this lab, it helps demonstrate that:
//
// - login chunks are close to login query vectors
// - payment chunks are close to payment query vectors
// - app crash chunks are close to app crash query vectors
//
// This makes the similarity search behavior easier for students to understand.
//
// -----------------------------------------------------------------------------
// Why the option keys use backticks
// -----------------------------------------------------------------------------
// The configuration keys are written as:
//
//     `vector.dimensions`
//     `vector.similarity_function`
//
// The backticks are used because these keys contain dots.
//
// In Cypher, dots normally indicate property access, like:
//
//     dc.embedding
//
// But here, the dot is part of the configuration key name itself.
//
// Backticks tell Cypher:
//
//     "Treat this entire text as one key name."
//
// Without backticks, Cypher may interpret the dot incorrectly.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query creates a vector index for semantic search over DocumentChunk
// embeddings.
//
// The main purpose is:
//
//     "Enable Neo4j to efficiently find document chunks whose embedding vectors
//      are similar to a query embedding."
//
// The query defines:
//
// - index name:
//     documentChunk_embedding_vector
//
// - target label:
//     DocumentChunk
//
// - indexed property:
//     embedding
//
// - vector dimension:
//     3
//
// - similarity function:
//     cosine
//
// After this query runs, the next important validation step is to check whether
// the vector index is online and fully populated:
//
//     SHOW VECTOR INDEXES
//     YIELD name, state, populationPercent, labelsOrTypes, properties
//     WHERE name = "documentChunk_embedding_vector"
//     RETURN name, state, populationPercent, labelsOrTypes, properties;
//
// Ideally, we want to see:
//
//     state = "ONLINE"
//     populationPercent = 100.0
//     labelsOrTypes includes "DocumentChunk"
//     properties includes "embedding"
//
// Once the index is online, the graph is ready for vector similarity search.
//
// In the overall RAG-style workflow, this step sits here:
//
//     1. Load KnowledgeArticle nodes.
//     2. Load DocumentChunk nodes.
//     3. Connect chunks to articles with PART_OF.
//     4. Add embeddings to chunks.
//     5. Create a vector index on DocumentChunk.embedding.
//     6. Verify the vector index is online.
//     7. Run vector similarity search.
//     8. Trace retrieved chunks back to articles and issues.
//
// Production note:
// These demo embeddings are only 3-dimensional for teaching.
// In a real system, the vector.dimensions value must match the actual embedding
// model output dimension exactly.
```

# Step 20 — Verify vector index status

```cypher
// =============================================================================
// VERIFY SPECIFIC VECTOR INDEX BY NAME
// =============================================================================
// This query checks whether the specific vector index named
// documentChunk_embedding_vector exists in the Neo4j database and whether it is
// ready for vector similarity search.
//
// In simple terms, it answers this question:
//
//     "Is my DocumentChunk embedding vector index present,
//      online,
//      fully populated,
//      and configured on the expected label and property?"
//
// This is an important validation step after creating a vector index with:
//
//     CREATE VECTOR INDEX documentChunk_embedding_vector IF NOT EXISTS
//     FOR (dc:DocumentChunk)
//     ON dc.embedding
//
// In a RAG-style workflow, this check is important because vector search depends
// on the index being available and ready.
//
// If the vector index is not online or not fully populated, similarity search
// queries may fail, return incomplete results, or behave differently than
// expected.
//
// This query is useful for:
// - Confirming that the vector index was created
// - Checking whether the vector index is ONLINE
// - Checking whether index population has reached 100%
// - Confirming the index applies to DocumentChunk nodes
// - Confirming the index is built on the embedding property
// - Verifying the vector index provider
// - Documenting index readiness in a lab guide or demo

SHOW VECTOR INDEXES

// =============================================================================
// SHOW ALL VECTOR INDEXES
// =============================================================================
// SHOW VECTOR INDEXES asks Neo4j to list the vector indexes currently defined in
// the active database.
//
// A vector index is a special kind of index used for similarity search over
// embedding vectors.
//
// In normal property lookup, we may ask:
//
//     "Find the chunk where chunkId equals C-K001-001."
//
// But in vector search, we ask:
//
//     "Find the chunks whose embedding vectors are most similar to this query
//      embedding."
//
// That is why vector indexes are very important in semantic search and RAG
// systems.
//
// For this project, the vector index is expected to support searches over:
//
//     (:DocumentChunk).embedding
//
// where embedding is a numeric list such as:
//
//     [0.95, 0.10, 0.05]
//
// This command does not create or modify any index.
// It only reads vector index metadata.

YIELD name, state, populationPercent, type, entityType, labelsOrTypes, properties, indexProvider

// =============================================================================
// SELECT VECTOR INDEX METADATA FIELDS
// =============================================================================
// YIELD chooses which columns from SHOW VECTOR INDEXES we want to inspect.
//
// SHOW VECTOR INDEXES can expose multiple metadata fields, but this query focuses
// on the most useful ones for validating whether the target vector index is
// ready for use.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// name is the vector index name.
//
// In this query, we are looking specifically for:
//
//     documentChunk_embedding_vector
//
// A clear index name is important because vector search queries usually call the
// index by name.
//
// For example, a later vector search query may use:
//
//     db.index.vector.queryNodes(
//       "documentChunk_embedding_vector",
//       3,
//       [0.94, 0.12, 0.04]
//     )
//
// If the index name is misspelled or different from what the search query uses,
// the vector search will not find the expected index.
//
// -----------------------------------------------------------------------------
// state
// -----------------------------------------------------------------------------
// state tells us the current lifecycle state of the vector index.
//
// The ideal value is:
//
//     ONLINE
//
// ONLINE means the index is ready to be used for vector search.
//
// If the state is not ONLINE, the index may still be building, may have failed,
// or may not be ready for query execution.
//
// This is one of the most important fields to check after creating the index.
//
// -----------------------------------------------------------------------------
// populationPercent
// -----------------------------------------------------------------------------
// populationPercent tells us how much of the index population process has
// completed.
//
// The ideal value is:
//
//     100.0
//
// This means Neo4j has fully populated the vector index using the existing
// DocumentChunk.embedding values.
//
// If the value is less than 100.0, the index may still be building.
//
// In that case, vector search should usually wait until the index is fully
// populated.
//
// For readiness, we ideally want:
//
//     state = ONLINE
//     populationPercent = 100.0
//
// Together, these two values tell us the index is ready for reliable use.
//
// -----------------------------------------------------------------------------
// type
// -----------------------------------------------------------------------------
// type tells us what kind of index this is.
//
// Since we are using SHOW VECTOR INDEXES, the type should indicate that this is
// a vector index.
//
// This helps distinguish it from other index types such as:
//
// - range indexes
// - text indexes
// - full-text indexes
// - lookup indexes
//
// A vector index is specifically designed for nearest-neighbor similarity search
// over numeric embedding vectors.
//
// -----------------------------------------------------------------------------
// entityType
// -----------------------------------------------------------------------------
// entityType tells us whether the index applies to nodes or relationships.
//
// For this project, we expect the index to apply to nodes because our embeddings
// are stored on DocumentChunk nodes:
//
//     (:DocumentChunk {embedding: [...]})
//
// So the expected entity type should indicate a node-based index.
//
// This confirms that Neo4j is indexing node properties, not relationship
// properties.
//
// -----------------------------------------------------------------------------
// labelsOrTypes
// -----------------------------------------------------------------------------
// labelsOrTypes tells us which node label or relationship type the vector index
// applies to.
//
// For this vector index, we expect:
//
//     DocumentChunk
//
// This confirms that the index is scoped to DocumentChunk nodes.
//
// That matters because embeddings could theoretically exist on many different
// node types, such as:
//
// - KnowledgeArticle
// - Issue
// - Ticket
// - Product
// - CustomerInteraction
//
// But this particular index is intended for chunk-level retrieval, so it should
// target DocumentChunk.
//
// -----------------------------------------------------------------------------
// properties
// -----------------------------------------------------------------------------
// properties tells us which property is indexed.
//
// For this vector index, we expect:
//
//     embedding
//
// This confirms that the index is built on the numeric vector property.
//
// This is important because the text itself is stored in:
//
//     dc.text
//
// but vector search does not compare plain text directly.
//
// Vector search compares:
//
//     dc.embedding
//
// If the index were built on the wrong property, similarity search would not
// work as intended.
//
// -----------------------------------------------------------------------------
// indexProvider
// -----------------------------------------------------------------------------
// indexProvider tells us which internal Neo4j provider is backing the vector
// index.
//
// This is useful for administration and troubleshooting.
//
// In most lab/demo scenarios, we do not need to manually tune the provider.
// But returning this field helps confirm that Neo4j created the index using a
// vector-capable indexing implementation.

WHERE name = "documentChunk_embedding_vector"

// =============================================================================
// FILTER TO THE TARGET VECTOR INDEX
// =============================================================================
// WHERE filters the vector index metadata so that we only inspect one specific
// index.
//
// Without this WHERE clause, Neo4j would return all vector indexes in the
// database.
//
// Here, we only want the index whose name exactly matches:
//
//     documentChunk_embedding_vector
//
// This makes the result focused and easy to validate.
//
// If the index exists:
// - Neo4j returns one row with the index metadata.
//
// If the index does not exist:
// - Neo4j returns no rows.
//
// A no-row result is useful because it clearly tells us that the expected vector
// index was not found.
//
// Possible reasons for no rows include:
// - The vector index was not created.
// - The index was created with a different name.
// - The query is running against a different database.
// - The connected user does not have permission to view indexes.
// - The CREATE VECTOR INDEX statement failed earlier.
//
// Important:
// The name comparison is exact. A small spelling difference means the filter
// will not match.
//
// For example, these would not match:
//
//     documentchunk_embedding_vector
//     documentChunk_embeddings_vector
//     documentChunk_embedding_index
//     DocumentChunk_embedding_vector

RETURN
  name,
  state,
  populationPercent,
  type,
  entityType,
  labelsOrTypes,
  properties,
  indexProvider;

// =============================================================================
// RETURN THE VECTOR INDEX DETAILS
// =============================================================================
// RETURN defines the final output table shown to the user.
//
// We return:
//
// - name
// - state
// - populationPercent
// - type
// - entityType
// - labelsOrTypes
// - properties
// - indexProvider
//
// These fields are enough to verify whether the vector index exists and whether
// it is ready for use.
//
// A healthy expected result should conceptually look like:
//
//     name                            state   populationPercent   type     entityType   labelsOrTypes       properties     indexProvider
//     ----------------------------------------------------------------------------------------------------------------------------------
//     documentChunk_embedding_vector  ONLINE  100.0               VECTOR   NODE         ["DocumentChunk"]   ["embedding"]  <provider>
//
// The most important checks are:
//
//     name
//       should be documentChunk_embedding_vector
//
//     state
//       should be ONLINE
//
//     populationPercent
//       should be 100.0
//
//     labelsOrTypes
//       should include DocumentChunk
//
//     properties
//       should include embedding
//
// If all of those are correct, the vector index is ready for similarity search.
//
// -----------------------------------------------------------------------------
// What if state is not ONLINE?
// -----------------------------------------------------------------------------
// If state is not ONLINE, the index may not be ready yet.
//
// In that case, wait and rerun this verification query.
//
// If it remains non-online or failed, investigate index creation errors,
// embedding data quality, Neo4j version compatibility, or database logs.
//
// -----------------------------------------------------------------------------
// What if populationPercent is less than 100?
// -----------------------------------------------------------------------------
// If populationPercent is less than 100.0, Neo4j may still be populating the
// index.
//
// This can happen when the index is newly created and existing nodes already
// contain embedding values.
//
// For a small lab dataset, population should usually complete quickly.
//
// For a large production dataset, index population can take more time.
//
// -----------------------------------------------------------------------------
// What if labelsOrTypes or properties are wrong?
// -----------------------------------------------------------------------------
// If labelsOrTypes does not include DocumentChunk, or properties does not include
// embedding, then the index was created on the wrong graph target.
//
// In that case, vector search may run against the wrong data.
//
// The schema should be reviewed and corrected before continuing.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a targeted vector index verification query.
//
// It does not create, update, delete, or query vector similarity results.
//
// It only checks whether the specific vector index:
//
//     documentChunk_embedding_vector
//
// exists and is ready.
//
// The main purpose is:
//
//     "Confirm that Neo4j has an ONLINE, fully populated vector index on
//      DocumentChunk.embedding."
//
// This is a good production-readiness habit because after creating any important
// schema object, especially a vector index, we should verify it before depending
// on it.
//
// In the full RAG-style workflow, this validation step comes after:
//
//     1. Loading DocumentChunk nodes
//     2. Adding embedding vectors
//     3. Creating the vector index
//
// And before:
//
//     4. Running vector similarity search
//     5. Retrieving relevant chunks
//     6. Tracing chunks back to KnowledgeArticle and Issue nodes
//
// Ideal readiness condition:
//
//     state = ONLINE
//     populationPercent = 100.0
//     labelsOrTypes includes DocumentChunk
//     properties includes embedding

```

# Step 21 — Verify vector index configuration

```cypher
// =============================================================================
// VERIFY VECTOR INDEX OPTIONS AND CREATE STATEMENT
// =============================================================================
// This query inspects the detailed definition of one specific Neo4j vector index:
//
//     documentChunk_embedding_vector
//
// In simple terms, it answers this question:
//
//     "What is the current state of my vector index,
//      what configuration options were used to create it,
//      and what Cypher statement can recreate the same index?"
//
// This is slightly different from the earlier vector index verification query.
//
// Earlier, we inspected operational metadata such as:
//
// - populationPercent
// - entityType
// - labelsOrTypes
// - properties
// - indexProvider
//
// In this query, we focus more on the index definition itself:
//
// - options
// - createStatement
//
// This is useful when we want to confirm whether the vector index was created
// with the correct configuration, such as:
//
// - vector dimension
// - similarity function
// - index provider/options
// - full schema creation syntax
//
// This query is especially helpful for:
// - Auditing vector index configuration
// - Documenting schema setup in a lab guide
// - Troubleshooting vector-search behavior
// - Comparing index definitions across environments
// - Recreating the same index in another database
// - Confirming that the index matches the intended RAG/vector-search design

SHOW VECTOR INDEXES

// =============================================================================
// SHOW ALL VECTOR INDEXES
// =============================================================================
// SHOW VECTOR INDEXES asks Neo4j to list vector indexes currently defined in the
// active database.
//
// A vector index is a specialized index used for similarity search over numeric
// embedding vectors.
//
// In this project, we created a vector index on:
//
//     (:DocumentChunk).embedding
//
// That means Neo4j can later search for DocumentChunk nodes whose embedding
// vectors are closest to a query vector.
//
// This is the foundation for semantic search and RAG-style retrieval.
//
// For example, after the index is ready, we can ask Neo4j:
//
//     "Find the top 3 chunks most similar to this query embedding."
//
// Instead of matching exact text, Neo4j compares vector similarity.
//
// This command is read-only.
// It does not create, update, or delete the index.
// It only shows metadata about vector indexes.

YIELD name, state, options, createStatement

// =============================================================================
// SELECT VECTOR INDEX DEFINITION FIELDS
// =============================================================================
// YIELD chooses which columns from SHOW VECTOR INDEXES we want to inspect.
//
// In this query, we are selecting four fields:
//
//     1. name
//     2. state
//     3. options
//     4. createStatement
//
// These fields are useful when we want to inspect the definition and
// configuration of a vector index, not just its existence.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// name is the name of the vector index.
//
// In this query, we expect the index name to be:
//
//     documentChunk_embedding_vector
//
// This name is important because vector search queries usually reference the
// index by name.
//
// For example:
//
//     db.index.vector.queryNodes(
//       "documentChunk_embedding_vector",
//       3,
//       [0.94, 0.12, 0.04]
//     )
//
// If the name is wrong, misspelled, or different from what the query expects,
// vector similarity search will not target the correct index.
//
// -----------------------------------------------------------------------------
// state
// -----------------------------------------------------------------------------
// state tells us the current lifecycle state of the index.
//
// The ideal value is:
//
//     ONLINE
//
// ONLINE means Neo4j considers the index ready for use.
//
// If the state is not ONLINE, the index may still be building, may have failed,
// or may not be ready for vector search yet.
//
// This field is important because having an index definition is not enough.
// The index also needs to be operationally ready.
//
// -----------------------------------------------------------------------------
// options
// -----------------------------------------------------------------------------
// options shows the configuration options associated with the vector index.
//
// This is one of the most important fields in this query.
//
// For a vector index, options can show configuration details such as:
//
//     vector.dimensions
//     vector.similarity_function
//
// In our lab, we expect the vector index configuration to represent:
//
//     vector.dimensions = 3
//     vector.similarity_function = cosine
//
// Why does this matter?
//
// Because vector search only works correctly when the index configuration
// matches the stored embedding vectors.
//
// For example, our sample DocumentChunk embeddings look like:
//
//     [0.95, 0.10, 0.05]
//
// This vector has 3 dimensions.
//
// So the index must also be configured with:
//
//     vector.dimensions = 3
//
// If the index expected a different dimension, such as 768 or 1536, then our
// 3-dimensional demo vectors would not match the index requirements.
//
// The similarity function also matters.
//
// In this lab, we used cosine similarity because it is commonly used for text
// embeddings and semantic similarity search.
//
// Cosine similarity compares the direction of vectors, which is useful when we
// care about semantic closeness rather than only raw numeric distance.
//
// -----------------------------------------------------------------------------
// createStatement
// -----------------------------------------------------------------------------
// createStatement shows the Cypher statement Neo4j can use to recreate this
// vector index.
//
// This is extremely useful for documentation and environment migration.
//
// For example, if we want to recreate the same vector index in another database,
// such as:
//
// - a fresh local Neo4j container
// - a test environment
// - a staging environment
// - a production environment
//
// then createStatement gives us the index creation syntax that represents the
// current index definition.
//
// This helps avoid guessing.
//
// Instead of manually reconstructing the index command from memory, we can
// inspect the actual statement Neo4j reports.
//
// This is also useful for lab guides because students can compare:
//
//     "What we intended to create"
//
// with:
//
//     "What Neo4j says currently exists."

WHERE name = "documentChunk_embedding_vector"

// =============================================================================
// FILTER TO THE SPECIFIC VECTOR INDEX
// =============================================================================
// WHERE filters the output so that we only inspect the vector index named:
//
//     documentChunk_embedding_vector
//
// Without this WHERE clause, Neo4j would return all vector indexes in the
// database.
//
// That may be useful for a full inventory, but here we want a targeted
// verification query.
//
// This line makes the output focused and easy to understand.
//
// If the index exists:
// - Neo4j returns one row with its state, options, and create statement.
//
// If the index does not exist:
// - Neo4j returns no rows.
//
// A no-row result is meaningful.
// It means Neo4j did not find an index with this exact name.
//
// Possible reasons include:
//
// - The vector index was not created.
// - The index was created with a different name.
// - The query is running against a different database.
// - The connected user does not have permission to view index metadata.
// - The index creation query failed earlier.
//
// Important:
// The name comparison is exact.
//
// These names would not match:
//
//     documentchunk_embedding_vector
//     documentChunk_embedding_vector
//     documentChunk_embeddings_vector
//     documentChunk_embedding_index
//
// Exact naming matters because later vector search queries also refer to the
// index by name.

RETURN
  name,
  state,
  options,
  createStatement;

// =============================================================================
// RETURN VECTOR INDEX CONFIGURATION DETAILS
// =============================================================================
// RETURN defines the final output table shown by Neo4j.
//
// Here we return:
//
// - name
// - state
// - options
// - createStatement
//
// This output helps us verify both operational readiness and index definition.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// The name column confirms which vector index was found.
//
// Expected value:
//
//     documentChunk_embedding_vector
//
// This confirms that the query found the intended index.
//
// -----------------------------------------------------------------------------
// state
// -----------------------------------------------------------------------------
// The state column confirms whether the index is ready.
//
// Expected value:
//
//     ONLINE
//
// If the state is ONLINE, the index is available for use.
//
// If the state is not ONLINE, we should wait, recheck, or troubleshoot before
// running vector similarity queries.
//
// -----------------------------------------------------------------------------
// options
// -----------------------------------------------------------------------------
// The options column shows the actual configuration stored for the index.
//
// For this lab, we expect to see configuration equivalent to:
//
//     vector.dimensions = 3
//     vector.similarity_function = cosine
//
// This confirms that the index matches our demo embedding vectors.
//
// The most important thing to check is that:
//
//     vector.dimensions
//
// matches:
//
//     size(dc.embedding)
//
// For this lab, both should be:
//
//     3
//
// If the dimensions do not match, the vector index is not compatible with the
// stored embedding values.
//
// -----------------------------------------------------------------------------
// createStatement
// -----------------------------------------------------------------------------
// The createStatement column gives us the Cypher command that represents how
// Neo4j would create this index.
//
// This is useful because it acts like schema documentation generated directly
// from the database.
//
// A conceptual create statement may look similar to:
//
//     CREATE VECTOR INDEX documentChunk_embedding_vector IF NOT EXISTS
//     FOR (dc:DocumentChunk)
//     ON dc.embedding
//     OPTIONS {
//       indexConfig: {
//         `vector.dimensions`: 3,
//         `vector.similarity_function`: 'cosine'
//       }
//     }
//
// This helps confirm that the database schema matches the intended lab design.
//
// -----------------------------------------------------------------------------
// Expected result for this lab
// -----------------------------------------------------------------------------
// A healthy result should conceptually show:
//
//     name                            state   options                     createStatement
//     ------------------------------------------------------------------------------------
//     documentChunk_embedding_vector  ONLINE  dimensions=3, cosine         CREATE VECTOR INDEX ...
//
// The exact formatting of options and createStatement may vary depending on the
// Neo4j version and client display format.
//
// But the important validations are:
//
// - The index name is correct.
// - The state is ONLINE.
// - The options show 3 dimensions.
// - The options show cosine similarity.
// - The createStatement targets DocumentChunk.embedding.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a targeted vector index definition verification query.
//
// It does not run vector search.
// It does not inspect individual embeddings.
// It does not create or modify any schema object.
//
// It only checks:
//
//     "What does Neo4j say about the exact vector index definition?"
//
// This is a very useful production-readiness and documentation habit.
//
// After creating a vector index, we should verify not only that it exists, but
// also that it was created with the correct configuration.
//
// For this lab, the ideal validation is:
//
//     name = documentChunk_embedding_vector
//     state = ONLINE
//     options include vector.dimensions = 3
//     options include vector.similarity_function = cosine
//     createStatement targets (:DocumentChunk).embedding
//
// Once this is confirmed, the database is ready for the next step:
//
//     Run vector similarity search against documentChunk_embedding_vector.
```

# Step 22 — Run first vector similarity search

```cypher
// =============================================================================
// RUN VECTOR SIMILARITY SEARCH AGAINST DOCUMENT CHUNK EMBEDDINGS
// =============================================================================
// This query performs a vector similarity search using Neo4j's vector index.
//
// In simple terms, it answers this question:
//
//     "Which DocumentChunk nodes are most semantically similar to the query
//      embedding [0.92, 0.12, 0.05]?"
//
// Earlier, we created:
//
//     1. DocumentChunk nodes
//     2. embedding vectors on those chunks
//     3. a vector index named documentChunk_embedding_vector
//
// Now we are using that vector index to retrieve the top matching chunks.
//
// The query embedding:
//
//     [0.92, 0.12, 0.05]
//
// is intentionally close to the login-related chunk embeddings:
//
//     C-K001-001 -> [0.95, 0.10, 0.05]
//     C-K001-002 -> [0.90, 0.15, 0.05]
//
// So we expect the top results to be login-related chunks.
//
// This query is useful for:
// - Testing whether the vector index works
// - Performing semantic search over document chunks
// - Finding chunks similar to a user question
// - Preparing retrieved chunks for RAG-style answers
// - Validating that embeddings were stored and indexed correctly
// - Demonstrating how vector search differs from exact keyword search

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  3,
  [0.92, 0.12, 0.05]
)

// =============================================================================
// CALL VECTOR INDEX QUERY PROCEDURE
// =============================================================================
// CALL is used to execute a Neo4j procedure.
//
// A procedure is like a built-in database function that performs a special task.
//
// Here we are calling:
//
//     db.index.vector.queryNodes()
//
// This procedure searches a vector index and returns nodes whose vector
// properties are most similar to the query vector we provide.
//
// In this case, the vector index is built on:
//
//     (:DocumentChunk).embedding
//
// So the procedure searches DocumentChunk nodes based on their embedding values.
//
// -----------------------------------------------------------------------------
// Parameter 1: 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// The first argument is the vector index name:
//
//     'documentChunk_embedding_vector'
//
// This tells Neo4j exactly which vector index to search.
//
// Earlier, we created this index using:
//
//     CREATE VECTOR INDEX documentChunk_embedding_vector IF NOT EXISTS
//     FOR (dc:DocumentChunk)
//     ON dc.embedding
//
// So this procedure call means:
//
//     "Search inside the vector index that was created for
//      DocumentChunk.embedding."
//
// The name must match exactly.
//
// If the index name is misspelled, Neo4j will not know which vector index to use.
//
// -----------------------------------------------------------------------------
// Parameter 2: 3
// -----------------------------------------------------------------------------
// The second argument tells Neo4j how many nearest results to return.
//
// Here we pass:
//
//     3
//
// That means:
//
//     "Return the top 3 most similar DocumentChunk nodes."
//
// This is often called top-k retrieval.
//
// In RAG systems, top-k controls how many pieces of context we retrieve before
// sending them to an LLM.
//
// For example:
// - top 1 gives only the single best match
// - top 3 gives a small set of likely relevant chunks
// - top 10 gives broader context but may include less relevant results
//
// Choosing top-k is a balance:
//
//     More results = more coverage
//     Fewer results = more focused context
//
// In this lab, top 3 is a good teaching value because it lets us see the best
// matches without flooding the output.
//
// -----------------------------------------------------------------------------
// Parameter 3: [0.92, 0.12, 0.05]
// -----------------------------------------------------------------------------
// The third argument is the query vector.
//
// This vector represents the meaning of the user's search intent.
//
// In a real application, this vector would usually be generated from a user's
// question using an embedding model.
//
// For example, a user may ask:
//
//     "Customer cannot login and OTP is not received"
//
// The application would convert that question into an embedding vector and then
// pass that vector into the vector search procedure.
//
// In this lab, we manually provide a simple 3-dimensional demo vector:
//
//     [0.92, 0.12, 0.05]
//
// This vector is close to the login-related vectors we stored earlier:
//
//     [0.95, 0.10, 0.05]
//     [0.90, 0.15, 0.05]
//
// Because the first dimension is high, this query vector represents the
// login-failure topic in our simplified demo embedding space.
//
// -----------------------------------------------------------------------------
// Why the query vector must match the index dimension
// -----------------------------------------------------------------------------
// The vector index was created with:
//
//     `vector.dimensions`: 3
//
// That means every vector used with this index must have exactly 3 numeric
// values.
//
// This query vector has three values:
//
//     [0.92, 0.12, 0.05]
//
// So it is compatible with the index.
//
// If we passed a vector with only two values:
//
//     [0.92, 0.12]
//
// or four values:
//
//     [0.92, 0.12, 0.05, 0.01]
//
// the vector search would not match the expected index dimension.
//
// Dimension consistency is critical in vector search.

YIELD node AS dc, score

// =============================================================================
// YIELD VECTOR SEARCH RESULTS
// =============================================================================
// YIELD defines which outputs we want from the vector search procedure.
//
// The procedure returns two important values:
//
//     1. node
//     2. score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node represents a matched node returned by the vector index.
//
// Since our vector index was created on DocumentChunk nodes, each returned node
// should be a DocumentChunk.
//
// We rename node to dc using:
//
//     node AS dc
//
// This is useful because dc is a meaningful variable name.
//
// It reminds us that the returned node is a DocumentChunk.
//
// After this alias, we can access its properties like:
//
//     dc.chunkId
//     dc.text
//     dc.embedding
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score represents how similar the returned node's embedding is to the query
// vector.
//
// Because our vector index uses cosine similarity, a higher score generally
// means the chunk embedding is more similar to the query embedding.
//
// In simple terms:
//
//     Higher score = better semantic match
//
// So if the query vector is login-related, login-related chunks should receive
// higher scores than payment or app-crash chunks.
//
// The score is extremely useful because it helps us rank the returned chunks by
// relevance.
//
// In a RAG system, this score can help decide which chunks should be passed to
// the LLM as context.

RETURN
  dc.chunkId AS chunkId,
  dc.text AS chunkText,
  dc.embedding AS embedding,
  score

// =============================================================================
// RETURN MATCHED CHUNKS AND SIMILARITY SCORES
// =============================================================================
// RETURN defines what Neo4j should show in the final result table.
//
// Here we return:
//
//     1. dc.chunkId AS chunkId
//     2. dc.text AS chunkText
//     3. dc.embedding AS embedding
//     4. score
//
// This gives us both the retrieved content and the reason it was ranked.
//
// -----------------------------------------------------------------------------
// dc.chunkId AS chunkId
// -----------------------------------------------------------------------------
// chunkId identifies the returned DocumentChunk.
//
// Example:
//
//     C-K001-001
//     C-K001-002
//
// This helps us know exactly which chunk was retrieved.
//
// Since chunkId is unique, it is safe to use it for tracing, debugging, and
// linking search results back to graph data.
//
// -----------------------------------------------------------------------------
// dc.text AS chunkText
// -----------------------------------------------------------------------------
// chunkText returns the human-readable text stored in the retrieved chunk.
//
// This is the actual content that a RAG system may pass to an LLM.
//
// For example, a returned chunk may say:
//
//     "If a customer cannot sign in, ask them to reset the password and verify
//      OTP delivery."
//
// This text is important because it becomes supporting evidence for the answer.
//
// The vector search finds the chunk mathematically, but the user or LLM still
// needs the readable text.
//
// -----------------------------------------------------------------------------
// dc.embedding AS embedding
// -----------------------------------------------------------------------------
// embedding returns the stored numeric vector for the chunk.
//
// Returning the embedding is useful in a lab because students can visually
// compare the stored vector with the query vector.
//
// Query vector:
//
//     [0.92, 0.12, 0.05]
//
// Login chunk vectors:
//
//     [0.95, 0.10, 0.05]
//     [0.90, 0.15, 0.05]
//
// We can see that these are very close, which explains why login chunks should
// rank highly.
//
// In production, embeddings are usually much larger, so we often do not return
// them in normal application responses.
//
// But for teaching and validation, returning the embedding is very useful.
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score shows the similarity score calculated by the vector index.
//
// Since we are using cosine similarity, the score tells us how close each stored
// embedding is to the query embedding.
//
// A higher score means a stronger match.
//
// Example conceptual output:
//
//     chunkId       chunkText                              embedding             score
//     ---------------------------------------------------------------------------------
//     C-K001-001    If a customer cannot sign in...        [0.95,0.10,0.05]     0.999...
//     C-K001-002    For login failure, clear cache...      [0.90,0.15,0.05]     0.999...
//     C-K003-002    For app crash issues...                [0.15,0.10,0.90]     lower
//
// The exact score values may vary depending on the vector search implementation,
// but the ranking should reflect semantic/vector closeness.

ORDER BY score DESC;

// =============================================================================
// SORT RESULTS BY HIGHEST SIMILARITY SCORE
// =============================================================================
// ORDER BY score DESC sorts the returned chunks from most similar to least
// similar.
//
// DESC means descending order.
//
// So the highest score appears first.
//
// This is exactly what we want in a search result because the most relevant
// chunk should be shown at the top.
//
// Without ORDER BY, the result order may not be as clear or predictable.
//
// Even though vector search procedures usually return results in relevance order,
// explicitly ordering by score makes the intent obvious and makes the query
// easier for students to understand.
//
// -----------------------------------------------------------------------------
// Expected behavior in this lab
// -----------------------------------------------------------------------------
// Since the query vector is:
//
//     [0.92, 0.12, 0.05]
//
// and our login chunks have embeddings:
//
//     C-K001-001 -> [0.95, 0.10, 0.05]
//     C-K001-002 -> [0.90, 0.15, 0.05]
//
// we expect login-related chunks to appear at the top.
//
// That means the top results should likely include:
//
//     C-K001-001
//     C-K001-002
//
// because their vectors are closest to the query vector.
//
// Payment chunks and app-crash chunks should score lower because their vectors
// point more strongly toward different dimensions.
//
// -----------------------------------------------------------------------------
// Why this matters for RAG
// -----------------------------------------------------------------------------
// In a RAG-style workflow, this query represents the retrieval step.
//
// The overall flow looks like:
//
//     1. User asks a question.
//     2. Application converts the question into an embedding.
//     3. Neo4j vector index retrieves the most similar DocumentChunk nodes.
//     4. The application collects the chunk text.
//     5. The chunk text is provided to an LLM as grounding context.
//     6. The LLM generates an answer based on the retrieved context.
//
// So this query is the bridge between:
//
//     "User intent as a vector"
//
// and:
//
//     "Relevant knowledge stored in the graph."
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query performs vector similarity search over DocumentChunk embeddings.
//
// The main flow is:
//
//     1. CALL db.index.vector.queryNodes() to search the vector index.
//     2. Provide the vector index name.
//     3. Ask for the top 3 closest chunks.
//     4. Provide a 3-dimensional query embedding.
//     5. YIELD the matched node and similarity score.
//     6. RETURN the chunk ID, text, embedding, and score.
//     7. ORDER BY score DESC to show the best matches first.
//
// The most important concept is:
//
//     Vector search does not require exact word matching.
//     It finds chunks whose embeddings are mathematically similar to the query
//     embedding.
//
// In this lab, the query vector is login-oriented, so the top results should be
// login-related chunks.
//
// This confirms that:
//
// - embeddings were assigned correctly
// - the vector index is usable
// - similarity search is working
// - the graph is ready for RAG-style retrieval experiments
//
// Production note:
// In a real system, the query vector should come from the same embedding model
// that generated the stored DocumentChunk embeddings. Mixing embedding models or
// dimensions can produce unreliable search results.
```

# Step 23 — Hybrid vector search with graph traversal

```cypher
// =============================================================================
// VECTOR SEARCH WITH GRAPH CONTEXT EXPANSION
// =============================================================================
// This query performs a vector similarity search over DocumentChunk embeddings,
// and then expands from each retrieved chunk to its parent KnowledgeArticle and
// the Issue solved by that article.
//
// In simple terms, it answers this question:
//
//     "Which chunks are most similar to my query vector,
//      which knowledge articles do those chunks belong to,
//      and which business issues do those articles solve?"
//
// This is a very important RAG-style graph query.
//
// The earlier vector search query only returned matching chunks:
//
//     DocumentChunk + score
//
// But this query goes one step further.
//
// After retrieving the most similar chunks, it follows graph relationships:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// This means the result is no longer just a semantic search result.
// It becomes a semantically retrieved result with business context.
//
// The query gives us:
// - the retrieved chunk text
// - the similarity score
// - the parent knowledge article
// - the issue solved by that article
// - the issue severity
//
// This is exactly why combining vector search with graph traversal is powerful.
// Vector search finds relevant text, and the graph explains where that text came
// from and what business problem it supports.

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  3,
  [0.92, 0.12, 0.05]
)

// =============================================================================
// STEP 1: RUN VECTOR SIMILARITY SEARCH
// =============================================================================
// CALL is used to execute a Neo4j procedure.
//
// Here we are calling:
//
//     db.index.vector.queryNodes()
//
// This procedure searches a vector index and returns nodes whose embedding
// vectors are most similar to the query vector provided.
//
// In this project, the vector index:
//
//     documentChunk_embedding_vector
//
// was created on:
//
//     (:DocumentChunk).embedding
//
// So this procedure searches DocumentChunk nodes based on their embedding values.
//
// -----------------------------------------------------------------------------
// Parameter 1: 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// The first parameter is the name of the vector index to search.
//
// This tells Neo4j:
//
//     "Use the vector index named documentChunk_embedding_vector."
//
// The name must match the index name exactly.
//
// If this name is misspelled, Neo4j will not be able to find the intended vector
// index.
//
// This index was designed to search embeddings stored on DocumentChunk nodes.
//
// -----------------------------------------------------------------------------
// Parameter 2: 3
// -----------------------------------------------------------------------------
// The second parameter is the number of nearest results to return.
//
// Here we pass:
//
//     3
//
// That means:
//
//     "Return the top 3 most similar chunks."
//
// This is commonly called top-k retrieval.
//
// In a RAG workflow, top-k controls how many chunks are retrieved as candidate
// context for answering a user question.
//
// A smaller top-k gives focused results.
// A larger top-k gives broader context but may include less relevant chunks.
//
// For this lab, top 3 is a good value because it keeps the output easy to read
// while still showing multiple related chunks.
//
// -----------------------------------------------------------------------------
// Parameter 3: [0.92, 0.12, 0.05]
// -----------------------------------------------------------------------------
// The third parameter is the query embedding.
//
// In a real application, this vector would normally be generated from a user's
// natural-language question using the same embedding model that generated the
// stored chunk embeddings.
//
// For example, a user may ask:
//
//     "Customer cannot login and OTP is not coming."
//
// The application would convert that question into an embedding vector and pass
// that vector into this procedure.
//
// In this lab, we manually provide a simple 3-dimensional vector:
//
//     [0.92, 0.12, 0.05]
//
// This vector is intentionally close to the login-related chunk embeddings:
//
//     C-K001-001 -> [0.95, 0.10, 0.05]
//     C-K001-002 -> [0.90, 0.15, 0.05]
//
// So we expect login-related chunks to appear near the top.
//
// -----------------------------------------------------------------------------
// Why the vector has exactly 3 numbers
// -----------------------------------------------------------------------------
// The vector index was created with:
//
//     `vector.dimensions`: 3
//
// That means the query vector must also contain exactly 3 numeric values.
//
// This query vector is valid because:
//
//     [0.92, 0.12, 0.05]
//
// contains exactly three values.
//
// If we passed a vector with a different number of values, it would not match
// the expected dimension of the index.

YIELD node AS dc, score

// =============================================================================
// STEP 2: CAPTURE VECTOR SEARCH RESULTS
// =============================================================================
// YIELD defines what outputs we want from the vector search procedure.
//
// The vector query procedure returns:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the graph node returned by the vector index.
//
// Since our vector index was created on DocumentChunk nodes, each returned node
// should be a DocumentChunk.
//
// We rename node to dc:
//
//     node AS dc
//
// This makes the rest of the query easier to read.
//
// Instead of writing a generic variable name like node, we use dc to remind
// ourselves:
//
//     "This returned node is a DocumentChunk."
//
// After this alias, we can use properties such as:
//
//     dc.chunkId
//     dc.text
//     dc.embedding
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score represents the similarity score between the query vector and the stored
// chunk embedding.
//
// Because the vector index uses cosine similarity, a higher score means the
// returned chunk is more similar to the query vector.
//
// In simple terms:
//
//     Higher score = stronger semantic match
//
// This score is important because it tells us how relevant the chunk is from the
// vector-search perspective.
//
// In RAG systems, this score is often used to rank retrieved context before
// passing it to an LLM.

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// STEP 3: EXPAND FROM RETRIEVED CHUNK TO ARTICLE AND ISSUE
// =============================================================================
// After vector search returns the most similar DocumentChunk nodes, this MATCH
// clause follows graph relationships from each retrieved chunk.
//
// The pattern is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// In plain English:
//
//     "For each retrieved chunk,
//      find the KnowledgeArticle it is part of,
//      then find the Issue solved by that KnowledgeArticle."
//
// This is the graph-context expansion step.
//
// Vector search gives us relevance.
// Graph traversal gives us meaning, lineage, and business context.
//
// -----------------------------------------------------------------------------
// (dc)-[:PART_OF]->(ka:KnowledgeArticle)
// -----------------------------------------------------------------------------
// This part of the pattern connects the retrieved chunk to its parent article.
//
// The relationship:
//
//     PART_OF
//
// means:
//
//     "This DocumentChunk is part of this KnowledgeArticle."
//
// Example:
//
//     (:DocumentChunk {chunkId: "C-K001-001"})
//         -[:PART_OF]->
//     (:KnowledgeArticle {articleId: "K001"})
//
// This is important because the chunk is only a small piece of text.
// The KnowledgeArticle gives us the larger source document or support article
// that the chunk came from.
//
// Without this relationship, the vector search result would return text, but we
// would not easily know which article the text belongs to.
//
// -----------------------------------------------------------------------------
// (ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
// -----------------------------------------------------------------------------
// This second part of the pattern connects the parent article to the issue it
// solves.
//
// The relationship:
//
//     SOLVES
//
// means:
//
//     "This KnowledgeArticle solves this Issue."
//
// Example:
//
//     (:KnowledgeArticle {articleId: "K001"})
//         -[:SOLVES]->
//     (:Issue {name: "Login Failure"})
//
// This adds business meaning to the retrieved chunk.
//
// Now we are not only saying:
//
//     "This chunk is similar to the query."
//
// We are also saying:
//
//     "This chunk comes from an article that solves a specific issue."
//
// That is much more useful in a support automation or RAG system.
//
// -----------------------------------------------------------------------------
// Why this MATCH comes after vector search
// -----------------------------------------------------------------------------
// The query first performs vector search because we want to retrieve only the
// most relevant chunks.
//
// Then, for only those retrieved chunks, it expands into the graph.
//
// This is efficient and meaningful because we do not traverse the entire graph
// first. We start from the semantically relevant chunks and then enrich those
// results with graph context.
//
// Conceptually:
//
//     Query vector
//         -> find similar chunks
//         -> follow PART_OF to article
//         -> follow SOLVES to issue
//         -> return enriched result
//
// -----------------------------------------------------------------------------
// Important behavior of MATCH
// -----------------------------------------------------------------------------
// MATCH behaves like an inner join.
//
// That means the result row continues only if the full pattern exists:
//
//     DocumentChunk -> KnowledgeArticle -> Issue
//
// If a retrieved chunk does not have a PART_OF relationship, it will be dropped.
//
// If the parent KnowledgeArticle does not have a SOLVES relationship to an Issue,
// it will also be dropped.
//
// This is good for strict validation because the query only returns complete,
// well-connected RAG context.
//
// However, in some production scenarios, you may want optional context.
//
// In that case, you could use OPTIONAL MATCH to keep the chunk even if article
// or issue context is missing.
//
// But for this lab, MATCH is better because it confirms that the graph has the
// expected relationships.

RETURN
  dc.chunkId AS chunkId,
  dc.text AS chunkText,
  score,
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS issueSeverity

// =============================================================================
// STEP 4: RETURN ENRICHED VECTOR SEARCH RESULTS
// =============================================================================
// RETURN defines the final output table.
//
// This query returns data from three levels of the graph:
//
//     1. DocumentChunk
//     2. KnowledgeArticle
//     3. Issue
//
// It also returns the vector similarity score.
//
// This makes the result useful for both technical validation and business
// explanation.
//
// -----------------------------------------------------------------------------
// dc.chunkId AS chunkId
// -----------------------------------------------------------------------------
// chunkId identifies the retrieved DocumentChunk.
//
// Example:
//
//     C-K001-001
//
// This helps us trace exactly which chunk was returned by vector search.
//
// Since chunkId is unique, it is useful for debugging, citations, audits, and
// retrieval traceability.
//
// -----------------------------------------------------------------------------
// dc.text AS chunkText
// -----------------------------------------------------------------------------
// chunkText is the actual text stored in the retrieved chunk.
//
// This is the content that may later be passed to an LLM as grounding context.
//
// For example:
//
//     "If a customer cannot sign in, ask them to reset the password and verify
//      OTP delivery."
//
// This field is important because vector search returns a node, but the LLM or
// user needs readable text.
//
// In a real RAG pipeline, this chunk text becomes part of the evidence used to
// generate the final answer.
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score shows how similar the chunk embedding is to the query embedding.
//
// Higher score means stronger vector similarity.
//
// This helps us rank the returned chunks.
//
// For this query vector:
//
//     [0.92, 0.12, 0.05]
//
// we expect login-related chunks to have high scores because their embeddings
// are close to the query vector.
//
// The score helps explain why a particular chunk was returned.
//
// -----------------------------------------------------------------------------
// ka.articleId AS articleId
// -----------------------------------------------------------------------------
// articleId identifies the parent KnowledgeArticle.
//
// Example:
//
//     K001
//
// This tells us which article the chunk belongs to.
//
// This is important because a chunk is only one small part of a larger knowledge
// article. The articleId lets us trace the result back to the full source.
//
// -----------------------------------------------------------------------------
// ka.title AS articleTitle
// -----------------------------------------------------------------------------
// articleTitle gives the readable title of the parent article.
//
// Example:
//
//     Fix login failure
//
// This makes the result more understandable to humans.
//
// Instead of only seeing a chunk ID, we can immediately understand the broader
// article topic.
//
// -----------------------------------------------------------------------------
// i.issueId AS issueId
// -----------------------------------------------------------------------------
// issueId identifies the Issue node connected to the article.
//
// Example:
//
//     I001
//
// This is useful for stable issue tracking.
//
// In production systems, IDs are often safer than names because names can change
// over time, but IDs should remain stable.
//
// -----------------------------------------------------------------------------
// i.name AS issueName
// -----------------------------------------------------------------------------
// issueName gives the readable name of the issue solved by the article.
//
// Example:
//
//     Login Failure
//
// This field explains the business problem associated with the retrieved chunk.
//
// If the query vector is login-oriented, we expect the top chunks to connect to
// an issue like:
//
//     Login Failure
//
// That confirms the vector search and graph traversal are aligned.
//
// -----------------------------------------------------------------------------
// i.severity AS issueSeverity
// -----------------------------------------------------------------------------
// issueSeverity shows the priority or seriousness of the issue.
//
// Example values might be:
//
//     High
//     Medium
//     Low
//     Critical
//
// Severity is useful because not all retrieved knowledge has the same business
// urgency.
//
// In a support system, severity can help decide:
// - how urgently to show the result
// - whether to escalate the issue
// - how to prioritize recommended actions
// - whether the answer should include stronger guidance or warnings
//
// This is a good example of why graph context matters.
//
// The vector index finds the relevant chunk.
// The graph tells us the business importance of the issue.

ORDER BY score DESC;

// =============================================================================
// STEP 5: SORT BY BEST VECTOR MATCH FIRST
// =============================================================================
// ORDER BY score DESC sorts the final results by similarity score in descending
// order.
//
// DESC means highest score first.
//
// This is exactly what we want for search results because the most relevant
// chunk should appear at the top.
//
// Expected behavior for this lab:
//
// The query vector is:
//
//     [0.92, 0.12, 0.05]
//
// This is close to login-related chunk embeddings:
//
//     C-K001-001 -> [0.95, 0.10, 0.05]
//     C-K001-002 -> [0.90, 0.15, 0.05]
//
// So the top results should likely be chunks from article:
//
//     K001 - Fix login failure
//
// and the connected issue should likely be:
//
//     Login Failure
//
// This result proves that the end-to-end flow is working:
//
//     vector query
//         -> relevant chunks
//         -> parent article
//         -> solved issue
//         -> severity context
//
// -----------------------------------------------------------------------------
// Why sorting by score matters
// -----------------------------------------------------------------------------
// In a RAG pipeline, the order of retrieved context can matter.
//
// Higher-ranked chunks are usually more relevant, so they may be placed earlier
// in the prompt or given more importance by downstream logic.
//
// Sorting by score makes the retrieval output predictable, explainable, and easy
// to validate.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query combines vector similarity search with graph traversal.
//
// It does not merely return the closest chunks.
// It enriches each vector-search result with article and issue context.
//
// The complete logic is:
//
//     1. Search the vector index for the top 3 chunks similar to the query vector.
//     2. Treat each returned node as a DocumentChunk.
//     3. Follow PART_OF from the chunk to its parent KnowledgeArticle.
//     4. Follow SOLVES from the article to the related Issue.
//     5. Return chunk text, similarity score, article details, and issue details.
//     6. Sort the results by highest similarity score.
//
// The key learning point is:
//
//     Vector search answers: "What text is semantically similar?"
//     Graph traversal answers: "Where did this text come from, and what does it
//     mean in the business domain?"
//
// Together, they create a stronger RAG retrieval pattern than vector search
// alone.
//
// In production, this pattern is powerful because retrieved chunks can be
// returned with traceability:
//
//     chunk -> article -> issue -> severity
//
// That means the system can produce answers that are not only relevant, but also
// explainable, auditable, and connected to business context.
//
// Production note:
// Your pasted query contains -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the real Cypher arrow:
//
//     MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
```

# Step 24A — Final Day 3 validation summary

```cypher
// =============================================================================
// END-TO-END DAY 3 GRAPH READINESS VALIDATION QUERY
// =============================================================================
// This query performs a complete health check of the Day 3 Neo4j knowledge graph.
//
// In simple terms, it answers this question:
//
//     "Is my graph ready for vector-search and RAG-style retrieval?"
//
// This query validates multiple important pieces together:
//
// 1. How many KnowledgeArticle nodes exist?
// 2. How many DocumentChunk nodes exist?
// 3. How many chunks have embeddings?
// 4. What embedding dimensions are present?
// 5. How many SOLVES relationships exist between KnowledgeArticle and Issue?
// 6. How many PART_OF relationships exist between DocumentChunk and KnowledgeArticle?
// 7. Is the vector index present?
// 8. What is the vector index state?
// 9. Is the vector index fully populated?
// 10. Is the vector index built on the expected label and property?
//
// This is a strong final validation query because it checks the graph from both
// a data-modeling perspective and a vector-search readiness perspective.
//
// By the end of this query, we should be able to confirm:
//
//     KnowledgeArticle nodes exist.
//     DocumentChunk nodes exist.
//     Chunks have embeddings.
//     Embeddings have the correct dimension.
//     Articles are connected to Issues using SOLVES.
//     Chunks are connected to Articles using PART_OF.
//     The vector index is available and ready for search.
//
// This kind of query is especially useful for:
// - lab completion checks
// - demo validation
// - troubleshooting
// - production-readiness review
// - validating repeatable setup scripts
// - confirming an end-to-end RAG graph pipeline

MATCH (ka:KnowledgeArticle)

// =============================================================================
// STEP 1: COUNT KNOWLEDGE ARTICLE NODES
// =============================================================================
// This MATCH finds all KnowledgeArticle nodes in the database.
//
// The pattern:
//
//     (ka:KnowledgeArticle)
//
// means:
//
//     "Find every node labelled KnowledgeArticle and call each one ka."
//
// KnowledgeArticle nodes represent the main support articles or knowledge-base
// entries loaded into the graph.
//
// For this lab, we previously loaded sample articles such as:
//
//     K001 - Fix login failure
//     K002 - Resolve payment failure
//     K003 - Fix app crash
//
// So we usually expect the KnowledgeArticle count to be:
//
//     3
//
// This count confirms that the article-level knowledge exists before we inspect
// chunks, relationships, embeddings, or vector indexes.

WITH count(ka) AS knowledgeArticleCount

// =============================================================================
// CARRY FORWARD KNOWLEDGE ARTICLE COUNT
// =============================================================================
// WITH is used to pass intermediate results from one part of a Cypher query to
// the next part.
//
// Here:
//
//     count(ka) AS knowledgeArticleCount
//
// counts all matched KnowledgeArticle nodes and stores the result in a variable
// called:
//
//     knowledgeArticleCount
//
// This is important because once we move to the next MATCH clause, we still want
// to remember the article count.
//
// Think of WITH as creating a checkpoint:
//
//     "Store this result, then continue with the next validation step."
//
// Without WITH, later parts of the query would not have access to this aggregated
// count in a clean and controlled way.

MATCH (dc:DocumentChunk)

// =============================================================================
// STEP 2: COUNT DOCUMENT CHUNKS AND EMBEDDING COVERAGE
// =============================================================================
// This MATCH finds all DocumentChunk nodes in the database.
//
// The pattern:
//
//     (dc:DocumentChunk)
//
// means:
//
//     "Find every node labelled DocumentChunk and call each one dc."
//
// DocumentChunk nodes represent smaller pieces of text split from the larger
// KnowledgeArticle records.
//
// In RAG-style systems, chunks are often the main retrieval unit because they are
// more focused than full articles.
//
// For this lab, we previously loaded six chunks:
//
//     C-K001-001
//     C-K001-002
//     C-K002-001
//     C-K002-002
//     C-K003-001
//     C-K003-002
//
// So we usually expect the DocumentChunk count to be:
//
//     6
//
// This step also checks embedding readiness, because vector search depends on
// chunks having numeric embedding vectors.

WITH
  knowledgeArticleCount,
  count(dc) AS documentChunkCount,
  count(dc.embedding) AS chunksWithEmbedding,
  collect(DISTINCT size(dc.embedding)) AS embeddingDimensions

// =============================================================================
// CARRY FORWARD ARTICLE COUNT AND CALCULATE CHUNK/EMBEDDING SUMMARY
// =============================================================================
// This WITH clause does several things at once.
//
// First, it carries forward:
//
//     knowledgeArticleCount
//
// from the previous step.
//
// Then it calculates three new values:
//
//     documentChunkCount
//     chunksWithEmbedding
//     embeddingDimensions
//
// -----------------------------------------------------------------------------
// count(dc) AS documentChunkCount
// -----------------------------------------------------------------------------
// count(dc) counts all DocumentChunk nodes matched in the previous MATCH clause.
//
// This tells us the total number of chunks in the graph.
//
// Expected lab value:
//
//     documentChunkCount = 6
//
// This confirms that the chunk-loading step completed successfully.
//
// -----------------------------------------------------------------------------
// count(dc.embedding) AS chunksWithEmbedding
// -----------------------------------------------------------------------------
// count(dc.embedding) counts how many DocumentChunk nodes have a non-null
// embedding property.
//
// This is different from count(dc):
//
// - count(dc) counts all chunks.
// - count(dc.embedding) counts only chunks where embedding exists.
//
// For a healthy vector-search setup, we usually want:
//
//     chunksWithEmbedding = documentChunkCount
//
// For this lab, the ideal result is:
//
//     chunksWithEmbedding = 6
//
// If this number is lower than documentChunkCount, then some chunks are missing
// embeddings and may not participate in vector search.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT size(dc.embedding)) AS embeddingDimensions
// -----------------------------------------------------------------------------
// size(dc.embedding) returns the number of values inside each embedding vector.
//
// For example:
//
//     size([0.95, 0.10, 0.05])
//
// returns:
//
//     3
//
// collect(DISTINCT ...) collects only the unique embedding dimensions present
// across all chunks.
//
// For this lab, we expect:
//
//     embeddingDimensions = [3]
//
// This is important because a vector index expects consistent vector dimensions.
//
// If this result shows:
//
//     [3, 4]
//
// that means some embeddings have 3 dimensions and some have 4 dimensions, which
// is usually a serious data-quality problem for vector search.
//
// If the list contains null, that may indicate some chunks are missing embeddings.

MATCH (:KnowledgeArticle)-[s:SOLVES]->(:Issue)

// =============================================================================
// STEP 3: COUNT SOLVES RELATIONSHIPS
// =============================================================================
// This MATCH finds SOLVES relationships from KnowledgeArticle nodes to Issue
// nodes.
//
// The pattern:
//
//     (:KnowledgeArticle)-[s:SOLVES]->(:Issue)
//
// means:
//
//     "Find every relationship where a KnowledgeArticle solves an Issue,
//      and call that relationship s."
//
// We use anonymous nodes here:
//
//     (:KnowledgeArticle)
//     (:Issue)
//
// because we do not need to return article or issue details in this summary.
// We only need to count the relationships.
//
// The SOLVES relationship represents this business meaning:
//
//     "This knowledge article solves this issue."
//
// For example:
//
//     (:KnowledgeArticle {articleId: "K001"})
//         -[:SOLVES]->
//     (:Issue {name: "Login Failure"})
//
// For this lab, if three articles each solve one issue, we usually expect:
//
//     solvesRelationshipCount = 3
//
// This count confirms that article-to-issue graph modeling was completed.

WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  count(s) AS solvesRelationshipCount

// =============================================================================
// CARRY FORWARD COUNTS AND STORE SOLVES RELATIONSHIP COUNT
// =============================================================================
// This WITH clause carries forward all previously calculated validation values:
//
// - knowledgeArticleCount
// - documentChunkCount
// - chunksWithEmbedding
// - embeddingDimensions
//
// Then it calculates:
//
//     count(s) AS solvesRelationshipCount
//
// count(s) counts how many SOLVES relationships were matched.
//
// This value confirms whether the article-to-issue relationship layer exists.
//
// Expected lab value:
//
//     solvesRelationshipCount = 3
//
// If this count is 0, it may mean:
//
// - SOLVES relationships were not created.
// - Issue nodes do not exist.
// - KnowledgeArticle.issueType did not match Issue.name.
// - Relationships were created in the wrong direction.
// - The query is running against a different database.

MATCH (:DocumentChunk)-[p:PART_OF]->(:KnowledgeArticle)

// =============================================================================
// STEP 4: COUNT PART_OF RELATIONSHIPS
// =============================================================================
// This MATCH finds PART_OF relationships from DocumentChunk nodes to
// KnowledgeArticle nodes.
//
// The pattern:
//
//     (:DocumentChunk)-[p:PART_OF]->(:KnowledgeArticle)
//
// means:
//
//     "Find every relationship where a DocumentChunk is part of a
//      KnowledgeArticle, and call that relationship p."
//
// The PART_OF relationship represents this meaning:
//
//     "This chunk belongs to this parent knowledge article."
//
// For example:
//
//     (:DocumentChunk {chunkId: "C-K001-001"})
//         -[:PART_OF]->
//     (:KnowledgeArticle {articleId: "K001"})
//
// For this lab, if we loaded six chunks and each chunk belongs to one article,
// we usually expect:
//
//     partOfRelationshipCount = 6
//
// This count is important because chunk lineage is essential in RAG systems.
//
// Vector search may retrieve a chunk, but the PART_OF relationship lets us trace
// that chunk back to its parent article.

WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  solvesRelationshipCount,
  count(p) AS partOfRelationshipCount

// =============================================================================
// CARRY FORWARD ALL COUNTS AND STORE PART_OF RELATIONSHIP COUNT
// =============================================================================
// This WITH clause carries forward the full validation state so far:
//
// - knowledgeArticleCount
// - documentChunkCount
// - chunksWithEmbedding
// - embeddingDimensions
// - solvesRelationshipCount
//
// Then it calculates:
//
//     count(p) AS partOfRelationshipCount
//
// count(p) counts all PART_OF relationships matched in the previous step.
//
// Expected lab value:
//
//     partOfRelationshipCount = 6
//
// A healthy result usually means:
//
//     partOfRelationshipCount = documentChunkCount
//
// because every chunk should normally be connected to exactly one parent article.
//
// If partOfRelationshipCount is lower than documentChunkCount, some chunks may
// be orphaned or not connected to their parent KnowledgeArticle.

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

// =============================================================================
// STEP 5: CHECK VECTOR INDEX READINESS USING A SUBQUERY
// =============================================================================
// CALL () { ... } defines a subquery.
//
// A subquery is like a smaller query inside the main query.
//
// We use a subquery here because SHOW VECTOR INDEXES is a schema-inspection
// command, and we want to combine its result with the counts calculated earlier.
//
// In simple terms, this subquery asks:
//
//     "Find the vector index named documentChunk_embedding_vector,
//      and return its state, population percentage, labels, and properties."
//
// -----------------------------------------------------------------------------
// SHOW VECTOR INDEXES
// -----------------------------------------------------------------------------
// SHOW VECTOR INDEXES lists vector indexes currently defined in the database.
//
// A vector index is a special index used for similarity search over embedding
// vectors.
//
// In this lab, the expected vector index is:
//
//     documentChunk_embedding_vector
//
// It should be built on:
//
//     (:DocumentChunk).embedding
//
// This index allows Neo4j to efficiently find chunks whose embeddings are close
// to a query embedding.
//
// -----------------------------------------------------------------------------
// YIELD name, state, populationPercent, labelsOrTypes, properties
// -----------------------------------------------------------------------------
// YIELD selects the vector index metadata fields we want to inspect.
//
// The fields are:
//
// - name
// - state
// - populationPercent
// - labelsOrTypes
// - properties
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// name is the vector index name.
//
// We use it in the WHERE clause to find the specific index:
//
//     documentChunk_embedding_vector
//
// -----------------------------------------------------------------------------
// state
// -----------------------------------------------------------------------------
// state tells us whether the vector index is ready.
//
// The ideal value is:
//
//     ONLINE
//
// ONLINE means Neo4j considers the index usable for vector search.
//
// -----------------------------------------------------------------------------
// populationPercent
// -----------------------------------------------------------------------------
// populationPercent tells us how much of the index population process has
// completed.
//
// The ideal value is:
//
//     100.0
//
// This means the index has fully processed the existing embedding values.
//
// -----------------------------------------------------------------------------
// labelsOrTypes
// -----------------------------------------------------------------------------
// labelsOrTypes tells us which node label or relationship type the index applies
// to.
//
// For this lab, we expect it to include:
//
//     DocumentChunk
//
// That confirms the index targets the correct node label.
//
// -----------------------------------------------------------------------------
// properties
// -----------------------------------------------------------------------------
// properties tells us which property is indexed.
//
// For this lab, we expect it to include:
//
//     embedding
//
// That confirms the index is built on DocumentChunk.embedding.
//
// -----------------------------------------------------------------------------
// WHERE name = "documentChunk_embedding_vector"
// -----------------------------------------------------------------------------
// This filters the vector index list to only the index we care about.
//
// If the index exists, the subquery returns its readiness metadata.
//
// If the index does not exist, the subquery returns no row.
//
// Important note:
// In a strict Cypher pipeline, if this subquery returns no rows, the whole query
// may return no final result. That is useful in one sense because it indicates
// the expected vector index is missing, but if we wanted the final summary to
// still appear even when the index is missing, we would need an OPTIONAL-style
// pattern or a different query structure.
//
// For this lab, this strict behavior is acceptable because we expect the vector
// index to exist by this stage.
//
// -----------------------------------------------------------------------------
// RETURN aliases from the subquery
// -----------------------------------------------------------------------------
// The subquery returns:
//
//     state AS vectorIndexState
//     populationPercent AS vectorIndexPopulation
//     labelsOrTypes AS vectorIndexLabels
//     properties AS vectorIndexProperties
//
// These aliases make the final result more readable.
//
// Instead of returning generic column names like:
//
//     state
//     populationPercent
//
// the final output clearly says:
//
//     vectorIndexState
//     vectorIndexPopulation

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

// =============================================================================
// FINAL VALIDATION SUMMARY OUTPUT
// =============================================================================
// RETURN defines the final summary table.
//
// This output combines graph data counts, relationship counts, embedding
// readiness, and vector index readiness into one result.
//
// -----------------------------------------------------------------------------
// knowledgeArticleCount
// -----------------------------------------------------------------------------
// Shows how many KnowledgeArticle nodes exist.
//
// Expected lab value:
//
//     3
//
// This confirms the article-level knowledge base exists.
//
// -----------------------------------------------------------------------------
// documentChunkCount
// -----------------------------------------------------------------------------
// Shows how many DocumentChunk nodes exist.
//
// Expected lab value:
//
//     6
//
// This confirms the chunk-level knowledge was loaded.
//
// -----------------------------------------------------------------------------
// solvesRelationshipCount
// -----------------------------------------------------------------------------
// Shows how many SOLVES relationships exist from KnowledgeArticle to Issue.
//
// Expected lab value:
//
//     3
//
// This confirms articles are connected to the issues they solve.
//
// -----------------------------------------------------------------------------
// partOfRelationshipCount
// -----------------------------------------------------------------------------
// Shows how many PART_OF relationships exist from DocumentChunk to
// KnowledgeArticle.
//
// Expected lab value:
//
//     6
//
// This confirms chunks are connected to their parent articles.
//
// -----------------------------------------------------------------------------
// chunksWithEmbedding
// -----------------------------------------------------------------------------
// Shows how many DocumentChunk nodes have an embedding property.
//
// Expected lab value:
//
//     6
//
// This should ideally match:
//
//     documentChunkCount
//
// If chunksWithEmbedding is less than documentChunkCount, some chunks are not
// ready for vector search.
//
// -----------------------------------------------------------------------------
// embeddingDimensions
// -----------------------------------------------------------------------------
// Shows the distinct embedding dimensions present across DocumentChunk nodes.
//
// Expected lab value:
//
//     [3]
//
// This confirms all embeddings have the same 3-dimensional structure used in
// this demo.
//
// A healthy vector-search setup should normally show one consistent dimension.
//
// -----------------------------------------------------------------------------
// vectorIndexState
// -----------------------------------------------------------------------------
// Shows the current state of the vector index.
//
// Expected value:
//
//     ONLINE
//
// ONLINE means the vector index is ready to be used.
//
// -----------------------------------------------------------------------------
// vectorIndexPopulation
// -----------------------------------------------------------------------------
// Shows how much of the vector index population is complete.
//
// Expected value:
//
//     100.0
//
// This means the index is fully populated.
//
// -----------------------------------------------------------------------------
// vectorIndexLabels
// -----------------------------------------------------------------------------
// Shows the label or labels targeted by the vector index.
//
// Expected value should include:
//
//     DocumentChunk
//
// This confirms the vector index is built on the correct node type.
//
// -----------------------------------------------------------------------------
// vectorIndexProperties
// -----------------------------------------------------------------------------
// Shows the property or properties indexed by the vector index.
//
// Expected value should include:
//
//     embedding
//
// This confirms the vector index is built on the correct vector property.
//
// =============================================================================
// EXPECTED HEALTHY RESULT FOR THIS LAB
// =============================================================================
// A healthy Day 3 graph should conceptually return:
//
//     knowledgeArticleCount      = 3
//     documentChunkCount         = 6
//     solvesRelationshipCount    = 3
//     partOfRelationshipCount    = 6
//     chunksWithEmbedding        = 6
//     embeddingDimensions        = [3]
//     vectorIndexState           = ONLINE
//     vectorIndexPopulation      = 100.0
//     vectorIndexLabels          = ["DocumentChunk"]
//     vectorIndexProperties      = ["embedding"]
//
// If the result matches these expectations, it means the graph is ready for
// vector-search and RAG-style retrieval.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is an end-to-end validation checkpoint.
//
// It does not create, update, or delete graph data.
// It only verifies whether the graph is structurally and semantically ready.
//
// The query validates:
//
//     Article layer:
//       KnowledgeArticle nodes exist.
//
//     Chunk layer:
//       DocumentChunk nodes exist.
//
//     Relationship layer:
//       SOLVES and PART_OF relationships exist.
//
//     Embedding layer:
//       chunks have embeddings with consistent dimensions.
//
//     Vector index layer:
//       the vector index exists, is online, and targets the expected label/property.
//
// This is exactly the kind of query we run at the end of a lab or ingestion
// pipeline to prove that all required components are in place.
//
// Production note:
// Your pasted query contains -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the real Cypher arrow:
//
//     MATCH (:KnowledgeArticle)-[s:SOLVES]->(:Issue)
//     MATCH (:DocumentChunk)-[p:PART_OF]->(:KnowledgeArticle)
```

# Step 24B — Final vector index validation summary

```cypher
// =============================================================================
// QUICK VECTOR INDEX READINESS CHECK
// =============================================================================
// This query verifies the readiness of one specific Neo4j vector index:
//
//     documentChunk_embedding_vector
//
// In simple terms, it answers this question:
//
//     "Is the vector index for DocumentChunk embeddings online,
//      fully populated,
//      and pointing to the expected label and property?"
//
// This is a focused version of the vector-index validation query.
//
// Instead of returning every metadata field, this query returns only the most
// important readiness fields:
//
// - vectorIndexState
// - vectorIndexPopulation
// - vectorIndexLabels
// - vectorIndexProperties
//
// These fields are enough to confirm whether the vector index is ready for
// semantic search / RAG retrieval.
//
// In this lab, the expected healthy result is:
//
//     vectorIndexState       = ONLINE
//     vectorIndexPopulation  = 100.0
//     vectorIndexLabels      = ["DocumentChunk"]
//     vectorIndexProperties  = ["embedding"]
//
// If these values are correct, then the vector index is ready to support
// similarity search using:
//
//     CALL db.index.vector.queryNodes(...)

SHOW VECTOR INDEXES

// =============================================================================
// SHOW VECTOR INDEX METADATA
// =============================================================================
// SHOW VECTOR INDEXES asks Neo4j to list vector indexes currently defined in the
// active database.
//
// A vector index is a special index designed for similarity search over numeric
// embedding vectors.
//
// In this project, we created a vector index on:
//
//     (:DocumentChunk).embedding
//
// That means Neo4j can use this index to quickly find DocumentChunk nodes whose
// embedding vectors are similar to a query vector.
//
// This is the core mechanism behind semantic retrieval in this lab.
//
// The command itself is read-only.
// It does not create, update, or delete any graph data or index definition.
// It only exposes metadata about existing vector indexes.

YIELD name, state, populationPercent, labelsOrTypes, properties

// =============================================================================
// SELECT ONLY THE IMPORTANT READINESS FIELDS
// =============================================================================
// YIELD chooses which columns from SHOW VECTOR INDEXES we want to use.
//
// Here we select:
//
//     name
//     state
//     populationPercent
//     labelsOrTypes
//     properties
//
// These are the key fields needed to verify that the vector index is usable.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// name is the vector index name.
//
// We need this field because the database may contain multiple vector indexes.
//
// For example, a larger graph could have separate vector indexes for:
//
// - DocumentChunk embeddings
// - KnowledgeArticle embeddings
// - Product embeddings
// - Ticket embeddings
//
// The name allows us to filter down to the one specific index used in this lab:
//
//     documentChunk_embedding_vector
//
// -----------------------------------------------------------------------------
// state
// -----------------------------------------------------------------------------
// state tells us the operational status of the vector index.
//
// The value we want is:
//
//     ONLINE
//
// ONLINE means the index is ready to be used by vector search queries.
//
// If the state is not ONLINE, the index may still be building, may have failed,
// or may not be available for search yet.
//
// This is one of the most important readiness checks.
//
// -----------------------------------------------------------------------------
// populationPercent
// -----------------------------------------------------------------------------
// populationPercent tells us how much of the index population process has
// completed.
//
// The value we want is:
//
//     100.0
//
// This means Neo4j has fully populated the index using the existing embedding
// values.
//
// If this value is below 100.0, the index may still be catching up.
//
// In that case, we should wait and rerun this query before depending on vector
// search results.
//
// -----------------------------------------------------------------------------
// labelsOrTypes
// -----------------------------------------------------------------------------
// labelsOrTypes tells us which node label or relationship type the index applies
// to.
//
// For this lab, we expect it to include:
//
//     DocumentChunk
//
// That confirms the vector index is scoped to the correct node type.
//
// This matters because the graph may contain many labels, but our semantic
// retrieval unit is specifically the DocumentChunk node.
//
// -----------------------------------------------------------------------------
// properties
// -----------------------------------------------------------------------------
// properties tells us which property is indexed.
//
// For this lab, we expect it to include:
//
//     embedding
//
// That confirms Neo4j is indexing the numeric vector property, not another field.
//
// This is important because vector search compares embeddings, not raw text.
//
// The readable text is stored in:
//
//     dc.text
//
// but the vector-search index is built on:
//
//     dc.embedding

WHERE name = "documentChunk_embedding_vector"

// =============================================================================
// FILTER TO THE VECTOR INDEX USED BY THIS LAB
// =============================================================================
// WHERE filters the SHOW VECTOR INDEXES output so we only inspect one specific
// index.
//
// Without this filter, Neo4j would return all vector indexes in the database.
//
// Here we only want:
//
//     documentChunk_embedding_vector
//
// This makes the output focused and easy to validate.
//
// If this index exists, Neo4j returns one row.
//
// If this index does not exist, Neo4j returns no rows.
//
// A no-row result means one of the following may be true:
//
// - The vector index was not created.
// - The index was created with a different name.
// - The query is running against a different database.
// - The connected user does not have permission to view index metadata.
// - The index creation command failed earlier.
//
// Important:
// The name comparison is exact.
// Even a small spelling or capitalization difference will prevent a match.

RETURN
  state AS vectorIndexState,
  populationPercent AS vectorIndexPopulation,
  labelsOrTypes AS vectorIndexLabels,
  properties AS vectorIndexProperties;

// =============================================================================
// RETURN CLEAN READINESS OUTPUT
// =============================================================================
// RETURN defines the final output shown by Neo4j.
//
// Instead of returning raw column names, we use aliases that clearly describe
// what each value means.
//
// -----------------------------------------------------------------------------
// state AS vectorIndexState
// -----------------------------------------------------------------------------
// This returns the index state using a more descriptive output column name.
//
// Expected value:
//
//     ONLINE
//
// If this shows ONLINE, the vector index is operationally ready.
//
// If it shows another value, wait or troubleshoot before running vector search.
//
// -----------------------------------------------------------------------------
// populationPercent AS vectorIndexPopulation
// -----------------------------------------------------------------------------
// This returns how fully populated the index is.
//
// Expected value:
//
//     100.0
//
// A value of 100.0 means the index has finished processing the stored embedding
// vectors.
//
// This is important because vector search should ideally run only after the
// index is fully populated.
//
// -----------------------------------------------------------------------------
// labelsOrTypes AS vectorIndexLabels
// -----------------------------------------------------------------------------
// This returns the label or relationship type targeted by the vector index.
//
// Expected value should include:
//
//     DocumentChunk
//
// That confirms the index is built for the correct graph entity.
//
// -----------------------------------------------------------------------------
// properties AS vectorIndexProperties
// -----------------------------------------------------------------------------
// This returns the indexed property.
//
// Expected value should include:
//
//     embedding
//
// That confirms the index is built on the correct vector property.
//
// =============================================================================
// EXPECTED HEALTHY RESULT
// =============================================================================
// For this lab, a healthy result should conceptually look like:
//
//     vectorIndexState       vectorIndexPopulation   vectorIndexLabels     vectorIndexProperties
//     ------------------------------------------------------------------------------------------
//     ONLINE                 100.0                   ["DocumentChunk"]     ["embedding"]
//
// This means:
//
// - The vector index exists.
// - The vector index is online.
// - The vector index is fully populated.
// - The vector index targets DocumentChunk nodes.
// - The vector index indexes the embedding property.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a compact vector-index readiness check.
//
// It does not inspect the actual vectors.
// It does not run similarity search.
// It does not create or modify the index.
//
// It only verifies:
//
//     "Is the vector index ready for use?"
//
// If the returned values match the expected healthy result, the graph is ready
// for the next step:
//
//     CALL db.index.vector.queryNodes(
//       'documentChunk_embedding_vector',
//       3,
//       [0.92, 0.12, 0.05]
//     )
//
// In the overall RAG workflow, this query confirms that the retrieval
// infrastructure is ready before we run semantic search.
```

# Addendum A — Step A1: Current knowledge graph audit

```cypher
// =============================================================================
// COMPACT GRAPH READINESS SUMMARY QUERY
// =============================================================================
// This query performs a compact validation check for the Day 3 Neo4j knowledge
// graph.
//
// In simple terms, it answers this question:
//
//     "Do we have the expected articles, chunks, embeddings, and relationships
//      needed for the graph-based RAG workflow?"
//
// This query checks five important areas:
//
// 1. KnowledgeArticle node count
// 2. DocumentChunk node count
// 3. Number of chunks that have embeddings
// 4. Distinct embedding dimensions present in the graph
// 5. Relationship counts for:
//    - KnowledgeArticle -[:SOLVES]-> Issue
//    - DocumentChunk -[:PART_OF]-> KnowledgeArticle
//
// This is a useful end-of-step validation query because it gives one clean row
// showing whether the core graph structure is ready.
//
// Expected healthy result for this lab is usually:
//
//     knowledgeArticleCount    = 3
//     documentChunkCount       = 6
//     chunksWithEmbedding      = 6
//     embeddingDimensions      = [3]
//     solvesRelationshipCount  = 3
//     partOfRelationshipCount  = 6
//
// If these values match expectations, it means:
// - Articles were loaded.
// - Chunks were loaded.
// - Chunks received embeddings.
// - Embedding dimensions are consistent.
// - Articles are connected to Issues.
// - Chunks are connected to Articles.
//
// This query does not check the vector index itself.
// It focuses only on graph data, embeddings, and relationships.

MATCH (ka:KnowledgeArticle)

// =============================================================================
// STEP 1: COUNT KNOWLEDGE ARTICLE NODES
// =============================================================================
// This MATCH finds all nodes labelled KnowledgeArticle.
//
// The pattern:
//
//     (ka:KnowledgeArticle)
//
// means:
//
//     "Find every KnowledgeArticle node in the graph and temporarily call each
//      one ka."
//
// KnowledgeArticle nodes represent the article-level knowledge base records.
//
// In this lab, examples may include:
//
//     K001 - Fix login failure
//     K002 - Resolve payment failure
//     K003 - Fix app crash
//
// Counting these nodes helps confirm that the knowledge article loading step
// completed successfully.
//
// If this count is lower than expected, then later steps such as chunk linking,
// issue linking, and RAG retrieval may not work correctly because the parent
// knowledge articles may be missing.

WITH count(ka) AS knowledgeArticleCount

// =============================================================================
// STORE ARTICLE COUNT AND CONTINUE
// =============================================================================
// WITH passes intermediate values from one part of the query to the next.
//
// Here:
//
//     count(ka) AS knowledgeArticleCount
//
// counts all matched KnowledgeArticle nodes and stores the count in a variable
// named:
//
//     knowledgeArticleCount
//
// Think of WITH like a checkpoint.
//
// It says:
//
//     "Save the article count, then continue to the next validation step."
//
// This is necessary because we want the final RETURN statement to include the
// article count along with chunk counts, embedding checks, and relationship
// counts.

MATCH (dc:DocumentChunk)

// =============================================================================
// STEP 2: MATCH DOCUMENT CHUNK NODES
// =============================================================================
// This MATCH finds all nodes labelled DocumentChunk.
//
// The pattern:
//
//     (dc:DocumentChunk)
//
// means:
//
//     "Find every DocumentChunk node in the graph and temporarily call each one
//      dc."
//
// DocumentChunk nodes represent the smaller text pieces created from the larger
// KnowledgeArticle records.
//
// In a RAG-style workflow, chunks are very important because they usually become
// the searchable retrieval units.
//
// Instead of retrieving an entire article, the system can retrieve the most
// relevant chunk.
//
// For this lab, we expect chunks such as:
//
//     C-K001-001
//     C-K001-002
//     C-K002-001
//     C-K002-002
//     C-K003-001
//     C-K003-002
//
// So the expected chunk count is usually:
//
//     6

WITH
  knowledgeArticleCount,
  count(dc) AS documentChunkCount,
  count(dc.embedding) AS chunksWithEmbedding,
  collect(DISTINCT size(dc.embedding)) AS embeddingDimensions

// =============================================================================
// STORE CHUNK COUNT AND EMBEDDING READINESS DETAILS
// =============================================================================
// This WITH clause carries forward the previous article count and calculates
// chunk-level validation values.
//
// It produces:
//
// - documentChunkCount
// - chunksWithEmbedding
// - embeddingDimensions
//
// -----------------------------------------------------------------------------
// knowledgeArticleCount
// -----------------------------------------------------------------------------
// We include knowledgeArticleCount here so it is not lost.
//
// In Cypher, each WITH clause controls which variables are passed forward.
//
// If we do not include a variable in WITH, it is not available in the next part
// of the query.
//
// So this line keeps the article count available for the final output.
//
// -----------------------------------------------------------------------------
// count(dc) AS documentChunkCount
// -----------------------------------------------------------------------------
// count(dc) counts all DocumentChunk nodes matched by the previous MATCH.
//
// This tells us the total number of chunks currently present in the graph.
//
// Expected lab value:
//
//     documentChunkCount = 6
//
// This confirms that the chunk-loading step completed successfully.
//
// -----------------------------------------------------------------------------
// count(dc.embedding) AS chunksWithEmbedding
// -----------------------------------------------------------------------------
// count(dc.embedding) counts only DocumentChunk nodes where the embedding
// property exists and is not null.
//
// This is different from count(dc):
//
// - count(dc) counts every DocumentChunk node.
// - count(dc.embedding) counts only chunks that actually have embeddings.
//
// This is an important readiness check for vector search.
//
// Ideally:
//
//     chunksWithEmbedding = documentChunkCount
//
// For this lab, the expected result is:
//
//     chunksWithEmbedding = 6
//
// If chunksWithEmbedding is lower than documentChunkCount, some chunks are
// missing embeddings and may not be available for vector similarity search.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT size(dc.embedding)) AS embeddingDimensions
// -----------------------------------------------------------------------------
// size(dc.embedding) returns the length of the embedding vector.
//
// For example:
//
//     size([0.95, 0.10, 0.05])
//
// returns:
//
//     3
//
// collect(DISTINCT ...) collects only the unique embedding dimensions found
// across all DocumentChunk nodes.
//
// In this lab, every embedding should be 3-dimensional, so the expected result is:
//
//     embeddingDimensions = [3]
//
// This is important because vector search requires consistent vector dimensions.
//
// If this result shows something like:
//
//     [3, 4]
//
// then some embeddings have different dimensions, which is usually a data-quality
// problem.
//
// If it shows:
//
//     [3, null]
//
// then some chunks may be missing embeddings.
//
// For a healthy vector-search setup, we want one clean dimension value:
//
//     [3]

MATCH (:KnowledgeArticle)-[s:SOLVES]->(:Issue)

// =============================================================================
// STEP 3: COUNT SOLVES RELATIONSHIPS
// =============================================================================
// This MATCH finds all SOLVES relationships from KnowledgeArticle nodes to Issue
// nodes.
//
// The pattern:
//
//     (:KnowledgeArticle)-[s:SOLVES]->(:Issue)
//
// means:
//
//     "Find every relationship where a KnowledgeArticle solves an Issue,
//      and call that relationship s."
//
// We use anonymous nodes here:
//
//     (:KnowledgeArticle)
//     (:Issue)
//
// because we do not need article or issue details in this summary.
// We only need to count the relationships.
//
// The SOLVES relationship represents this business meaning:
//
//     "This knowledge article solves this issue."
//
// Example:
//
//     (:KnowledgeArticle {articleId: "K001"})
//         -[:SOLVES]->
//     (:Issue {name: "Login Failure"})
//
// For this lab, if three knowledge articles each solve one issue, the expected
// count is:
//
//     solvesRelationshipCount = 3
//
// If this count is lower than expected, possible reasons include:
// - SOLVES relationships were not created.
// - Issue nodes are missing.
// - KnowledgeArticle.issueType did not match Issue.name.
// - Relationships were created in the wrong direction.
// - The query is being run against a different database.

WITH
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  count(s) AS solvesRelationshipCount

// =============================================================================
// STORE SOLVES COUNT AND CONTINUE
// =============================================================================
// This WITH clause carries forward all previously calculated values:
//
// - knowledgeArticleCount
// - documentChunkCount
// - chunksWithEmbedding
// - embeddingDimensions
//
// It also calculates:
//
//     count(s) AS solvesRelationshipCount
//
// count(s) counts how many SOLVES relationships were matched.
//
// This value confirms whether the article-to-issue relationship layer exists.
//
// Expected lab value:
//
//     solvesRelationshipCount = 3
//
// This is important because the RAG graph should not only retrieve chunks.
// It should also understand which business issue the parent article solves.

MATCH (:DocumentChunk)-[p:PART_OF]->(:KnowledgeArticle)

// =============================================================================
// STEP 4: COUNT PART_OF RELATIONSHIPS
// =============================================================================
// This MATCH finds all PART_OF relationships from DocumentChunk nodes to
// KnowledgeArticle nodes.
//
// The pattern:
//
//     (:DocumentChunk)-[p:PART_OF]->(:KnowledgeArticle)
//
// means:
//
//     "Find every relationship where a DocumentChunk is part of a
//      KnowledgeArticle, and call that relationship p."
//
// The PART_OF relationship represents chunk lineage.
//
// In plain English:
//
//     "This chunk belongs to this parent article."
//
// Example:
//
//     (:DocumentChunk {chunkId: "C-K001-001"})
//         -[:PART_OF]->
//     (:KnowledgeArticle {articleId: "K001"})
//
// This relationship is very important in RAG-style systems.
//
// Vector search may retrieve a chunk, but after retrieving it, we usually need
// to trace it back to:
//
// - the parent article
// - the article title
// - the source
// - the issue solved by the article
//
// The PART_OF relationship enables that traceability.
//
// For this lab, if six chunks were loaded and each chunk belongs to one article,
// the expected count is:
//
//     partOfRelationshipCount = 6

RETURN
  knowledgeArticleCount,
  documentChunkCount,
  chunksWithEmbedding,
  embeddingDimensions,
  solvesRelationshipCount,
  count(p) AS partOfRelationshipCount;

// =============================================================================
// FINAL SUMMARY OUTPUT
// =============================================================================
// RETURN produces the final validation result.
//
// This query returns one compact row containing:
//
// - knowledgeArticleCount
// - documentChunkCount
// - chunksWithEmbedding
// - embeddingDimensions
// - solvesRelationshipCount
// - partOfRelationshipCount
//
// -----------------------------------------------------------------------------
// knowledgeArticleCount
// -----------------------------------------------------------------------------
// Shows how many KnowledgeArticle nodes exist.
//
// Expected lab value:
//
//     3
//
// This confirms that the article-level knowledge base was loaded.
//
// -----------------------------------------------------------------------------
// documentChunkCount
// -----------------------------------------------------------------------------
// Shows how many DocumentChunk nodes exist.
//
// Expected lab value:
//
//     6
//
// This confirms that the article content was split into chunk-level nodes.
//
// -----------------------------------------------------------------------------
// chunksWithEmbedding
// -----------------------------------------------------------------------------
// Shows how many DocumentChunk nodes have an embedding property.
//
// Expected lab value:
//
//     6
//
// This should ideally match:
//
//     documentChunkCount
//
// If it does not match, some chunks are missing embeddings.
//
// -----------------------------------------------------------------------------
// embeddingDimensions
// -----------------------------------------------------------------------------
// Shows the distinct embedding dimensions found across DocumentChunk nodes.
//
// Expected lab value:
//
//     [3]
//
// This confirms all embeddings have the same expected 3-dimensional structure
// used in this lab.
//
// A healthy vector-search setup should normally have only one consistent
// embedding dimension.
//
// -----------------------------------------------------------------------------
// solvesRelationshipCount
// -----------------------------------------------------------------------------
// Shows how many SOLVES relationships exist from KnowledgeArticle to Issue.
//
// Expected lab value:
//
//     3
//
// This confirms that knowledge articles are connected to the issues they solve.
//
// -----------------------------------------------------------------------------
// count(p) AS partOfRelationshipCount
// -----------------------------------------------------------------------------
// count(p) counts all PART_OF relationships matched in the previous MATCH.
//
// Expected lab value:
//
//     6
//
// This confirms that each DocumentChunk is connected back to its parent
// KnowledgeArticle.
//
// A healthy result usually means:
//
//     partOfRelationshipCount = documentChunkCount
//
// because every chunk should normally belong to one parent article.
//
// =============================================================================
// EXPECTED HEALTHY RESULT FOR THIS LAB
// =============================================================================
// A healthy result should conceptually look like:
//
//     knowledgeArticleCount      = 3
//     documentChunkCount         = 6
//     chunksWithEmbedding        = 6
//     embeddingDimensions        = [3]
//     solvesRelationshipCount    = 3
//     partOfRelationshipCount    = 6
//
// This means:
//
// - The article layer is present.
// - The chunk layer is present.
// - All chunks have embeddings.
// - Embedding dimensions are consistent.
// - Articles are connected to Issues.
// - Chunks are connected to Articles.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a compact graph readiness validation query.
//
// It does not create, update, or delete anything.
// It only reads the current graph state and summarizes whether the required
// Day 3 graph components exist.
//
// The query validates:
//
//     1. Article data
//     2. Chunk data
//     3. Embedding coverage
//     4. Embedding dimension consistency
//     5. Article-to-Issue relationships
//     6. Chunk-to-Article relationships
//
// This is a good checkpoint before running graph-enhanced vector search because
// it confirms that both the semantic-search data and the relationship context
// are present.
//
// Production note:
// Your pasted query contains -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the real Cypher arrow:
//
//     MATCH (:KnowledgeArticle)-[s:SOLVES]->(:Issue)
//     MATCH (:DocumentChunk)-[p:PART_OF]->(:KnowledgeArticle)
```

# Addendum A — Step A2: Detect orphan knowledge graph records

```cypher
// =============================================================================
// DATA QUALITY CHECK FOR MISSING GRAPH RELATIONSHIPS AND EMBEDDINGS
// =============================================================================
// This query performs a data-quality validation check across the Day 3 Neo4j
// knowledge graph.
//
// In simple terms, it answers this question:
//
//     "Are there any important records that are incomplete or disconnected?"
//
// Specifically, this query checks for three possible problems:
//
// 1. KnowledgeArticle nodes that do NOT have a SOLVES relationship to an Issue.
// 2. DocumentChunk nodes that do NOT have a PART_OF relationship to a KnowledgeArticle.
// 3. DocumentChunk nodes that do NOT have an embedding vector.
//
// These are important checks because a RAG-style graph depends on connected and
// enriched data.
//
// For example:
//
// - A KnowledgeArticle without a SOLVES relationship may exist, but the graph
//   does not know which Issue it solves.
// - A DocumentChunk without a PART_OF relationship may contain useful text, but
//   the graph cannot trace it back to its parent article.
// - A DocumentChunk without an embedding may exist in the graph, but it may not
//   participate in vector similarity search.
//
// This query combines all three checks into one result using UNION ALL.
//
// The output is designed like a troubleshooting report:
//
//     issueType             -> what kind of data-quality issue was found
//     recordId              -> which record has the issue
//     recordTitleOrText     -> readable title/text to help identify the record
//
// If this query returns no rows, that is usually a good sign.
// It means no missing SOLVES relationships, missing PART_OF relationships, or
// missing embeddings were found by these checks.

MATCH (ka:KnowledgeArticle)

// =============================================================================
// CHECK 1: FIND KNOWLEDGE ARTICLES
// =============================================================================
// This MATCH finds all KnowledgeArticle nodes in the database.
//
// The pattern:
//
//     (ka:KnowledgeArticle)
//
// means:
//
//     "Find every node labelled KnowledgeArticle and temporarily call each one ka."
//
// KnowledgeArticle nodes represent the article-level knowledge records.
//
// In this lab, examples include:
//
//     K001 - Fix login failure
//     K002 - Resolve payment failure
//     K003 - Fix app crash
//
// Each KnowledgeArticle should ideally be connected to an Issue using:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// That relationship tells the graph:
//
//     "This article solves this issue."
//
// The next WHERE clause checks whether that expected relationship is missing.

WHERE NOT EXISTS {
  MATCH (ka)-[:SOLVES]->(:Issue)
}

// =============================================================================
// FILTER KNOWLEDGE ARTICLES MISSING SOLVES RELATIONSHIP
// =============================================================================
// WHERE NOT EXISTS { ... } is used here as a negative existence check.
//
// In simple terms, this block means:
//
//     "Keep only those KnowledgeArticle nodes for which this SOLVES pattern
//      does NOT exist."
//
// The pattern inside the NOT EXISTS block is:
//
//     MATCH (ka)-[:SOLVES]->(:Issue)
//
// This asks:
//
//     "Does this KnowledgeArticle have a SOLVES relationship pointing to an
//      Issue node?"
//
// If the answer is yes:
// - The KnowledgeArticle is healthy for this check.
// - It is filtered out and not returned.
//
// If the answer is no:
// - The KnowledgeArticle is missing its SOLVES relationship.
// - It is returned as a data-quality issue.
//
// -----------------------------------------------------------------------------
// Why this matters
// -----------------------------------------------------------------------------
// A KnowledgeArticle without a SOLVES relationship is incomplete from a graph
// modeling perspective.
//
// It may still have useful properties like:
//
//     ka.articleId
//     ka.title
//     ka.content
//
// But without the relationship:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// the graph cannot answer connected questions such as:
//
//     "Which issue does this article solve?"
//     "Which articles solve Login Failure?"
//     "Which high-severity issues have recommended articles?"
//
// This relationship is especially important when vector search results are
// expanded into graph context.
//
// If an article is not connected to an Issue, then a retrieved chunk may be
// traceable to the article, but not to the business issue it supports.

RETURN
  "KnowledgeArticle without SOLVES relationship" AS issueType,
  ka.articleId AS recordId,
  ka.title AS recordTitleOrText

// =============================================================================
// RETURN KNOWLEDGE ARTICLE DATA-QUALITY ISSUE
// =============================================================================
// This RETURN produces rows for KnowledgeArticle nodes that failed the first
// validation check.
//
// Each returned row has three columns:
//
//     issueType
//     recordId
//     recordTitleOrText
//
// -----------------------------------------------------------------------------
// "KnowledgeArticle without SOLVES relationship" AS issueType
// -----------------------------------------------------------------------------
// This hardcoded text describes the problem found.
//
// It makes the final output self-explanatory.
//
// Instead of only returning an article ID, the output clearly says:
//
//     "This KnowledgeArticle is missing a SOLVES relationship."
//
// This is useful because the full query combines multiple checks using UNION ALL.
// The issueType column tells us which check produced each row.
//
// -----------------------------------------------------------------------------
// ka.articleId AS recordId
// -----------------------------------------------------------------------------
// recordId identifies the affected KnowledgeArticle.
//
// Example:
//
//     K001
//
// Using the generic alias recordId is intentional because later UNION branches
// will return DocumentChunk chunk IDs in the same column.
//
// This keeps the final output shape consistent across different checks.
//
// -----------------------------------------------------------------------------
// ka.title AS recordTitleOrText
// -----------------------------------------------------------------------------
// recordTitleOrText gives a readable description of the affected record.
//
// For KnowledgeArticle nodes, the readable value is the article title.
//
// Example:
//
//     Fix login failure
//
// This helps a human quickly understand which record needs attention.

UNION ALL

// =============================================================================
// COMBINE WITH NEXT DATA-QUALITY CHECK
// =============================================================================
// UNION ALL combines the result rows from this check with the result rows from
// the next check.
//
// This is useful because we want one consolidated data-quality report instead
// of running three separate queries.
//
// Important rule:
//
// All UNION branches must return the same number of columns with compatible
// column names.
//
// That is why each branch returns:
//
//     issueType
//     recordId
//     recordTitleOrText
//
// -----------------------------------------------------------------------------
// Why UNION ALL instead of UNION?
// -----------------------------------------------------------------------------
// UNION removes duplicate rows.
// UNION ALL keeps all rows.
//
// For validation and troubleshooting, UNION ALL is usually better because we do
// not want Neo4j to silently remove rows that happen to look similar.
//
// If multiple records have the same title or text, we still want to see every
// issue row.

MATCH (dc:DocumentChunk)

// =============================================================================
// CHECK 2: FIND DOCUMENT CHUNKS
// =============================================================================
// This MATCH finds all DocumentChunk nodes in the database.
//
// The pattern:
//
//     (dc:DocumentChunk)
//
// means:
//
//     "Find every node labelled DocumentChunk and temporarily call each one dc."
//
// DocumentChunk nodes represent smaller pieces of text split from larger
// KnowledgeArticle records.
//
// In a RAG-style system, chunks are usually the main retrieval unit.
//
// Each DocumentChunk should ideally be connected back to its parent
// KnowledgeArticle using:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)
//
// That relationship provides lineage.
//
// It tells the graph:
//
//     "This chunk is part of this knowledge article."
//
// The next WHERE clause checks whether that expected relationship is missing.

WHERE NOT EXISTS {
  MATCH (dc)-[:PART_OF]->(:KnowledgeArticle)
}

// =============================================================================
// FILTER DOCUMENT CHUNKS MISSING PART_OF RELATIONSHIP
// =============================================================================
// This WHERE NOT EXISTS block keeps only those DocumentChunk nodes that do not
// have a PART_OF relationship to a KnowledgeArticle.
//
// The pattern inside the NOT EXISTS block is:
//
//     MATCH (dc)-[:PART_OF]->(:KnowledgeArticle)
//
// This asks:
//
//     "Does this DocumentChunk point to a parent KnowledgeArticle using PART_OF?"
//
// If the answer is yes:
// - The chunk has proper lineage.
// - It is not returned by this check.
//
// If the answer is no:
// - The chunk is disconnected from its parent article.
// - It is returned as a data-quality issue.
//
// -----------------------------------------------------------------------------
// Why this matters
// -----------------------------------------------------------------------------
// A DocumentChunk without PART_OF may still contain useful text.
//
// But it becomes difficult to trace that text back to:
//
// - the parent article
// - the article title
// - the article source
// - the issue solved by the article
// - the broader business context
//
// In RAG systems, this traceability is very important.
//
// If vector search retrieves an orphan chunk, the system may know the chunk text,
// but it may not know where that chunk came from.
//
// That weakens explainability, citation quality, and troubleshooting.
//
// A healthy graph should usually satisfy:
//
//     Every DocumentChunk has one PART_OF relationship to a KnowledgeArticle.

RETURN
  "DocumentChunk without PART_OF relationship" AS issueType,
  dc.chunkId AS recordId,
  dc.text AS recordTitleOrText

// =============================================================================
// RETURN DOCUMENT CHUNK RELATIONSHIP ISSUE
// =============================================================================
// This RETURN produces rows for DocumentChunk nodes that failed the PART_OF
// relationship validation check.
//
// -----------------------------------------------------------------------------
// "DocumentChunk without PART_OF relationship" AS issueType
// -----------------------------------------------------------------------------
// This hardcoded text describes the exact issue found.
//
// It tells us:
//
//     "This chunk exists, but it is not connected to a parent article."
//
// This makes the final report easy to read.
//
// -----------------------------------------------------------------------------
// dc.chunkId AS recordId
// -----------------------------------------------------------------------------
// recordId identifies the affected DocumentChunk.
//
// Example:
//
//     C-K001-001
//
// This is useful because chunkId is the stable identifier we can use to inspect
// or repair the chunk.
//
// -----------------------------------------------------------------------------
// dc.text AS recordTitleOrText
// -----------------------------------------------------------------------------
// recordTitleOrText returns the chunk text.
//
// For DocumentChunk records, the text is more useful than a title because chunks
// usually do not have their own title.
//
// Returning the text helps us understand what content is affected.

UNION ALL

// =============================================================================
// COMBINE WITH FINAL DATA-QUALITY CHECK
// =============================================================================
// This second UNION ALL combines the first two issue checks with the final check.
//
// At this point, the final output may include:
//
// - KnowledgeArticle records missing SOLVES relationships
// - DocumentChunk records missing PART_OF relationships
// - DocumentChunk records missing embeddings
//
// Because all branches return the same three columns, Neo4j can combine them
// into one consistent result table.

MATCH (dc:DocumentChunk)

// =============================================================================
// CHECK 3: FIND DOCUMENT CHUNKS AGAIN FOR EMBEDDING VALIDATION
// =============================================================================
// This MATCH again finds all DocumentChunk nodes.
//
// We run a separate MATCH here because this is a separate validation branch in
// the UNION ALL query.
//
// This branch focuses only on embedding coverage.
//
// A DocumentChunk should ideally have an embedding property like:
//
//     dc.embedding = [0.95, 0.10, 0.05]
//
// In this lab, embeddings are 3-dimensional demo vectors.
//
// In production, embeddings may have hundreds or thousands of dimensions,
// depending on the embedding model.
//
// The next WHERE clause checks whether the embedding is missing.

WHERE dc.embedding IS NULL

// =============================================================================
// FILTER DOCUMENT CHUNKS WITHOUT EMBEDDINGS
// =============================================================================
// WHERE dc.embedding IS NULL keeps only DocumentChunk nodes where the embedding
// property is missing or null.
//
// In simple terms, this means:
//
//     "Show me chunks that do not have an embedding vector."
//
// -----------------------------------------------------------------------------
// Why this matters
// -----------------------------------------------------------------------------
// Vector search depends on embeddings.
//
// If a DocumentChunk does not have an embedding, it cannot be properly compared
// against a query vector in the vector index.
//
// That means the chunk may be invisible to semantic search, even though it
// exists in the graph.
//
// For example, a chunk may contain useful troubleshooting text, but without an
// embedding, a vector-search query may not retrieve it.
//
// A healthy vector-search setup usually requires:
//
//     chunksWithEmbedding = total DocumentChunk count
//
// This check helps identify the exact chunks that are missing embeddings.

RETURN
  "DocumentChunk without embedding" AS issueType,
  dc.chunkId AS recordId,
  dc.text AS recordTitleOrText;

// =============================================================================
// RETURN DOCUMENT CHUNKS MISSING EMBEDDINGS
// =============================================================================
// This RETURN produces rows for DocumentChunk nodes that do not have an
// embedding property.
//
// -----------------------------------------------------------------------------
// "DocumentChunk without embedding" AS issueType
// -----------------------------------------------------------------------------
// This hardcoded text describes the issue found.
//
// It tells us:
//
//     "This chunk exists, but it has no embedding vector."
//
// This is important because embedding coverage is required before reliable
// vector similarity search.
//
// -----------------------------------------------------------------------------
// dc.chunkId AS recordId
// -----------------------------------------------------------------------------
// recordId identifies the affected DocumentChunk.
//
// Example:
//
//     C-K002-001
//
// This helps us locate the exact chunk that needs embedding generation or update.
//
// -----------------------------------------------------------------------------
// dc.text AS recordTitleOrText
// -----------------------------------------------------------------------------
// recordTitleOrText returns the text of the affected chunk.
//
// This helps us understand what content is currently missing an embedding.
//
// =============================================================================
// EXPECTED HEALTHY RESULT
// =============================================================================
// In a healthy Day 3 graph, this query should return:
//
//     no rows
//
// No rows means:
//
// - Every KnowledgeArticle has a SOLVES relationship to an Issue.
// - Every DocumentChunk has a PART_OF relationship to a KnowledgeArticle.
// - Every DocumentChunk has an embedding.
//
// This is one of the few cases where an empty result is a good result.
//
// It means the query did not find any of the data-quality issues it was looking
// for.
//
// =============================================================================
// IF ROWS ARE RETURNED, HOW TO READ THEM
// =============================================================================
// If rows are returned, read them like a repair checklist.
//
// Example output:
//
//     issueType                                      recordId      recordTitleOrText
//     --------------------------------------------------------------------------------
//     KnowledgeArticle without SOLVES relationship   K004          Troubleshoot refund delay
//     DocumentChunk without PART_OF relationship     C-K005-001    Customer cannot update profile...
//     DocumentChunk without embedding                C-K002-002    For payment failure, confirm...
//
// This tells us exactly:
//
// - what kind of issue exists
// - which record has the issue
// - what readable title or text helps identify it
//
// After reviewing the rows, we would fix the graph by creating the missing
// relationships or adding the missing embeddings.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a consolidated data-quality issue report.
//
// It does not create, update, or delete anything.
// It only finds records that are incomplete from the perspective of the graph
// and vector-search workflow.
//
// The three checks are:
//
//     1. KnowledgeArticle without SOLVES relationship
//     2. DocumentChunk without PART_OF relationship
//     3. DocumentChunk without embedding
//
// These checks are important because a RAG-ready graph needs:
//
//     connected articles,
//     connected chunks,
//     and embedded chunks.
//
// A clean result means the graph is structurally connected and semantically
// ready for vector retrieval.
//
// Production note:
// Your pasted query contains -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the real Cypher arrow:
//
//     MATCH (ka)-[:SOLVES]->(:Issue)
//     MATCH (dc)-[:PART_OF]->(:KnowledgeArticle)

```

# Addendum A — Step A3: Validate Issue-to-KnowledgeArticle coverage

```cypher
// =============================================================================
// ISSUE-TO-KNOWLEDGE ARTICLE COVERAGE REPORT
// =============================================================================
// This query creates a coverage report for Issue nodes and their related
// KnowledgeArticle nodes.
//
// In simple terms, it answers this question:
//
//     "For each Issue in the graph, how many KnowledgeArticles solve it,
//      and which articles are they?"
//
// This is an important validation and reporting query because it starts from
// Issue nodes and checks whether support knowledge exists for each issue.
//
// Earlier, we created relationships like:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// That relationship means:
//
//     "This knowledge article solves this issue."
//
// This query reads that relationship in reverse direction:
//
//     (:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// but conceptually reports from the Issue perspective:
//
//     Issue -> articles that solve it
//
// This is useful for:
// - Checking support knowledge coverage by issue
// - Finding issues that have no linked articles
// - Verifying that each issue has the expected troubleshooting content
// - Building dashboards or reports for knowledge-base completeness
// - Identifying gaps where new knowledge articles need to be created
// - Preparing graph data for customer support or RAG-style workflows
//
// A key point is that this query uses OPTIONAL MATCH.
// That means it returns every Issue node even if no KnowledgeArticle is linked
// to it.
//
// This is very important for coverage reporting because missing relationships
// are often just as important as existing relationships.

MATCH (i:Issue)

// =============================================================================
// STEP 1: MATCH ALL ISSUE NODES
// =============================================================================
// This MATCH finds every Issue node in the graph.
//
// The pattern:
//
//     (i:Issue)
//
// means:
//
//     "Find all nodes labelled Issue and temporarily call each one i."
//
// Issue nodes represent the business problems or support categories in the
// graph.
//
// Examples may include:
//
//     I001 - Login Failure
//     I002 - Payment Failure
//     I003 - App Crash
//
// Starting from Issue nodes is intentional.
//
// If we started from KnowledgeArticle nodes, we would only see issues that
// already have linked articles.
//
// But for a coverage report, we want to see all issues, including issues that
// have zero articles.
//
// That is why the query begins with:
//
//     MATCH (i:Issue)
//
// This establishes the complete list of issues first.

OPTIONAL MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)

// =============================================================================
// STEP 2: OPTIONALLY MATCH KNOWLEDGE ARTICLES THAT SOLVE EACH ISSUE
// =============================================================================
// OPTIONAL MATCH tries to find a pattern, but it does not remove the current row
// if the pattern is missing.
//
// The pattern is:
//
//     (ka:KnowledgeArticle)-[:SOLVES]->(i)
//
// In plain English:
//
//     "For the current Issue i, find KnowledgeArticle nodes that have a SOLVES
//      relationship pointing to this Issue."
//
// -----------------------------------------------------------------------------
// Why OPTIONAL MATCH is used instead of MATCH
// -----------------------------------------------------------------------------
// A normal MATCH behaves like an inner join.
//
// If we wrote:
//
//     MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)
//
// then only issues with at least one matching KnowledgeArticle would appear.
//
// Issues with no linked articles would disappear from the result.
//
// That would be bad for a coverage report because missing coverage is exactly
// what we may want to detect.
//
// OPTIONAL MATCH behaves more like a left outer join.
//
// It keeps the Issue row even when no matching KnowledgeArticle exists.
//
// If no article is found, ka becomes null for that issue.
//
// This allows the final output to show:
//
//     knowledgeArticleCount = 0
//
// for issues that have no support article coverage.
//
// -----------------------------------------------------------------------------
// Relationship direction
// -----------------------------------------------------------------------------
// The relationship pattern is:
//
//     (ka)-[:SOLVES]->(i)
//
// This direction means:
//
//     KnowledgeArticle SOLVES Issue
//
// Even though the report is issue-focused, the stored relationship still points
// from article to issue.
//
// This is common in graph queries:
//
//     We can start from one side of the graph,
//     but still match relationships using their stored direction.
//
// -----------------------------------------------------------------------------
// Why this pattern is useful
// -----------------------------------------------------------------------------
// This step connects each Issue to the articles that solve it.
//
// For example:
//
//     (:KnowledgeArticle {articleId: "K001", title: "Fix login failure"})
//         -[:SOLVES]->
//     (:Issue {issueId: "I001", name: "Login Failure"})
//
// The final report can then say:
//
//     Issue I001 / Login Failure
//     has 1 linked article:
//     K001 - Fix login failure
//
// This is very useful when checking whether the knowledge base has enough
// troubleshooting content for each issue category.

RETURN
  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS severity,
  count(ka) AS knowledgeArticleCount,
  collect(ka.articleId) AS knowledgeArticleIds,
  collect(ka.title) AS knowledgeArticleTitles

// =============================================================================
// STEP 3: RETURN ISSUE DETAILS AND ARTICLE COVERAGE SUMMARY
// =============================================================================
// RETURN defines the final output table.
//
// This query returns issue-level fields and aggregated article information.
//
// The output columns are:
//
// - issueId
// - issueName
// - severity
// - knowledgeArticleCount
// - knowledgeArticleIds
// - knowledgeArticleTitles
//
// -----------------------------------------------------------------------------
// i.issueId AS issueId
// -----------------------------------------------------------------------------
// issueId is the stable identifier for the Issue node.
//
// Example:
//
//     I001
//     I002
//     I003
//
// This helps uniquely identify each issue in the report.
//
// IDs are important because names may change over time, but stable IDs should
// normally remain consistent.
//
// -----------------------------------------------------------------------------
// i.name AS issueName
// -----------------------------------------------------------------------------
// issueName is the readable name of the issue.
//
// Example:
//
//     Login Failure
//     Payment Failure
//     App Crash
//
// This makes the report understandable to humans.
//
// Instead of only seeing I001, the user can immediately understand the business
// meaning of the issue.
//
// -----------------------------------------------------------------------------
// i.severity AS severity
// -----------------------------------------------------------------------------
// severity shows the priority or seriousness of the issue.
//
// Example values may include:
//
//     High
//     Medium
//     Low
//     Critical
//
// Including severity is useful because a high-severity issue with no linked
// knowledge article may require urgent attention.
//
// For example:
//
//     Login Failure / High severity / 0 articles
//
// is more concerning than:
//
//     Cosmetic UI Issue / Low severity / 0 articles
//
// This helps prioritize knowledge-base improvements.
//
// -----------------------------------------------------------------------------
// count(ka) AS knowledgeArticleCount
// -----------------------------------------------------------------------------
// count(ka) counts how many KnowledgeArticle nodes were matched for each Issue.
//
// Because OPTIONAL MATCH is used, this count behaves nicely:
//
// - If one or more articles solve the issue, count(ka) returns that number.
// - If no article solves the issue, ka is null and count(ka) returns 0.
//
// This makes the count ideal for coverage reporting.
//
// Example:
//
//     issueName        knowledgeArticleCount
//     --------------------------------------
//     Login Failure    1
//     Payment Failure  1
//     App Crash        1
//     Refund Delay     0
//
// A count of 0 identifies an issue with missing knowledge coverage.
//
// -----------------------------------------------------------------------------
// collect(ka.articleId) AS knowledgeArticleIds
// -----------------------------------------------------------------------------
// collect(ka.articleId) gathers the article IDs linked to each Issue into a list.
//
// Example:
//
//     ["K001"]
//
// or, if multiple articles solve the same issue:
//
//     ["K001", "K004", "K007"]
//
// This is useful because the report does not just say how many articles exist.
// It also tells us which specific articles are linked.
//
// Important note:
// If an Issue has no linked KnowledgeArticle, ka.articleId is null.
// Depending on Neo4j behavior and data shape, the collected list may appear as
// an empty list or may include null-like values.
//
// If you want a cleaner production report, you can filter nulls in a later
// refinement.
//
// For the current lab query, this version is simple and easy to understand.
//
// -----------------------------------------------------------------------------
// collect(ka.title) AS knowledgeArticleTitles
// -----------------------------------------------------------------------------
// collect(ka.title) gathers the titles of linked KnowledgeArticle nodes into a
// list.
//
// Example:
//
//     ["Fix login failure"]
//
// This makes the report readable because article IDs are precise, but titles
// explain the content in human language.
//
// If multiple articles solve one issue, this list helps quickly review whether
// the linked articles make sense for that issue.
//
// For example:
//
//     Issue: Login Failure
//     Titles: ["Fix login failure", "Troubleshoot OTP delivery"]
//
// That gives a quick coverage summary for the issue.

ORDER BY issueId;

// =============================================================================
// STEP 4: SORT THE REPORT BY ISSUE ID
// =============================================================================
// ORDER BY issueId sorts the final result by the issueId column.
//
// Sorting does not change the graph data.
// It only controls how the report is displayed.
//
// Without ORDER BY, Neo4j may return rows in an order that is not predictable.
//
// Sorting by issueId makes the output stable and easy to compare.
//
// Expected output may look conceptually like:
//
//     issueId   issueName        severity   knowledgeArticleCount   knowledgeArticleIds   knowledgeArticleTitles
//     ----------------------------------------------------------------------------------------------------------
//     I001      Login Failure    High       1                       ["K001"]              ["Fix login failure"]
//     I002      Payment Failure  Medium     1                       ["K002"]              ["Resolve payment failure"]
//     I003      App Crash        High       1                       ["K003"]              ["Fix app crash"]
//
// If an issue has no linked article, it may look like:
//
//     I004      Refund Delay     Medium     0                       []                    []
//
// This is useful because it immediately highlights coverage gaps.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is an issue coverage report.
//
// It does not create, update, or delete anything.
// It only reads the graph and summarizes how well Issues are covered by
// KnowledgeArticles.
//
// The query follows this flow:
//
//     1. MATCH all Issue nodes.
//     2. OPTIONAL MATCH KnowledgeArticles that SOLVE each Issue.
//     3. RETURN issue details, article count, article IDs, and article titles.
//     4. ORDER the output by issueId.
//
// The most important concept is:
//
//     OPTIONAL MATCH keeps issues in the result even when no article is linked.
//
// That makes this query especially valuable for finding knowledge-base gaps.
//
// In a production knowledge graph, this kind of report can help teams answer:
//
//     "Which high-severity issues do not yet have support articles?"
//
// or:
//
//     "Which issues have enough troubleshooting coverage?"
//
// Production note:
// Your pasted query contains -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the real Cypher arrow:
//
//     OPTIONAL MATCH (ka:KnowledgeArticle)-[:SOLVES]->(i)
```

# Addendum A — Step A4: Validate chunk coverage per article

```cypher
// =============================================================================
// ARTICLE-TO-CHUNK COVERAGE AND EMBEDDING SUMMARY REPORT
// =============================================================================
// This query creates a summary report showing how DocumentChunk nodes are
// distributed across KnowledgeArticle nodes.
//
// In simple terms, it answers these questions:
//
//     "For each KnowledgeArticle, how many chunks belong to it?"
//     "How many of those chunks have embeddings?"
//     "Which chunk IDs belong to each article?"
//     "What order do those chunks appear in?"
//     "What embedding dimensions are present for the chunks of each article?"
//
// This is an important validation query in a RAG-style graph workflow.
//
// Earlier, we created relationships like:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)
//
// That relationship means:
//
//     "This document chunk is part of this knowledge article."
//
// This query uses that relationship to group chunks by their parent article and
// produce one summary row per article.
//
// This is useful for:
// - Confirming that each article has the expected number of chunks
// - Checking whether all chunks have embeddings
// - Reviewing chunk IDs for each article
// - Validating chunk order
// - Confirming embedding dimensions are consistent
// - Preparing the graph for vector search and RAG retrieval
//
// A healthy result for this lab may look conceptually like:
//
//     K001 -> 2 chunks -> 2 chunks with embeddings -> dimensions [3]
//     K002 -> 2 chunks -> 2 chunks with embeddings -> dimensions [3]
//     K003 -> 2 chunks -> 2 chunks with embeddings -> dimensions [3]

MATCH (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)

// =============================================================================
// MATCH DOCUMENT CHUNKS CONNECTED TO THEIR PARENT KNOWLEDGE ARTICLES
// =============================================================================
// MATCH tells Neo4j what graph pattern we want to find.
//
// The pattern is:
//
//     (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)
//
// In plain English, this means:
//
//     "Find DocumentChunk nodes that are connected to KnowledgeArticle nodes
//      using a PART_OF relationship."
//
// -----------------------------------------------------------------------------
// (dc:DocumentChunk)
// -----------------------------------------------------------------------------
// This is the starting node in the pattern.
//
// - Parentheses () represent a node.
// - dc is the variable name assigned to each matched DocumentChunk.
// - DocumentChunk is the node label.
//
// DocumentChunk nodes represent smaller pieces of article text.
//
// In a RAG-style system, chunks are usually the units that get embedded,
// indexed, searched, and retrieved.
//
// Example chunk IDs may be:
//
//     C-K001-001
//     C-K001-002
//     C-K002-001
//
// -----------------------------------------------------------------------------
// -[:PART_OF]->
// -----------------------------------------------------------------------------
// This is the relationship pattern.
//
// - Square brackets [] represent a relationship.
// - PART_OF is the relationship type.
// - The arrow -> shows the relationship direction.
//
// The direction reads naturally as:
//
//     DocumentChunk PART_OF KnowledgeArticle
//
// or:
//
//     "This chunk is part of this article."
//
// This relationship is important because it gives each chunk lineage.
// If vector search retrieves a chunk, the graph can trace that chunk back to
// the original article.
//
// -----------------------------------------------------------------------------
// (ka:KnowledgeArticle)
// -----------------------------------------------------------------------------
// This is the ending node in the pattern.
//
// - ka is the variable name assigned to the matched KnowledgeArticle.
// - KnowledgeArticle is the node label.
//
// KnowledgeArticle nodes represent the parent support articles or knowledge-base
// entries.
//
// Examples may include:
//
//     K001 - Fix login failure
//     K002 - Resolve payment failure
//     K003 - Fix app crash
//
// -----------------------------------------------------------------------------
// Why this query starts from the relationship
// -----------------------------------------------------------------------------
// This query only counts chunks that are actually connected to articles through
// PART_OF.
//
// That means if a DocumentChunk exists but does not have a PART_OF relationship,
// it will not be included in this report.
//
// This is intentional for this report because we are summarizing article-to-chunk
// coverage based on actual graph connections.
//
// If we wanted to find orphan chunks, we would use a separate data-quality query
// with WHERE NOT EXISTS.

RETURN
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  count(dc) AS chunkCount,
  count(dc.embedding) AS chunksWithEmbedding,
  collect(dc.chunkId) AS chunkIds,
  collect(dc.chunkOrder) AS chunkOrders,
  collect(DISTINCT size(dc.embedding)) AS embeddingDimensions

// =============================================================================
// RETURN ARTICLE-LEVEL CHUNK AND EMBEDDING SUMMARY
// =============================================================================
// RETURN defines the final output table.
//
// Because this query returns article properties along with aggregate functions
// like count() and collect(), Neo4j groups the results by the non-aggregated
// article fields:
//
//     ka.articleId
//     ka.title
//
// So the output produces one row per KnowledgeArticle.
//
// Each row summarizes the chunks connected to that article.
//
// -----------------------------------------------------------------------------
// ka.articleId AS articleId
// -----------------------------------------------------------------------------
// articleId is the stable identifier of the KnowledgeArticle.
//
// Example:
//
//     K001
//     K002
//     K003
//
// This tells us which article each summary row belongs to.
//
// The alias articleId makes the output clean and easy to read.
//
// -----------------------------------------------------------------------------
// ka.title AS articleTitle
// -----------------------------------------------------------------------------
// articleTitle is the human-readable title of the KnowledgeArticle.
//
// Example:
//
//     Fix login failure
//     Resolve payment failure
//     Fix app crash
//
// This makes the report easier for humans to understand.
//
// Instead of only seeing an article ID, we can immediately see what topic the
// article covers.
//
// -----------------------------------------------------------------------------
// count(dc) AS chunkCount
// -----------------------------------------------------------------------------
// count(dc) counts how many DocumentChunk nodes are connected to the current
// KnowledgeArticle through the PART_OF relationship.
//
// In simple terms:
//
//     "How many chunks belong to this article?"
//
// For this lab, each article may have two chunks, so we may expect:
//
//     K001 -> chunkCount = 2
//     K002 -> chunkCount = 2
//     K003 -> chunkCount = 2
//
// This validates whether the article was split into the expected number of
// chunks.
//
// If chunkCount is lower than expected, possible reasons include:
// - Some chunks were not created.
// - Some chunks were not connected using PART_OF.
// - The query is running against a different database.
// - The relationship direction is different from expected.
//
// -----------------------------------------------------------------------------
// count(dc.embedding) AS chunksWithEmbedding
// -----------------------------------------------------------------------------
// count(dc.embedding) counts how many connected chunks have a non-null embedding
// property.
//
// This is different from count(dc):
//
// - count(dc) counts all chunks connected to the article.
// - count(dc.embedding) counts only chunks that actually have embeddings.
//
// This is important for vector search readiness.
//
// Ideally, for each article:
//
//     chunksWithEmbedding = chunkCount
//
// That means every chunk belonging to the article has an embedding and can
// participate in vector similarity search.
//
// If chunksWithEmbedding is lower than chunkCount, then some chunks exist but
// are missing embeddings.
//
// That would be a problem for RAG because those chunks may not be retrievable
// through vector search.
//
// -----------------------------------------------------------------------------
// collect(dc.chunkId) AS chunkIds
// -----------------------------------------------------------------------------
// collect(dc.chunkId) gathers all chunk IDs connected to the article into a list.
//
// Example:
//
//     ["C-K001-001", "C-K001-002"]
//
// This is useful because the report does not only show the number of chunks.
// It also shows exactly which chunks belong to each article.
//
// This helps with:
// - debugging
// - validation
// - lab screenshots
// - checking chunk lineage
// - confirming that the expected chunk IDs were created
//
// -----------------------------------------------------------------------------
// collect(dc.chunkOrder) AS chunkOrders
// -----------------------------------------------------------------------------
// collect(dc.chunkOrder) gathers the chunk order values into a list.
//
// Example:
//
//     [1, 2]
//
// chunkOrder tells us the sequence of chunks inside the parent article.
//
// This is important because when an article is split into multiple chunks, the
// order helps reconstruct the original reading sequence.
//
// For example:
//
//     C-K001-001 -> chunkOrder 1
//     C-K001-002 -> chunkOrder 2
//
// If the collected chunk orders look unusual, such as:
//
//     [1, 3]
//
// then chunk 2 may be missing.
//
// If the list contains duplicate values, such as:
//
//     [1, 1]
//
// then the chunking process may have assigned incorrect ordering.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT size(dc.embedding)) AS embeddingDimensions
// -----------------------------------------------------------------------------
// size(dc.embedding) returns the number of values inside each chunk's embedding
// vector.
//
// For example:
//
//     size([0.95, 0.10, 0.05])
//
// returns:
//
//     3
//
// collect(DISTINCT ...) gathers only the unique embedding dimensions found among
// the chunks for each article.
//
// For this lab, the expected result is:
//
//     [3]
//
// This means all embedded chunks for that article have 3-dimensional vectors.
//
// -----------------------------------------------------------------------------
// Why DISTINCT is important here
// -----------------------------------------------------------------------------
// Without DISTINCT, if an article has two chunks with 3-dimensional embeddings,
// the result would be:
//
//     [3, 3]
//
// That is technically correct, but noisy.
//
// With DISTINCT, Neo4j returns:
//
//     [3]
//
// This is cleaner and easier to validate.
//
// -----------------------------------------------------------------------------
// What if multiple dimensions appear?
// -----------------------------------------------------------------------------
// If the result shows:
//
//     [3, 4]
//
// that means chunks under the same article have inconsistent embedding
// dimensions.
//
// That is usually a serious issue because vector indexes expect consistent
// dimensions for the indexed vector property.
//
// In production, this could mean:
// - embeddings were generated by different models
// - old and new embeddings were mixed
// - some embedding update logic failed
// - bad data was loaded accidentally
//
// -----------------------------------------------------------------------------
// What if null appears?
// -----------------------------------------------------------------------------
// If some chunks are missing embeddings, size(dc.embedding) may produce null.
//
// In that case, embeddingDimensions may show a null-like value depending on the
// data and Neo4j version/client behavior.
//
// That should be reviewed together with:
//
//     chunkCount
//     chunksWithEmbedding
//
// If chunksWithEmbedding is lower than chunkCount, at least one chunk under that
// article is missing an embedding.

ORDER BY articleId;

// =============================================================================
// SORT THE REPORT BY ARTICLE ID
// =============================================================================
// ORDER BY articleId sorts the final report by the articleId column.
//
// Sorting does not change the graph data.
// It only controls how the output is displayed.
//
// Without ORDER BY, Neo4j may return rows in an order that is not predictable.
//
// Sorting by articleId makes the report easier to read, compare, and document.
//
// Expected output may look conceptually like:
//
//     articleId   articleTitle             chunkCount   chunksWithEmbedding   chunkIds                      chunkOrders   embeddingDimensions
//     -------------------------------------------------------------------------------------------------------------------------------------
//     K001        Fix login failure        2            2                     ["C-K001-001","C-K001-002"]   [1,2]         [3]
//     K002        Resolve payment failure  2            2                     ["C-K002-001","C-K002-002"]   [1,2]         [3]
//     K003        Fix app crash            2            2                     ["C-K003-001","C-K003-002"]   [1,2]         [3]
//
// This stable ordering is useful for:
// - lab guide screenshots
// - comparing expected vs actual output
// - validating chunk coverage per article
// - confirming embedding readiness per article
// - preparing for vector search demos
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is an article-level chunk and embedding coverage report.
//
// It does not create, update, or delete anything.
// It only reads the graph and summarizes each article's connected chunks.
//
// The query validates:
//
//     1. Which chunks belong to each article.
//     2. How many chunks each article has.
//     3. How many chunks have embeddings.
//     4. Whether embedding dimensions are consistent.
//     5. Whether chunk ordering looks complete and sensible.
//
// The most important health checks are:
//
//     chunkCount = expected number of chunks per article
//     chunksWithEmbedding = chunkCount
//     embeddingDimensions = [3] for this lab
//
// If those checks pass, the article-to-chunk layer is ready for vector-search
// and RAG-style retrieval.
//
// Production note:
// Your pasted query contains -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the real Cypher arrow:
//
//     MATCH (dc:DocumentChunk)-[:PART_OF]->(ka:KnowledgeArticle)
```

# Addendum A — Step A5: Validate vector index readiness

```cypher
// =============================================================================
// DETAILED VECTOR INDEX INSPECTION QUERY
// =============================================================================
// This query inspects one specific Neo4j vector index in detail:
//
//     documentChunk_embedding_vector
//
// In simple terms, it answers this question:
//
//     "Does my DocumentChunk embedding vector index exist,
//      is it online,
//      is it fully populated,
//      what label/property does it index,
//      which provider backs it,
//      and what configuration options were used?"
//
// This is a more detailed version of the earlier vector-index readiness checks.
//
// Earlier queries may have checked only:
//
// - state
// - populationPercent
// - labelsOrTypes
// - properties
//
// This query also includes:
//
// - type
// - entityType
// - indexProvider
// - options
//
// That makes it useful when we want to verify not only that the index is ready,
// but also that its internal configuration matches the intended design.
//
// In this lab, the expected vector index should be:
//
//     Name:       documentChunk_embedding_vector
//     Target:     (:DocumentChunk).embedding
//     Dimension:  3
//     Similarity: cosine
//
// This query is useful for:
// - verifying vector index readiness
// - checking whether the index is ONLINE
// - checking whether population is 100% complete
// - confirming the index targets DocumentChunk nodes
// - confirming the indexed property is embedding
// - inspecting index provider details
// - validating vector index configuration options
// - documenting the schema state in a lab guide or demo

SHOW VECTOR INDEXES

// =============================================================================
// SHOW ALL VECTOR INDEXES
// =============================================================================
// SHOW VECTOR INDEXES asks Neo4j to list vector indexes currently defined in
// the active database.
//
// A vector index is a specialized index used for similarity search over numeric
// embedding vectors.
//
// In this project, DocumentChunk nodes store text and embeddings:
//
//     dc.text       -> human-readable chunk text
//     dc.embedding  -> numeric vector representation of that text
//
// The vector index is built on:
//
//     (:DocumentChunk).embedding
//
// This allows Neo4j to efficiently answer questions like:
//
//     "Which document chunks are closest to this query embedding?"
//
// This is the foundation for semantic search and RAG-style retrieval.
//
// Important:
// This command is read-only.
// It does not create, update, delete, or rebuild the index.
// It only displays metadata about existing vector indexes.

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

// =============================================================================
// SELECT DETAILED VECTOR INDEX METADATA FIELDS
// =============================================================================
// YIELD chooses which columns from SHOW VECTOR INDEXES we want to inspect.
//
// Here we select detailed metadata fields so that we can fully verify the vector
// index definition and readiness.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// name is the vector index name.
//
// In this query, we are interested in:
//
//     documentChunk_embedding_vector
//
// This name is important because vector-search procedures call the index by name.
//
// For example:
//
//     CALL db.index.vector.queryNodes(
//       'documentChunk_embedding_vector',
//       3,
//       [0.92, 0.12, 0.05]
//     )
//
// If the name is wrong or misspelled, the vector search query will not use the
// intended index.
//
// -----------------------------------------------------------------------------
// state
// -----------------------------------------------------------------------------
// state tells us the operational status of the vector index.
//
// The ideal value is:
//
//     ONLINE
//
// ONLINE means Neo4j considers the index ready for use.
//
// If the state is not ONLINE, the index may still be building, may have failed,
// or may not be available for vector search yet.
//
// This is one of the most important readiness checks.
//
// -----------------------------------------------------------------------------
// populationPercent
// -----------------------------------------------------------------------------
// populationPercent tells us how much of the index population process has
// completed.
//
// The ideal value is:
//
//     100.0
//
// This means Neo4j has finished indexing the existing embedding values.
//
// If the value is less than 100.0, the index may still be populating.
//
// In a production workflow, we should normally wait until:
//
//     state = ONLINE
//     populationPercent = 100.0
//
// before relying on vector-search results.
//
// -----------------------------------------------------------------------------
// type
// -----------------------------------------------------------------------------
// type tells us what kind of index this is.
//
// Since this query uses SHOW VECTOR INDEXES, the type should indicate that this
// is a vector index.
//
// This helps distinguish the index from other Neo4j index types such as:
//
// - range indexes
// - text indexes
// - full-text indexes
// - lookup indexes
//
// A vector index is specifically designed for nearest-neighbor similarity search
// over numeric vectors.
//
// -----------------------------------------------------------------------------
// entityType
// -----------------------------------------------------------------------------
// entityType tells us whether the index applies to nodes or relationships.
//
// In this lab, embeddings are stored on DocumentChunk nodes, so we expect the
// entity type to indicate a node-based index.
//
// Conceptually, the indexed entity is:
//
//     (:DocumentChunk)
//
// not a relationship.
//
// This confirms that the vector index is built on node properties.
//
// -----------------------------------------------------------------------------
// labelsOrTypes
// -----------------------------------------------------------------------------
// labelsOrTypes tells us which node label or relationship type the index applies
// to.
//
// For this lab, we expect:
//
//     ["DocumentChunk"]
//
// This confirms that the vector index is scoped to DocumentChunk nodes.
//
// This matters because the database may contain many labels, such as:
//
// - KnowledgeArticle
// - Issue
// - DocumentChunk
// - Customer
// - Ticket
//
// But this particular vector index should target the chunk-level retrieval unit:
//
//     DocumentChunk
//
// -----------------------------------------------------------------------------
// properties
// -----------------------------------------------------------------------------
// properties tells us which property is indexed.
//
// For this lab, we expect:
//
//     ["embedding"]
//
// This confirms that Neo4j is indexing the numeric vector property.
//
// This is important because vector search does not compare the raw text property.
//
// The readable text is stored in:
//
//     dc.text
//
// But semantic search compares:
//
//     dc.embedding
//
// If the index were built on the wrong property, vector search would not work as
// intended.
//
// -----------------------------------------------------------------------------
// indexProvider
// -----------------------------------------------------------------------------
// indexProvider tells us which internal Neo4j index provider backs this vector
// index.
//
// Think of the provider as the underlying implementation Neo4j uses to store
// and search the index data.
//
// For most lab work, we do not need to manually change the provider.
//
// But returning it is useful for:
//
// - troubleshooting
// - environment comparison
// - database administration
// - confirming which index implementation Neo4j is using
//
// In production environments, index provider details can be useful when comparing
// behavior across Neo4j versions or deployments.
//
// -----------------------------------------------------------------------------
// options
// -----------------------------------------------------------------------------
// options shows the configuration options used for the vector index.
//
// This is especially important for vector indexes because configuration controls
// how embeddings are interpreted and compared.
//
// For this lab, the options should show configuration equivalent to:
//
//     vector.dimensions = 3
//     vector.similarity_function = cosine
//
// -----------------------------------------------------------------------------
// Why vector dimensions matter
// -----------------------------------------------------------------------------
// The index dimension must match the size of the stored embedding vectors.
//
// Earlier, our DocumentChunk embeddings looked like:
//
//     [0.95, 0.10, 0.05]
//
// This vector has 3 values.
//
// So the vector index must also be configured with:
//
//     vector.dimensions = 3
//
// If the index expects 3 dimensions but a query vector has 2 or 4 dimensions,
// vector search will not be compatible.
//
// -----------------------------------------------------------------------------
// Why similarity function matters
// -----------------------------------------------------------------------------
// The similarity function controls how Neo4j compares vectors.
//
// In this lab, we use:
//
//     cosine
//
// Cosine similarity compares vectors based on direction.
//
// This is commonly useful for text embeddings because semantic similarity is
// often represented by vectors pointing in similar directions.
//
// For example, a login-related query vector:
//
//     [0.92, 0.12, 0.05]
//
// should be close to login-related chunk vectors:
//
//     [0.95, 0.10, 0.05]
//     [0.90, 0.15, 0.05]
//
// That is why checking the options field is important.
// It confirms the index is configured the way our vector-search demo expects.

WHERE name = "documentChunk_embedding_vector"

// =============================================================================
// FILTER TO THE SPECIFIC VECTOR INDEX
// =============================================================================
// WHERE filters the vector index metadata so that we only inspect the index
// named:
//
//     documentChunk_embedding_vector
//
// Without this WHERE clause, Neo4j would return all vector indexes in the
// database.
//
// That may be useful for a full inventory, but here we want a targeted
// verification query.
//
// If the index exists:
// - Neo4j returns one row with its metadata.
//
// If the index does not exist:
// - Neo4j returns no rows.
//
// A no-row result is important because it means Neo4j did not find an index with
// this exact name.
//
// Possible reasons include:
//
// - The vector index was not created.
// - The index was created with a different name.
// - The query is running against the wrong database.
// - The connected user does not have permission to view index metadata.
// - The index creation command failed earlier.
//
// Important:
// The name comparison is exact.
//
// These names would not match:
//
//     documentchunk_embedding_vector
//     documentChunk_embedding_vector
//     documentChunk_embeddings_vector
//     documentChunk_embedding_index
//
// Exact naming matters because vector-search procedures also reference the index
// by exact name.

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

// =============================================================================
// RETURN DETAILED VECTOR INDEX INFORMATION
// =============================================================================
// RETURN defines the final output table shown by Neo4j.
//
// This query returns:
//
// - name
// - state
// - populationPercent
// - type
// - entityType
// - labelsOrTypes
// - properties
// - indexProvider
// - options
//
// Together, these fields provide a complete readiness and configuration view of
// the vector index.
//
// -----------------------------------------------------------------------------
// name
// -----------------------------------------------------------------------------
// Confirms which vector index was found.
//
// Expected value:
//
//     documentChunk_embedding_vector
//
// -----------------------------------------------------------------------------
// state
// -----------------------------------------------------------------------------
// Confirms whether the index is operationally ready.
//
// Expected value:
//
//     ONLINE
//
// If this is not ONLINE, wait or troubleshoot before running vector search.
//
// -----------------------------------------------------------------------------
// populationPercent
// -----------------------------------------------------------------------------
// Confirms whether index population has completed.
//
// Expected value:
//
//     100.0
//
// This means existing embedding values have been indexed.
//
// -----------------------------------------------------------------------------
// type
// -----------------------------------------------------------------------------
// Confirms the index type.
//
// Expected value should indicate a vector index.
//
// -----------------------------------------------------------------------------
// entityType
// -----------------------------------------------------------------------------
// Confirms whether the index is built for nodes or relationships.
//
// For this lab, we expect a node-based index because embeddings are stored on
// DocumentChunk nodes.
//
// -----------------------------------------------------------------------------
// labelsOrTypes
// -----------------------------------------------------------------------------
// Confirms the label or type targeted by the index.
//
// Expected value should include:
//
//     DocumentChunk
//
// -----------------------------------------------------------------------------
// properties
// -----------------------------------------------------------------------------
// Confirms the property indexed by the vector index.
//
// Expected value should include:
//
//     embedding
//
// -----------------------------------------------------------------------------
// indexProvider
// -----------------------------------------------------------------------------
// Shows the Neo4j index provider backing the vector index.
//
// This is mainly useful for troubleshooting, administration, and environment
// comparison.
//
// -----------------------------------------------------------------------------
// options
// -----------------------------------------------------------------------------
// Shows the vector index configuration.
//
// For this lab, we expect options equivalent to:
//
//     vector.dimensions = 3
//     vector.similarity_function = cosine
//
// This confirms that the vector index matches the sample embeddings and the
// intended similarity behavior.
//
// =============================================================================
// EXPECTED HEALTHY RESULT FOR THIS LAB
// =============================================================================
// A healthy output should conceptually look like:
//
//     name                            = documentChunk_embedding_vector
//     state                           = ONLINE
//     populationPercent               = 100.0
//     type                            = VECTOR
//     entityType                      = NODE
//     labelsOrTypes                   = ["DocumentChunk"]
//     properties                      = ["embedding"]
//     indexProvider                   = <Neo4j vector index provider>
//     options                         = { vector.dimensions: 3,
//                                         vector.similarity_function: "cosine" }
//
// The exact formatting of type, provider, and options may vary depending on the
// Neo4j version and client display format.
//
// The most important checks are:
//
//     state = ONLINE
//     populationPercent = 100.0
//     labelsOrTypes includes DocumentChunk
//     properties includes embedding
//     options show 3 dimensions
//     options show cosine similarity
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a detailed vector index inspection query.
//
// It does not run vector similarity search.
// It does not inspect individual DocumentChunk embeddings.
// It does not create or modify the index.
//
// It only verifies:
//
//     "Is the vector index present, ready, and configured correctly?"
//
// This is a good production-readiness habit because vector search depends on
// both data correctness and index correctness.
//
// In the full RAG-style workflow, this query validates the vector index layer:
//
//     1. KnowledgeArticle nodes were loaded.
//     2. DocumentChunk nodes were loaded.
//     3. PART_OF relationships connected chunks to articles.
//     4. SOLVES relationships connected articles to issues.
//     5. Embeddings were added to chunks.
//     6. A vector index was created on DocumentChunk.embedding.
//     7. This query verifies that the vector index is ready and correctly configured.
//
// Once this query confirms the expected values, the graph is ready for
// similarity search using:
//
//     CALL db.index.vector.queryNodes(
//       'documentChunk_embedding_vector',
//       3,
//       [0.92, 0.12, 0.05]
//     )
```

# Addendum A — Step A6: Controlled vector retrieval quality test

```cypher
// =============================================================================
// RUN LOGIN-STYLE VECTOR SEARCH AND EXPAND RESULTS INTO ARTICLE + ISSUE CONTEXT
// =============================================================================
// This query performs a vector similarity search using a login-oriented test
// vector, then expands each retrieved DocumentChunk into its parent
// KnowledgeArticle and the Issue solved by that article.
//
// In simple terms, it answers this question:
//
//     "When I search with a login-style query vector,
//      which chunks are most similar,
//      which articles do those chunks belong to,
//      and which issues do those articles solve?"
//
// This is a strong end-to-end RAG validation query because it tests three layers
// together:
//
// 1. Vector search layer:
//    The query searches the vector index using a test embedding.
//
// 2. Chunk lineage layer:
//    Each retrieved chunk is traced back to its parent KnowledgeArticle through
//    the PART_OF relationship.
//
// 3. Business context layer:
//    Each parent KnowledgeArticle is traced to the Issue it solves through the
//    SOLVES relationship.
//
// The graph path used after retrieval is:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// This means the query does not only return the nearest chunks.
// It also returns the larger article and issue context behind those chunks.
//
// The test vector:
//
//     [0.92, 0.12, 0.05]
//
// is intentionally close to the login-related sample embeddings:
//
//     C-K001-001 -> [0.95, 0.10, 0.05]
//     C-K001-002 -> [0.90, 0.15, 0.05]
//
// So the expected top results should be login-related chunks from:
//
//     K001 - Fix login failure
//
// and the connected issue should be:
//
//     Login Failure
//
// This query is useful for:
// - validating vector search behavior
// - confirming the vector index returns expected chunks
// - checking chunk-to-article traceability
// - checking article-to-issue context
// - demonstrating graph-enhanced semantic search
// - preparing for a RAG-style retrieval workflow

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  3,
  [0.92, 0.12, 0.05]
)

// =============================================================================
// STEP 1: SEARCH THE VECTOR INDEX
// =============================================================================
// CALL executes a Neo4j procedure.
//
// Here we call:
//
//     db.index.vector.queryNodes()
//
// This procedure searches a vector index and returns nodes whose stored embedding
// vectors are most similar to the query vector we provide.
//
// In this lab, the vector index:
//
//     documentChunk_embedding_vector
//
// was created on:
//
//     (:DocumentChunk).embedding
//
// So the procedure searches DocumentChunk nodes based on their embedding values.
//
// -----------------------------------------------------------------------------
// Parameter 1: 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// The first parameter is the vector index name.
//
// This tells Neo4j:
//
//     "Search inside the vector index named documentChunk_embedding_vector."
//
// The index name must match exactly.
//
// If the name is misspelled, Neo4j cannot use the intended index.
//
// -----------------------------------------------------------------------------
// Parameter 2: 3
// -----------------------------------------------------------------------------
// The second parameter is the number of nearest results to return.
//
// Here we pass:
//
//     3
//
// This means:
//
//     "Return the top 3 most similar DocumentChunk nodes."
//
// This is commonly called top-k retrieval.
//
// In a RAG system, top-k controls how many chunks are retrieved as candidate
// context before an LLM generates an answer.
//
// A smaller top-k gives more focused context.
// A larger top-k gives broader context, but may include weaker matches.
//
// -----------------------------------------------------------------------------
// Parameter 3: [0.92, 0.12, 0.05]
// -----------------------------------------------------------------------------
// The third parameter is the query vector.
//
// In a real application, this vector would normally be generated from a user's
// question by an embedding model.
//
// For example, a user question like:
//
//     "Customer cannot sign in and OTP is not arriving"
//
// would be converted into an embedding vector.
//
// In this lab, we manually provide a simple 3-dimensional test vector:
//
//     [0.92, 0.12, 0.05]
//
// This vector is login-oriented because its first dimension is very high.
//
// Earlier, the login-related chunks were assigned similar vectors:
//
//     C-K001-001 -> [0.95, 0.10, 0.05]
//     C-K001-002 -> [0.90, 0.15, 0.05]
//
// Because these vectors are close to the query vector, those login chunks should
// appear near the top of the result set.
//
// -----------------------------------------------------------------------------
// Why the query vector has exactly 3 numbers
// -----------------------------------------------------------------------------
// The vector index was created with:
//
//     `vector.dimensions`: 3
//
// That means the query vector must also contain exactly 3 numeric values.
//
// This query vector is valid because it contains:
//
//     0.92
//     0.12
//     0.05
//
// If the query vector had a different number of values, it would not match the
// configured vector dimension of the index.

YIELD node AS dc, score

// =============================================================================
// STEP 2: RECEIVE VECTOR SEARCH RESULTS
// =============================================================================
// YIELD defines which outputs we want from the vector-search procedure.
//
// The procedure returns:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the matched node returned by the vector index.
//
// Because this index was created on DocumentChunk.embedding, each returned node
// should be a DocumentChunk.
//
// We rename node to dc:
//
//     node AS dc
//
// This makes the rest of the query easier to understand.
//
// dc clearly means:
//
//     DocumentChunk
//
// After this alias, we can access chunk properties such as:
//
//     dc.chunkId
//     dc.text
//     dc.embedding
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the similarity score between the query vector and the stored chunk
// embedding.
//
// Because the vector index was configured with cosine similarity, a higher score
// means the returned chunk is more similar to the query vector.
//
// In simple terms:
//
//     Higher score = stronger semantic match
//
// The score helps us rank the retrieved chunks and explain why a chunk appeared
// in the result.

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// STEP 3: EXPAND EACH RETRIEVED CHUNK INTO GRAPH CONTEXT
// =============================================================================
// After vector search finds the most similar chunks, this MATCH clause follows
// graph relationships from each retrieved DocumentChunk.
//
// The pattern is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// In plain English, this means:
//
//     "For each retrieved chunk,
//      find the KnowledgeArticle it belongs to,
//      then find the Issue that article solves."
//
// This is the graph-enhancement part of the query.
//
// Vector search tells us:
//
//     "This chunk is semantically similar."
//
// Graph traversal tells us:
//
//     "This chunk belongs to this article,
//      and this article solves this issue."
//
// Together, they create an explainable retrieval result.
//
// -----------------------------------------------------------------------------
// (dc)-[:PART_OF]->(ka:KnowledgeArticle)
// -----------------------------------------------------------------------------
// This part traces the retrieved chunk to its parent KnowledgeArticle.
//
// The PART_OF relationship means:
//
//     "This DocumentChunk is part of this KnowledgeArticle."
//
// This is important because the retrieved chunk is only a small piece of text.
// The article gives us the larger source context.
//
// -----------------------------------------------------------------------------
// (ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
// -----------------------------------------------------------------------------
// This part traces the parent article to the Issue it solves.
//
// The SOLVES relationship means:
//
//     "This KnowledgeArticle solves this Issue."
//
// This gives business meaning to the retrieved chunk.
//
// For a login-style vector, we expect the retrieved chunks to connect to:
//
//     K001 - Fix login failure
//
// and then to an Issue such as:
//
//     Login Failure
//
// -----------------------------------------------------------------------------
// Why MATCH is used here
// -----------------------------------------------------------------------------
// MATCH requires the full graph pattern to exist.
//
// That means the result row is kept only when:
//
// - the retrieved DocumentChunk has a PART_OF relationship to a KnowledgeArticle
// - that KnowledgeArticle has a SOLVES relationship to an Issue
//
// This is useful for strict validation.
//
// If a retrieved chunk is missing article or issue context, it will not appear
// in the final result.
//
// In production, if you wanted to keep chunks even when context is missing, you
// could use OPTIONAL MATCH instead. But for this lab validation, MATCH is better
// because it confirms the complete graph path exists.

RETURN
  "Login-style test vector" AS testName,
  dc.chunkId AS chunkId,
  dc.text AS chunkText,
  score,
  ka.articleId AS articleId,
  ka.title AS articleTitle,
  i.issueId AS issueId,
  i.name AS issueName

// =============================================================================
// STEP 4: RETURN LABELED, ENRICHED SEARCH RESULTS
// =============================================================================
// RETURN defines the final output table.
//
// This query returns:
//
// - a fixed test name
// - the retrieved chunk details
// - the similarity score
// - the parent article details
// - the connected issue details
//
// -----------------------------------------------------------------------------
// "Login-style test vector" AS testName
// -----------------------------------------------------------------------------
// This hardcoded value labels the test case.
//
// It tells anyone reading the output:
//
//     "These results came from the login-style test vector."
//
// This is useful when running multiple test vectors, such as:
//
// - Login-style test vector
// - Payment-style test vector
// - App-crash-style test vector
//
// The testName column makes screenshots and validation outputs easier to
// understand.
//
// -----------------------------------------------------------------------------
// dc.chunkId AS chunkId
// -----------------------------------------------------------------------------
// chunkId identifies the retrieved DocumentChunk.
//
// Example:
//
//     C-K001-001
//
// This helps us trace exactly which chunk was returned by vector search.
//
// -----------------------------------------------------------------------------
// dc.text AS chunkText
// -----------------------------------------------------------------------------
// chunkText is the readable text stored in the retrieved chunk.
//
// This is the content that would normally be passed to an LLM as grounding
// context in a RAG workflow.
//
// For a login-style query vector, we expect the top chunk text to mention things
// like:
//
// - sign in
// - reset password
// - OTP delivery
// - login failure
// - app cache
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score shows how similar the returned chunk embedding is to the query vector.
//
// Higher scores should appear first after ORDER BY score DESC.
//
// This score helps explain the ranking of results.
//
// -----------------------------------------------------------------------------
// ka.articleId AS articleId
// -----------------------------------------------------------------------------
// articleId identifies the parent KnowledgeArticle.
//
// Example:
//
//     K001
//
// This tells us which article the retrieved chunk belongs to.
//
// -----------------------------------------------------------------------------
// ka.title AS articleTitle
// -----------------------------------------------------------------------------
// articleTitle gives the readable title of the parent article.
//
// Example:
//
//     Fix login failure
//
// This makes the result easier for humans to understand.
//
// -----------------------------------------------------------------------------
// i.issueId AS issueId
// -----------------------------------------------------------------------------
// issueId identifies the Issue node connected to the parent article.
//
// Example:
//
//     I001
//
// This provides stable issue traceability.
//
// -----------------------------------------------------------------------------
// i.name AS issueName
// -----------------------------------------------------------------------------
// issueName gives the readable issue name.
//
// Example:
//
//     Login Failure
//
// For this login-style test vector, this is the key business-context validation.
//
// If the top results return issueName = "Login Failure", then the vector search
// and graph traversal are aligned with the expected topic.

ORDER BY score DESC;

// =============================================================================
// STEP 5: SORT BY BEST MATCH FIRST
// =============================================================================
// ORDER BY score DESC sorts the final output from highest similarity score to
// lowest similarity score.
//
// DESC means descending order.
//
// This ensures the best vector matches appear at the top.
//
// Expected behavior:
//
// Since the query vector is:
//
//     [0.92, 0.12, 0.05]
//
// and the login chunks are:
//
//     C-K001-001 -> [0.95, 0.10, 0.05]
//     C-K001-002 -> [0.90, 0.15, 0.05]
//
// we expect those chunks to rank highly.
//
// Conceptual expected output:
//
//     testName                  chunkId       articleId   articleTitle       issueName
//     --------------------------------------------------------------------------------
//     Login-style test vector   C-K001-001    K001        Fix login failure  Login Failure
//     Login-style test vector   C-K001-002    K001        Fix login failure  Login Failure
//     ...
//
// The third result may be less relevant because the query asks for top 3, and
// after the two strongest login chunks, Neo4j still returns the next closest
// available chunk.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query validates graph-enhanced vector retrieval.
//
// The complete flow is:
//
//     1. Search DocumentChunk embeddings using a login-style vector.
//     2. Retrieve the top 3 most similar chunks.
//     3. Capture each chunk and its similarity score.
//     4. Traverse from chunk to parent KnowledgeArticle using PART_OF.
//     5. Traverse from article to Issue using SOLVES.
//     6. Return chunk, article, and issue context.
//     7. Sort results by highest similarity score.
//
// The key learning point is:
//
//     Vector search finds relevant text.
//     Graph traversal explains that text.
//
// In this lab, a healthy result should show that a login-style query vector
// retrieves login-related chunks and connects them to:
//
//     K001 - Fix login failure
//     Login Failure
//
// This proves that the graph is not only searchable, but also explainable and
// ready for RAG-style retrieval demonstrations.
//
// Production note:
// Your pasted query contains -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the real Cypher arrow:
//
//     MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

```

# Addendum A — Step A7: Controlled retrieval tests for Payment Failure and App Crash

```cypher
// =============================================================================
// RUN MULTIPLE VECTOR SEARCH TEST CASES AND COMPARE RETRIEVAL RESULTS
// =============================================================================
// This query runs two vector similarity search test cases in one combined query:
//
//     1. Payment-style test vector
//     2. App-crash-style test vector
//
// In simple terms, it answers this question:
//
//     "If I search the same DocumentChunk vector index using different test
//      vectors, do I retrieve the expected topic-specific chunks and their
//      related article/issue context?"
//
// This is a very useful RAG validation query because it tests whether the vector
// search behaves differently for different semantic intents.
//
// Earlier, we created demo embeddings like this:
//
//     Login-related chunks:
//       C-K001-001 -> [0.95, 0.10, 0.05]
//       C-K001-002 -> [0.90, 0.15, 0.05]
//
//     Payment-related chunks:
//       C-K002-001 -> [0.05, 0.95, 0.10]
//       C-K002-002 -> [0.10, 0.90, 0.15]
//
//     App-crash-related chunks:
//       C-K003-001 -> [0.10, 0.10, 0.95]
//       C-K003-002 -> [0.15, 0.10, 0.90]
//
// This query uses two test vectors:
//
//     Payment-style vector:
//       [0.08, 0.92, 0.12]
//
//     App-crash-style vector:
//       [0.12, 0.10, 0.92]
//
// The payment-style vector has a high second dimension, so it should retrieve
// payment-related chunks near the top.
//
// The app-crash-style vector has a high third dimension, so it should retrieve
// app-crash-related chunks near the top.
//
// After each vector search, the query expands the retrieved chunk into graph
// context using:
//
//     (:DocumentChunk)-[:PART_OF]->(:KnowledgeArticle)-[:SOLVES]->(:Issue)
//
// This gives us not only the matching chunk, but also:
// - the parent article
// - the issue solved by that article
// - the similarity score
// - the test case name

CALL {

  // ===========================================================================
  // SUBQUERY BLOCK
  // ===========================================================================
  // CALL { ... } creates a subquery.
  //
  // A subquery lets us run a smaller query inside the main query and then return
  // its results to the outer query.
  //
  // In this case, the subquery contains two separate vector-search test cases,
  // combined using UNION ALL.
  //
  // Why use a subquery here?
  //
  // Because we want both test cases to produce the same output columns:
  //
  //     testName
  //     chunkId
  //     chunkText
  //     score
  //     articleId
  //     articleTitle
  //     issueId
  //     issueName
  //
  // Then the outer RETURN can display both test results together in one final
  // table.
  //
  // Think of this like running two experiments and placing their results into
  // one combined report.

  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    3,
    [0.08, 0.92, 0.12]
  )

  // ===========================================================================
  // TEST CASE 1: PAYMENT-STYLE VECTOR SEARCH
  // ===========================================================================
  // This procedure call searches the vector index using a payment-oriented test
  // vector.
  //
  // The procedure is:
  //
  //     db.index.vector.queryNodes()
  //
  // It searches a vector index and returns nodes whose stored embedding vectors
  // are closest to the query vector.
  //
  // ---------------------------------------------------------------------------
  // Parameter 1: 'documentChunk_embedding_vector'
  // ---------------------------------------------------------------------------
  // This is the name of the vector index to search.
  //
  // Earlier, this index was created on:
  //
  //     (:DocumentChunk).embedding
  //
  // So this procedure searches DocumentChunk nodes using their embedding values.
  //
  // The index name must match exactly. If the name is misspelled, Neo4j will not
  // search the intended vector index.
  //
  // ---------------------------------------------------------------------------
  // Parameter 2: 3
  // ---------------------------------------------------------------------------
  // This tells Neo4j to return the top 3 most similar chunks.
  //
  // This is called top-k retrieval.
  //
  // In RAG systems, top-k controls how many chunks are retrieved as candidate
  // context.
  //
  // Here, top 3 is useful because it lets us see the two strongest expected
  // payment chunks and one additional nearest chunk for comparison.
  //
  // ---------------------------------------------------------------------------
  // Parameter 3: [0.08, 0.92, 0.12]
  // ---------------------------------------------------------------------------
  // This is the payment-style query vector.
  //
  // It has a high second dimension:
  //
  //     0.92
  //
  // That makes it close to the payment-related demo chunk embeddings:
  //
  //     C-K002-001 -> [0.05, 0.95, 0.10]
  //     C-K002-002 -> [0.10, 0.90, 0.15]
  //
  // So we expect the top results to be payment-related chunks.
  //
  // In a real application, this vector would normally be generated from a user
  // question such as:
  //
  //     "Customer payment failed during checkout."
  //
  // But in this lab, we manually provide a 3-dimensional vector so students can
  // clearly see how vector closeness works.

  YIELD node AS dc, score

  // ===========================================================================
  // RECEIVE PAYMENT VECTOR SEARCH RESULTS
  // ===========================================================================
  // YIELD receives the output from the vector search procedure.
  //
  // The procedure returns:
  //
  //     node
  //     score
  //
  // ---------------------------------------------------------------------------
  // node AS dc
  // ---------------------------------------------------------------------------
  // node is the matched node returned by the vector index.
  //
  // Because this index is built on DocumentChunk.embedding, each returned node
  // should be a DocumentChunk.
  //
  // We rename node to dc because dc clearly means:
  //
  //     DocumentChunk
  //
  // This makes the rest of the query easier to read.
  //
  // ---------------------------------------------------------------------------
  // score
  // ---------------------------------------------------------------------------
  // score tells us how similar the returned chunk's embedding is to the query
  // vector.
  //
  // Since this vector index uses cosine similarity, a higher score means a
  // stronger semantic/vector match.
  //
  // In simple terms:
  //
  //     Higher score = better match
  //
  // For this payment-style vector, payment-related chunks should score highest.

  MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

  // ===========================================================================
  // EXPAND PAYMENT RESULTS INTO ARTICLE AND ISSUE CONTEXT
  // ===========================================================================
  // After vector search retrieves the matching DocumentChunk nodes, this MATCH
  // follows graph relationships from each retrieved chunk.
  //
  // The pattern is:
  //
  //     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
  //
  // In plain English:
  //
  //     "For each retrieved chunk, find the article it belongs to,
  //      and then find the issue that article solves."
  //
  // ---------------------------------------------------------------------------
  // (dc)-[:PART_OF]->(ka:KnowledgeArticle)
  // ---------------------------------------------------------------------------
  // This connects the retrieved chunk to its parent article.
  //
  // Example:
  //
  //     (:DocumentChunk {chunkId: "C-K002-001"})
  //         -[:PART_OF]->
  //     (:KnowledgeArticle {articleId: "K002"})
  //
  // This is important because a chunk is only a small piece of text.
  // The KnowledgeArticle gives us the larger source context.
  //
  // ---------------------------------------------------------------------------
  // (ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
  // ---------------------------------------------------------------------------
  // This connects the parent article to the Issue it solves.
  //
  // Example:
  //
  //     (:KnowledgeArticle {articleId: "K002"})
  //         -[:SOLVES]->
  //     (:Issue {name: "Payment Failure"})
  //
  // This gives business meaning to the vector search result.
  //
  // The vector index tells us:
  //
  //     "This chunk is similar to the query vector."
  //
  // The graph tells us:
  //
  //     "This chunk belongs to a payment article that solves Payment Failure."

  RETURN
    "Payment-style test vector" AS testName,
    dc.chunkId AS chunkId,
    dc.text AS chunkText,
    score,
    ka.articleId AS articleId,
    ka.title AS articleTitle,
    i.issueId AS issueId,
    i.name AS issueName

  // ===========================================================================
  // RETURN PAYMENT TEST RESULT ROWS
  // ===========================================================================
  // This RETURN creates result rows for the payment-style test case.
  //
  // ---------------------------------------------------------------------------
  // "Payment-style test vector" AS testName
  // ---------------------------------------------------------------------------
  // This hardcoded label identifies which test case produced the row.
  //
  // This is important because the final output combines payment and app-crash
  // results together.
  //
  // Without testName, it would be harder to tell which vector produced which
  // result.
  //
  // ---------------------------------------------------------------------------
  // dc.chunkId AS chunkId
  // ---------------------------------------------------------------------------
  // chunkId identifies the retrieved DocumentChunk.
  //
  // Expected top payment chunks may include:
  //
  //     C-K002-001
  //     C-K002-002
  //
  // ---------------------------------------------------------------------------
  // dc.text AS chunkText
  // ---------------------------------------------------------------------------
  // chunkText is the actual retrieved text.
  //
  // For the payment-style vector, we expect text mentioning things like:
  //
  // - payment fails
  // - checkout
  // - card status
  // - available balance
  // - payment gateway response
  // - retry transaction
  //
  // ---------------------------------------------------------------------------
  // score
  // ---------------------------------------------------------------------------
  // score shows the similarity between the query vector and the chunk embedding.
  //
  // Higher score means stronger match.
  //
  // ---------------------------------------------------------------------------
  // ka.articleId AS articleId
  // ---------------------------------------------------------------------------
  // articleId identifies the parent KnowledgeArticle.
  //
  // For payment-related chunks, we expect:
  //
  //     K002
  //
  // ---------------------------------------------------------------------------
  // ka.title AS articleTitle
  // ---------------------------------------------------------------------------
  // articleTitle gives the readable title of the parent article.
  //
  // For payment-related chunks, we expect something like:
  //
  //     Resolve payment failure
  //
  // ---------------------------------------------------------------------------
  // i.issueId AS issueId
  // ---------------------------------------------------------------------------
  // issueId identifies the connected Issue node.
  //
  // ---------------------------------------------------------------------------
  // i.name AS issueName
  // ---------------------------------------------------------------------------
  // issueName gives the readable issue name.
  //
  // For this test case, we expect:
  //
  //     Payment Failure

  UNION ALL

  // ===========================================================================
  // UNION ALL: COMBINE PAYMENT TEST RESULTS WITH APP-CRASH TEST RESULTS
  // ===========================================================================
  // UNION ALL combines the rows from the first vector-search test case with the
  // rows from the second vector-search test case.
  //
  // Why UNION ALL?
  //
  // Because we want a single combined report containing both test cases.
  //
  // UNION ALL keeps all rows exactly as returned.
  //
  // This is better than UNION for validation because UNION may remove duplicate
  // rows if two rows look identical.
  //
  // For testing and troubleshooting, we usually want to see every returned row.
  //
  // Important rule:
  //
  // Both sides of UNION ALL must return the same number of columns with compatible
  // names and types.
  //
  // That is why both test cases return:
  //
  //     testName
  //     chunkId
  //     chunkText
  //     score
  //     articleId
  //     articleTitle
  //     issueId
  //     issueName

  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    3,
    [0.12, 0.10, 0.92]
  )

  // ===========================================================================
  // TEST CASE 2: APP-CRASH-STYLE VECTOR SEARCH
  // ===========================================================================
  // This second procedure call searches the same vector index, but with an
  // app-crash-oriented test vector.
  //
  // The index is the same:
  //
  //     documentChunk_embedding_vector
  //
  // The top-k value is also the same:
  //
  //     3
  //
  // The query vector is different:
  //
  //     [0.12, 0.10, 0.92]
  //
  // This vector has a high third dimension:
  //
  //     0.92
  //
  // That makes it close to the app-crash-related demo embeddings:
  //
  //     C-K003-001 -> [0.10, 0.10, 0.95]
  //     C-K003-002 -> [0.15, 0.10, 0.90]
  //
  // So we expect the top results to be app-crash-related chunks.
  //
  // In a real application, this vector might come from a user question like:
  //
  //     "The mobile app keeps crashing on my phone."
  //
  // But here we manually use a simple vector to make the test easy to understand.

  YIELD node AS dc, score

  // ===========================================================================
  // RECEIVE APP-CRASH VECTOR SEARCH RESULTS
  // ===========================================================================
  // As before, the procedure returns:
  //
  //     node
  //     score
  //
  // node is renamed to dc because the returned node is expected to be a
  // DocumentChunk.
  //
  // score tells us how close each chunk is to the app-crash-style query vector.
  //
  // For this test vector, app-crash chunks should have the highest scores.

  MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

  // ===========================================================================
  // EXPAND APP-CRASH RESULTS INTO ARTICLE AND ISSUE CONTEXT
  // ===========================================================================
  // This MATCH follows the same graph-context path:
  //
  //     DocumentChunk -> KnowledgeArticle -> Issue
  //
  // In plain English:
  //
  //     "For each app-crash-like retrieved chunk,
  //      find the parent article and the issue solved by that article."
  //
  // For expected app-crash results, the graph context should point to:
  //
  //     K003 - Fix app crash
  //
  // and:
  //
  //     App Crash
  //
  // This proves that the vector search result is not only mathematically close,
  // but also connected to the correct business topic in the graph.

  RETURN
    "App-crash-style test vector" AS testName,
    dc.chunkId AS chunkId,
    dc.text AS chunkText,
    score,
    ka.articleId AS articleId,
    ka.title AS articleTitle,
    i.issueId AS issueId,
    i.name AS issueName

  // ===========================================================================
  // RETURN APP-CRASH TEST RESULT ROWS
  // ===========================================================================
  // This RETURN creates result rows for the app-crash-style test case.
  //
  // ---------------------------------------------------------------------------
  // "App-crash-style test vector" AS testName
  // ---------------------------------------------------------------------------
  // This labels the row as belonging to the app-crash test.
  //
  // ---------------------------------------------------------------------------
  // dc.chunkId AS chunkId
  // ---------------------------------------------------------------------------
  // Expected top app-crash chunks may include:
  //
  //     C-K003-001
  //     C-K003-002
  //
  // ---------------------------------------------------------------------------
  // dc.text AS chunkText
  // ---------------------------------------------------------------------------
  // For the app-crash-style vector, we expect chunk text mentioning things like:
  //
  // - mobile app crashes
  // - update the app
  // - clear cache
  // - restart device
  // - device compatibility
  //
  // ---------------------------------------------------------------------------
  // score
  // ---------------------------------------------------------------------------
  // Higher score means the chunk embedding is more similar to the app-crash test
  // vector.
  //
  // ---------------------------------------------------------------------------
  // ka.articleId AS articleId
  // ---------------------------------------------------------------------------
  // For app-crash-related chunks, we expect:
  //
  //     K003
  //
  // ---------------------------------------------------------------------------
  // ka.title AS articleTitle
  // ---------------------------------------------------------------------------
  // For app-crash-related chunks, we expect something like:
  //
  //     Fix app crash
  //
  // ---------------------------------------------------------------------------
  // i.issueId AS issueId
  // ---------------------------------------------------------------------------
  // This identifies the Issue node connected to the article.
  //
  // ---------------------------------------------------------------------------
  // i.name AS issueName
  // ---------------------------------------------------------------------------
  // For this test case, we expect:
  //
  //     App Crash
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

// =============================================================================
// OUTER RETURN: DISPLAY THE COMBINED TEST RESULTS
// =============================================================================
// After the subquery finishes, the outer RETURN displays the combined results
// from both vector-search test cases.
//
// This final output contains rows from:
//
//     Payment-style test vector
//     App-crash-style test vector
//
// Each row includes:
//
// - testName
// - chunkId
// - chunkText
// - score
// - articleId
// - articleTitle
// - issueId
// - issueName
//
// This makes the output easy to inspect because each result row shows both:
//
//     1. The vector-search match
//     2. The graph context behind that match
//
// -----------------------------------------------------------------------------
// testName
// -----------------------------------------------------------------------------
// testName tells us which test vector produced the row.
//
// This is important because the output contains multiple test scenarios.
//
// -----------------------------------------------------------------------------
// chunkId
// -----------------------------------------------------------------------------
// chunkId identifies the retrieved chunk.
//
// This helps us verify whether the expected chunks were retrieved.
//
// -----------------------------------------------------------------------------
// chunkText
// -----------------------------------------------------------------------------
// chunkText shows the actual text content of the retrieved chunk.
//
// This lets us manually confirm whether the retrieved text matches the test
// intent.
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score shows the similarity score from the vector index.
//
// Higher score means stronger vector similarity.
//
// -----------------------------------------------------------------------------
// articleId and articleTitle
// -----------------------------------------------------------------------------
// These fields trace the chunk back to its parent KnowledgeArticle.
//
// This proves the PART_OF relationship is working during retrieval expansion.
//
// -----------------------------------------------------------------------------
// issueId and issueName
// -----------------------------------------------------------------------------
// These fields trace the parent article to the Issue it solves.
//
// This proves the SOLVES relationship is working during retrieval expansion.
//
// Together, these fields show the complete graph-enhanced retrieval chain:
//
//     test vector
//       -> matching chunk
//       -> parent article
//       -> solved issue

ORDER BY
  testName,
  score DESC;

// =============================================================================
// SORT RESULTS BY TEST CASE AND BEST MATCH
// =============================================================================
// ORDER BY controls the display order of the final combined result.
//
// Here we sort by:
//
//     testName,
//     score DESC
//
// This means:
//
//     1. Group rows by test case name.
//     2. Within each test case, show the highest-scoring result first.
//
// -----------------------------------------------------------------------------
// Why sort by testName first?
// -----------------------------------------------------------------------------
// Sorting by testName keeps payment results and app-crash results grouped
// together.
//
// This makes the output easier to read.
//
// Instead of mixing both test cases, the result appears like:
//
//     App-crash-style test vector
//       top result
//       second result
//       third result
//
//     Payment-style test vector
//       top result
//       second result
//       third result
//
// Depending on alphabetical order, App-crash may appear before Payment.
//
// -----------------------------------------------------------------------------
// Why sort by score DESC inside each test?
// -----------------------------------------------------------------------------
// score DESC means highest similarity score first.
//
// This is exactly what we want because the strongest match should appear at the
// top of each test group.
//
// -----------------------------------------------------------------------------
// Expected result behavior
// -----------------------------------------------------------------------------
// For the payment-style vector:
//
//     [0.08, 0.92, 0.12]
//
// Expected top results should include:
//
//     C-K002-001
//     C-K002-002
//
// with article/issue context:
//
//     K002 - Resolve payment failure
//     Payment Failure
//
// For the app-crash-style vector:
//
//     [0.12, 0.10, 0.92]
//
// Expected top results should include:
//
//     C-K003-001
//     C-K003-002
//
// with article/issue context:
//
//     K003 - Fix app crash
//     App Crash
//
// The third result in each test group may be less relevant because we asked for
// top 3 results, and after the strongest two topic-specific chunks, Neo4j still
// returns the next nearest available chunk.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a multi-scenario vector retrieval validation query.
//
// It tests whether the same vector index can correctly retrieve different
// topic-specific chunks when given different query vectors.
//
// The complete flow is:
//
//     1. Run payment-style vector search.
//     2. Expand retrieved payment chunks to article and issue context.
//     3. Run app-crash-style vector search.
//     4. Expand retrieved app-crash chunks to article and issue context.
//     5. Combine both result sets using UNION ALL.
//     6. Return one clean report.
//     7. Sort by test name and score.
//
// The key learning point is:
//
//     Vector search finds semantically similar chunks.
//     Graph traversal explains those chunks with article and issue context.
//     UNION ALL lets us compare multiple retrieval tests in one output.
//
// In a healthy lab result:
//
//     Payment-style vector
//       should retrieve payment chunks linked to Payment Failure.
//
//     App-crash-style vector
//       should retrieve app-crash chunks linked to App Crash.
//
// This proves that the graph is ready for explainable, graph-enhanced RAG
// retrieval across multiple intent types.
//
// Production note:
// Your pasted query contains -&gt;, which is the HTML-escaped form of ->.
// In Neo4j Browser or Cypher Shell, use the real Cypher arrow:
//
//     MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

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
