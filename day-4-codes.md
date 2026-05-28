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
// =============================================================================
// EXPLAINABLE VECTOR RETRIEVAL WITH GRAPH CONTEXT ENRICHMENT
// =============================================================================
// This query performs multiple vector similarity searches against DocumentChunk
// nodes and then enriches each retrieved chunk with connected graph context.
//
// In simple terms, it tells Neo4j:
//
//     "Find the most semantically similar document chunks for different
//      test-style query vectors, then explain those results using the
//      surrounding knowledge graph."
//
// This is more powerful than a plain vector search.
//
// A plain vector search can only answer:
//
//     "Which chunks are closest to this query embedding?"
//
// But this query answers:
//
//     "Which chunks are closest,
//      which knowledge articles do they belong to,
//      which issues do those articles solve,
//      which tickets are connected to those issues,
//      which customers raised those tickets,
//      which products are involved,
//      which agents are assigned,
//      and what graph analytics context exists for those tickets?"
//
// This query is useful for:
// - Testing vector retrieval quality
// - Demonstrating explainable RAG
// - Connecting semantic search results to graph relationships
// - Showing operational ticket context behind retrieved knowledge
// - Validating that DocumentChunk, KnowledgeArticle, Issue, Ticket, Customer,
//   Product, and Agent nodes are connected correctly
// - Combining vector similarity with graph-based reasoning
// - Building production-style customer support intelligence workflows

CALL {

// =============================================================================
// SUBQUERY FOR COMBINING MULTIPLE VECTOR SEARCH TESTS
// =============================================================================
// CALL { ... } starts a Cypher subquery.
//
// A subquery is like a temporary workspace inside the main query.
//
// Here, we use the subquery to run three different vector retrieval tests:
//
//     1. Login-style retrieval
//     2. Payment-style retrieval
//     3. App-crash-style retrieval
//
// Each test uses a different query vector.
//
// Even though the vectors are different, every test returns the same structure:
//
//     testName
//     dc
//     score
//
// This common structure is important because the results are combined later
// using UNION ALL.
//
// Think of this subquery as creating one combined stream of vector results.
// After the subquery finishes, the rest of the query can treat all retrieved
// chunks in the same way, regardless of which test produced them.
//
// This design is useful in labs and demos because it allows us to compare
// multiple semantic retrieval scenarios in one single query.

  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    2,
    [0.92, 0.12, 0.05]
  )

// =============================================================================
// LOGIN-STYLE VECTOR SEARCH
// =============================================================================
// This procedure call queries the Neo4j vector index named:
//
//     documentChunk_embedding_vector
//
// The procedure:
//
//     db.index.vector.queryNodes()
//
// searches for nodes whose stored embedding vectors are closest to the input
// query vector.
//
// Let us break down the arguments:
//
// -----------------------------------------------------------------------------
// 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// This is the name of the vector index.
//
// The index is expected to be created on DocumentChunk nodes, most likely on
// the embedding property.
//
// In simple terms, this index allows Neo4j to quickly search for chunks by
// semantic similarity instead of scanning every chunk manually.
//
// -----------------------------------------------------------------------------
// 2
// -----------------------------------------------------------------------------
// This means:
//
//     "Return the top 2 most similar nodes."
//
// So for this login-style query vector, Neo4j will return only the two closest
// matching DocumentChunk nodes.
//
// In production, this value is often called topK.
//
// A smaller topK gives fewer but more focused results.
// A larger topK gives broader recall but may include weaker matches.
//
// -----------------------------------------------------------------------------
// [0.92, 0.12, 0.05]
// -----------------------------------------------------------------------------
// This is the query embedding vector.
//
// In this lab dataset, this vector appears to represent a login-style meaning.
//
// For example, it may be designed to retrieve chunks related to:
// - Login failures
// - Authentication problems
// - Password issues
// - User access problems
//
// Important point:
//
//     The vector itself is not human-readable text.
//
// It is a numeric representation of meaning.
//
// In a real application, this vector would normally be generated from a user
// question using an embedding model. For example:
//
//     User question: "Why can the customer not log in?"
//     Embedding model: converts that sentence into a numeric vector
//     Neo4j vector index: finds chunks with similar vectors
//
// In this lab, the vector is hardcoded so that we can test and explain the
// retrieval behavior clearly.

  YIELD node AS dc, score

// =============================================================================
// CAPTURE VECTOR SEARCH OUTPUT
// =============================================================================
// YIELD extracts the values returned by db.index.vector.queryNodes().
//
// The procedure returns two important things:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the matching graph node returned by the vector index.
//
// We rename it to:
//
//     dc
//
// because this node represents a DocumentChunk.
//
// Using dc as the variable name makes the rest of the query easier to read.
//
// It tells us:
//
//     "This variable holds the retrieved document chunk."
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the similarity score between the query vector and the stored
// DocumentChunk embedding.
//
// A higher score generally means a stronger semantic match.
//
// If the vector index uses cosine similarity, then the score represents how
// close the direction of the two vectors is.
//
// In simple terms:
//
//     Higher score = more semantically similar
//     Lower score  = less semantically similar
//
// Production note:
//
// Vector score is useful, but it should not be the only explanation.
// That is why this query later joins the chunk to KnowledgeArticle, Issue,
// Ticket, Customer, Product, and Agent context.

  RETURN
    "Login-style cleaned explainable retrieval" AS testName,
    dc,
    score

// =============================================================================
// RETURN LOGIN-STYLE RETRIEVAL RESULT
// =============================================================================
// This RETURN statement outputs the result of the first vector search branch.
//
// It returns three fields:
//
//     testName
//     dc
//     score
//
// -----------------------------------------------------------------------------
// testName
// -----------------------------------------------------------------------------
// This is a human-readable label for the retrieval test.
//
// Here the value is:
//
//     "Login-style cleaned explainable retrieval"
//
// This label helps us identify which vector search produced the row.
//
// This is important because this query runs three different vector searches.
// Without testName, the final output would mix login, payment, and app-crash
// results together without a clear source label.
//
// -----------------------------------------------------------------------------
// dc
// -----------------------------------------------------------------------------
// This is the retrieved DocumentChunk node.
//
// It will later be matched to its parent KnowledgeArticle.
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// This is the vector similarity score for the retrieved chunk.
//
// It will later be returned as vectorScore in the final output.

  UNION ALL

// =============================================================================
// UNION ALL TO COMBINE VECTOR SEARCH RESULT SETS
// =============================================================================
// UNION ALL combines the results of multiple query branches.
//
// In this query, it combines:
//
//     Login-style retrieval results
//     Payment-style retrieval results
//     App-crash-style retrieval results
//
// Why UNION ALL instead of UNION?
//
//     UNION removes duplicate rows.
//     UNION ALL keeps all rows.
//
// For retrieval testing, UNION ALL is usually better because the same chunk
// may appear in more than one semantic test.
//
// For example, a chunk might be relevant to both:
// - Login troubleshooting
// - Application crash troubleshooting
//
// If we used UNION, Neo4j might remove duplicate-looking rows and hide useful
// diagnostic information.
//
// UNION ALL preserves the full retrieval evidence.
//
// Important Cypher rule:
//
//     Every UNION branch must return the same number of columns
//     with the same column names.
//
// That is why every branch returns:
//
//     testName,
//     dc,
//     score

  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    2,
    [0.08, 0.92, 0.12]
  )

// =============================================================================
// PAYMENT-STYLE VECTOR SEARCH
// =============================================================================
// This is the second vector retrieval test.
//
// It queries the same vector index:
//
//     documentChunk_embedding_vector
//
// It again asks for:
//
//     top 2 matching DocumentChunk nodes
//
// But this time it uses a different query vector:
//
//     [0.08, 0.92, 0.12]
//
// In this lab dataset, this vector appears to represent payment-style meaning.
//
// For example, it may retrieve chunks related to:
// - Payment failures
// - Billing issues
// - Transaction errors
// - Checkout problems
//
// This helps us test whether the vector index can separate payment-related
// chunks from login-related or app-crash-related chunks.
//
// Production note:
//
// In a real support application, this vector would be generated dynamically
// from a user's question such as:
//
//     "The customer says their payment failed during checkout."
//
// The embedding model would convert that sentence into a vector, and Neo4j
// would use the vector index to find semantically similar chunks.

  YIELD node AS dc, score

// =============================================================================
// CAPTURE PAYMENT-STYLE VECTOR SEARCH OUTPUT
// =============================================================================
// Just like the first branch, this YIELD captures:
//
//     node AS dc
//     score
//
// The retrieved node is renamed to dc because it is expected to be a
// DocumentChunk.
//
// The score tells us how strongly the chunk matched the payment-style query
// vector.
//
// Keeping the variable names the same across all UNION branches is important.
// It allows the outer query to process all retrieved chunks in one common way.

  RETURN
    "Payment-style cleaned explainable retrieval" AS testName,
    dc,
    score

// =============================================================================
// RETURN PAYMENT-STYLE RETRIEVAL RESULT
// =============================================================================
// This RETURN statement gives the second branch the same output shape as the
// first branch:
//
//     testName
//     dc
//     score
//
// The testName value:
//
//     "Payment-style cleaned explainable retrieval"
//
// allows the final output to clearly show which rows came from the payment
// retrieval scenario.
//
// This is especially useful when comparing vector scores across multiple test
// cases in the same result table.

  UNION ALL

// =============================================================================
// UNION ALL FOR THE THIRD VECTOR TEST
// =============================================================================
// This UNION ALL adds one more retrieval scenario to the combined result stream.
//
// At this point, the subquery has already included:
//
//     Login-style results
//     Payment-style results
//
// The next branch adds:
//
//     App-crash-style results
//
// Again, UNION ALL keeps all rows without deduplication.
//
// This is helpful for explainability because if the same chunk appears in
// multiple retrieval scenarios, that overlap itself may be meaningful.

  CALL db.index.vector.queryNodes(
    'documentChunk_embedding_vector',
    2,
    [0.12, 0.10, 0.92]
  )

// =============================================================================
// APP-CRASH-STYLE VECTOR SEARCH
// =============================================================================
// This is the third vector retrieval test.
//
// It uses the same vector index:
//
//     documentChunk_embedding_vector
//
// It asks for the top 2 nearest chunks using this query vector:
//
//     [0.12, 0.10, 0.92]
//
// In this lab dataset, this vector appears to represent app-crash-style meaning.
//
// For example, it may retrieve chunks related to:
// - Mobile app crash
// - Application instability
// - Runtime failures
// - Crash after login or checkout
// - Error handling problems
//
// The purpose is to verify that the vector index retrieves chunks that are
// semantically close to application crash issues.
//
// Production note:
//
// Hardcoded vectors are useful for controlled demos because they make the
// behavior predictable.
//
// In production, vectors should normally come from an embedding model rather
// than being manually typed into the query.

  YIELD node AS dc, score

// =============================================================================
// CAPTURE APP-CRASH-STYLE VECTOR SEARCH OUTPUT
// =============================================================================
// This YIELD captures the matching DocumentChunk node and its vector similarity
// score.
//
// node AS dc means:
//
//     "Take the returned node and call it dc."
//
// score means:
//
//     "Keep the similarity score so we can rank and inspect the result later."
//
// This keeps the third branch compatible with the previous UNION ALL branches.

  RETURN
    "App-crash-style cleaned explainable retrieval" AS testName,
    dc,
    score

// =============================================================================
// RETURN APP-CRASH-STYLE RETRIEVAL RESULT
// =============================================================================
// This final branch returns:
//
//     testName
//     dc
//     score
//
// The testName value:
//
//     "App-crash-style cleaned explainable retrieval"
//
// makes it easy to identify app-crash retrieval results in the final output.
//
// After this point, the CALL subquery ends and the combined vector result stream
// becomes available to the main query.

}

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// CONNECT RETRIEVED CHUNK TO KNOWLEDGE ARTICLE AND ISSUE
// =============================================================================
// This MATCH clause enriches each retrieved DocumentChunk with graph context.
//
// The pattern is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// Let us break this down:
//
// -----------------------------------------------------------------------------
// (dc)
// -----------------------------------------------------------------------------
// dc is the DocumentChunk retrieved by vector search.
//
// It came from one of the three vector query branches inside the subquery.
//
// -----------------------------------------------------------------------------
// -[:PART_OF]->
// -----------------------------------------------------------------------------
// PART_OF is a relationship showing that the chunk belongs to a larger
// KnowledgeArticle.
//
// This is important because chunks are usually smaller pieces of a larger
// document.
//
// A chunk alone may not provide enough source context.
// Connecting it back to its KnowledgeArticle gives traceability.
//
// -----------------------------------------------------------------------------
// (ka:KnowledgeArticle)
// -----------------------------------------------------------------------------
// ka represents the parent knowledge article.
//
// This is the source article from which the retrieved chunk was created.
//
// Returning article details later helps answer:
//
//     "Which knowledge article supported this retrieved result?"
//
// -----------------------------------------------------------------------------
// -[:SOLVES]->
// -----------------------------------------------------------------------------
// SOLVES connects a KnowledgeArticle to the Issue it addresses.
//
// This means the article is not just random content.
// It is specifically meant to solve or explain a known issue.
//
// -----------------------------------------------------------------------------
// (i:Issue)
// -----------------------------------------------------------------------------
// i represents the operational issue solved by the article.
//
// This is the key step that makes the retrieval explainable.
//
// Instead of only saying:
//
//     "This chunk matched the vector query."
//
// We can now say:
//
//     "This chunk matched the vector query,
//      it belongs to this article,
//      and this article solves this specific issue."
//
// Production relevance:
//
// This is the bridge between semantic search and business meaning.
// It makes the vector retrieval result traceable, explainable, and useful for
// downstream support workflows.

OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

// =============================================================================
// OPTIONALLY FIND TICKETS RELATED TO THE ISSUE
// =============================================================================
// This OPTIONAL MATCH finds Ticket nodes connected to the retrieved Issue.
//
// The pattern is:
//
//     (t:Ticket)-[:HAS_ISSUE]->(i)
//
// Meaning:
//
//     "Find tickets that have this issue."
//
// -----------------------------------------------------------------------------
// Why OPTIONAL MATCH?
// -----------------------------------------------------------------------------
// OPTIONAL MATCH works like a left join in SQL.
//
// It means:
//
//     "Try to find this pattern.
//      If it exists, return it.
//      If it does not exist, keep the existing row anyway and use null."
//
// This is important because not every Issue is guaranteed to have a related
// Ticket.
//
// If we used a normal MATCH here, then any retrieved issue without a ticket
// would disappear from the final result.
//
// That would be dangerous because a valid vector result could be removed only
// because operational ticket context was missing.
//
// OPTIONAL MATCH keeps the result complete.
//
// -----------------------------------------------------------------------------
// Why ticket context matters
// -----------------------------------------------------------------------------
// Tickets provide real operational evidence.
//
// They help answer:
// - Has this issue actually occurred for customers?
// - How many tickets are related to it?
// - What is the ticket status?
// - What priority was assigned?
// - Is this issue still open or already resolved?
//
// This turns knowledge retrieval into operational intelligence.

OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)

// =============================================================================
// OPTIONALLY FIND CUSTOMERS WHO RAISED RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Customer nodes connected to the related tickets.
//
// The pattern is:
//
//     (c:Customer)-[:RAISED]->(t)
//
// Meaning:
//
//     "Find customers who raised these tickets."
//
// -----------------------------------------------------------------------------
// Why this is optional
// -----------------------------------------------------------------------------
// A ticket may exist without a connected Customer node.
//
// This can happen in:
// - Demo datasets
// - Incomplete imports
// - Anonymized support data
// - Partially migrated systems
// - Tickets created by automated monitoring instead of customers
//
// OPTIONAL MATCH prevents the query from losing valid ticket or issue rows when
// customer data is missing.
//
// -----------------------------------------------------------------------------
// Why customer context matters
// -----------------------------------------------------------------------------
// Customer context helps measure business impact.
//
// It helps answer:
// - Which customers are affected?
// - Is one customer repeatedly facing the same issue?
// - Are multiple customers impacted by the same issue?
// - Should this issue be escalated because of customer impact?
//
// In production support systems, customer context is often one of the most
// important enrichment dimensions.

OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)

// =============================================================================
// OPTIONALLY FIND PRODUCTS RELATED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Product nodes connected to the related tickets.
//
// The pattern is:
//
//     (t)-[:ABOUT]->(p:Product)
//
// Meaning:
//
//     "Find the product that this ticket is about."
//
// -----------------------------------------------------------------------------
// Why product context matters
// -----------------------------------------------------------------------------
// Product context helps identify the affected system or business area.
//
// It helps answer:
// - Which product is impacted?
// - Is the issue isolated to one product?
// - Does the same issue appear across multiple products?
// - Which product team may need to investigate?
//
// For example, a payment issue in one product may be less severe than the same
// issue appearing across several products.
//
// -----------------------------------------------------------------------------
// Why OPTIONAL MATCH is used
// -----------------------------------------------------------------------------
// Not every ticket may have product information.
//
// If the product relationship is missing, we still want to keep the ticket and
// issue context in the result.

OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

// =============================================================================
// OPTIONALLY FIND AGENTS ASSIGNED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Agent nodes assigned to the related tickets.
//
// The pattern is:
//
//     (t)-[:ASSIGNED_TO]->(a:Agent)
//
// Meaning:
//
//     "Find the support agent assigned to this ticket."
//
// -----------------------------------------------------------------------------
// Why agent context matters
// -----------------------------------------------------------------------------
// Agent context helps with ownership and operational follow-up.
//
// It helps answer:
// - Who is handling the ticket?
// - Which agents have worked on this issue?
// - Are related tickets concentrated under one agent?
// - Is escalation or reassignment needed?
//
// -----------------------------------------------------------------------------
// Why this is optional
// -----------------------------------------------------------------------------
// Some tickets may not be assigned yet.
//
// If we used a normal MATCH, unassigned tickets would disappear from the result.
//
// OPTIONAL MATCH keeps those tickets visible while showing null for missing
// agent details.

RETURN

// =============================================================================
// FINAL RETURN SHAPE
// =============================================================================
// The RETURN clause defines the final output columns.
//
// Instead of returning raw graph nodes only, this query returns a clean,
// readable, report-friendly structure.
//
// This is useful for:
// - Neo4j Browser result tables
// - Demo walkthroughs
// - RAG explainability output
// - API responses
// - Support dashboards
// - Troubleshooting analysis
//
// The returned columns are grouped into several logical areas:
//
//     1. Retrieval test metadata
//     2. Retrieved chunk details
//     3. Knowledge article context
//     4. Issue context
//     5. Related operational context
//     6. Ticket analytics context
//     7. Human-readable operational status

  testName,

// =============================================================================
// RETURN TEST NAME
// =============================================================================
// testName identifies which vector retrieval scenario produced the row.
//
// Possible values are:
//
//     "Login-style cleaned explainable retrieval"
//     "Payment-style cleaned explainable retrieval"
//     "App-crash-style cleaned explainable retrieval"
//
// This is important because all three vector searches are combined into one
// final result table.
//
// Without testName, we would not know whether a retrieved chunk came from the
// login, payment, or app-crash test.
//
// This makes comparison and debugging much easier.

  dc.chunkId AS chunkId,

// =============================================================================
// RETURN RETRIEVED CHUNK ID
// =============================================================================
// dc.chunkId returns the unique identifier of the retrieved DocumentChunk.
//
// The alias:
//
//     AS chunkId
//
// makes the output column cleaner and easier to read.
//
// This field helps trace exactly which chunk was retrieved by vector search.
//
// Traceability is important in explainable RAG because users should be able to
// inspect the source chunk behind a generated or recommended answer.

  dc.text AS retrievedChunk,

// =============================================================================
// RETURN RETRIEVED CHUNK TEXT
// =============================================================================
// dc.text returns the actual text stored in the retrieved DocumentChunk.
//
// The alias:
//
//     AS retrievedChunk
//
// makes it clear that this text is the chunk retrieved from vector similarity
// search.
//
// This is the core content that matched the query embedding.
//
// In a RAG workflow, this text could be passed to an LLM as supporting context.
//
// Production note:
//
// Always inspect retrievedChunk during testing.
// A high vector score is useful, but the actual text should still be reviewed
// to confirm that the retrieved content is relevant and safe to use.

  score AS vectorScore,

// =============================================================================
// RETURN VECTOR SIMILARITY SCORE
// =============================================================================
// score is returned as vectorScore.
//
// This value tells us how similar the retrieved chunk embedding was to the
// query embedding.
//
// Higher score generally means stronger semantic similarity.
//
// This helps rank retrieved chunks within each test scenario.
//
// Important reminder:
//
// Vector score should be treated as a relevance signal, not absolute truth.
//
// That is why this query also returns article, issue, ticket, customer, product,
// agent, and analytics context.

  ka.articleId AS articleId,

// =============================================================================
// RETURN KNOWLEDGE ARTICLE ID
// =============================================================================
// ka.articleId returns the business identifier of the KnowledgeArticle.
//
// This helps identify the parent article from which the retrieved chunk came.
//
// This is useful for source traceability.
//
// Instead of only seeing a chunk, we can say:
//
//     "This chunk belongs to article KA-xxxx."
//
// In production, articleId is often used for lookups, documentation links,
// support references, or audit trails.

  ka.title AS articleTitle,

// =============================================================================
// RETURN KNOWLEDGE ARTICLE TITLE
// =============================================================================
// ka.title returns the title of the KnowledgeArticle.
//
// The article title gives human-readable context for the retrieved chunk.
//
// This helps users quickly understand what the source article is about without
// opening the full article node.

  i.issueId AS issueId,

// =============================================================================
// RETURN ISSUE ID
// =============================================================================
// i.issueId returns the identifier of the Issue solved by the KnowledgeArticle.
//
// This connects the retrieved knowledge content to a known operational problem.
//
// This is important because it allows the system to move from:
//
//     "Relevant text was found."
//
// to:
//
//     "Relevant text was found for this specific issue."

  i.name AS issueName,

// =============================================================================
// RETURN ISSUE NAME
// =============================================================================
// i.name returns the human-readable name of the Issue.
//
// This helps the user understand what problem the article is addressing.
//
// For example, the issue name may be something like:
//
//     "Login Failure"
//     "Payment Declined"
//     "Mobile App Crash"
//
// The exact values depend on your graph data.

  i.severity AS issueSeverity,

// =============================================================================
// RETURN ISSUE SEVERITY
// =============================================================================
// i.severity returns the severity level of the Issue.
//
// Severity is important because two retrieved chunks may have similar vector
// scores, but the issue severity may influence which one should be prioritized.
//
// For example:
//
//     A high-severity payment outage may need faster attention than a
//     low-severity UI warning.
//
// This is a good example of why graph context improves vector search.
// Vector search finds semantic similarity.
// Graph context helps determine operational priority.

  collect(DISTINCT t.ticketId) AS relatedTicketIds,

// =============================================================================
// RETURN RELATED TICKET IDS
// =============================================================================
// collect(DISTINCT t.ticketId) gathers all ticket IDs connected to the issue.
//
// Why collect?
//
//     One issue can be connected to many tickets.
//
// If we returned t.ticketId directly, we could get multiple rows for the same
// retrieved chunk.
//
// collect() groups those ticket IDs into a list.
//
// -----------------------------------------------------------------------------
// Why DISTINCT?
// -----------------------------------------------------------------------------
// DISTINCT removes duplicates.
//
// Duplicates can appear because multiple OPTIONAL MATCH clauses may multiply
// rows internally.
//
// For example, if a ticket is connected to a customer, product, and agent,
// intermediate row expansion can cause repeated ticket IDs.
//
// DISTINCT keeps the final list clean.
//
// -----------------------------------------------------------------------------
// Why this field matters
// -----------------------------------------------------------------------------
// relatedTicketIds shows whether the retrieved issue has operational evidence.
//
// It helps answer:
//
//     "Which real tickets are connected to this retrieved issue?"

  collect(DISTINCT c.name) AS relatedCustomers,

// =============================================================================
// RETURN RELATED CUSTOMERS
// =============================================================================
// collect(DISTINCT c.name) gathers the names of customers who raised related
// tickets.
//
// This provides customer impact context.
//
// Why collect?
//
//     Multiple customers may have raised tickets for the same issue.
//
// Why DISTINCT?
//
//     The same customer may appear more than once through row expansion.
//
// This field helps answer:
// - Which customers are affected?
// - Is the same customer repeatedly affected?
// - Are many customers reporting the same issue?
// - Should this issue be escalated due to customer impact?

  collect(DISTINCT p.name) AS relatedProducts,

// =============================================================================
// RETURN RELATED PRODUCTS
// =============================================================================
// collect(DISTINCT p.name) gathers product names associated with related tickets.
//
// This provides product impact context.
//
// It helps answer:
// - Which product is affected?
// - Is the issue limited to one product?
// - Is the same issue affecting multiple products?
// - Which product team may need to investigate?

  collect(DISTINCT a.name) AS assignedAgents,

// =============================================================================
// RETURN ASSIGNED AGENTS
// =============================================================================
// collect(DISTINCT a.name) gathers the names of agents assigned to related
// tickets.
//
// This provides operational ownership context.
//
// It helps answer:
// - Who is working on the related tickets?
// - Are multiple agents involved?
// - Is one agent handling many related tickets?
// - Is escalation or coordination needed?

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

// =============================================================================
// RETURN STRUCTURED TICKET ANALYTICS CONTEXT
// =============================================================================
// This expression returns a list of structured ticket analytics maps.
//
// The overall expression is:
//
//     collect(DISTINCT CASE ... END) AS ticketAnalyticsContext
//
// Let us break this down carefully.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT ...)
// -----------------------------------------------------------------------------
// collect() groups multiple ticket analytics objects into a list.
//
// DISTINCT removes duplicate analytics maps that may appear because of optional
// graph expansion.
//
// This is useful because one retrieved issue may have many related tickets.
//
// -----------------------------------------------------------------------------
// CASE WHEN t IS NULL THEN null
// -----------------------------------------------------------------------------
// Because t comes from an OPTIONAL MATCH, it may be null.
//
// That means there may be no related ticket for the issue.
//
// The CASE expression handles that safely.
//
// If no ticket exists:
//
//     WHEN t IS NULL THEN null
//
// This prevents the query from creating a misleading ticket map with all fields
// set to null.
//
// -----------------------------------------------------------------------------
// ELSE { ... }
// -----------------------------------------------------------------------------
// If a ticket exists, the query creates a map containing important ticket
// details and graph analytics scores.
//
// A map is a structured object with key-value pairs.
//
// For example:
//
//     {
//       ticketId: "T-1001",
//       priority: "High",
//       status: "Open"
//     }
//
// This is cleaner than returning many separate repeated columns.
//
// -----------------------------------------------------------------------------
// ticketId
// -----------------------------------------------------------------------------
// ticketId identifies the related ticket.
//
// -----------------------------------------------------------------------------
// priority
// -----------------------------------------------------------------------------
// priority indicates how urgent or important the ticket is from an operational
// support perspective.
//
// -----------------------------------------------------------------------------
// status
// -----------------------------------------------------------------------------
// status tells whether the ticket is open, closed, in progress, resolved, etc.
//
// -----------------------------------------------------------------------------
// fullDegreeScore
// -----------------------------------------------------------------------------
// fullDegreeScore usually represents how connected the ticket is in the graph.
//
// A higher degree score can mean the ticket is connected to many other entities,
// such as customers, products, agents, or issues.
//
// -----------------------------------------------------------------------------
// pageRankScore
// -----------------------------------------------------------------------------
// pageRankScore usually represents graph-based importance or influence.
//
// A ticket with a high PageRank score may be connected to other important nodes.
//
// -----------------------------------------------------------------------------
// betweennessScore
// -----------------------------------------------------------------------------
// betweennessScore usually identifies bridge-like nodes.
//
// A ticket with high betweenness may connect different communities or parts of
// the graph.
//
// This can be useful for finding tickets that explain relationships between
// otherwise separate issue clusters.
//
// -----------------------------------------------------------------------------
// louvainCommunityId
// -----------------------------------------------------------------------------
// louvainCommunityId identifies the community detected by the Louvain algorithm.
//
// Community IDs help group related nodes together based on graph structure.
//
// -----------------------------------------------------------------------------
// labelPropagationCommunityId
// -----------------------------------------------------------------------------
// labelPropagationCommunityId identifies the community detected by the Label
// Propagation algorithm.
//
// This gives another way to understand clusters of related graph entities.
//
// -----------------------------------------------------------------------------
// Production relevance
// -----------------------------------------------------------------------------
// This ticketAnalyticsContext field is useful for prioritization and
// explainability.
//
// For example, two retrieved chunks may have similar vector scores.
//
// But if one chunk connects to an issue whose tickets have high PageRank,
// high betweenness, and high priority, that issue may deserve more attention.

  CASE
    WHEN count(t) = 0 THEN "No related ticket found for this issue"
    ELSE "Related ticket context found"
  END AS operationalContextStatus

// =============================================================================
// RETURN HUMAN-READABLE OPERATIONAL CONTEXT STATUS
// =============================================================================
// This CASE expression creates a simple human-readable status message.
//
// It checks:
//
//     count(t) = 0
//
// Meaning:
//
//     "Were any related Ticket nodes found?"
//
// -----------------------------------------------------------------------------
// WHEN count(t) = 0
// -----------------------------------------------------------------------------
// If no tickets were found, return:
//
//     "No related ticket found for this issue"
//
// This means the vector result still found a chunk, article, and issue, but no
// operational ticket context exists for that issue.
//
// -----------------------------------------------------------------------------
// ELSE
// -----------------------------------------------------------------------------
// If one or more tickets were found, return:
//
//     "Related ticket context found"
//
// This means the retrieved issue is connected to real ticket data.
//
// -----------------------------------------------------------------------------
// Why this field is useful
// -----------------------------------------------------------------------------
// This field makes the result easier to read during demos and troubleshooting.
//
// Instead of manually checking whether relatedTicketIds is empty, the query
// directly tells us whether operational context was found.
//
// Production note:
//
// This kind of derived status is useful for dashboards and API consumers because
// it turns raw graph data into an immediately understandable message.

ORDER BY
  testName,
  vectorScore DESC;

// =============================================================================
// ORDER FINAL RESULTS
// =============================================================================
// ORDER BY controls the order of rows in the final output.
//
// This query orders by:
//
//     testName
//     vectorScore DESC
//
// -----------------------------------------------------------------------------
// testName
// -----------------------------------------------------------------------------
// Ordering by testName groups rows from the same retrieval scenario together.
//
// This means login-style results appear together, payment-style results appear
// together, and app-crash-style results appear together.
//
// This makes the output easier to read and compare.
//
// -----------------------------------------------------------------------------
// vectorScore DESC
// -----------------------------------------------------------------------------
// DESC means descending order.
//
// So within each testName group, the highest vectorScore appears first.
//
// This is important because higher vector scores generally represent stronger
// semantic matches.
//
// -----------------------------------------------------------------------------
// Final behavior
// -----------------------------------------------------------------------------
// The final result is grouped by retrieval test and ranked by semantic
// similarity inside each group.
//
// This gives a clean explainable retrieval output:
//
//     test scenario -> best matching chunks -> article context -> issue context
//     -> ticket/customer/product/agent context -> analytics context
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a complete example of graph-enriched vector retrieval.
//
// The main purpose is:
//
//     "Retrieve semantically relevant document chunks and explain them using
//      the connected knowledge graph and operational support context."
//
// It combines:
//
// - Vector similarity search
// - Knowledge article traceability
// - Issue-level explanation
// - Ticket-level operational evidence
// - Customer impact context
// - Product impact context
// - Agent ownership context
// - Graph analytics context
//
// This is much closer to a production-ready explainable RAG pattern than simple
// vector search alone.
```

# Step 2 — Add a controlled natural-language question for Login Failure

```cypher
// =============================================================================
// PARAMETERIZED LOGIN-STYLE EXPLAINABLE VECTOR RETRIEVAL QUERY
// =============================================================================
// This query performs a login-focused vector similarity search and then enriches
// the retrieved document chunks with connected graph context.
//
// In simple terms, it tells Neo4j:
//
//     "Treat this user question as a login-related query,
//      search the vector index for the most relevant document chunks,
//      then explain the result using KnowledgeArticle, Issue, Ticket,
//      Customer, Product, Agent, and ticket analytics context."
//
// This is a very useful pattern for explainable RAG.
//
// A normal vector search can only answer:
//
//     "Which chunks are semantically closest to the question?"
//
// But this graph-enriched query answers:
//
//     "Which chunks are closest,
//      which knowledge article do they belong to,
//      which issue does that article solve,
//      which real support tickets are related,
//      which customers are affected,
//      which products are involved,
//      which agents are assigned,
//      and what analytics scores exist for those tickets?"
//
// This query is useful for:
// - Testing login-related vector retrieval
// - Demonstrating graph-powered RAG explainability
// - Connecting a natural-language user question to graph context
// - Validating chunk-to-article-to-issue relationships
// - Checking whether retrieved issues have operational ticket evidence
// - Building production-style customer support intelligence workflows

WITH
  "Why can't customers log in?" AS userQuestion,
  "login" AS queryType,
  [0.92, 0.12, 0.05] AS queryVector,
  2 AS topK

// =============================================================================
// DEFINE QUERY INPUTS USING WITH
// =============================================================================
// The WITH clause creates variables that will be used later in the query.
//
// Think of this section as the "input configuration" for the retrieval flow.
//
// Instead of hardcoding values directly inside every part of the query, we first
// define meaningful variables:
//
//     userQuestion
//     queryType
//     queryVector
//     topK
//
// This makes the query easier to read, easier to modify, and easier to convert
// later into an application/API-driven query.
//
// -----------------------------------------------------------------------------
// "Why can't customers log in?" AS userQuestion
// -----------------------------------------------------------------------------
// userQuestion stores the original natural-language question.
//
// This is the human-readable question that the user is asking.
//
// In this example:
//
//     "Why can't customers log in?"
//
// This question is not directly used by the vector index procedure, because
// Neo4j vector search needs a numeric vector, not raw text.
//
// However, returning userQuestion in the final output is still very useful
// because it keeps the retrieved result traceable to the original user intent.
//
// In production, this value would usually come from:
// - A chatbot question
// - A support analyst search box
// - An API request
// - A helpdesk assistant workflow
//
// -----------------------------------------------------------------------------
// "login" AS queryType
// -----------------------------------------------------------------------------
// queryType labels the type of query being executed.
//
// Here the value is:
//
//     "login"
//
// This helps classify the retrieval scenario.
//
// In a larger demo or production system, queryType could be used for:
// - Logging
// - Debugging
// - Routing
// - Evaluation reports
// - Comparing retrieval quality across issue categories
//
// For example, other queryType values might be:
//
//     "payment"
//     "app-crash"
//     "performance"
//     "account-access"
//
// -----------------------------------------------------------------------------
// [0.92, 0.12, 0.05] AS queryVector
// -----------------------------------------------------------------------------
// queryVector stores the numeric embedding used for vector search.
//
// This vector is the machine-readable representation of the user question.
//
// In this lab dataset, the vector:
//
//     [0.92, 0.12, 0.05]
//
// appears designed to represent login-style meaning.
//
// For example, it may retrieve chunks related to:
// - Login failures
// - Authentication problems
// - Password reset issues
// - Account access problems
// - Customer sign-in errors
//
// Important beginner note:
//
//     The vector itself is not readable like normal English.
//
// It is a list of numbers that represents meaning in embedding space.
//
// In a real production system, you would usually not manually write this vector.
// Instead, the flow would look like:
//
//     User question:
//       "Why can't customers log in?"
//
//     Embedding model:
//       Converts that question into a numeric vector.
//
//     Neo4j vector index:
//       Finds document chunks with similar vectors.
//
// Here, the vector is hardcoded because this is a controlled lab/demo query.
//
// -----------------------------------------------------------------------------
// 2 AS topK
// -----------------------------------------------------------------------------
// topK controls how many similar chunks should be retrieved.
//
// Here:
//
//     topK = 2
//
// This means:
//
//     "Return the top 2 most similar DocumentChunk nodes."
//
// In vector search, topK is an important tuning parameter.
//
// A smaller topK gives:
// - Fewer results
// - More focused output
// - Less noise
//
// A larger topK gives:
// - More candidate chunks
// - Better recall
// - But possibly more irrelevant results
//
// In production RAG systems, topK is usually tuned based on:
// - Dataset size
// - Chunk quality
// - Embedding quality
// - LLM context window size
// - Desired precision versus recall balance

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  topK,
  queryVector
)

// =============================================================================
// RUN VECTOR SIMILARITY SEARCH AGAINST DOCUMENT CHUNKS
// =============================================================================
// This procedure call performs vector similarity search using Neo4j's vector
// index.
//
// The procedure:
//
//     db.index.vector.queryNodes()
//
// searches for nodes whose stored embedding vectors are closest to the input
// query vector.
//
// In simple terms, it asks Neo4j:
//
//     "Find the DocumentChunk nodes whose embeddings are most similar
//      to this login-style query vector."
//
// -----------------------------------------------------------------------------
// First argument: 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// This is the name of the vector index.
//
// The index is expected to be created on DocumentChunk nodes, most likely on
// their embedding property.
//
// The index makes similarity search efficient.
//
// Without a vector index, Neo4j would have to compare the query vector against
// many stored embeddings manually, which would not scale well for larger
// datasets.
//
// -----------------------------------------------------------------------------
// Second argument: topK
// -----------------------------------------------------------------------------
// topK tells the procedure how many nearest matching nodes to return.
//
// Because topK was defined earlier as:
//
//     2
//
// this procedure returns the top 2 most similar DocumentChunk nodes.
//
// This makes the query flexible.
//
// If we later change:
//
//     2 AS topK
//
// to:
//
//     5 AS topK
//
// then the vector search will return 5 chunks without changing this procedure
// call.
//
// -----------------------------------------------------------------------------
// Third argument: queryVector
// -----------------------------------------------------------------------------
// queryVector is the embedding used as the search input.
//
// Because queryVector was defined earlier as:
//
//     [0.92, 0.12, 0.05]
//
// Neo4j searches for chunks whose stored embeddings are close to that vector.
//
// In this demo, that vector is login-oriented.
//
// Production note:
//
// In a real application, queryVector would usually be generated dynamically
// from userQuestion by an embedding model before this Cypher query is executed.

YIELD node AS dc, score

// =============================================================================
// CAPTURE VECTOR SEARCH OUTPUT
// =============================================================================
// YIELD extracts the values returned by db.index.vector.queryNodes().
//
// The vector search procedure returns two important outputs:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the matching graph node returned by the vector index.
//
// We rename node to:
//
//     dc
//
// because the returned node represents a DocumentChunk.
//
// Using dc makes the query easier to understand.
//
// It tells the reader:
//
//     "This variable contains the retrieved document chunk."
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the similarity score between:
//
//     queryVector
//
// and the retrieved DocumentChunk's stored embedding.
//
// A higher score generally means the chunk is more semantically similar to the
// query vector.
//
// In simple terms:
//
//     Higher score = stronger match
//     Lower score  = weaker match
//
// Important production note:
//
// Vector score is useful, but it is not enough by itself.
//
// That is why this query does not stop here.
//
// It continues by connecting the retrieved chunk to:
// - KnowledgeArticle
// - Issue
// - Ticket
// - Customer
// - Product
// - Agent
// - Ticket analytics fields
//
// This makes the retrieval explainable and operationally meaningful.

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// CONNECT RETRIEVED CHUNK TO KNOWLEDGE ARTICLE AND ISSUE
// =============================================================================
// This MATCH clause enriches each retrieved DocumentChunk with graph context.
//
// The pattern is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// Let us break this down.
//
// -----------------------------------------------------------------------------
// (dc)
// -----------------------------------------------------------------------------
// dc is the DocumentChunk returned from the vector search.
//
// It represents a small piece of text that was semantically similar to the
// login-style query vector.
//
// -----------------------------------------------------------------------------
// -[:PART_OF]->
// -----------------------------------------------------------------------------
// PART_OF shows that the retrieved chunk belongs to a larger KnowledgeArticle.
//
// This is important because chunks are usually smaller fragments of a full
// article.
//
// A chunk may contain the relevant sentence or paragraph, but the article gives
// broader source context.
//
// -----------------------------------------------------------------------------
// (ka:KnowledgeArticle)
// -----------------------------------------------------------------------------
// ka represents the parent KnowledgeArticle.
//
// This is the article from which the retrieved chunk came.
//
// Returning the article later helps answer:
//
//     "Which knowledge article supported this retrieval result?"
//
// This is very important for explainable RAG because users should be able to
// trace answers back to source material.
//
// -----------------------------------------------------------------------------
// -[:SOLVES]->
// -----------------------------------------------------------------------------
// SOLVES connects the KnowledgeArticle to the Issue it addresses.
//
// This relationship gives business meaning to the article.
//
// It tells us:
//
//     "This article is intended to solve this issue."
//
// -----------------------------------------------------------------------------
// (i:Issue)
// -----------------------------------------------------------------------------
// i represents the Issue solved by the article.
//
// This is the key explainability step.
//
// Without this MATCH, we only know:
//
//     "This chunk is semantically similar."
//
// With this MATCH, we know:
//
//     "This chunk is semantically similar,
//      it belongs to this article,
//      and that article solves this specific issue."
//
// Production relevance:
//
// This is what makes graph-powered retrieval stronger than plain vector search.
// It connects semantic similarity to structured domain knowledge.

OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

// =============================================================================
// OPTIONALLY FIND TICKETS RELATED TO THE ISSUE
// =============================================================================
// This OPTIONAL MATCH finds Ticket nodes connected to the retrieved Issue.
//
// The pattern is:
//
//     (t:Ticket)-[:HAS_ISSUE]->(i)
//
// Meaning:
//
//     "Find tickets that have this issue."
//
// -----------------------------------------------------------------------------
// Why OPTIONAL MATCH is used
// -----------------------------------------------------------------------------
// OPTIONAL MATCH behaves like a left join in SQL.
//
// It means:
//
//     "Try to find this pattern.
//      If it exists, return the matched data.
//      If it does not exist, keep the existing row and use null."
//
// This is important because not every Issue may have related tickets.
//
// If we used a normal MATCH here, then any retrieved issue without a ticket
// would disappear from the final result.
//
// That would be dangerous because the vector retrieval result may still be
// valid even if operational ticket context is missing.
//
// OPTIONAL MATCH keeps the retrieval result and simply shows null ticket data
// when no tickets exist.
//
// -----------------------------------------------------------------------------
// Why ticket context matters
// -----------------------------------------------------------------------------
// Tickets provide real operational evidence.
//
// They help answer:
// - Has this issue happened in real support cases?
// - How many tickets are related to this issue?
// - What priority were those tickets assigned?
// - Are the tickets open, resolved, or in progress?
// - Is this issue active in the support environment?
//
// This turns semantic retrieval into operational intelligence.

OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)

// =============================================================================
// OPTIONALLY FIND CUSTOMERS WHO RAISED RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Customer nodes connected to the related tickets.
//
// The pattern is:
//
//     (c:Customer)-[:RAISED]->(t)
//
// Meaning:
//
//     "Find customers who raised these tickets."
//
// -----------------------------------------------------------------------------
// Why this is optional
// -----------------------------------------------------------------------------
// A ticket may exist without a connected Customer node.
//
// This can happen in:
// - Demo datasets
// - Incomplete imports
// - Anonymized customer data
// - Partially migrated systems
// - Tickets created by automated monitoring instead of a customer
//
// OPTIONAL MATCH prevents the query from dropping valid ticket or issue rows
// just because customer information is missing.
//
// -----------------------------------------------------------------------------
// Why customer context matters
// -----------------------------------------------------------------------------
// Customer context helps measure business impact.
//
// It helps answer:
// - Which customers are affected?
// - Are multiple customers reporting the same issue?
// - Is one customer repeatedly facing login problems?
// - Should the issue be escalated because of customer impact?
//
// In production support analytics, customer context is often critical for
// prioritization and escalation.

OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)

// =============================================================================
// OPTIONALLY FIND PRODUCTS RELATED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Product nodes connected to the related tickets.
//
// The pattern is:
//
//     (t)-[:ABOUT]->(p:Product)
//
// Meaning:
//
//     "Find the product that this ticket is about."
//
// -----------------------------------------------------------------------------
// Why product context matters
// -----------------------------------------------------------------------------
// Product context helps identify the affected application, service, or business
// area.
//
// It helps answer:
// - Which product is impacted?
// - Is the login issue isolated to one product?
// - Is the same issue appearing across multiple products?
// - Which product team may need to investigate?
//
// For example, a login issue in one product may be local.
// But a login issue across multiple products may indicate a shared
// authentication service problem.
//
// -----------------------------------------------------------------------------
// Why OPTIONAL MATCH is used
// -----------------------------------------------------------------------------
// Not every ticket may have product information.
//
// If product information is missing, we still want to keep the ticket and issue
// context in the final result.

OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

// =============================================================================
// OPTIONALLY FIND AGENTS ASSIGNED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Agent nodes assigned to related tickets.
//
// The pattern is:
//
//     (t)-[:ASSIGNED_TO]->(a:Agent)
//
// Meaning:
//
//     "Find the support agent assigned to this ticket."
//
// -----------------------------------------------------------------------------
// Why agent context matters
// -----------------------------------------------------------------------------
// Agent context helps with ownership and follow-up.
//
// It helps answer:
// - Who is handling the related login tickets?
// - Are multiple agents involved?
// - Is one agent handling many related issues?
// - Is escalation or reassignment needed?
// - Who has prior context if a similar issue appears again?
//
// -----------------------------------------------------------------------------
// Why this is optional
// -----------------------------------------------------------------------------
// Some tickets may not be assigned yet.
//
// If we used a normal MATCH, unassigned tickets would disappear from the result.
//
// OPTIONAL MATCH keeps those tickets visible while showing null for missing
// agent data.

RETURN

// =============================================================================
// FINAL RETURN SHAPE
// =============================================================================
// The RETURN clause defines the final output of the query.
//
// Instead of returning raw graph nodes only, this query returns a clean,
// readable, report-friendly structure.
//
// The output includes:
//
//     1. Original query input
//     2. Retrieved chunk details
//     3. Vector similarity score
//     4. Knowledge article source context
//     5. Issue context
//     6. Related tickets
//     7. Customer impact context
//     8. Product impact context
//     9. Agent ownership context
//    10. Ticket analytics context
//    11. Human-readable operational context status
//
// This structure is useful for:
// - Neo4j Browser result tables
// - Demo walkthroughs
// - RAG explainability output
// - API responses
// - Support dashboards
// - Retrieval quality validation

  userQuestion,

// =============================================================================
// RETURN ORIGINAL USER QUESTION
// =============================================================================
// userQuestion returns the original natural-language question:
//
//     "Why can't customers log in?"
//
// This is useful because the final result remains connected to the user's
// original intent.
//
// In a real RAG or support assistant workflow, this helps with:
// - Debugging
// - Audit trails
// - Evaluation
// - Logging
// - User-facing explainability
//
// Without this field, the output would show retrieved chunks but not the
// original question that caused those chunks to be retrieved.

  queryType,

// =============================================================================
// RETURN QUERY TYPE
// =============================================================================
// queryType returns the category assigned to this query.
//
// Here the value is:
//
//     "login"
//
// This helps classify the retrieval result.
//
// In a larger system, queryType can help compare retrieval performance across
// multiple issue categories such as:
//
//     login
//     payment
//     app-crash
//     performance
//     account-access
//
// It also makes the output easier to filter and analyze.

  queryVector,

// =============================================================================
// RETURN QUERY VECTOR
// =============================================================================
// queryVector returns the numeric embedding used for the vector search.
//
// Here the vector is:
//
//     [0.92, 0.12, 0.05]
//
// Returning the query vector is useful in lab/demo scenarios because it makes
// the retrieval experiment fully visible.
//
// It helps answer:
//
//     "Which exact vector produced these results?"
//
// Production note:
//
// In real systems, you may or may not return the raw vector to end users.
//
// Raw vectors are usually more useful for debugging, evaluation, and internal
// observability than for normal business users.

  topK,

// =============================================================================
// RETURN TOP K VALUE
// =============================================================================
// topK returns the number of nearest chunks requested from the vector index.
//
// Here:
//
//     topK = 2
//
// This tells the reader:
//
//     "The vector search was configured to return the top 2 nearest chunks."
//
// Returning topK is useful for debugging and repeatability.
//
// If retrieval results look too narrow or too broad, topK is one of the first
// parameters to inspect.

  dc.chunkId AS chunkId,

// =============================================================================
// RETURN RETRIEVED CHUNK ID
// =============================================================================
// dc.chunkId returns the unique identifier of the retrieved DocumentChunk.
//
// The alias:
//
//     AS chunkId
//
// makes the output column clean and readable.
//
// This field is important for source traceability.
//
// It allows us to identify exactly which chunk was retrieved by vector search.

  dc.text AS retrievedChunk,

// =============================================================================
// RETURN RETRIEVED CHUNK TEXT
// =============================================================================
// dc.text returns the actual text stored inside the retrieved DocumentChunk.
//
// The alias:
//
//     AS retrievedChunk
//
// makes it clear that this is the text retrieved by semantic vector search.
//
// In a RAG workflow, this text could be used as supporting context for an LLM.
//
// Production note:
//
// Always inspect retrievedChunk during testing.
// A high vector score is useful, but the actual retrieved text should still be
// reviewed to confirm relevance and correctness.

  score AS vectorScore,

// =============================================================================
// RETURN VECTOR SIMILARITY SCORE
// =============================================================================
// score is returned as:
//
//     vectorScore
//
// This value tells us how similar the retrieved chunk embedding was to the query
// embedding.
//
// Higher score generally means stronger semantic similarity.
//
// This helps rank retrieved chunks.
//
// Important reminder:
//
// Vector score is a relevance signal, not absolute truth.
//
// That is why this query also returns article, issue, ticket, customer, product,
// agent, and analytics context.

  ka.articleId AS articleId,

// =============================================================================
// RETURN KNOWLEDGE ARTICLE ID
// =============================================================================
// ka.articleId returns the business identifier of the KnowledgeArticle.
//
// This helps identify the source article from which the retrieved chunk came.
//
// This is useful for:
// - Source traceability
// - Documentation lookup
// - Support references
// - Audit trails
// - Linking back to a knowledge base article

  ka.title AS articleTitle,

// =============================================================================
// RETURN KNOWLEDGE ARTICLE TITLE
// =============================================================================
// ka.title returns the title of the KnowledgeArticle.
//
// The title gives human-readable context for the retrieved chunk.
//
// Instead of only seeing a chunk of text, the user can also see the larger
// article it belongs to.

  i.issueId AS issueId,

// =============================================================================
// RETURN ISSUE ID
// =============================================================================
// i.issueId returns the identifier of the Issue solved by the KnowledgeArticle.
//
// This connects retrieved knowledge content to a known operational issue.
//
// This is important because it moves the result from:
//
//     "Relevant text was found."
//
// to:
//
//     "Relevant text was found for this specific issue."

  i.name AS issueName,

// =============================================================================
// RETURN ISSUE NAME
// =============================================================================
// i.name returns the human-readable name of the Issue.
//
// For a login query, this may represent something like:
//
//     "Login Failure"
//     "Authentication Error"
//     "Account Access Issue"
//
// The exact value depends on your graph data.
//
// This field helps business users quickly understand what problem the retrieved
// knowledge article is solving.

  i.severity AS issueSeverity,

// =============================================================================
// RETURN ISSUE SEVERITY
// =============================================================================
// i.severity returns the severity level of the Issue.
//
// Severity is important because two retrieved chunks may have similar vector
// scores, but their operational priority may be different.
//
// For example:
//
//     A high-severity login outage affecting many customers should be handled
//     differently from a low-severity login UI message.
//
// This is another example of why graph context improves vector search.
//
// Vector search finds semantic similarity.
// Graph context helps determine operational importance.

  collect(DISTINCT t.ticketId) AS relatedTicketIds,

// =============================================================================
// RETURN RELATED TICKET IDS
// =============================================================================
// collect(DISTINCT t.ticketId) gathers all ticket IDs connected to the issue.
//
// Why collect?
//
//     One issue can be connected to many tickets.
//
// If we returned t.ticketId directly, the result could produce many separate
// rows for the same retrieved chunk.
//
// collect() groups those ticket IDs into a list.
//
// -----------------------------------------------------------------------------
// Why DISTINCT?
// -----------------------------------------------------------------------------
// DISTINCT removes duplicate ticket IDs.
//
// Duplicates can appear because multiple OPTIONAL MATCH clauses may multiply
// rows internally.
//
// For example, one ticket connected to one customer, one product, and one agent
// may appear multiple times during graph expansion.
//
// DISTINCT keeps the final list clean.
//
// -----------------------------------------------------------------------------
// Why this field matters
// -----------------------------------------------------------------------------
// relatedTicketIds shows whether the retrieved issue has real operational
// evidence.
//
// It helps answer:
//
//     "Which support tickets are connected to this login-related issue?"

  collect(DISTINCT c.name) AS relatedCustomers,

// =============================================================================
// RETURN RELATED CUSTOMERS
// =============================================================================
// collect(DISTINCT c.name) gathers the names of customers who raised related
// tickets.
//
// This provides customer impact context.
//
// Why collect?
//
//     Multiple customers may have raised tickets for the same issue.
//
// Why DISTINCT?
//
//     The same customer may appear more than once because of row expansion.
//
// This field helps answer:
// - Which customers are affected?
// - Are many customers reporting login problems?
// - Is one customer repeatedly affected?
// - Should this issue be escalated because of customer impact?

  collect(DISTINCT p.name) AS relatedProducts,

// =============================================================================
// RETURN RELATED PRODUCTS
// =============================================================================
// collect(DISTINCT p.name) gathers product names associated with related tickets.
//
// This provides product impact context.
//
// It helps answer:
// - Which product is affected?
// - Is the login issue limited to one product?
// - Is the same issue affecting multiple products?
// - Which product team may need to investigate?

  collect(DISTINCT a.name) AS assignedAgents,

// =============================================================================
// RETURN ASSIGNED AGENTS
// =============================================================================
// collect(DISTINCT a.name) gathers the names of agents assigned to related
// tickets.
//
// This provides operational ownership context.
//
// It helps answer:
// - Who is working on the related login tickets?
// - Are multiple agents involved?
// - Is one agent handling many related tickets?
// - Is escalation or coordination needed?

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

// =============================================================================
// RETURN STRUCTURED TICKET ANALYTICS CONTEXT
// =============================================================================
// This expression returns a list of structured ticket analytics maps.
//
// The overall expression is:
//
//     collect(DISTINCT CASE ... END) AS ticketAnalyticsContext
//
// Let us break it down.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT ...)
// -----------------------------------------------------------------------------
// collect() groups multiple ticket analytics objects into a list.
//
// DISTINCT removes duplicate analytics maps that may appear because of optional
// graph expansion.
//
// This is useful because one retrieved issue may have many related tickets.
//
// -----------------------------------------------------------------------------
// CASE WHEN t IS NULL THEN null
// -----------------------------------------------------------------------------
// Because t comes from an OPTIONAL MATCH, it may be null.
//
// That means there may be no related ticket for the issue.
//
// The CASE expression handles that safely.
//
// If no ticket exists:
//
//     WHEN t IS NULL THEN null
//
// This prevents the query from creating a misleading ticket map with all fields
// set to null.
//
// -----------------------------------------------------------------------------
// ELSE { ... }
// -----------------------------------------------------------------------------
// If a ticket exists, the query creates a map containing important ticket
// details and graph analytics scores.
//
// A map is a structured object with key-value pairs.
//
// For example:
//
//     {
//       ticketId: "T-1001",
//       priority: "High",
//       status: "Open"
//     }
//
// This is cleaner than returning many separate repeated columns.
//
// -----------------------------------------------------------------------------
// ticketId
// -----------------------------------------------------------------------------
// ticketId identifies the related ticket.
//
// -----------------------------------------------------------------------------
// priority
// -----------------------------------------------------------------------------
// priority indicates how urgent or important the ticket is from an operational
// support perspective.
//
// -----------------------------------------------------------------------------
// status
// -----------------------------------------------------------------------------
// status tells whether the ticket is open, closed, in progress, resolved, etc.
//
// -----------------------------------------------------------------------------
// fullDegreeScore
// -----------------------------------------------------------------------------
// fullDegreeScore usually represents how connected the ticket is in the graph.
//
// A higher degree score can mean the ticket is connected to many other entities,
// such as customers, products, agents, or issues.
//
// -----------------------------------------------------------------------------
// pageRankScore
// -----------------------------------------------------------------------------
// pageRankScore usually represents graph-based importance or influence.
//
// A ticket with a high PageRank score may be connected to other important nodes.
//
// -----------------------------------------------------------------------------
// betweennessScore
// -----------------------------------------------------------------------------
// betweennessScore usually identifies bridge-like nodes.
//
// A ticket with high betweenness may connect different communities or parts of
// the graph.
//
// This can be useful for finding tickets that explain relationships between
// otherwise separate issue clusters.
//
// -----------------------------------------------------------------------------
// louvainCommunityId
// -----------------------------------------------------------------------------
// louvainCommunityId identifies the community detected by the Louvain algorithm.
//
// Community IDs help group related nodes together based on graph structure.
//
// -----------------------------------------------------------------------------
// labelPropagationCommunityId
// -----------------------------------------------------------------------------
// labelPropagationCommunityId identifies the community detected by the Label
// Propagation algorithm.
//
// This gives another way to understand clusters of related graph entities.
//
// -----------------------------------------------------------------------------
// Production relevance
// -----------------------------------------------------------------------------
// This ticketAnalyticsContext field is useful for prioritization and
// explainability.
//
// For example, two retrieved chunks may have similar vector scores.
//
// But if one chunk connects to an issue whose tickets have high PageRank,
// high betweenness, high degree, and high priority, that issue may deserve more
// attention.

  CASE
    WHEN count(t) = 0 THEN "No related ticket found for this issue"
    ELSE "Related ticket context found"
  END AS operationalContextStatus

// =============================================================================
// RETURN HUMAN-READABLE OPERATIONAL CONTEXT STATUS
// =============================================================================
// This CASE expression creates a simple human-readable status message.
//
// It checks:
//
//     count(t) = 0
//
// Meaning:
//
//     "Were any related Ticket nodes found?"
//
// -----------------------------------------------------------------------------
// WHEN count(t) = 0
// -----------------------------------------------------------------------------
// If no tickets were found, return:
//
//     "No related ticket found for this issue"
//
// This means the vector result found a chunk, article, and issue, but no
// operational ticket context exists for that issue.
//
// -----------------------------------------------------------------------------
// ELSE
// -----------------------------------------------------------------------------
// If one or more tickets were found, return:
//
//     "Related ticket context found"
//
// This means the retrieved issue is connected to real ticket data.
//
// -----------------------------------------------------------------------------
// Why this field is useful
// -----------------------------------------------------------------------------
// This field makes the result easier to read during demos and troubleshooting.
//
// Instead of manually checking whether relatedTicketIds is empty, the query
// directly tells us whether operational context was found.
//
// Production note:
//
// Derived status fields like this are useful for dashboards and API consumers
// because they convert raw graph data into an immediately understandable message.

ORDER BY
  vectorScore DESC;

// =============================================================================
// ORDER FINAL RESULTS BY VECTOR SCORE
// =============================================================================
// ORDER BY controls the order of rows in the final output.
//
// This query orders by:
//
//     vectorScore DESC
//
// -----------------------------------------------------------------------------
// vectorScore
// -----------------------------------------------------------------------------
// vectorScore is the similarity score from the vector search.
//
// It tells us how close the retrieved DocumentChunk was to the login-style query
// vector.
//
// -----------------------------------------------------------------------------
// DESC
// -----------------------------------------------------------------------------
// DESC means descending order.
//
// So the highest vectorScore appears first.
//
// This is important because the strongest semantic match should usually be
// reviewed before weaker matches.
//
// -----------------------------------------------------------------------------
// Final behavior
// -----------------------------------------------------------------------------
// The final output shows the top login-related retrieved chunks ranked by
// semantic similarity.
//
// Each result is enriched with:
//
// - Original user question
// - Query type
// - Query vector
// - topK value
// - Retrieved chunk text
// - Vector score
// - Knowledge article source
// - Issue details
// - Related tickets
// - Related customers
// - Related products
// - Assigned agents
// - Ticket analytics context
// - Operational context status
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a focused login-style graph-enriched vector retrieval workflow.
//
// The main purpose is:
//
//     "Use a login-related query vector to retrieve relevant document chunks,
//      then explain those chunks through the connected support knowledge graph."
//
// It combines:
//
// - Natural-language question tracking
// - Query classification
// - Vector similarity search
// - Knowledge article traceability
// - Issue-level explanation
// - Ticket-level operational evidence
// - Customer impact context
// - Product impact context
// - Agent ownership context
// - Graph analytics context
//
// This is much closer to a production-ready explainable RAG pattern than simple
// vector search alone.
```

# Step 3 — Add controlled questions for all three retrieval scenarios

```cypher
// =============================================================================
// MULTI-REQUEST EXPLAINABLE VECTOR RETRIEVAL WITH GRAPH CONTEXT ENRICHMENT
// =============================================================================
// This query performs multiple vector similarity searches in one execution.
//
// In simple terms, it tells Neo4j:
//
//     "Take several user-style questions,
//      run vector search for each question,
//      retrieve the most relevant document chunks,
//      then explain each result using the surrounding knowledge graph."
//
// This query is an improved version of running three separate vector searches.
//
// Instead of writing three different queries for:
//
//     1. Login problems
//     2. Payment failures
//     3. App crashes
//
// we define all three retrieval requests inside one list and use UNWIND to
// process them one by one.
//
// This is useful because it demonstrates a scalable retrieval pattern.
//
// In production, a similar structure could be used when:
// - Testing multiple retrieval scenarios
// - Benchmarking vector search quality
// - Running batch evaluation queries
// - Comparing query categories
// - Building explainable RAG validation workflows
// - Enriching semantic search results with graph context
//
// The query combines:
//
// - A natural-language user question
// - A query type/category
// - A query embedding vector
// - A topK retrieval limit
// - Vector similarity search
// - Knowledge article source traceability
// - Issue-level explanation
// - Ticket-level operational evidence
// - Customer, product, and agent context
// - Graph analytics scores
// - A human-readable operational status message

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

// =============================================================================
// DEFINE MULTIPLE RETRIEVAL REQUESTS USING UNWIND
// =============================================================================
// UNWIND takes a list and expands it into individual rows.
//
// Here, the list contains three maps.
//
// Each map represents one vector retrieval request.
//
// In simple terms, this section says:
//
//     "Create three retrieval requests,
//      then process each request as a separate row."
//
// After UNWIND runs, Neo4j will process the query as if it had three input rows:
//
//     Row 1 -> login request
//     Row 2 -> payment request
//     Row 3 -> app_crash request
//
// Each row is stored in the variable:
//
//     request
//
// -----------------------------------------------------------------------------
// Why use UNWIND here?
// -----------------------------------------------------------------------------
// Without UNWIND, we might need to write three separate vector search queries
// or use multiple UNION ALL branches.
//
// UNWIND makes the query cleaner and more scalable.
//
// If we want to test another query type later, such as:
//
//     "performance"
//     "refund"
//     "account_locked"
//
// we only need to add another map to the list.
//
// The rest of the query remains unchanged.
//
// -----------------------------------------------------------------------------
// Why use maps inside the list?
// -----------------------------------------------------------------------------
// Each request is written as a map because a map allows related values to stay
// grouped together.
//
// Each request contains:
//
//     queryType
//     userQuestion
//     queryVector
//     topK
//
// This is cleaner than managing separate lists for query types, questions,
// vectors, and topK values.
//
// -----------------------------------------------------------------------------
// request.queryType
// -----------------------------------------------------------------------------
// queryType classifies the retrieval scenario.
//
// In this query, there are three query types:
//
//     login
//     payment
//     app_crash
//
// This value helps identify which scenario produced each final result.
//
// -----------------------------------------------------------------------------
// request.userQuestion
// -----------------------------------------------------------------------------
// userQuestion stores the original natural-language question.
//
// Examples:
//
//     "Why can't customers log in?"
//     "Why are payments failing?"
//     "Why does the app crash?"
//
// Returning this value later helps preserve traceability between the user's
// question and the retrieved results.
//
// -----------------------------------------------------------------------------
// request.queryVector
// -----------------------------------------------------------------------------
// queryVector stores the numeric embedding used for vector search.
//
// In a real production system, this vector would usually be generated by an
// embedding model from the userQuestion.
//
// For example:
//
//     User question:
//       "Why are payments failing?"
//
//     Embedding model:
//       Converts the question into a vector.
//
//     Neo4j vector index:
//       Uses that vector to find similar document chunks.
//
// In this lab/demo query, the vectors are hardcoded so the retrieval behavior is
// predictable and easy to explain.
//
// -----------------------------------------------------------------------------
// request.topK
// -----------------------------------------------------------------------------
// topK controls how many nearest document chunks should be returned for each
// request.
//
// Here, every request uses:
//
//     topK: 2
//
// This means Neo4j will retrieve the top 2 most similar chunks for each query
// type.
//
// Since there are three requests and each asks for top 2 results, the vector
// search stage can produce up to:
//
//     3 query types * 2 chunks each = 6 retrieved chunk rows
//
// before graph expansion and aggregation.

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)

// =============================================================================
// RUN VECTOR SIMILARITY SEARCH FOR EACH REQUEST
// =============================================================================
// This procedure performs vector similarity search using Neo4j's vector index.
//
// The procedure:
//
//     db.index.vector.queryNodes()
//
// searches for nodes whose stored embedding vectors are closest to the input
// query vector.
//
// Because this procedure is executed after UNWIND, it runs once for each
// request row.
//
// So Neo4j effectively runs vector search for:
//
//     1. login query vector
//     2. payment query vector
//     3. app_crash query vector
//
// -----------------------------------------------------------------------------
// First argument: 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// This is the name of the vector index.
//
// The index is expected to be created on DocumentChunk nodes, most likely on
// their embedding property.
//
// In simple terms:
//
//     "Use the vector index built for document chunk embeddings."
//
// The index makes similarity search efficient.
//
// Without a vector index, Neo4j would need to compare vectors manually across
// many nodes, which would not scale well.
//
// -----------------------------------------------------------------------------
// Second argument: request.topK
// -----------------------------------------------------------------------------
// request.topK tells Neo4j how many nearest matching chunks to return for the
// current request.
//
// For all three requests in this query:
//
//     request.topK = 2
//
// So each query type retrieves the two most similar DocumentChunk nodes.
//
// In production, topK is an important tuning parameter.
//
// A smaller topK gives:
// - More focused results
// - Less noise
// - Lower downstream processing cost
//
// A larger topK gives:
// - Broader recall
// - More candidate chunks
// - Higher chance of including weaker matches
//
// -----------------------------------------------------------------------------
// Third argument: request.queryVector
// -----------------------------------------------------------------------------
// request.queryVector is the embedding vector for the current request.
//
// For example:
//
//     login     -> [0.92, 0.12, 0.05]
//     payment   -> [0.08, 0.92, 0.12]
//     app_crash -> [0.12, 0.10, 0.92]
//
// These vectors are numeric representations of meaning.
//
// Important beginner note:
//
//     Neo4j is not directly comparing the English question text here.
//
// It is comparing the request.queryVector against stored chunk embeddings.
//
// In a real RAG system, the natural-language question would first be converted
// into a vector by an embedding model.

YIELD node AS dc, score

// =============================================================================
// CAPTURE VECTOR SEARCH OUTPUT
// =============================================================================
// YIELD extracts the values returned by db.index.vector.queryNodes().
//
// The vector search procedure returns two important outputs:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the matching graph node returned by the vector index.
//
// We rename node to:
//
//     dc
//
// because the returned node represents a DocumentChunk.
//
// Using the name dc makes the query easier to read because it clearly means:
//
//     "document chunk"
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the similarity score between:
//
//     request.queryVector
//
// and the retrieved DocumentChunk's stored embedding.
//
// A higher score generally means a stronger semantic match.
//
// In simple terms:
//
//     Higher score = more relevant chunk
//     Lower score  = weaker match
//
// Production note:
//
// Vector score is useful, but vector score alone is not enough for strong
// explainability.
//
// That is why this query continues by joining each retrieved chunk to:
// - KnowledgeArticle
// - Issue
// - Ticket
// - Customer
// - Product
// - Agent
// - Ticket analytics fields

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// CONNECT RETRIEVED DOCUMENT CHUNK TO KNOWLEDGE ARTICLE AND ISSUE
// =============================================================================
// This MATCH clause enriches each retrieved DocumentChunk with source and issue
// context.
//
// The pattern is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// Let us break this down.
//
// -----------------------------------------------------------------------------
// (dc)
// -----------------------------------------------------------------------------
// dc is the DocumentChunk retrieved by vector search.
//
// It is a small piece of text that was semantically similar to the current
// request.queryVector.
//
// -----------------------------------------------------------------------------
// -[:PART_OF]->
// -----------------------------------------------------------------------------
// PART_OF shows that the retrieved chunk belongs to a larger KnowledgeArticle.
//
// This is important because chunks are usually created by splitting larger
// documents or articles into smaller pieces.
//
// A chunk may be useful for retrieval, but the KnowledgeArticle gives it source
// traceability.
//
// -----------------------------------------------------------------------------
// (ka:KnowledgeArticle)
// -----------------------------------------------------------------------------
// ka represents the parent KnowledgeArticle.
//
// This is the article from which the retrieved chunk came.
//
// Returning article information later helps answer:
//
//     "Which knowledge article supported this retrieved result?"
//
// This is very important for explainable RAG because a user should be able to
// trace an answer back to its source material.
//
// -----------------------------------------------------------------------------
// -[:SOLVES]->
// -----------------------------------------------------------------------------
// SOLVES connects a KnowledgeArticle to the Issue it addresses.
//
// This relationship tells us that the article is intended to solve or explain a
// specific operational problem.
//
// -----------------------------------------------------------------------------
// (i:Issue)
// -----------------------------------------------------------------------------
// i represents the Issue solved by the KnowledgeArticle.
//
// This is the key explainability step.
//
// Without this graph match, we only know:
//
//     "This chunk is similar to the query vector."
//
// With this graph match, we know:
//
//     "This chunk is similar,
//      it belongs to this knowledge article,
//      and that article solves this issue."
//
// Production relevance:
//
// This turns plain semantic retrieval into graph-enriched retrieval.
// It connects machine-learned similarity with structured business meaning.

OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

// =============================================================================
// OPTIONALLY FIND TICKETS RELATED TO THE ISSUE
// =============================================================================
// This OPTIONAL MATCH finds Ticket nodes connected to the retrieved Issue.
//
// The pattern is:
//
//     (t:Ticket)-[:HAS_ISSUE]->(i)
//
// Meaning:
//
//     "Find tickets that have this issue."
//
// -----------------------------------------------------------------------------
// Why OPTIONAL MATCH is used
// -----------------------------------------------------------------------------
// OPTIONAL MATCH behaves like a left join in SQL.
//
// It means:
//
//     "Try to find this pattern.
//      If it exists, return the matching data.
//      If it does not exist, keep the existing row and use null."
//
// This is important because not every Issue is guaranteed to have related
// tickets.
//
// If we used a normal MATCH here, any retrieved issue without tickets would be
// removed from the result set.
//
// That would be wrong for retrieval analysis because a vector result can still
// be valid even if operational ticket context is missing.
//
// OPTIONAL MATCH keeps the retrieved chunk, article, and issue visible.
//
// -----------------------------------------------------------------------------
// Why ticket context matters
// -----------------------------------------------------------------------------
// Tickets provide real operational evidence.
//
// They help answer:
// - Has this issue occurred in actual support cases?
// - How many tickets are connected to this issue?
// - What priority do those tickets have?
// - Are the tickets open, resolved, or in progress?
// - Is the issue active in the support environment?
//
// This makes retrieval results operationally meaningful.

OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)

// =============================================================================
// OPTIONALLY FIND CUSTOMERS WHO RAISED RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Customer nodes connected to related tickets.
//
// The pattern is:
//
//     (c:Customer)-[:RAISED]->(t)
//
// Meaning:
//
//     "Find customers who raised these tickets."
//
// -----------------------------------------------------------------------------
// Why this is optional
// -----------------------------------------------------------------------------
// A ticket may exist without a connected Customer node.
//
// This can happen in:
// - Demo datasets
// - Partial imports
// - Anonymized support data
// - Migrated systems with missing relationships
// - Tickets created by internal monitoring instead of customers
//
// OPTIONAL MATCH prevents valid ticket or issue rows from disappearing when
// customer information is missing.
//
// -----------------------------------------------------------------------------
// Why customer context matters
// -----------------------------------------------------------------------------
// Customer context helps measure business impact.
//
// It helps answer:
// - Which customers are affected?
// - Are multiple customers reporting the same issue?
// - Is one customer repeatedly affected?
// - Should the issue be escalated because of customer impact?
//
// In production support workflows, customer impact is often a major
// prioritization factor.

OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)

// =============================================================================
// OPTIONALLY FIND PRODUCTS RELATED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Product nodes connected to the related tickets.
//
// The pattern is:
//
//     (t)-[:ABOUT]->(p:Product)
//
// Meaning:
//
//     "Find the product that this ticket is about."
//
// -----------------------------------------------------------------------------
// Why product context matters
// -----------------------------------------------------------------------------
// Product context identifies the affected application, service, or business
// capability.
//
// It helps answer:
// - Which product is impacted?
// - Is the issue limited to one product?
// - Does the same issue affect multiple products?
// - Which product team should investigate?
//
// For example:
//
//     A login problem affecting one product may be product-specific.
//     A login problem affecting many products may indicate a shared
//     authentication service issue.
//
// -----------------------------------------------------------------------------
// Why OPTIONAL MATCH is used
// -----------------------------------------------------------------------------
// Some tickets may not have product relationships.
//
// If product data is missing, we still want to keep the related ticket and issue
// context in the output.

OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

// =============================================================================
// OPTIONALLY FIND AGENTS ASSIGNED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Agent nodes assigned to related tickets.
//
// The pattern is:
//
//     (t)-[:ASSIGNED_TO]->(a:Agent)
//
// Meaning:
//
//     "Find the support agent assigned to this ticket."
//
// -----------------------------------------------------------------------------
// Why agent context matters
// -----------------------------------------------------------------------------
// Agent context helps with ownership and operational follow-up.
//
// It helps answer:
// - Who is handling the related ticket?
// - Are multiple agents involved?
// - Is one agent handling many similar tickets?
// - Who has prior context for this issue?
// - Is reassignment or escalation needed?
//
// -----------------------------------------------------------------------------
// Why this is optional
// -----------------------------------------------------------------------------
// Some tickets may not be assigned yet.
//
// If we used a normal MATCH, unassigned tickets would disappear from the result.
//
// OPTIONAL MATCH keeps those tickets visible while showing null for missing
// agent information.

RETURN

// =============================================================================
// FINAL RETURN SHAPE
// =============================================================================
// The RETURN clause defines the final output columns.
//
// Instead of returning raw nodes only, this query returns a clean,
// report-friendly structure.
//
// The output includes:
//
//     1. Query request metadata
//     2. Retrieved chunk details
//     3. Vector similarity score
//     4. Knowledge article source context
//     5. Issue context
//     6. Related ticket IDs
//     7. Customer impact context
//     8. Product impact context
//     9. Agent ownership context
//    10. Ticket analytics context
//    11. Human-readable operational context status
//
// This output shape is useful for:
// - Neo4j Browser demos
// - Retrieval quality testing
// - RAG explainability reports
// - Support dashboards
// - API responses
// - Graph-based customer support intelligence

  request.queryType AS queryType,

// =============================================================================
// RETURN QUERY TYPE
// =============================================================================
// request.queryType returns the category of the current retrieval request.
//
// Example values are:
//
//     login
//     payment
//     app_crash
//
// The alias:
//
//     AS queryType
//
// makes the output column clean and easy to read.
//
// This field is important because all request types are processed together in
// one query.
//
// Without queryType, the final result table would mix login, payment, and
// app-crash results without clearly showing which scenario produced each row.

  request.userQuestion AS userQuestion,

// =============================================================================
// RETURN ORIGINAL USER QUESTION
// =============================================================================
// request.userQuestion returns the natural-language question for the current
// request.
//
// Example values are:
//
//     "Why can't customers log in?"
//     "Why are payments failing?"
//     "Why does the app crash?"
//
// This keeps the final result traceable to the original user intent.
//
// In production RAG systems, this is useful for:
// - Debugging
// - Logging
// - Evaluation
// - Audit trails
// - Explaining why certain chunks were retrieved

  request.queryVector AS queryVector,

// =============================================================================
// RETURN QUERY VECTOR
// =============================================================================
// request.queryVector returns the numeric embedding used for the vector search.
//
// Returning the vector is useful in lab/demo scenarios because it makes the
// experiment fully visible.
//
// It helps answer:
//
//     "Which exact vector produced this retrieval result?"
//
// Production note:
//
// In real user-facing systems, raw vectors are usually more useful for internal
// debugging and evaluation than for business users.
//
// However, keeping them in technical demos is helpful because it makes the
// retrieval pipeline transparent.

  request.topK AS topK,

// =============================================================================
// RETURN TOP K VALUE
// =============================================================================
// request.topK returns the number of nearest chunks requested for the current
// vector search.
//
// In this query, every request uses:
//
//     topK = 2
//
// This means each query type asks for the top 2 matching chunks.
//
// Returning topK improves repeatability.
//
// If someone reviews the results later, they can immediately see how many
// matches were requested for each scenario.

  dc.chunkId AS chunkId,

// =============================================================================
// RETURN RETRIEVED CHUNK ID
// =============================================================================
// dc.chunkId returns the unique identifier of the retrieved DocumentChunk.
//
// The alias:
//
//     AS chunkId
//
// makes the output column easier to read.
//
// This field is important for source traceability.
//
// It allows us to identify exactly which chunk was retrieved by vector search.

  dc.text AS retrievedChunk,

// =============================================================================
// RETURN RETRIEVED CHUNK TEXT
// =============================================================================
// dc.text returns the actual text stored in the retrieved DocumentChunk.
//
// The alias:
//
//     AS retrievedChunk
//
// makes it clear that this is the text retrieved by semantic vector search.
//
// In a RAG workflow, this text could be passed to an LLM as supporting context.
//
// Production note:
//
// Always inspect retrievedChunk during testing.
// A high vector score is helpful, but the actual retrieved text should still be
// reviewed for relevance, correctness, and usefulness.

  score AS vectorScore,

// =============================================================================
// RETURN VECTOR SIMILARITY SCORE
// =============================================================================
// score is returned as:
//
//     vectorScore
//
// This value tells us how similar the retrieved chunk embedding was to the
// current request.queryVector.
//
// Higher score generally means stronger semantic similarity.
//
// This helps rank results inside each query type.
//
// Important reminder:
//
// Vector score is a relevance signal, not absolute truth.
//
// That is why this query also returns graph context such as article, issue,
// ticket, customer, product, agent, and analytics information.

  ka.articleId AS articleId,

// =============================================================================
// RETURN KNOWLEDGE ARTICLE ID
// =============================================================================
// ka.articleId returns the business identifier of the KnowledgeArticle.
//
// This identifies the parent article from which the retrieved chunk came.
//
// This is useful for:
// - Source traceability
// - Documentation lookup
// - Support references
// - Audit trails
// - Linking back to a knowledge base article

  ka.title AS articleTitle,

// =============================================================================
// RETURN KNOWLEDGE ARTICLE TITLE
// =============================================================================
// ka.title returns the title of the KnowledgeArticle.
//
// The title provides human-readable source context.
//
// Instead of only seeing a retrieved chunk, the user can also see the larger
// article that chunk belongs to.

  i.issueId AS issueId,

// =============================================================================
// RETURN ISSUE ID
// =============================================================================
// i.issueId returns the identifier of the Issue solved by the KnowledgeArticle.
//
// This connects the retrieved knowledge content to a known operational issue.
//
// It moves the result from:
//
//     "Relevant text was found."
//
// to:
//
//     "Relevant text was found for this specific issue."

  i.name AS issueName,

// =============================================================================
// RETURN ISSUE NAME
// =============================================================================
// i.name returns the human-readable name of the Issue.
//
// Depending on the query type, issue names may represent concepts such as:
//
//     "Login Failure"
//     "Payment Declined"
//     "Mobile App Crash"
//
// The exact values depend on your graph data.
//
// This field helps users quickly understand what problem the retrieved
// knowledge article is solving.

  i.severity AS issueSeverity,

// =============================================================================
// RETURN ISSUE SEVERITY
// =============================================================================
// i.severity returns the severity level of the Issue.
//
// Severity is important because two chunks may have similar vector scores, but
// their operational priorities may be very different.
//
// For example:
//
//     A high-severity payment outage may require faster attention than a
//     low-severity informational login warning.
//
// This is one reason graph context improves vector retrieval.
//
// Vector search finds semantic similarity.
// Graph context helps evaluate operational importance.

  collect(DISTINCT t.ticketId) AS relatedTicketIds,

// =============================================================================
// RETURN RELATED TICKET IDS
// =============================================================================
// collect(DISTINCT t.ticketId) gathers all ticket IDs connected to the issue.
//
// Why collect?
//
//     One issue can be connected to many tickets.
//
// If we returned t.ticketId directly, the result could produce many rows for
// the same retrieved chunk.
//
// collect() groups those ticket IDs into a list.
//
// -----------------------------------------------------------------------------
// Why DISTINCT?
// -----------------------------------------------------------------------------
// DISTINCT removes duplicate ticket IDs.
//
// Duplicates can appear because multiple OPTIONAL MATCH clauses can multiply
// rows internally.
//
// For example, one ticket connected to one customer, one product, and one agent
// may appear multiple times during graph expansion.
//
// DISTINCT keeps the final list clean.
//
// -----------------------------------------------------------------------------
// Why this field matters
// -----------------------------------------------------------------------------
// relatedTicketIds shows whether the retrieved issue has real operational
// evidence.
//
// It helps answer:
//
//     "Which support tickets are connected to this retrieved issue?"

  collect(DISTINCT c.name) AS relatedCustomers,

// =============================================================================
// RETURN RELATED CUSTOMERS
// =============================================================================
// collect(DISTINCT c.name) gathers customer names connected to related tickets.
//
// This provides customer impact context.
//
// Why collect?
//
//     Multiple customers may have raised tickets for the same issue.
//
// Why DISTINCT?
//
//     The same customer may appear more than once because of row expansion.
//
// This field helps answer:
// - Which customers are affected?
// - Are many customers reporting the same issue?
// - Is one customer repeatedly affected?
// - Should this issue be escalated because of customer impact?

  collect(DISTINCT p.name) AS relatedProducts,

// =============================================================================
// RETURN RELATED PRODUCTS
// =============================================================================
// collect(DISTINCT p.name) gathers product names associated with related tickets.
//
// This provides product impact context.
//
// It helps answer:
// - Which product is affected?
// - Is the issue limited to one product?
// - Is the same issue affecting multiple products?
// - Which product team should investigate?

  collect(DISTINCT a.name) AS assignedAgents,

// =============================================================================
// RETURN ASSIGNED AGENTS
// =============================================================================
// collect(DISTINCT a.name) gathers names of agents assigned to related tickets.
//
// This provides operational ownership context.
//
// It helps answer:
// - Who is working on the related tickets?
// - Are multiple agents involved?
// - Is one agent handling many similar tickets?
// - Is escalation or coordination needed?

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

// =============================================================================
// RETURN STRUCTURED TICKET ANALYTICS CONTEXT
// =============================================================================
// This expression returns a list of structured ticket analytics maps.
//
// The overall expression is:
//
//     collect(DISTINCT CASE ... END) AS ticketAnalyticsContext
//
// Let us break this down carefully.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT ...)
// -----------------------------------------------------------------------------
// collect() groups multiple ticket analytics objects into a list.
//
// DISTINCT removes duplicate maps that may appear due to optional graph
// expansion.
//
// This is useful because one retrieved issue may have many related tickets.
//
// -----------------------------------------------------------------------------
// CASE WHEN t IS NULL THEN null
// -----------------------------------------------------------------------------
// Because t comes from an OPTIONAL MATCH, it may be null.
//
// That means there may be no related ticket for the issue.
//
// The CASE expression handles that safely.
//
// If no ticket exists:
//
//     WHEN t IS NULL THEN null
//
// This prevents the query from creating a misleading ticket analytics map with
// all fields set to null.
//
// -----------------------------------------------------------------------------
// ELSE { ... }
// -----------------------------------------------------------------------------
// If a ticket exists, the query creates a map containing useful ticket details
// and graph analytics scores.
//
// A map is a structured object with key-value pairs.
//
// For example:
//
//     {
//       ticketId: "T-1001",
//       priority: "High",
//       status: "Open"
//     }
//
// This is cleaner than returning many repeated columns.
//
// -----------------------------------------------------------------------------
// ticketId
// -----------------------------------------------------------------------------
// ticketId identifies the related ticket.
//
// -----------------------------------------------------------------------------
// priority
// -----------------------------------------------------------------------------
// priority indicates how urgent or important the ticket is from an operational
// support perspective.
//
// -----------------------------------------------------------------------------
// status
// -----------------------------------------------------------------------------
// status tells whether the ticket is open, closed, in progress, resolved, etc.
//
// -----------------------------------------------------------------------------
// fullDegreeScore
// -----------------------------------------------------------------------------
// fullDegreeScore usually represents how connected the ticket is in the graph.
//
// A higher degree score can mean the ticket is connected to many other entities,
// such as customers, products, agents, or issues.
//
// -----------------------------------------------------------------------------
// pageRankScore
// -----------------------------------------------------------------------------
// pageRankScore usually represents graph-based importance or influence.
//
// A ticket with a high PageRank score may be connected to other important nodes.
//
// -----------------------------------------------------------------------------
// betweennessScore
// -----------------------------------------------------------------------------
// betweennessScore usually identifies bridge-like nodes.
//
// A ticket with high betweenness may connect different communities or parts of
// the graph.
//
// This can help identify tickets that act as important connectors between
// otherwise separate issue clusters.
//
// -----------------------------------------------------------------------------
// louvainCommunityId
// -----------------------------------------------------------------------------
// louvainCommunityId identifies the community detected by the Louvain algorithm.
//
// Community IDs help group related nodes based on graph structure.
//
// -----------------------------------------------------------------------------
// labelPropagationCommunityId
// -----------------------------------------------------------------------------
// labelPropagationCommunityId identifies the community detected by the Label
// Propagation algorithm.
//
// This gives another way to understand clusters of related graph entities.
//
// -----------------------------------------------------------------------------
// Production relevance
// -----------------------------------------------------------------------------
// ticketAnalyticsContext is useful for prioritization and explainability.
//
// For example, two retrieved chunks may have similar vector scores.
//
// But if one connects to tickets with higher priority, higher PageRank, higher
// betweenness, or large community membership, that result may deserve more
// operational attention.

  CASE
    WHEN count(t) = 0 THEN "No related ticket found for this issue"
    ELSE "Related ticket context found"
  END AS operationalContextStatus

// =============================================================================
// RETURN HUMAN-READABLE OPERATIONAL CONTEXT STATUS
// =============================================================================
// This CASE expression creates a simple status message.
//
// It checks:
//
//     count(t) = 0
//
// Meaning:
//
//     "Were any related Ticket nodes found for this issue?"
//
// -----------------------------------------------------------------------------
// WHEN count(t) = 0
// -----------------------------------------------------------------------------
// If no tickets were found, return:
//
//     "No related ticket found for this issue"
//
// This means the vector result found a chunk, article, and issue, but no
// operational ticket context exists for that issue.
//
// -----------------------------------------------------------------------------
// ELSE
// -----------------------------------------------------------------------------
// If one or more tickets were found, return:
//
//     "Related ticket context found"
//
// This means the retrieved issue is connected to real ticket data.
//
// -----------------------------------------------------------------------------
// Why this field is useful
// -----------------------------------------------------------------------------
// This field makes the result easier to read during demos and troubleshooting.
//
// Instead of manually checking whether relatedTicketIds is empty, the output
// directly tells us whether operational context exists.
//
// Production note:
//
// Derived status fields like this are useful for dashboards and API consumers
// because they translate raw graph data into an immediately understandable
// message.

ORDER BY
  queryType,
  vectorScore DESC;

// =============================================================================
// ORDER FINAL RESULTS BY QUERY TYPE AND VECTOR SCORE
// =============================================================================
// ORDER BY controls the order of rows in the final output.
//
// This query orders by:
//
//     queryType
//     vectorScore DESC
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// Ordering by queryType groups results from the same request category together.
//
// This means:
//
//     app_crash results appear together
//     login results appear together
//     payment results appear together
//
// The exact alphabetical order depends on the queryType values.
//
// Grouping by queryType makes the result table easier to read and compare.
//
// -----------------------------------------------------------------------------
// vectorScore DESC
// -----------------------------------------------------------------------------
// Within each queryType group, results are ordered by vectorScore in descending
// order.
//
// DESC means:
//
//     highest score first
//
// This ensures that the strongest semantic match for each query type appears
// before weaker matches.
//
// -----------------------------------------------------------------------------
// Final behavior
// -----------------------------------------------------------------------------
// The final output is grouped by retrieval scenario and ranked by semantic
// similarity inside each scenario.
//
// This gives a clean explainable retrieval result:
//
//     query type
//       -> original question
//       -> best matching chunks
//       -> source article
//       -> solved issue
//       -> related tickets
//       -> customers/products/agents
//       -> analytics context
//       -> operational status
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query is a batch-style graph-enriched vector retrieval workflow.
//
// The main purpose is:
//
//     "Run multiple semantic retrieval requests in one query,
//      then explain every retrieved chunk using the connected support graph."
//
// It is more scalable than writing separate queries for each scenario because
// new retrieval tests can be added by inserting more maps into the UNWIND list.
//
// It combines:
//
// - Batch query request handling
// - Query type classification
// - Natural-language question traceability
// - Vector similarity search
// - topK-based retrieval control
// - Knowledge article traceability
// - Issue-level explanation
// - Ticket-level operational evidence
// - Customer impact context
// - Product impact context
// - Agent ownership context
// - Graph analytics context
// - Human-readable operational status
//
// This is a strong production-style pattern for explainable RAG, retrieval
// testing, and graph-powered support intelligence.
```

# Step 4 — Add a GDS-aware retrieval ranking score

```cypher
// =============================================================================
// MULTI-REQUEST EXPLAINABLE VECTOR RETRIEVAL WITH GRAPH-BASED RANK BOOSTING
// =============================================================================
// This query performs batch-style vector retrieval for multiple user questions
// and then improves the final ranking by combining:
//
//     1. Vector similarity score
//     2. PageRank-based ticket importance
//     3. Degree-based ticket connectivity
//     4. Betweenness-based bridge importance
//
// In simple terms, it tells Neo4j:
//
//     "For each user question,
//      retrieve the most semantically similar document chunks,
//      connect those chunks to knowledge articles and issues,
//      collect related operational ticket context,
//      calculate graph-based ranking boosts,
//      and then return an explainable final retrieval ranking."
//
// This is more advanced than plain vector search.
//
// A plain vector search only ranks by semantic similarity:
//
//     retrievalRank = vectorScore
//
// But this query creates a hybrid ranking score:
//
//     retrievalRankScore = vectorScore
//                        + pageRankBoost
//                        + degreeBoost
//                        + betweennessBoost
//
// This is useful because two chunks may be semantically similar, but the chunk
// connected to more important operational tickets may deserve higher priority.
//
// For example:
//
//     Chunk A has high vector similarity but no related tickets.
//     Chunk B has slightly lower vector similarity but is connected to high
//     PageRank tickets, high-degree tickets, or bridge-like tickets.
//
// In a support intelligence system, Chunk B may be more operationally important.
//
// This query is useful for:
// - Explainable RAG
// - Hybrid vector + graph ranking
// - Support ticket intelligence
// - Retrieval quality evaluation
// - Batch testing multiple query categories
// - Prioritizing retrieved knowledge using graph analytics
// - Demonstrating how graph data science scores can influence retrieval ranking

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

// =============================================================================
// DEFINE MULTIPLE RETRIEVAL REQUESTS USING UNWIND
// =============================================================================
// UNWIND takes a list and expands each item in that list into a separate row.
//
// Here, the list contains three maps.
//
// Each map represents one retrieval request:
//
//     1. login
//     2. payment
//     3. app_crash
//
// After UNWIND runs, Neo4j processes the query as if it has three separate
// input rows:
//
//     Row 1 -> login request
//     Row 2 -> payment request
//     Row 3 -> app_crash request
//
// Each row is stored in the variable:
//
//     request
//
// -----------------------------------------------------------------------------
// Why use UNWIND here?
// -----------------------------------------------------------------------------
// UNWIND makes this query scalable.
//
// Instead of writing three separate queries or three UNION ALL branches, we put
// all retrieval requests into one list.
//
// If we want to add another test later, such as:
//
//     refund
//     performance
//     account_locked
//     password_reset
//
// we only need to add one more map to this list.
//
// The rest of the query can stay the same.
//
// -----------------------------------------------------------------------------
// Why use maps inside the list?
// -----------------------------------------------------------------------------
// Each request is represented as a map because a map keeps related fields
// together.
//
// Each request contains:
//
//     queryType
//     userQuestion
//     queryVector
//     topK
//
// This is cleaner than keeping separate lists for query types, questions,
// vectors, and topK values.
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// queryType identifies the category of the request.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This value is returned later so that we can clearly see which scenario
// produced each retrieval result.
//
// -----------------------------------------------------------------------------
// userQuestion
// -----------------------------------------------------------------------------
// userQuestion stores the original natural-language question.
//
// Example:
//
//     "Why can't customers log in?"
//
// In this query, the text itself is not sent directly to the vector index.
//
// The vector index uses queryVector.
//
// Still, keeping userQuestion is useful because it preserves human-readable
// traceability between the user's question and the retrieved results.
//
// -----------------------------------------------------------------------------
// queryVector
// -----------------------------------------------------------------------------
// queryVector stores the numeric embedding used for vector search.
//
// In a real production system, this vector would usually be generated by an
// embedding model from the userQuestion.
//
// For example:
//
//     User question:
//       "Why are payments failing?"
//
//     Embedding model:
//       Converts the text into a vector.
//
//     Neo4j vector index:
//       Uses that vector to find semantically similar chunks.
//
// In this lab/demo query, the vectors are hardcoded so that retrieval behavior
// is predictable and easy to explain.
//
// -----------------------------------------------------------------------------
// topK
// -----------------------------------------------------------------------------
// topK controls how many nearest chunks should be returned for each request.
//
// Here, every request uses:
//
//     topK: 2
//
// So each query type retrieves the top 2 closest DocumentChunk nodes.
//
// Since there are three requests and each request retrieves two chunks, the
// vector search stage may produce up to:
//
//     3 requests * 2 chunks = 6 retrieved chunk rows

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)

// =============================================================================
// RUN VECTOR SIMILARITY SEARCH FOR EACH REQUEST
// =============================================================================
// This procedure performs vector similarity search using Neo4j's vector index.
//
// The procedure:
//
//     db.index.vector.queryNodes()
//
// searches for graph nodes whose stored embedding vectors are closest to the
// input query vector.
//
// Because this procedure appears after UNWIND, it runs once for each request.
//
// So Neo4j effectively performs three vector searches:
//
//     login     -> [0.92, 0.12, 0.05]
//     payment   -> [0.08, 0.92, 0.12]
//     app_crash -> [0.12, 0.10, 0.92]
//
// -----------------------------------------------------------------------------
// First argument: 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// This is the name of the vector index.
//
// The index is expected to be created on DocumentChunk nodes, most likely on
// their embedding property.
//
// In simple terms, this says:
//
//     "Use the vector index built for document chunk embeddings."
//
// The vector index makes similarity search efficient.
//
// Without the index, Neo4j would need to compare the query vector against many
// stored vectors manually, which would not scale well.
//
// -----------------------------------------------------------------------------
// Second argument: request.topK
// -----------------------------------------------------------------------------
// request.topK tells Neo4j how many nearest matching nodes to return for the
// current request.
//
// Since each request has:
//
//     topK: 2
//
// the procedure returns the two most similar DocumentChunk nodes for each query
// type.
//
// -----------------------------------------------------------------------------
// Third argument: request.queryVector
// -----------------------------------------------------------------------------
// request.queryVector is the current request's embedding vector.
//
// Neo4j compares this vector against stored DocumentChunk embeddings.
//
// Important beginner note:
//
//     Neo4j is not comparing English sentences directly here.
//
// It is comparing numeric vectors that represent meaning.
//
// Production note:
//
// In a real RAG system, request.queryVector would usually be produced by an
// embedding model before this Cypher query is executed.

YIELD node AS dc, score

// =============================================================================
// CAPTURE VECTOR SEARCH OUTPUT
// =============================================================================
// YIELD extracts the output returned by db.index.vector.queryNodes().
//
// This procedure returns:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the matching graph node returned by the vector search.
//
// We rename it to:
//
//     dc
//
// because it represents a DocumentChunk.
//
// This makes the rest of the query easier to read.
//
// Whenever we see dc, we know:
//
//     "This is the retrieved document chunk."
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the vector similarity score between:
//
//     request.queryVector
//
// and the retrieved chunk's stored embedding.
//
// A higher score usually means a stronger semantic match.
//
// In simple terms:
//
//     Higher score = more semantically similar
//     Lower score  = less semantically similar
//
// However, this query does not rely only on vector score.
//
// Later, it adds graph-based boosts using ticket analytics scores.
//
// This is what makes the query a hybrid vector + graph ranking query.

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// CONNECT RETRIEVED CHUNK TO KNOWLEDGE ARTICLE AND ISSUE
// =============================================================================
// This MATCH clause enriches each retrieved DocumentChunk with source and issue
// context.
//
// The pattern is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// Let us break this down.
//
// -----------------------------------------------------------------------------
// (dc)
// -----------------------------------------------------------------------------
// dc is the DocumentChunk retrieved by vector similarity search.
//
// It is the chunk that matched the current request.queryVector.
//
// -----------------------------------------------------------------------------
// -[:PART_OF]->
// -----------------------------------------------------------------------------
// PART_OF shows that the retrieved chunk belongs to a larger KnowledgeArticle.
//
// This is important because chunks are usually created by splitting larger
// documents into smaller pieces.
//
// The chunk is useful for retrieval, but the article gives source traceability.
//
// -----------------------------------------------------------------------------
// (ka:KnowledgeArticle)
// -----------------------------------------------------------------------------
// ka represents the parent KnowledgeArticle.
//
// This helps answer:
//
//     "Which knowledge article did this retrieved chunk come from?"
//
// Source traceability is very important in explainable RAG.
//
// -----------------------------------------------------------------------------
// -[:SOLVES]->
// -----------------------------------------------------------------------------
// SOLVES connects a KnowledgeArticle to the Issue it addresses.
//
// This relationship gives business meaning to the article.
//
// It tells us:
//
//     "This article is intended to solve this issue."
//
// -----------------------------------------------------------------------------
// (i:Issue)
// -----------------------------------------------------------------------------
// i represents the Issue solved by the article.
//
// This is the key explainability step.
//
// Without this MATCH, we only know:
//
//     "This chunk is similar to the query vector."
//
// With this MATCH, we know:
//
//     "This chunk is similar,
//      it belongs to this knowledge article,
//      and that article solves this issue."
//
// Production relevance:
//
// This transforms plain vector retrieval into graph-enriched retrieval.

OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

// =============================================================================
// OPTIONALLY FIND TICKETS RELATED TO THE ISSUE
// =============================================================================
// This OPTIONAL MATCH finds tickets connected to the retrieved issue.
//
// The pattern is:
//
//     (t:Ticket)-[:HAS_ISSUE]->(i)
//
// Meaning:
//
//     "Find tickets that have this issue."
//
// -----------------------------------------------------------------------------
// Why OPTIONAL MATCH is used
// -----------------------------------------------------------------------------
// OPTIONAL MATCH behaves like a left join.
//
// It tries to find the pattern, but if the pattern does not exist, the existing
// row is preserved and the missing values become null.
//
// This is important because not every Issue may have related Ticket nodes.
//
// If we used a normal MATCH here, then retrieved issues without tickets would
// disappear from the result.
//
// That would be dangerous because a vector retrieval result may still be valid
// even if ticket context is missing.
//
// -----------------------------------------------------------------------------
// Why ticket context matters
// -----------------------------------------------------------------------------
// Tickets provide operational evidence.
//
// They help answer:
// - Has this issue happened in real support cases?
// - How many tickets are connected to it?
// - What priority do those tickets have?
// - Are the tickets open or resolved?
// - Do graph analytics scores suggest that the tickets are important?

OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)

// =============================================================================
// OPTIONALLY FIND CUSTOMERS WHO RAISED RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds customers connected to related tickets.
//
// The pattern is:
//
//     (c:Customer)-[:RAISED]->(t)
//
// Meaning:
//
//     "Find customers who raised these tickets."
//
// This is optional because some tickets may not have customer relationships.
//
// That can happen in:
// - Demo datasets
// - Partial imports
// - Anonymized datasets
// - Internal monitoring tickets
// - Migration scenarios where relationships are incomplete
//
// Customer context helps answer:
// - Which customers are affected?
// - Are multiple customers reporting the same issue?
// - Is one customer repeatedly affected?
// - Should this issue be escalated because of customer impact?

OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)

// =============================================================================
// OPTIONALLY FIND PRODUCTS RELATED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds products connected to the related tickets.
//
// The pattern is:
//
//     (t)-[:ABOUT]->(p:Product)
//
// Meaning:
//
//     "Find the product that this ticket is about."
//
// Product context helps answer:
// - Which product is impacted?
// - Is the issue limited to one product?
// - Does the same issue appear across multiple products?
// - Which product team may need to investigate?
//
// This is optional because some tickets may not have product information.

OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

// =============================================================================
// OPTIONALLY FIND AGENTS ASSIGNED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds agents assigned to related tickets.
//
// The pattern is:
//
//     (t)-[:ASSIGNED_TO]->(a:Agent)
//
// Meaning:
//
//     "Find the support agent assigned to this ticket."
//
// Agent context helps answer:
// - Who owns the related ticket?
// - Which agents have worked on this issue?
// - Are multiple agents involved?
// - Is one agent handling many similar tickets?
// - Is escalation or reassignment needed?
//
// This is optional because some tickets may not be assigned yet.

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

// =============================================================================
// AGGREGATE OPERATIONAL CONTEXT AND EXTRACT GRAPH ANALYTICS MAX SCORES
// =============================================================================
// This WITH clause performs two important jobs:
//
//     1. It groups related graph context into clean lists.
//     2. It calculates maximum graph analytics scores from related tickets.
//
// This is a major transition point in the query.
//
// Before this WITH clause, OPTIONAL MATCH clauses may create multiple internal
// rows because one issue can be connected to many tickets, customers, products,
// and agents.
//
// This WITH clause compresses those rows into a cleaner grouped structure.
//
// -----------------------------------------------------------------------------
// request
// -----------------------------------------------------------------------------
// request is carried forward so that the final output still knows which
// retrieval request produced the result.
//
// This includes:
//
//     request.queryType
//     request.userQuestion
//     request.queryVector
//     request.topK
//
// -----------------------------------------------------------------------------
// dc
// -----------------------------------------------------------------------------
// dc is the retrieved DocumentChunk.
//
// It must be carried forward because the final output returns:
//
//     dc.chunkId
//     dc.text
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the original vector similarity score.
//
// It is carried forward because it becomes:
//
//     vectorScore
//
// and also contributes to:
//
//     retrievalRankScore
//
// -----------------------------------------------------------------------------
// ka
// -----------------------------------------------------------------------------
// ka is the parent KnowledgeArticle.
//
// It is carried forward so the final output can show:
//
//     ka.articleId
//     ka.title
//
// -----------------------------------------------------------------------------
// i
// -----------------------------------------------------------------------------
// i is the Issue solved by the KnowledgeArticle.
//
// It is carried forward so the final output can show:
//
//     i.issueId
//     i.name
//     i.severity
//
// -----------------------------------------------------------------------------
// collect(DISTINCT t.ticketId) AS relatedTicketIds
// -----------------------------------------------------------------------------
// This collects all related ticket IDs into a list.
//
// DISTINCT removes duplicates caused by optional graph expansion.
//
// This gives a clean list of tickets connected to the retrieved issue.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT c.name) AS relatedCustomers
// -----------------------------------------------------------------------------
// This collects all customer names connected through related tickets.
//
// DISTINCT prevents repeated customer names from appearing multiple times.
//
// This provides customer impact context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT p.name) AS relatedProducts
// -----------------------------------------------------------------------------
// This collects all related product names.
//
// This helps identify which products are affected by the retrieved issue.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT a.name) AS assignedAgents
// -----------------------------------------------------------------------------
// This collects all assigned agent names.
//
// This helps identify operational ownership.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT CASE ... END) AS ticketAnalyticsContext
// -----------------------------------------------------------------------------
// This creates a structured list of ticket analytics maps.
//
// If t is null, the CASE expression returns null.
//
// If t exists, it creates a map containing:
//
//     ticketId
//     priority
//     status
//     fullDegreeScore
//     pageRankScore
//     betweennessScore
//     louvainCommunityId
//     labelPropagationCommunityId
//
// This gives detailed ticket-level operational and graph analytics context.
//
// -----------------------------------------------------------------------------
// count(t) AS ticketCount
// -----------------------------------------------------------------------------
// ticketCount counts how many ticket rows exist for the grouped result.
//
// It is later used to produce a human-readable operational status.
//
// If ticketCount = 0, then no related tickets were found.
//
// -----------------------------------------------------------------------------
// max(t.pageRankScore) AS maxPageRankScore
// -----------------------------------------------------------------------------
// This finds the maximum PageRank score among related tickets.
//
// Why max?
//
//     If multiple tickets are related to the issue,
//     we want the strongest PageRank signal to influence the rank.
//
// PageRank usually represents graph importance or influence.
//
// A high PageRank ticket may be connected to other important nodes.
//
// -----------------------------------------------------------------------------
// max(t.fullDegreeScore) AS maxFullDegreeScore
// -----------------------------------------------------------------------------
// This finds the maximum degree score among related tickets.
//
// Degree usually represents how connected a node is.
//
// A ticket with high degree may be connected to many customers, products,
// agents, issues, or other important entities.
//
// -----------------------------------------------------------------------------
// max(t.betweennessScore) AS maxBetweennessScore
// -----------------------------------------------------------------------------
// This finds the maximum betweenness score among related tickets.
//
// Betweenness usually identifies bridge-like nodes.
//
// A high-betweenness ticket may connect different parts of the graph.
//
// Such a ticket can be operationally important because it may reveal a link
// between issue clusters that otherwise look separate.

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

// =============================================================================
// CALCULATE GRAPH-BASED RANKING BOOSTS
// =============================================================================
// This WITH clause converts raw graph analytics scores into ranking boost values.
//
// These boost values are later added to the vector score to calculate:
//
//     retrievalRankScore
//
// This is the hybrid ranking idea:
//
//     retrievalRankScore = vectorScore
//                        + pageRankBoost
//                        + degreeBoost
//                        + betweennessBoost
//
// -----------------------------------------------------------------------------
// Why use graph-based boosts?
// -----------------------------------------------------------------------------
// Vector score tells us semantic similarity.
//
// But semantic similarity alone does not always tell us operational importance.
//
// A retrieved chunk may be highly similar but connected to no real tickets.
//
// Another chunk may be slightly less similar but connected to highly important,
// highly connected, or bridge-like tickets.
//
// Graph boosts help the ranking consider operational importance.
//
// -----------------------------------------------------------------------------
// coalesce(maxPageRankScore, 0.0) AS pageRankBoost
// -----------------------------------------------------------------------------
// coalesce() handles null values.
//
// If maxPageRankScore is null, it returns 0.0 instead.
//
// This is important because if there are no related tickets, then:
//
//     max(t.pageRankScore)
//
// may be null.
//
// Without coalesce, adding null to the final score could make the whole
// retrievalRankScore null.
//
// pageRankBoost uses the raw PageRank value directly.
//
// Meaning:
//
//     Higher PageRank = stronger importance boost
//
// Production note:
//
// Whether raw PageRank should be used directly depends on score scale.
// In real production systems, PageRank may need normalization.
//
// -----------------------------------------------------------------------------
// coalesce(maxFullDegreeScore, 0.0) * 0.01 AS degreeBoost
// -----------------------------------------------------------------------------
// This converts the maximum fullDegreeScore into a smaller boost.
//
// The multiplication by:
//
//     0.01
//
// scales down the degree score.
//
// Why scale it?
//
//     Degree values can be much larger than vector similarity scores.
//
// If we added raw degree directly, it might dominate the final rank too much.
//
// Multiplying by 0.01 means:
//
//     "Use degree as a small influence, not as the entire ranking decision."
//
// -----------------------------------------------------------------------------
// coalesce(maxBetweennessScore, 0.0) * 0.01 AS betweennessBoost
// -----------------------------------------------------------------------------
// This converts the maximum betweenness score into a smaller boost.
//
// Like degreeBoost, it is multiplied by:
//
//     0.01
//
// This prevents betweenness from overwhelming vector similarity.
//
// Betweenness is useful because it identifies bridge nodes.
//
// A result connected to a high-betweenness ticket may be important because that
// ticket connects different parts of the support graph.
//
// -----------------------------------------------------------------------------
// Important production note about boost weights
// -----------------------------------------------------------------------------
// The weights used here are simple demo weights:
//
//     degreeBoost      = fullDegreeScore * 0.01
//     betweennessBoost = betweennessScore * 0.01
//
// In production, these weights should be tuned carefully.
//
// Common approaches include:
// - Normalize all scores to the same range
// - Use configurable weights
// - Evaluate ranking quality against labeled test cases
// - Avoid letting graph scores overpower semantic relevance
// - Monitor whether boosted results remain useful to users

RETURN
  request.queryType AS queryType,
  request.userQuestion AS userQuestion,

// =============================================================================
// RETURN QUERY METADATA
// =============================================================================
// This section returns the request metadata.
//
// -----------------------------------------------------------------------------
// request.queryType AS queryType
// -----------------------------------------------------------------------------
// queryType identifies which retrieval scenario produced this result.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This is important because all scenarios are processed in one batch query.
//
// Without queryType, the final output would mix all results together without
// showing which request they came from.
//
// -----------------------------------------------------------------------------
// request.userQuestion AS userQuestion
// -----------------------------------------------------------------------------
// userQuestion returns the original natural-language question.
//
// This keeps the result traceable to the user intent.
//
// It helps with:
// - Debugging
// - Evaluation
// - Demo explanation
// - Audit trails
// - RAG observability

  dc.chunkId AS chunkId,
  dc.text AS retrievedChunk,
  score AS vectorScore,

// =============================================================================
// RETURN RETRIEVED CHUNK DETAILS AND VECTOR SCORE
// =============================================================================
// This section returns the core retrieval result.
//
// -----------------------------------------------------------------------------
// dc.chunkId AS chunkId
// -----------------------------------------------------------------------------
// chunkId uniquely identifies the retrieved DocumentChunk.
//
// This is important for source traceability.
//
// It helps answer:
//
//     "Exactly which chunk did the vector index retrieve?"
//
// -----------------------------------------------------------------------------
// dc.text AS retrievedChunk
// -----------------------------------------------------------------------------
// retrievedChunk is the actual text stored in the retrieved DocumentChunk.
//
// This is the content that matched the query vector.
//
// In a RAG system, this text may be passed to an LLM as supporting context.
//
// -----------------------------------------------------------------------------
// score AS vectorScore
// -----------------------------------------------------------------------------
// vectorScore is the original similarity score from vector search.
//
// This represents semantic closeness between:
//
//     request.queryVector
//
// and the chunk embedding.
//
// Important point:
//
//     vectorScore is still returned separately even though we also calculate
//     retrievalRankScore.
//
// This is a good practice because it lets us compare:
//
//     pure semantic relevance
//
// versus:
//
//     graph-boosted ranking relevance

  pageRankBoost,
  degreeBoost,
  betweennessBoost,

// =============================================================================
// RETURN GRAPH-BASED BOOST COMPONENTS
// =============================================================================
// This section returns the individual graph boost values used in the final rank.
//
// Returning these separately is very important for explainability.
//
// Instead of only returning a final score, we show how that score was built.
//
// -----------------------------------------------------------------------------
// pageRankBoost
// -----------------------------------------------------------------------------
// pageRankBoost comes from the maximum PageRank score among related tickets.
//
// It represents graph-based importance or influence.
//
// A higher pageRankBoost means the retrieved issue is connected to at least one
// ticket that is important in the graph structure.
//
// -----------------------------------------------------------------------------
// degreeBoost
// -----------------------------------------------------------------------------
// degreeBoost comes from:
//
//     maxFullDegreeScore * 0.01
//
// It represents a scaled connectivity boost.
//
// A higher degreeBoost means the retrieved issue is connected to a ticket that
// has many graph connections.
//
// -----------------------------------------------------------------------------
// betweennessBoost
// -----------------------------------------------------------------------------
// betweennessBoost comes from:
//
//     maxBetweennessScore * 0.01
//
// It represents a scaled bridge-importance boost.
//
// A higher betweennessBoost means the retrieved issue is connected to a ticket
// that may act as a bridge between different graph communities or clusters.
//
// -----------------------------------------------------------------------------
// Why return boost components?
// -----------------------------------------------------------------------------
// Returning these fields makes the ranking explainable.
//
// A user can see whether a result ranked high because of:
//
//     - strong vector similarity,
//     - high PageRank,
//     - high degree,
//     - high betweenness,
//     - or a combination of all of them.

  score + pageRankBoost + degreeBoost + betweennessBoost AS retrievalRankScore,

// =============================================================================
// RETURN FINAL HYBRID RETRIEVAL RANK SCORE
// =============================================================================
// retrievalRankScore is the final ranking score used by this query.
//
// It is calculated as:
//
//     vectorScore
//       + pageRankBoost
//       + degreeBoost
//       + betweennessBoost
//
// In the query, that is written as:
//
//     score + pageRankBoost + degreeBoost + betweennessBoost
//
// -----------------------------------------------------------------------------
// What this means
// -----------------------------------------------------------------------------
// vectorScore captures semantic similarity.
//
// pageRankBoost captures graph importance.
//
// degreeBoost captures graph connectivity.
//
// betweennessBoost captures bridge-like importance.
//
// Together, they produce a hybrid rank that considers both:
//
//     "Is this chunk semantically relevant?"
//
// and:
//
//     "Is the connected operational context important?"
//
// -----------------------------------------------------------------------------
// Why this is powerful
// -----------------------------------------------------------------------------
// This ranking method allows the graph to influence retrieval quality.
//
// For example:
//
//     Result A:
//       vectorScore = 0.95
//       graph boosts = 0.00
//       retrievalRankScore = 0.95
//
//     Result B:
//       vectorScore = 0.90
//       graph boosts = 0.15
//       retrievalRankScore = 1.05
//
// Even though Result B has a slightly lower vector score, it may rank higher
// because the graph says its related tickets are more important.
//
// -----------------------------------------------------------------------------
// Production note
// -----------------------------------------------------------------------------
// This is a simple scoring formula for teaching and demonstration.
//
// In production, this formula should be tuned and validated.
//
// Good production practices include:
// - Normalizing vector and graph scores
// - Using configurable weights
// - Measuring ranking quality
// - Testing against known expected results
// - Avoiding score dominance by one metric
// - Logging boost components for explainability

  ka.articleId AS articleId,
  ka.title AS articleTitle,

// =============================================================================
// RETURN KNOWLEDGE ARTICLE CONTEXT
// =============================================================================
// This section returns the parent KnowledgeArticle information.
//
// -----------------------------------------------------------------------------
// ka.articleId AS articleId
// -----------------------------------------------------------------------------
// articleId is the business identifier of the KnowledgeArticle.
//
// It helps trace the retrieved chunk back to its source article.
//
// This is useful for:
// - Documentation lookup
// - Audit trails
// - Support references
// - Knowledge base navigation
//
// -----------------------------------------------------------------------------
// ka.title AS articleTitle
// -----------------------------------------------------------------------------
// articleTitle is the human-readable title of the KnowledgeArticle.
//
// It helps users quickly understand the broader article context without reading
// only the chunk text.

  i.issueId AS issueId,
  i.name AS issueName,
  i.severity AS issueSeverity,

// =============================================================================
// RETURN ISSUE CONTEXT
// =============================================================================
// This section returns the issue solved by the KnowledgeArticle.
//
// -----------------------------------------------------------------------------
// i.issueId AS issueId
// -----------------------------------------------------------------------------
// issueId uniquely identifies the Issue node.
//
// This connects retrieved knowledge to a known operational problem.
//
// -----------------------------------------------------------------------------
// i.name AS issueName
// -----------------------------------------------------------------------------
// issueName is the human-readable name of the issue.
//
// Example issue names may be:
//
//     Login Failure
//     Payment Declined
//     Mobile App Crash
//
// The actual values depend on your graph data.
//
// -----------------------------------------------------------------------------
// i.severity AS issueSeverity
// -----------------------------------------------------------------------------
// issueSeverity indicates the seriousness of the issue.
//
// Severity helps determine operational priority.
//
// For example, a high-severity issue with a slightly lower vector score may
// deserve more attention than a low-severity issue with a slightly higher score.

  relatedTicketIds,
  relatedCustomers,
  relatedProducts,
  assignedAgents,
  ticketAnalyticsContext,

// =============================================================================
// RETURN AGGREGATED OPERATIONAL CONTEXT
// =============================================================================
// This section returns the aggregated context collected earlier in the WITH
// clause.
//
// -----------------------------------------------------------------------------
// relatedTicketIds
// -----------------------------------------------------------------------------
// relatedTicketIds is a list of ticket IDs connected to the retrieved issue.
//
// This helps answer:
//
//     "Which real support tickets are related to this result?"
//
// -----------------------------------------------------------------------------
// relatedCustomers
// -----------------------------------------------------------------------------
// relatedCustomers is a list of customer names connected through related
// tickets.
//
// This helps measure customer impact.
//
// -----------------------------------------------------------------------------
// relatedProducts
// -----------------------------------------------------------------------------
// relatedProducts is a list of product names connected through related tickets.
//
// This helps identify affected products or services.
//
// -----------------------------------------------------------------------------
// assignedAgents
// -----------------------------------------------------------------------------
// assignedAgents is a list of support agents assigned to related tickets.
//
// This helps identify ownership and follow-up responsibility.
//
// -----------------------------------------------------------------------------
// ticketAnalyticsContext
// -----------------------------------------------------------------------------
// ticketAnalyticsContext is a list of maps containing ticket details and graph
// analytics scores.
//
// Each map may include:
//
//     ticketId
//     priority
//     status
//     fullDegreeScore
//     pageRankScore
//     betweennessScore
//     louvainCommunityId
//     labelPropagationCommunityId
//
// This provides detailed operational and graph analytics evidence behind the
// final retrieval ranking.

  CASE
    WHEN ticketCount = 0 THEN "No related ticket found for this issue"
    ELSE "Related ticket context found"
  END AS operationalContextStatus

// =============================================================================
// RETURN HUMAN-READABLE OPERATIONAL CONTEXT STATUS
// =============================================================================
// This CASE expression creates a simple status message.
//
// It checks:
//
//     ticketCount = 0
//
// -----------------------------------------------------------------------------
// WHEN ticketCount = 0
// -----------------------------------------------------------------------------
// If no related tickets were found, return:
//
//     "No related ticket found for this issue"
//
// This means the vector search found a chunk, article, and issue, but there is
// no ticket-based operational evidence connected to that issue.
//
// -----------------------------------------------------------------------------
// ELSE
// -----------------------------------------------------------------------------
// If ticketCount is greater than zero, return:
//
//     "Related ticket context found"
//
// This means the retrieved issue is connected to one or more support tickets.
//
// -----------------------------------------------------------------------------
// Why this field is useful
// -----------------------------------------------------------------------------
// This field makes the output easier to understand during demos and debugging.
//
// Instead of manually inspecting relatedTicketIds, the user can immediately see
// whether operational context exists.
//
// Production note:
//
// Human-readable derived status fields are helpful for dashboards, APIs, and
// support analysts because they convert raw graph structure into a clear
// operational message.

ORDER BY
  queryType,
  retrievalRankScore DESC,
  vectorScore DESC;

// =============================================================================
// ORDER FINAL RESULTS BY QUERY TYPE AND HYBRID RANKING SCORE
// =============================================================================
// ORDER BY controls how the final rows are sorted.
//
// This query orders by:
//
//     queryType
//     retrievalRankScore DESC
//     vectorScore DESC
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// queryType groups results by retrieval scenario.
//
// This means results are grouped by categories such as:
//
//     app_crash
//     login
//     payment
//
// Grouping by queryType makes it easier to compare results within each
// retrieval scenario.
//
// -----------------------------------------------------------------------------
// retrievalRankScore DESC
// -----------------------------------------------------------------------------
// retrievalRankScore is the final hybrid score.
//
// It includes:
//
//     vectorScore
//     pageRankBoost
//     degreeBoost
//     betweennessBoost
//
// DESC means descending order.
//
// So within each queryType, the result with the highest hybrid rank appears
// first.
//
// This is the main ranking logic of the query.
//
// -----------------------------------------------------------------------------
// vectorScore DESC
// -----------------------------------------------------------------------------
// vectorScore is used as a secondary tie-breaker.
//
// If two results have the same retrievalRankScore, the one with the higher pure
// vector similarity score appears first.
//
// This is a good design because semantic relevance should still matter strongly,
// even when graph boosts are used.
//
// -----------------------------------------------------------------------------
// Final behavior
// -----------------------------------------------------------------------------
// The final output is:
//
//     grouped by query type,
//     ranked by graph-boosted retrieval score,
//     and tie-broken by vector similarity.
//
// This gives a clean explainable ranking output:
//
//     query type
//       -> best hybrid-ranked chunks
//       -> vector score
//       -> graph boost components
//       -> final retrieval rank score
//       -> article context
//       -> issue context
//       -> operational ticket context
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query demonstrates an advanced production-style retrieval pattern.
//
// The main purpose is:
//
//     "Retrieve semantically relevant chunks,
//      enrich them with graph context,
//      use graph analytics signals to boost ranking,
//      and return an explainable final result."
//
// It combines:
//
// - Batch retrieval using UNWIND
// - Vector similarity search
// - Knowledge article traceability
// - Issue-level explanation
// - Ticket-level operational context
// - Customer impact context
// - Product impact context
// - Agent ownership context
// - PageRank-based importance boosting
// - Degree-based connectivity boosting
// - Betweenness-based bridge boosting
// - Hybrid retrieval ranking
// - Explainable score decomposition
//
// This is much closer to a real enterprise explainable RAG design than plain
// vector search alone, because the final result is influenced by both semantic
// meaning and operational graph importance.
```

# Step 5 — Assemble retrieved results into grouped subgraph context

```cypher
// =============================================================================
// BATCH EXPLAINABLE VECTOR RETRIEVAL WITH HYBRID RANKING AND GROUPED CONTEXT
// =============================================================================
// This query performs multiple vector retrieval requests in one execution and
// returns the retrieved results as grouped contextual objects.
//
// In simple terms, it tells Neo4j:
//
//     "For each user question,
//      retrieve the most semantically similar document chunks,
//      enrich those chunks with knowledge article, issue, ticket, customer,
//      product, agent, and analytics context,
//      calculate a hybrid ranking score,
//      then package the retrieved context into a structured list."
//
// This query is more advanced than a simple vector search because it does not
// only return flat rows.
//
// Instead, it builds a structured response:
//
//     queryType
//     userQuestion
//     topK
//     retrievedContext: [
//       {
//         chunk details,
//         vector score,
//         graph boost scores,
//         final rank score,
//         article context,
//         issue context,
//         operational context,
//         analytics context
//       }
//     ]
//
// This is very close to how an API response or RAG backend might prepare
// retrieval results before sending them to an application or LLM.
//
// The query combines:
// - Batch request handling using UNWIND
// - Vector similarity search
// - Knowledge article traceability
// - Issue-level explanation
// - Ticket/customer/product/agent enrichment
// - Graph analytics-based ranking boosts
// - Hybrid retrieval ranking
// - Structured context packaging

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

// =============================================================================
// DEFINE MULTIPLE RETRIEVAL REQUESTS USING UNWIND
// =============================================================================
// UNWIND takes a list and expands each item into its own row.
//
// Here, the list contains three maps:
//
//     1. login request
//     2. payment request
//     3. app_crash request
//
// After UNWIND runs, Neo4j processes each map one by one as:
//
//     request
//
// This means the rest of the query is executed once for each request.
//
// -----------------------------------------------------------------------------
// Why this is useful
// -----------------------------------------------------------------------------
// Without UNWIND, we would need separate queries or UNION ALL branches for each
// retrieval scenario.
//
// With UNWIND, we can scale easily.
//
// If we want to add more scenarios later, such as:
//
//     refund
//     password_reset
//     performance
//     account_locked
//
// we only add another map to this list.
//
// The rest of the query remains unchanged.
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// queryType identifies the category of the request.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This is useful for grouping and comparing retrieval results.
//
// -----------------------------------------------------------------------------
// userQuestion
// -----------------------------------------------------------------------------
// userQuestion stores the human-readable question.
//
// This is useful for traceability.
//
// Even though the vector index uses queryVector, not raw text, keeping the
// original question helps explain what user intent produced the result.
//
// -----------------------------------------------------------------------------
// queryVector
// -----------------------------------------------------------------------------
// queryVector is the numeric embedding used for semantic search.
//
// In a real production system, this vector would normally be generated by an
// embedding model from the userQuestion.
//
// Here, the vectors are hardcoded for controlled demo/testing behavior.
//
// -----------------------------------------------------------------------------
// topK
// -----------------------------------------------------------------------------
// topK controls how many matching chunks should be retrieved for each request.
//
// Since each request has:
//
//     topK: 2
//
// Neo4j retrieves the top 2 matching chunks per query type.

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)

// =============================================================================
// RUN VECTOR SEARCH FOR THE CURRENT REQUEST
// =============================================================================
// db.index.vector.queryNodes() searches a Neo4j vector index and returns the
// nearest nodes to the given query vector.
//
// In simple terms:
//
//     "Find the DocumentChunk nodes whose embeddings are closest to this
//      request's query vector."
//
// -----------------------------------------------------------------------------
// 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// This is the name of the vector index.
//
// It is expected to be created on DocumentChunk nodes, likely on their embedding
// property.
//
// The vector index allows Neo4j to search by semantic similarity efficiently.
//
// -----------------------------------------------------------------------------
// request.topK
// -----------------------------------------------------------------------------
// This tells Neo4j how many nearest matching chunks to return for the current
// request.
//
// Because topK is stored inside each request map, different requests could use
// different topK values if needed.
//
// -----------------------------------------------------------------------------
// request.queryVector
// -----------------------------------------------------------------------------
// This is the embedding vector for the current request.
//
// Neo4j compares this vector with stored DocumentChunk embeddings.
//
// Important point:
//
//     Neo4j is not directly comparing the English userQuestion here.
//
// It is comparing numeric vectors that represent meaning.

YIELD node AS dc, score

// =============================================================================
// CAPTURE VECTOR SEARCH OUTPUT
// =============================================================================
// YIELD extracts the output from db.index.vector.queryNodes().
//
// The procedure returns:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the retrieved graph node.
//
// We rename it to dc because it represents a DocumentChunk.
//
// This makes the query easier to read.
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the vector similarity score.
//
// A higher score generally means the retrieved chunk is more semantically
// similar to the query vector.
//
// This score later becomes:
//
//     vectorScore
//
// and also contributes to:
//
//     retrievalRankScore

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// CONNECT RETRIEVED CHUNK TO KNOWLEDGE ARTICLE AND ISSUE
// =============================================================================
// This MATCH clause enriches each retrieved chunk with graph context.
//
// The path is:
//
//     DocumentChunk -> KnowledgeArticle -> Issue
//
// Meaning:
//
//     "This retrieved chunk is part of a knowledge article,
//      and that knowledge article solves a specific issue."
//
// This is the key explainability step.
//
// Without this MATCH, we only know:
//
//     "This chunk is semantically similar."
//
// With this MATCH, we know:
//
//     "This chunk is semantically similar,
//      it came from this article,
//      and the article solves this issue."
//
// This makes vector retrieval traceable and business meaningful.

OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

// =============================================================================
// OPTIONALLY FIND TICKETS RELATED TO THE ISSUE
// =============================================================================
// This OPTIONAL MATCH finds tickets connected to the issue.
//
// The pattern means:
//
//     "Find Ticket nodes that have this Issue."
//
// OPTIONAL MATCH is used because some issues may not have related tickets.
//
// If we used a normal MATCH, issues without tickets would disappear from the
// result.
//
// OPTIONAL MATCH keeps the retrieved result and uses null when ticket context is
// missing.

OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)

// =============================================================================
// OPTIONALLY FIND CUSTOMERS WHO RAISED THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds customers connected to the related tickets.
//
// The pattern means:
//
//     "Find customers who raised these tickets."
//
// Customer context helps measure impact.
//
// It helps answer:
// - Which customers are affected?
// - Are multiple customers reporting the same issue?
// - Is one customer repeatedly affected?
// - Should the issue be escalated?

OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)

// =============================================================================
// OPTIONALLY FIND PRODUCTS RELATED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds products associated with the tickets.
//
// The pattern means:
//
//     "Find the product this ticket is about."
//
// Product context helps identify which product, application, or service is
// affected by the issue.

OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

// =============================================================================
// OPTIONALLY FIND AGENTS ASSIGNED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds agents assigned to related tickets.
//
// The pattern means:
//
//     "Find the support agent assigned to this ticket."
//
// Agent context helps identify ownership and follow-up responsibility.

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

// =============================================================================
// AGGREGATE OPERATIONAL CONTEXT AND CALCULATE GRAPH BOOSTS
// =============================================================================
// This WITH clause groups the optional graph context and calculates boost values.
//
// Before this point, OPTIONAL MATCH clauses may create multiple rows because one
// issue can connect to many tickets, customers, products, and agents.
//
// This WITH clause compresses those expanded rows into grouped collections.
//
// -----------------------------------------------------------------------------
// request
// -----------------------------------------------------------------------------
// Carries the original request forward.
//
// We need it later to return:
//
//     queryType
//     userQuestion
//     topK
//
// -----------------------------------------------------------------------------
// dc
// -----------------------------------------------------------------------------
// Carries the retrieved DocumentChunk forward.
//
// We need it later for:
//
//     chunkId
//     text
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// Carries the vector similarity score forward.
//
// This score is used both as the raw vectorScore and as part of the final hybrid
// retrievalRankScore.
//
// -----------------------------------------------------------------------------
// ka
// -----------------------------------------------------------------------------
// Carries the KnowledgeArticle forward.
//
// We need it later to return article context.
//
// -----------------------------------------------------------------------------
// i
// -----------------------------------------------------------------------------
// Carries the Issue forward.
//
// We need it later to return issue context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT t.ticketId) AS relatedTicketIds
// -----------------------------------------------------------------------------
// Collects related ticket IDs into a clean list.
//
// DISTINCT removes duplicates caused by optional graph expansion.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT c.name) AS relatedCustomers
// -----------------------------------------------------------------------------
// Collects customer names connected through related tickets.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT p.name) AS relatedProducts
// -----------------------------------------------------------------------------
// Collects product names connected through related tickets.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT a.name) AS assignedAgents
// -----------------------------------------------------------------------------
// Collects agent names assigned to related tickets.
//
// -----------------------------------------------------------------------------
// ticketAnalyticsContext
// -----------------------------------------------------------------------------
// This creates a structured list of ticket analytics maps.
//
// For each ticket, the map may contain:
//
//     ticketId
//     priority
//     status
//     fullDegreeScore
//     pageRankScore
//     betweennessScore
//     louvainCommunityId
//     labelPropagationCommunityId
//
// This gives detailed operational and graph analytics evidence behind the
// retrieval result.
//
// -----------------------------------------------------------------------------
// count(t) AS ticketCount
// -----------------------------------------------------------------------------
// Counts how many ticket rows were found.
//
// This is later used to decide whether operational ticket context exists.
//
// -----------------------------------------------------------------------------
// pageRankBoost
// -----------------------------------------------------------------------------
// Uses the maximum PageRank score among related tickets.
//
// coalesce(..., 0.0) ensures that if no ticket exists, the boost becomes 0.0
// instead of null.
//
// PageRank usually represents graph importance or influence.
//
// -----------------------------------------------------------------------------
// degreeBoost
// -----------------------------------------------------------------------------
// Uses the maximum fullDegreeScore among related tickets and scales it by 0.01.
//
// The scaling prevents degree from dominating the ranking too much.
//
// -----------------------------------------------------------------------------
// betweennessBoost
// -----------------------------------------------------------------------------
// Uses the maximum betweennessScore among related tickets and scales it by 0.01.
//
// Betweenness identifies bridge-like nodes that connect different parts of the
// graph.
//
// Scaling keeps this as a controlled boost instead of overwhelming vector
// similarity.

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

// =============================================================================
// CALCULATE FINAL HYBRID RETRIEVAL RANK SCORE
// =============================================================================
// This WITH clause calculates the final ranking score.
//
// The formula is:
//
//     retrievalRankScore = vectorScore
//                        + pageRankBoost
//                        + degreeBoost
//                        + betweennessBoost
//
// In the query:
//
//     score + pageRankBoost + degreeBoost + betweennessBoost
//
// -----------------------------------------------------------------------------
// Why this matters
// -----------------------------------------------------------------------------
// Vector score measures semantic similarity.
//
// Graph boosts measure operational importance.
//
// By combining them, the final score considers both:
//
//     "Is this chunk relevant to the question?"
//
// and:
//
//     "Is the connected operational graph context important?"
//
// -----------------------------------------------------------------------------
// Why keep individual boost fields?
// -----------------------------------------------------------------------------
// We carry pageRankBoost, degreeBoost, and betweennessBoost forward separately.
//
// This is important for explainability.
//
// Instead of only seeing a final score, users can inspect how the score was
// formed.
//
// Production note:
//
// In a real production system, these boost weights should be tuned carefully.
// Scores may need normalization so that one metric does not dominate the final
// ranking unfairly.

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

// =============================================================================
// PACKAGE RETRIEVED RESULTS INTO STRUCTURED CONTEXT OBJECTS
// =============================================================================
// This WITH clause transforms multiple retrieved rows into a structured list.
//
// The key expression is:
//
//     collect({ ... }) AS retrievedContext
//
// This means:
//
//     "For each request, collect all retrieved chunks and their context into
//      a list of maps."
//
// This is different from returning one flat row per chunk.
//
// Instead, the output becomes grouped by request.
//
// -----------------------------------------------------------------------------
// Why this is useful
// -----------------------------------------------------------------------------
// This structure is very useful for API-style responses.
//
// For example, instead of returning:
//
//     login row 1
//     login row 2
//     payment row 1
//     payment row 2
//
// we can return:
//
//     login -> [retrieved context objects]
//     payment -> [retrieved context objects]
//     app_crash -> [retrieved context objects]
//
// This is cleaner for applications, dashboards, and RAG pipelines.
//
// -----------------------------------------------------------------------------
// chunkId
// -----------------------------------------------------------------------------
// Stores the unique identifier of the retrieved DocumentChunk.
//
// This supports traceability.
//
// -----------------------------------------------------------------------------
// text
// -----------------------------------------------------------------------------
// Stores the retrieved chunk text.
//
// This is the actual semantic content that matched the query vector.
//
// -----------------------------------------------------------------------------
// vectorScore
// -----------------------------------------------------------------------------
// Stores the original vector similarity score.
//
// This shows pure semantic relevance.
//
// -----------------------------------------------------------------------------
// pageRankBoost
// -----------------------------------------------------------------------------
// Stores the PageRank-based graph importance boost.
//
// -----------------------------------------------------------------------------
// degreeBoost
// -----------------------------------------------------------------------------
// Stores the scaled degree-based connectivity boost.
//
// -----------------------------------------------------------------------------
// betweennessBoost
// -----------------------------------------------------------------------------
// Stores the scaled betweenness-based bridge importance boost.
//
// -----------------------------------------------------------------------------
// retrievalRankScore
// -----------------------------------------------------------------------------
// Stores the final hybrid ranking score.
//
// This is the combined score used to represent both semantic relevance and graph
// importance.
//
// -----------------------------------------------------------------------------
// article
// -----------------------------------------------------------------------------
// article is a nested map containing:
//
//     articleId
//     title
//
// This shows the parent KnowledgeArticle source for the retrieved chunk.
//
// -----------------------------------------------------------------------------
// issue
// -----------------------------------------------------------------------------
// issue is a nested map containing:
//
//     issueId
//     name
//     severity
//
// This explains what operational issue the knowledge article solves.
//
// -----------------------------------------------------------------------------
// operationalContext
// -----------------------------------------------------------------------------
// operationalContext is a nested map containing:
//
//     status
//     ticketIds
//     customers
//     products
//     assignedAgents
//
// This gives business and support context around the retrieved issue.
//
// -----------------------------------------------------------------------------
// status
// -----------------------------------------------------------------------------
// The status field is calculated using:
//
//     CASE WHEN ticketCount = 0 THEN ...
//
// If no tickets exist, it says:
//
//     "No related ticket found for this issue"
//
// Otherwise, it says:
//
//     "Related ticket context found"
//
// -----------------------------------------------------------------------------
// analyticsContext
// -----------------------------------------------------------------------------
// analyticsContext contains the detailed ticket analytics maps collected earlier.
//
// This preserves graph data science outputs such as:
//
//     fullDegreeScore
//     pageRankScore
//     betweennessScore
//     louvainCommunityId
//     labelPropagationCommunityId

RETURN
  request.queryType AS queryType,
  request.userQuestion AS userQuestion,
  request.topK AS top

// =============================================================================
// RETURN GROUPED REQUEST-LEVEL OUTPUT
// =============================================================================
// This RETURN clause begins returning one grouped result per request.
//
// -----------------------------------------------------------------------------
// request.queryType AS queryType
// -----------------------------------------------------------------------------
// Returns the category of the request.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This tells us which retrieval scenario the grouped context belongs to.
//
// -----------------------------------------------------------------------------
// request.userQuestion AS userQuestion
// -----------------------------------------------------------------------------
// Returns the original user question.
//
// This keeps the final grouped result connected to the user's original intent.
//
// -----------------------------------------------------------------------------
// request.topK AS top
// -----------------------------------------------------------------------------
// Returns the topK value used for the request.
//
// In your pasted query, this line ends as:
//
//     request.topK AS top
//
// This is syntactically valid as an alias, but your full query appears to be
// cut off here.
//
// Most likely, the final RETURN was intended to continue with:
//
//     retrievedContext
//
// For example:
//
//     RETURN
//       request.queryType AS queryType,
//       request.userQuestion AS userQuestion,
//       request.topK AS topK,
//       retrievedContext
//
// If you want the structured collected context to appear in the output, make
// sure retrievedContext is included in the final RETURN.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query builds a structured, graph-enriched retrieval response.
//
// The main purpose is:
//
//     "Run multiple vector searches,
//      enrich each retrieved chunk with graph context,
//      calculate graph-boosted ranking scores,
//      and package the results as grouped retrieval context per user question."
//
// This is a strong pattern for:
// - Explainable RAG
// - Retrieval APIs
// - Support intelligence dashboards
// - Batch retrieval testing
// - Graph-enhanced semantic search
// - Hybrid vector + graph ranking
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
// =============================================================================
// BATCH EXPLAINABLE VECTOR RETRIEVAL WITH COMPACT SUMMARY OUTPUT
// =============================================================================
// This query runs multiple vector retrieval requests in one execution and then
// returns a compact summary for each request.
//
// In simple terms, it tells Neo4j:
//
//     "For each user question,
//      retrieve the most relevant document chunks using vector search,
//      enrich each chunk with article, issue, ticket, customer, product,
//      and agent context,
//      calculate a graph-boosted retrieval score,
//      group the retrieved context per request,
//      and finally return summary lists for quick inspection."
//
// This query is slightly different from earlier detailed versions.
//
// Earlier queries returned one row per retrieved chunk or returned full nested
// context objects.
//
// This query returns one row per query type and summarizes the retrieved context
// into lists such as:
//
//     retrievedChunkIds
//     retrievedIssueNames
//     retrievedTicketIds
//     operationalStatuses
//     retrievalRankScores
//
// This is useful when we want a clean overview instead of a very detailed
// row-by-row result.
//
// For example, instead of seeing:
//
//     login     -> chunk 1
//     login     -> chunk 2
//     payment   -> chunk 1
//     payment   -> chunk 2
//     app_crash -> chunk 1
//     app_crash -> chunk 2
//
// we get:
//
//     login     -> [chunk 1, chunk 2]
//     payment   -> [chunk 1, chunk 2]
//     app_crash -> [chunk 1, chunk 2]
//
// This makes the result easier to read during demos, validation checks, and
// retrieval quality testing.
//
// This query is useful for:
// - Batch vector retrieval testing
// - Compact RAG result inspection
// - Comparing multiple query types side by side
// - Checking whether graph context exists for each retrieved result
// - Verifying chunk-to-article-to-issue relationships
// - Summarizing ticket/customer/product/agent enrichment
// - Validating graph-boosted retrieval rank scores

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

// =============================================================================
// DEFINE MULTIPLE RETRIEVAL REQUESTS USING UNWIND
// =============================================================================
// UNWIND takes a list and expands each item in that list into its own row.
//
// Here, the list contains three maps.
//
// Each map represents one retrieval request:
//
//     1. login
//     2. payment
//     3. app_crash
//
// After UNWIND runs, Neo4j processes the rest of the query once per request.
//
// The current request is stored in the variable:
//
//     request
//
// -----------------------------------------------------------------------------
// Why use UNWIND here?
// -----------------------------------------------------------------------------
// UNWIND is useful when we want to process multiple inputs using the same logic.
//
// Without UNWIND, we would need to write separate queries or UNION ALL branches
// for each query type.
//
// With UNWIND, we define the requests once and let Cypher process them in a
// repeatable way.
//
// If we want to add another query type later, such as:
//
//     refund
//     password_reset
//     account_locked
//     performance
//
// we only add another map to the list.
//
// The remaining query does not need to change.
//
// -----------------------------------------------------------------------------
// Why each item is a map
// -----------------------------------------------------------------------------
// Each request is written as a map because a map keeps related fields together.
//
// Each request contains:
//
//     queryType
//     userQuestion
//     queryVector
//     topK
//
// This is cleaner than maintaining separate lists for query types, questions,
// vectors, and topK values.
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// queryType identifies the request category.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This field is returned later so that each summary row clearly shows which
// retrieval scenario it belongs to.
//
// -----------------------------------------------------------------------------
// userQuestion
// -----------------------------------------------------------------------------
// userQuestion stores the original human-readable question.
//
// Example:
//
//     "Why can't customers log in?"
//
// The vector index does not use this raw English text directly.
//
// The vector index uses:
//
//     queryVector
//
// Still, keeping userQuestion is important for traceability.
//
// It helps us understand which human question produced the vector search.
//
// -----------------------------------------------------------------------------
// queryVector
// -----------------------------------------------------------------------------
// queryVector stores the numeric embedding used for semantic search.
//
// In this lab/demo query, the vectors are hardcoded.
//
// In a production RAG system, this vector would usually be generated from the
// userQuestion by an embedding model.
//
// Example production flow:
//
//     User asks a question
//       -> embedding model converts it into a vector
//       -> Neo4j vector index retrieves similar chunks
//       -> graph context enriches the result
//
// -----------------------------------------------------------------------------
// topK
// -----------------------------------------------------------------------------
// topK controls how many chunks should be retrieved for each request.
//
// Here each request uses:
//
//     topK: 2
//
// So each query type asks Neo4j to retrieve the top 2 most similar chunks.

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)

// =============================================================================
// RUN VECTOR SIMILARITY SEARCH FOR EACH REQUEST
// =============================================================================
// db.index.vector.queryNodes() performs vector search against a Neo4j vector
// index.
//
// In simple terms, it asks:
//
//     "Which DocumentChunk nodes have embeddings closest to this request's
//      query vector?"
//
// Because this CALL comes after UNWIND, it runs once for each request.
//
// So this single query performs three vector searches:
//
//     login     -> [0.92, 0.12, 0.05]
//     payment   -> [0.08, 0.92, 0.12]
//     app_crash -> [0.12, 0.10, 0.92]
//
// -----------------------------------------------------------------------------
// 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// This is the vector index name.
//
// It is expected to be created on DocumentChunk embeddings.
//
// The index allows Neo4j to find semantically similar chunks efficiently.
//
// -----------------------------------------------------------------------------
// request.topK
// -----------------------------------------------------------------------------
// This value controls how many nearest chunks to return for the current request.
//
// Since request.topK is 2 for each request, Neo4j returns up to two matching
// chunks per query type.
//
// -----------------------------------------------------------------------------
// request.queryVector
// -----------------------------------------------------------------------------
// This is the current request's embedding vector.
//
// Neo4j compares this vector against stored chunk embeddings.
//
// Important beginner note:
//
//     Neo4j is not comparing the English sentence directly here.
//
// It is comparing numeric vectors that represent semantic meaning.

YIELD node AS dc, score

// =============================================================================
// CAPTURE VECTOR SEARCH OUTPUT
// =============================================================================
// YIELD extracts values returned by the vector search procedure.
//
// The procedure returns:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the graph node returned by the vector index.
//
// We rename it to:
//
//     dc
//
// because it represents a DocumentChunk.
//
// This makes the rest of the query easier to read.
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the similarity score between the request.queryVector and the
// retrieved chunk's stored embedding.
//
// A higher score usually means a stronger semantic match.
//
// This score is later stored as:
//
//     vectorScore
//
// and also contributes to:
//
//     retrievalRankScore

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// CONNECT RETRIEVED CHUNK TO KNOWLEDGE ARTICLE AND ISSUE
// =============================================================================
// This MATCH clause enriches the retrieved chunk with graph context.
//
// The pattern is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// This means:
//
//     "The retrieved document chunk is part of a knowledge article,
//      and that knowledge article solves a specific issue."
//
// -----------------------------------------------------------------------------
// Why this matters
// -----------------------------------------------------------------------------
// Vector search alone only tells us:
//
//     "This chunk is semantically similar."
//
// This graph pattern tells us:
//
//     "This chunk is semantically similar,
//      it came from this knowledge article,
//      and that article solves this issue."
//
// This makes the result explainable and traceable.
//
// -----------------------------------------------------------------------------
// PART_OF
// -----------------------------------------------------------------------------
// PART_OF connects a smaller chunk back to its parent KnowledgeArticle.
//
// This is important because chunks are often created by splitting long articles
// into smaller pieces for retrieval.
//
// -----------------------------------------------------------------------------
// SOLVES
// -----------------------------------------------------------------------------
// SOLVES connects a KnowledgeArticle to the Issue it addresses.
//
// This gives business meaning to the retrieved chunk.
//
// It shows what problem the retrieved knowledge is intended to solve.

OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

// =============================================================================
// OPTIONALLY FIND TICKETS RELATED TO THE ISSUE
// =============================================================================
// This OPTIONAL MATCH finds tickets connected to the retrieved issue.
//
// The pattern means:
//
//     "Find Ticket nodes that have this Issue."
//
// OPTIONAL MATCH is used because not every issue may have related tickets.
//
// If a normal MATCH were used, any retrieved issue without tickets would be
// removed from the result.
//
// OPTIONAL MATCH keeps the retrieved result and uses null where ticket context
// is missing.
//
// Ticket context is useful because it provides operational evidence behind the
// retrieved issue.

OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)

// =============================================================================
// OPTIONALLY FIND CUSTOMERS WHO RAISED RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds customers connected to related tickets.
//
// The pattern means:
//
//     "Find customers who raised these tickets."
//
// Customer context helps answer:
//
//     "Who was affected by this issue?"
//
// This is useful for business impact analysis and escalation decisions.

OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)

// =============================================================================
// OPTIONALLY FIND PRODUCTS RELATED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds products connected to related tickets.
//
// The pattern means:
//
//     "Find the product this ticket is about."
//
// Product context helps identify which application, service, or business
// capability is impacted.

OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

// =============================================================================
// OPTIONALLY FIND AGENTS ASSIGNED TO RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds agents assigned to related tickets.
//
// The pattern means:
//
//     "Find the support agent assigned to this ticket."
//
// Agent context helps identify ownership and follow-up responsibility.

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

// =============================================================================
// AGGREGATE RELATED CONTEXT AND CALCULATE GRAPH-BASED BOOSTS
// =============================================================================
// This WITH clause groups the optional graph context and calculates ranking
// boost values.
//
// Before this WITH clause, the OPTIONAL MATCH clauses may create multiple rows
// because one issue can be connected to multiple tickets, customers, products,
// and agents.
//
// This WITH clause compresses those rows into one grouped result per retrieved
// chunk/article/issue combination.
//
// -----------------------------------------------------------------------------
// request
// -----------------------------------------------------------------------------
// Carries the original request forward.
//
// We need this later to return:
//
//     queryType
//     userQuestion
//     topK
//
// -----------------------------------------------------------------------------
// dc
// -----------------------------------------------------------------------------
// Carries the retrieved DocumentChunk forward.
//
// We need this later for:
//
//     dc.chunkId
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// Carries the vector similarity score forward.
//
// This becomes:
//
//     vectorScore
//
// and contributes to:
//
//     retrievalRankScore
//
// -----------------------------------------------------------------------------
// ka
// -----------------------------------------------------------------------------
// Carries the parent KnowledgeArticle forward.
//
// We need its articleId later.
//
// -----------------------------------------------------------------------------
// i
// -----------------------------------------------------------------------------
// Carries the Issue forward.
//
// We need its name later.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT t.ticketId) AS relatedTicketIds
// -----------------------------------------------------------------------------
// Collects all related ticket IDs into a list.
//
// DISTINCT removes duplicates caused by row expansion.
//
// This gives us a clean list of tickets connected to the retrieved issue.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT c.name) AS relatedCustomers
// -----------------------------------------------------------------------------
// Collects customer names connected through those tickets.
//
// This provides customer impact context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT p.name) AS relatedProducts
// -----------------------------------------------------------------------------
// Collects product names connected through those tickets.
//
// This provides product impact context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT a.name) AS assignedAgents
// -----------------------------------------------------------------------------
// Collects agent names assigned to those tickets.
//
// This provides ownership context.
//
// -----------------------------------------------------------------------------
// count(t) AS ticketCount
// -----------------------------------------------------------------------------
// Counts how many ticket rows were matched.
//
// This value is later used to decide whether operational context exists.
//
// If ticketCount is 0, then no related tickets were found.
//
// -----------------------------------------------------------------------------
// coalesce(max(t.pageRankScore), 0.0) AS pageRankBoost
// -----------------------------------------------------------------------------
// This calculates the PageRank-based boost.
//
// max(t.pageRankScore) finds the highest PageRank score among related tickets.
//
// coalesce(..., 0.0) converts null to 0.0 when no tickets or no PageRank values
// exist.
//
// PageRank usually represents graph importance or influence.
//
// A higher PageRank boost means the retrieved issue is connected to at least one
// graph-important ticket.
//
// -----------------------------------------------------------------------------
// coalesce(max(t.fullDegreeScore), 0.0) * 0.01 AS degreeBoost
// -----------------------------------------------------------------------------
// This calculates the degree-based boost.
//
// fullDegreeScore usually represents how connected a ticket is in the graph.
//
// The multiplication by:
//
//     0.01
//
// scales the boost down.
//
// This is important because raw degree values can be much larger than vector
// similarity scores.
//
// Scaling keeps degree as a controlled ranking signal instead of letting it
// dominate the final score.
//
// -----------------------------------------------------------------------------
// coalesce(max(t.betweennessScore), 0.0) * 0.01 AS betweennessBoost
// -----------------------------------------------------------------------------
// This calculates the betweenness-based boost.
//
// Betweenness usually identifies bridge-like nodes.
//
// A high-betweenness ticket may connect different issue clusters or customer
// groups.
//
// The multiplication by 0.01 keeps this boost controlled.
//
// -----------------------------------------------------------------------------
// Production note
// -----------------------------------------------------------------------------
// These boost formulas are simple and useful for a demo.
//
// In production, the boost values should usually be normalized and tuned.
//
// Otherwise, one graph metric may accidentally dominate semantic similarity.

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

// =============================================================================
// BUILD A COMPACT CONTEXT ITEM FOR EACH RETRIEVED CHUNK
// =============================================================================
// This WITH clause creates one map called:
//
//     contextItem
//
// A map is a structured object containing key-value pairs.
//
// In simple terms, this section says:
//
//     "For this retrieved chunk, package the most important fields into one
//      compact object."
//
// This is useful because later we can collect all contextItem maps into a list
// per request.
//
// -----------------------------------------------------------------------------
// chunkId
// -----------------------------------------------------------------------------
// Stores the ID of the retrieved DocumentChunk.
//
// This helps identify exactly which chunk was retrieved.
//
// -----------------------------------------------------------------------------
// articleId
// -----------------------------------------------------------------------------
// Stores the ID of the parent KnowledgeArticle.
//
// This gives source traceability.
//
// -----------------------------------------------------------------------------
// issueName
// -----------------------------------------------------------------------------
// Stores the name of the Issue solved by the article.
//
// This tells us what problem the retrieved knowledge relates to.
//
// -----------------------------------------------------------------------------
// vectorScore
// -----------------------------------------------------------------------------
// Stores the original semantic similarity score from vector search.
//
// This helps us understand pure vector relevance.
//
// -----------------------------------------------------------------------------
// retrievalRankScore
// -----------------------------------------------------------------------------
// Stores the hybrid ranking score.
//
// The formula is:
//
//     vectorScore + pageRankBoost + degreeBoost + betweennessBoost
//
// This combines semantic similarity with graph-based operational importance.
//
// -----------------------------------------------------------------------------
// ticketIds
// -----------------------------------------------------------------------------
// Stores the list of related ticket IDs.
//
// This shows whether the issue has real support ticket evidence.
//
// -----------------------------------------------------------------------------
// customers
// -----------------------------------------------------------------------------
// Stores the list of related customer names.
//
// This helps measure customer impact.
//
// -----------------------------------------------------------------------------
// products
// -----------------------------------------------------------------------------
// Stores the list of related product names.
//
// This helps identify affected products or services.
//
// -----------------------------------------------------------------------------
// assignedAgents
// -----------------------------------------------------------------------------
// Stores the list of assigned support agents.
//
// This helps identify operational ownership.
//
// -----------------------------------------------------------------------------
// operationalStatus
// -----------------------------------------------------------------------------
// Stores a human-readable status message.
//
// If no tickets are found, the value is:
//
//     "No related ticket found for this issue"
//
// Otherwise, the value is:
//
//     "Related ticket context found"
//
// This makes the output easier to understand without manually inspecting the
// ticketIds list.

WITH
  request,
  collect(contextItem) AS retrievedContext

// =============================================================================
// GROUP CONTEXT ITEMS BY REQUEST
// =============================================================================
// This WITH clause collects all contextItem maps for each request.
//
// The result is:
//
//     retrievedContext
//
// which is a list of compact context objects.
//
// Because request is carried forward and contextItem is collected, the query now
// produces one grouped row per request.
//
// For example:
//
//     login     -> [contextItem1, contextItem2]
//     payment   -> [contextItem1, contextItem2]
//     app_crash -> [contextItem1, contextItem2]
//
// -----------------------------------------------------------------------------
// Why this is useful
// -----------------------------------------------------------------------------
// Grouping retrieved context per request makes the final output compact.
//
// This is especially helpful when comparing multiple query types in one result
// table.
//
// It also resembles how an API might return retrieval results grouped under the
// original request.

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

// =============================================================================
// RETURN COMPACT SUMMARY OUTPUT
// =============================================================================
// This RETURN clause produces one final summary row per request.
//
// Instead of returning full nested context objects, it extracts selected fields
// from retrievedContext into separate summary lists.
//
// This is useful when we want to quickly inspect retrieval behavior across
// multiple query types.
//
// -----------------------------------------------------------------------------
// request.queryType AS queryType
// -----------------------------------------------------------------------------
// Returns the query category.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This identifies which retrieval scenario the summary row belongs to.
//
// -----------------------------------------------------------------------------
// request.userQuestion AS userQuestion
// -----------------------------------------------------------------------------
// Returns the original human-readable question.
//
// This keeps the summary connected to user intent.
//
// -----------------------------------------------------------------------------
// request.topK AS topK
// -----------------------------------------------------------------------------
// Returns the topK value used for vector retrieval.
//
// Here topK is 2 for each request.
//
// This tells us how many chunks were requested from the vector index.
//
// -----------------------------------------------------------------------------
// size(retrievedContext) AS retrievedContextCount
// -----------------------------------------------------------------------------
// size() returns how many context items were collected.
//
// Since topK is 2, this count will usually be 2 per request if both retrieved
// chunks successfully matched to article and issue context.
//
// If the count is lower than topK, it may indicate that some retrieved chunks
// did not have the expected PART_OF or SOLVES relationships.
//
// This is useful for validation.
//
// -----------------------------------------------------------------------------
// [item IN retrievedContext | item.chunkId] AS retrievedChunkIds
// -----------------------------------------------------------------------------
// This is a list comprehension.
//
// It loops through each item in retrievedContext and extracts:
//
//     item.chunkId
//
// The result is a list of retrieved chunk IDs.
//
// This helps quickly verify which chunks were retrieved for each query type.
//
// -----------------------------------------------------------------------------
// [item IN retrievedContext | item.issueName] AS retrievedIssueNames
// -----------------------------------------------------------------------------
// This extracts the issue name from each context item.
//
// It helps quickly verify whether the vector search returned issue-appropriate
// chunks.
//
// For example:
// - login query should ideally return login-related issue names
// - payment query should ideally return payment-related issue names
// - app_crash query should ideally return app-crash-related issue names
//
// -----------------------------------------------------------------------------
// [item IN retrievedContext | item.ticketIds] AS retrievedTicketIds
// -----------------------------------------------------------------------------
// This extracts the ticketIds list from each context item.
//
// Because each context item can itself contain multiple ticket IDs, the final
// value may be a list of lists.
//
// Example shape:
//
//     [
//       ["T-001", "T-002"],
//       ["T-003"]
//     ]
//
// This helps inspect operational evidence behind each retrieved chunk.
//
// -----------------------------------------------------------------------------
// [item IN retrievedContext | item.operationalStatus] AS operationalStatuses
// -----------------------------------------------------------------------------
// This extracts the operational status for each retrieved context item.
//
// Example values:
//
//     "Related ticket context found"
//     "No related ticket found for this issue"
//
// This helps quickly see whether the retrieved issues have ticket context.
//
// -----------------------------------------------------------------------------
// [item IN retrievedContext | item.retrievalRankScore] AS retrievalRankScores
// -----------------------------------------------------------------------------
// This extracts the final hybrid ranking score for each retrieved context item.
//
// This helps compare graph-boosted ranking values within each query type.
//
// Important note:
//
// The retrievedContext list is collected without an explicit ORDER BY before
// collection.
//
// If you need retrievalRankScores and chunk IDs to appear in strict ranked
// order inside the lists, add an ORDER BY before the collect(contextItem) step.
//
// For example, sort by:
//
//     request.queryType,
//     contextItem.retrievalRankScore DESC
//
// before collecting.

ORDER BY
  queryType;

// =============================================================================
// ORDER FINAL SUMMARY ROWS BY QUERY TYPE
// =============================================================================
// ORDER BY controls the order of the final result rows.
//
// Here the query orders by:
//
//     queryType
//
// This groups the output alphabetically by query type.
//
// Example order may be:
//
//     app_crash
//     login
//     payment
//
// This makes the final summary easier to scan.
//
// -----------------------------------------------------------------------------
// Important note about ordering inside lists
// -----------------------------------------------------------------------------
// This ORDER BY only controls the order of the final rows.
//
// It does not guarantee the order of items inside:
//
//     retrievedChunkIds
//     retrievedIssueNames
//     retrievedTicketIds
//     operationalStatuses
//     retrievalRankScores
//
// Those internal list orders depend on the order in which rows were collected.
//
// If deterministic ordering inside the lists is important, introduce an ORDER BY
// before:
//
//     collect(contextItem)
//
// using the desired ranking field.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query creates a compact, grouped summary of batch vector retrieval
// results.
//
// The main purpose is:
//
//     "Run multiple vector searches,
//      enrich each result with graph context,
//      compute hybrid retrieval rank scores,
//      group results per query type,
//      and return summary lists for quick validation."
//
// It is useful for:
// - Checking retrieval coverage
// - Comparing multiple query types
// - Validating expected issue matches
// - Verifying ticket context availability
// - Reviewing graph-boosted ranking scores
// - Producing concise demo-friendly output
//
// This is a good pattern when the goal is not to inspect every detail, but to
// quickly validate whether each query type retrieved the expected chunks,
// issues, ticket context, and ranking scores.
```

# Step 6 — Format grouped context as LLM-ready context

```cypher
// =============================================================================
// BATCH EXPLAINABLE VECTOR RETRIEVAL WITH LLM-READY CONTEXT PREPARATION
// =============================================================================
// This query runs multiple vector retrieval requests and prepares a structured,
// LLM-ready context package for each request.
//
// In simple terms, it tells Neo4j:
//
//     "For each user question,
//      retrieve the most relevant document chunks,
//      enrich them with graph context,
//      calculate ranking signals,
//      separate evidence from issue context,
//      and return a compact package that can be passed to an LLM."
//
// This query is very close to a production-style RAG backend pattern.
//
// It does not simply return raw vector search rows.
//
// Instead, it organizes the output into:
//
//     1. retrievedEvidence
//        The actual retrieved text chunks and their scores.
//
//     2. issueContext
//        The structured issue objects connected to those chunks.
//
//     3. llmReadyContext
//        A clear instruction string telling an LLM how to use the retrieved
//        evidence safely.
//
//     4. warning and count fields
//        These help detect retrieval or context-quality issues.
//
// This is useful because LLMs should not receive unstructured database output
// without guidance.
//
// A good RAG pipeline should clearly separate:
// - What was retrieved
// - Why it was retrieved
// - What business issue it relates to
// - What operational context exists
// - What limitations or warnings should be respected

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

// =============================================================================
// DEFINE MULTIPLE RETRIEVAL REQUESTS USING UNWIND
// =============================================================================
// UNWIND expands a list into individual rows.
//
// Here, the list contains three maps.
//
// Each map represents one retrieval request:
//
//     1. login
//     2. payment
//     3. app_crash
//
// After UNWIND executes, Neo4j processes the remaining query once per request.
//
// The current request is stored in the variable:
//
//     request
//
// -----------------------------------------------------------------------------
// Why use UNWIND here?
// -----------------------------------------------------------------------------
// UNWIND lets us run the same retrieval pipeline for multiple inputs.
//
// Without UNWIND, we would need to write separate queries or UNION ALL blocks
// for each query type.
//
// With UNWIND, the query becomes easier to extend.
//
// For example, if we later want to test:
//
//     refund
//     password_reset
//     account_locked
//     performance
//
// we can simply add more maps to this list.
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// queryType identifies the category of the user question.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This is useful because the final result will contain one row per request
// category.
//
// -----------------------------------------------------------------------------
// userQuestion
// -----------------------------------------------------------------------------
// userQuestion stores the original human-readable question.
//
// Example:
//
//     "Why can't customers log in?"
//
// The vector index does not directly search this text.
//
// Instead, it searches using:
//
//     queryVector
//
// Still, keeping userQuestion is important for traceability, logging,
// debugging, evaluation, and LLM prompt construction.
//
// -----------------------------------------------------------------------------
// queryVector
// -----------------------------------------------------------------------------
// queryVector stores the numeric embedding used for semantic vector search.
//
// In this lab/demo, the vectors are hardcoded.
//
// In a real production RAG system, these vectors would normally be generated by
// an embedding model from the userQuestion.
//
// Production flow:
//
//     userQuestion
//       -> embedding model
//       -> queryVector
//       -> Neo4j vector index
//       -> retrieved chunks
//       -> graph-enriched context
//       -> LLM-ready answer context
//
// -----------------------------------------------------------------------------
// topK
// -----------------------------------------------------------------------------
// topK controls how many nearest chunks should be retrieved for each request.
//
// Here each request uses:
//
//     topK: 2
//
// This means Neo4j will retrieve the top 2 closest DocumentChunk nodes for each
// query type.

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)

// =============================================================================
// RUN VECTOR SIMILARITY SEARCH FOR EACH REQUEST
// =============================================================================
// db.index.vector.queryNodes() searches a Neo4j vector index for nodes whose
// embeddings are closest to the given query vector.
//
// In simple terms, this says:
//
//     "Find the DocumentChunk nodes that are semantically closest to the current
//      request's query vector."
//
// Because this CALL happens after UNWIND, it runs once per request.
//
// So this single query performs vector search for:
//
//     login     -> [0.92, 0.12, 0.05]
//     payment   -> [0.08, 0.92, 0.12]
//     app_crash -> [0.12, 0.10, 0.92]
//
// -----------------------------------------------------------------------------
// First argument: 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// This is the name of the vector index.
//
// The index is expected to be created on DocumentChunk nodes, most likely on
// their embedding property.
//
// The vector index makes semantic search efficient.
//
// Without the index, Neo4j would need to compare the query vector against many
// stored vectors manually.
//
// -----------------------------------------------------------------------------
// Second argument: request.topK
// -----------------------------------------------------------------------------
// request.topK tells Neo4j how many nearest matching chunks to return.
//
// Since every request has:
//
//     topK: 2
//
// the procedure returns up to two matching chunks per query type.
//
// -----------------------------------------------------------------------------
// Third argument: request.queryVector
// -----------------------------------------------------------------------------
// request.queryVector is the numeric embedding for the current request.
//
// Important beginner note:
//
//     Neo4j is not directly comparing English text here.
//
// It is comparing vectors.
//
// Vectors are numeric representations of meaning.

YIELD node AS dc, score

// =============================================================================
// CAPTURE VECTOR SEARCH OUTPUT
// =============================================================================
// YIELD extracts values returned by the vector search procedure.
//
// The procedure returns:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the matching graph node returned by the vector index.
//
// We rename it to:
//
//     dc
//
// because the returned node represents a DocumentChunk.
//
// This makes the query easier to understand.
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the similarity score between:
//
//     request.queryVector
//
// and the stored embedding of the retrieved DocumentChunk.
//
// A higher score generally means stronger semantic similarity.
//
// This score later appears as:
//
//     vectorScore
//
// and also contributes to:
//
//     retrievalRankScore

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// CONNECT RETRIEVED CHUNK TO KNOWLEDGE ARTICLE AND ISSUE
// =============================================================================
// This MATCH clause enriches the retrieved DocumentChunk with graph context.
//
// The pattern is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// Meaning:
//
//     "The retrieved chunk is part of a knowledge article,
//      and that knowledge article solves a specific issue."
//
// -----------------------------------------------------------------------------
// Why this matters
// -----------------------------------------------------------------------------
// A vector result by itself only tells us:
//
//     "This chunk is semantically similar."
//
// But after this MATCH, we can say:
//
//     "This chunk is semantically similar,
//      it belongs to this knowledge article,
//      and that article solves this issue."
//
// This is the difference between plain vector search and explainable
// graph-powered retrieval.
//
// -----------------------------------------------------------------------------
// Production relevance
// -----------------------------------------------------------------------------
// In a production RAG system, this traceability is very important.
//
// Users and auditors should be able to understand:
// - Which source chunk was retrieved
// - Which article it came from
// - Which issue the article is meant to solve
// - Why the result is relevant to the question

OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

// =============================================================================
// OPTIONALLY FIND TICKETS RELATED TO THE ISSUE
// =============================================================================
// This OPTIONAL MATCH finds Ticket nodes connected to the retrieved Issue.
//
// The pattern means:
//
//     "Find tickets that have this issue."
//
// -----------------------------------------------------------------------------
// Why OPTIONAL MATCH?
// -----------------------------------------------------------------------------
// OPTIONAL MATCH behaves like a left join.
//
// It tries to find the pattern, but if the pattern does not exist, the existing
// row is preserved and the missing values become null.
//
// This is important because not every Issue is guaranteed to have related
// tickets.
//
// If we used normal MATCH, then any issue without tickets would disappear from
// the retrieval results.
//
// That would be dangerous because a retrieved chunk can still be valid even if
// no operational ticket context exists.
//
// -----------------------------------------------------------------------------
// Why ticket context matters
// -----------------------------------------------------------------------------
// Tickets provide operational evidence.
//
// They help answer:
// - Has this issue happened in real support cases?
// - How many tickets are connected to it?
// - What priority do those tickets have?
// - Are those tickets open, closed, or in progress?
// - Is this issue operationally active?

OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)

// =============================================================================
// OPTIONALLY FIND CUSTOMERS WHO RAISED RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Customer nodes connected to related tickets.
//
// The pattern means:
//
//     "Find customers who raised these tickets."
//
// Customer context helps answer:
//
//     "Who was affected by this issue?"
//
// This is important for impact analysis.
//
// For example, an issue affecting many customers may deserve higher priority
// than an issue affecting only one internal test case.

OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)

// =============================================================================
// OPTIONALLY FIND PRODUCTS RELATED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Product nodes connected to related tickets.
//
// The pattern means:
//
//     "Find the product that this ticket is about."
//
// Product context helps answer:
// - Which product is affected?
// - Is the issue isolated to one product?
// - Is the same issue affecting multiple products?
// - Which product team may need to investigate?

OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

// =============================================================================
// OPTIONALLY FIND AGENTS ASSIGNED TO RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Agent nodes assigned to related tickets.
//
// The pattern means:
//
//     "Find the support agent assigned to this ticket."
//
// Agent context helps answer:
// - Who owns the ticket?
// - Who has handled this issue before?
// - Is reassignment needed?
// - Is escalation needed?
//
// This is optional because some tickets may not be assigned yet.

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

// =============================================================================
// AGGREGATE OPERATIONAL CONTEXT AND CALCULATE GRAPH-BASED BOOSTS
// =============================================================================
// This WITH clause performs two jobs:
//
//     1. It groups operational graph context into clean lists.
//     2. It calculates graph-based ranking boost values.
//
// Before this WITH clause, OPTIONAL MATCH clauses may create multiple rows
// because one issue can connect to many tickets, customers, products, and
// agents.
//
// This WITH clause compresses those expanded rows into one grouped result per
// retrieved chunk/article/issue combination.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT t.ticketId) AS relatedTicketIds
// -----------------------------------------------------------------------------
// Collects all related ticket IDs into a clean list.
//
// DISTINCT removes duplicates caused by graph expansion.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT c.name) AS relatedCustomers
// -----------------------------------------------------------------------------
// Collects customer names connected through related tickets.
//
// This gives customer impact context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT p.name) AS relatedProducts
// -----------------------------------------------------------------------------
// Collects product names connected through related tickets.
//
// This gives product impact context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT a.name) AS assignedAgents
// -----------------------------------------------------------------------------
// Collects support agent names assigned to related tickets.
//
// This gives ownership context.
//
// -----------------------------------------------------------------------------
// count(t) AS ticketCount
// -----------------------------------------------------------------------------
// Counts how many ticket rows were found.
//
// This value is later used to create a human-readable operational status.
//
// -----------------------------------------------------------------------------
// pageRankBoost
// -----------------------------------------------------------------------------
// pageRankBoost uses the maximum PageRank score among related tickets.
//
// PageRank usually represents graph importance or influence.
//
// coalesce(..., 0.0) protects the calculation from null values.
//
// If there are no related tickets, max(t.pageRankScore) may be null.
// coalesce changes that null into 0.0.
//
// -----------------------------------------------------------------------------
// degreeBoost
// -----------------------------------------------------------------------------
// degreeBoost uses the maximum fullDegreeScore and scales it by 0.01.
//
// Degree usually represents how connected a ticket is.
//
// The multiplication by 0.01 prevents the degree value from overwhelming the
// vector similarity score.
//
// -----------------------------------------------------------------------------
// betweennessBoost
// -----------------------------------------------------------------------------
// betweennessBoost uses the maximum betweennessScore and scales it by 0.01.
//
// Betweenness usually identifies bridge-like nodes.
//
// A ticket with high betweenness may connect different issue clusters or
// customer groups.
//
// -----------------------------------------------------------------------------
// Production note
// -----------------------------------------------------------------------------
// These boost formulas are useful for a lab/demo.
//
// In production, these values should usually be normalized and tuned carefully
// so that one metric does not dominate the final ranking unfairly.

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

// =============================================================================
// BUILD A STRUCTURED CONTEXT ITEM FOR EACH RETRIEVED CHUNK
// =============================================================================
// This WITH clause packages each retrieved chunk and its graph context into one
// structured map called:
//
//     contextItem
//
// A map is a key-value object.
//
// In simple terms, this section says:
//
//     "For this retrieved chunk,
//      create one clean object containing the evidence, source, issue,
//      operational context, and ranking signals."
//
// -----------------------------------------------------------------------------
// chunkId
// -----------------------------------------------------------------------------
// Stores the unique ID of the retrieved DocumentChunk.
//
// This supports source traceability.
//
// -----------------------------------------------------------------------------
// text
// -----------------------------------------------------------------------------
// Stores the actual retrieved chunk text.
//
// This is the evidence that may later be passed to an LLM.
//
// -----------------------------------------------------------------------------
// vectorScore
// -----------------------------------------------------------------------------
// Stores the original semantic similarity score.
//
// This shows how close the chunk was to the query vector.
//
// -----------------------------------------------------------------------------
// retrievalRankScore
// -----------------------------------------------------------------------------
// Stores the hybrid ranking score.
//
// The formula is:
//
//     vectorScore + pageRankBoost + degreeBoost + betweennessBoost
//
// This combines semantic relevance with graph-based operational importance.
//
// -----------------------------------------------------------------------------
// article
// -----------------------------------------------------------------------------
// article is a nested map containing:
//
//     articleId
//     title
//
// This identifies the source KnowledgeArticle.
//
// -----------------------------------------------------------------------------
// issue
// -----------------------------------------------------------------------------
// issue is a nested map containing:
//
//     issueId
//     name
//     severity
//
// This explains the operational issue solved by the article.
//
// -----------------------------------------------------------------------------
// operationalContext
// -----------------------------------------------------------------------------
// operationalContext is a nested map containing:
//
//     status
//     ticketIds
//     customers
//     products
//     assignedAgents
//
// This tells us whether related ticket context exists and who/what is affected.
//
// -----------------------------------------------------------------------------
// rankingSignals
// -----------------------------------------------------------------------------
// rankingSignals is a nested map containing the score components:
//
//     vectorScore
//     pageRankBoost
//     degreeBoost
//     betweennessBoost
//     retrievalRankScore
//
// This is very important for explainability.
//
// Instead of returning only a final rank, the query shows how the final rank was
// calculated.

WITH
  request,
  collect(contextItem) AS retrievedContext

// =============================================================================
// GROUP ALL CONTEXT ITEMS BY REQUEST
// =============================================================================
// This WITH clause collects all contextItem maps for each request.
//
// The result is:
//
//     retrievedContext
//
// retrievedContext is a list of context objects.
//
// Because request is carried forward and contextItem is collected, the query now
// has one grouped row per request.
//
// Example conceptual shape:
//
//     login -> [
//       {chunk 1 context},
//       {chunk 2 context}
//     ]
//
//     payment -> [
//       {chunk 1 context},
//       {chunk 2 context}
//     ]
//
//     app_crash -> [
//       {chunk 1 context},
//       {chunk 2 context}
//     ]
//
// -----------------------------------------------------------------------------
// Why this is useful
// -----------------------------------------------------------------------------
// This grouped structure is useful for RAG and APIs.
//
// A downstream application can receive one request row and its full retrieval
// context as a list.
//
// This is easier to consume than many separate flat rows.

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
  [
    item IN retrievedContext
    WHERE item.operationalContext.status = "No related ticket found for this issue"
    | "Warning: Retrieved issue has no related ticket context for chunk " + item.chunkId
  ] AS warnings

// =============================================================================
// CREATE LLM-READY EVIDENCE, ISSUE CONTEXT, SIGNALS, AND WARNINGS
// =============================================================================
// This WITH clause reshapes the grouped retrievedContext into separate logical
// sections.
//
// This is an important RAG design step.
//
// Instead of giving the LLM one messy object, we separate the output into:
//
//     retrievedEvidence
//     issueContext
//     operationalContext
//     rankingSignals
//     warnings
//
// This makes the final result easier for humans, applications, and LLMs to
// understand.
//
// -----------------------------------------------------------------------------
// retrievedEvidence
// -----------------------------------------------------------------------------
// retrievedEvidence extracts only the evidence fields needed for answer
// generation:
//
//     chunkId
//     text
//     vectorScore
//     retrievalRankScore
//
// This is the content-oriented part of the retrieval result.
//
// The text field is especially important because it contains the actual
// retrieved knowledge chunk.
//
// -----------------------------------------------------------------------------
// issueContext
// -----------------------------------------------------------------------------
// issueContext extracts the issue maps from each retrieved context item.
//
// Each issue object contains:
//
//     issueId
//     name
//     severity
//
// This helps the LLM understand what operational issue the retrieved evidence
// is connected to.
//
// -----------------------------------------------------------------------------
// operationalContext
// -----------------------------------------------------------------------------
// operationalContext extracts ticket/customer/product/agent context.
//
// This gives the LLM supporting operational information, such as:
// - Whether related tickets exist
// - Which tickets are related
// - Which customers are affected
// - Which products are involved
// - Which agents are assigned
//
// -----------------------------------------------------------------------------
// rankingSignals
// -----------------------------------------------------------------------------
// rankingSignals extracts the score breakdown.
//
// This helps explain why a chunk ranked where it did.
//
// It includes:
// - vectorScore
// - pageRankBoost
// - degreeBoost
// - betweiesBoost
// - retrievalRankScore
//
// Note:
// The field in the actual query is correctly named:
//
//     betweennessBoost
//
// This score helps explain bridge-like graph importance.
//
// -----------------------------------------------------------------------------
// warnings
// -----------------------------------------------------------------------------
// warnings is created using a list comprehension with a WHERE filter.
//
// It checks each retrieved context item.
//
// If the item has:
//
//     "No related ticket found for this issue"
//
// then the query creates a warning message:
//
//     "Warning: Retrieved issue has no related ticket context for chunk <chunkId>"
//
// This is very useful for LLM safety and quality.
//
// It tells the downstream assistant:
//
//     "The retrieved evidence exists,
//      but operational ticket support is missing for this item."
//
// This prevents the LLM from overstating operational certainty.

RETURN
  request.queryType AS queryType,
  request.userQuestion AS userQuestion,
  request.topK AS topK,
  size(retrievedContext) AS retrievedContextCount,
  retrievedEvidence,
  issueContext,
  operationalContext,
  rankingSignals,
  warnings,
  "Use only the provided retrieved evidence and graph context. If warnings exist, mention the data-quality limitation." AS llmReadyContext,
  size(warnings) AS warningCount

// =============================================================================
// RETURN FINAL LLM-READY RETRIEVAL PACKAGE
// =============================================================================
// This RETURN clause produces the final output.
//
// The output is one row per request.
//
// Each row contains:
// - The original query metadata
// - A compact evidence list
// - Structured issue context
// - Operational graph context
// - Ranking signals
// - Warning messages
// - LLM usage guidance
// - Warning count
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// Returns the request category.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This helps identify which retrieval scenario the row belongs to.
//
// -----------------------------------------------------------------------------
// userQuestion
// -----------------------------------------------------------------------------
// Returns the original natural-language question.
//
// This keeps the final result connected to the user's intent.
//
// -----------------------------------------------------------------------------
// topK
// -----------------------------------------------------------------------------
// Returns how many chunks were requested from the vector index.
//
// Here, each request uses:
//
//     topK = 2
//
// -----------------------------------------------------------------------------
// retrievedContextCount
// -----------------------------------------------------------------------------
// Returns the number of retrieved context items collected for the request.
//
// In an ideal case, this should match topK.
//
// If retrievedContextCount is lower than topK, it may indicate that some vector
// results did not have the expected graph relationships, such as:
//
//     PART_OF
//     SOLVES
//
// This makes it useful for validation.
//
// -----------------------------------------------------------------------------
// retrievedEvidence
// -----------------------------------------------------------------------------
// Returns the retrieved chunk evidence.
//
// This contains:
// - chunk ID
// - retrieved text
// - vector score
// - hybrid retrieval rank score
//
// This is the main evidence an LLM should use when answering.
//
// -----------------------------------------------------------------------------
// issueContext
// -----------------------------------------------------------------------------
// Returns the structured issue information connected to the evidence.
//
// This helps the LLM understand what issue the retrieved chunks relate to.
//
// -----------------------------------------------------------------------------
// operationalContext
// -----------------------------------------------------------------------------
// Returns ticket/customer/product/agent context.
//
// This helps the LLM understand real-world support impact.
//
// -----------------------------------------------------------------------------
// rankingSignals
// -----------------------------------------------------------------------------
// Returns the score breakdown for each retrieved item.
//
// This supports explainability because we can see how the final ranking was
// influenced by vector similarity and graph boosts.
//
// -----------------------------------------------------------------------------
// warnings
// -----------------------------------------------------------------------------
// Returns warning messages for retrieved items that do not have related ticket
// context.
//
// This is important because it tells the LLM and the user when operational
// evidence is incomplete.
//
// -----------------------------------------------------------------------------
// llmReadyContext
// -----------------------------------------------------------------------------
// This returns a fixed instruction string:
//
//     "Use only the provided retrieved evidence and graph context.
//      If warnings exist, mention the data-quality limitation."
//
// This is a guardrail for downstream LLM usage.
//
// It tells the LLM:
// - Do not invent facts.
// - Use only retrieved evidence.
// - Use graph context only.
// - If warnings exist, disclose the limitation.
//
// This is very important for safe and explainable RAG behavior.
//
// -----------------------------------------------------------------------------
// warningCount
// -----------------------------------------------------------------------------
// Returns the number of warning messages.
//
// This allows applications or dashboards to quickly detect whether the response
// has data-quality limitations.

ORDER BY
  queryType;

// =============================================================================
// ORDER FINAL RESULTS BY QUERY TYPE
// =============================================================================
// ORDER BY controls the order of final rows.
//
// Here the query orders by:
//
//     queryType
//
// This sorts the result rows alphabetically by query type.
//
// Example order may be:
//
//     app_crash
//     login
//     payment
//
// This makes the output predictable and easy to compare.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query prepares a clean, explainable, LLM-ready retrieval package.
//
// The main purpose is:
//
//     "Run batch vector retrieval,
//      enrich results with graph context,
//      calculate hybrid ranking signals,
//      separate evidence and context,
//      detect missing operational support,
//      and return instructions for safe LLM usage."
//
// This is a strong production-style RAG pattern because it includes:
//
// - Batch query handling
// - Vector similarity search
// - Source traceability
// - Issue-level explanation
// - Operational ticket context
// - Customer/product/agent enrichment
// - Hybrid vector + graph ranking
// - Ranking signal transparency
// - Warning generation
// - LLM guardrail instruction
//
// In production, this pattern helps reduce hallucination risk because the LLM is
// explicitly instructed to use only the retrieved evidence and graph context.
```

# Step 6B — Deduplicate repeated LLM-ready context fields

```cypher
// =============================================================================
// BATCH EXPLAINABLE VECTOR RETRIEVAL WITH DEDUPED LLM-READY CONTEXT
// =============================================================================
// This query runs multiple vector retrieval requests and prepares a clean,
// deduplicated, LLM-ready context object for each request.
//
// In simple terms, it tells Neo4j:
//
//     "For each user question,
//      retrieve relevant document chunks using vector search,
//      connect those chunks to knowledge articles and issues,
//      enrich them with operational graph context,
//      calculate ranking signals,
//      deduplicate repeated issue and operational context,
//      and return a structured package that can be safely passed to an LLM."
//
// This query is useful because raw graph retrieval can produce repeated context.
//
// For example, two retrieved chunks may belong to the same article or solve the
// same issue. If we pass duplicate issue details repeatedly to an LLM, the prompt
// becomes noisy and less efficient.
//
// This query solves that by separating:
//
// - retrievedEvidence:
//     The actual retrieved chunk-level evidence.
//
// - issueContext:
//     Deduplicated issue-level context.
//
// - operationalContext:
//     Deduplicated ticket/customer/product/agent context.
//
// - rankingSignals:
//     Score breakdowns that explain why each chunk ranked where it did.
//
// - warnings:
//     Deduplicated warnings about missing operational ticket context.
//
// - instructionForDay5LLM:
//     A guardrail instruction telling the LLM to use only retrieved evidence and
//     graph context.
//
// This is a strong production-style RAG pattern because it prepares structured,
// explainable, and safer context instead of sending unorganized database rows.

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

// =============================================================================
// DEFINE MULTIPLE RETRIEVAL REQUESTS USING UNWIND
// =============================================================================
// UNWIND expands a list into individual rows.
//
// Here, the list contains three maps:
//
//     1. login request
//     2. payment request
//     3. app_crash request
//
// After UNWIND runs, Neo4j processes the rest of the query once for each map.
//
// The current map is stored in the variable:
//
//     request
//
// -----------------------------------------------------------------------------
// Why use UNWIND?
// -----------------------------------------------------------------------------
// UNWIND lets us process multiple retrieval requests using one shared query
// pipeline.
//
// Without UNWIND, we would need separate queries or UNION ALL branches for each
// query type.
//
// With UNWIND, adding another retrieval test is simple.
//
// For example, to add:
//
//     refund
//     password_reset
//     performance
//
// we only add another map to the list.
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// queryType identifies the request category.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This helps group and interpret the final result.
//
// -----------------------------------------------------------------------------
// userQuestion
// -----------------------------------------------------------------------------
// userQuestion stores the original natural-language question.
//
// The vector search itself does not directly use this English text.
//
// Instead, it uses:
//
//     queryVector
//
// Still, keeping userQuestion is important for traceability because it tells us
// what human question produced the retrieval result.
//
// -----------------------------------------------------------------------------
// queryVector
// -----------------------------------------------------------------------------
// queryVector is the numeric embedding used for semantic search.
//
// In this lab/demo query, the vectors are hardcoded.
//
// In a real production RAG system, these vectors would normally be generated by
// an embedding model from userQuestion.
//
// -----------------------------------------------------------------------------
// topK
// -----------------------------------------------------------------------------
// topK controls how many nearest document chunks should be retrieved.
//
// Here, each request uses:
//
//     topK: 2
//
// So each query type asks Neo4j to retrieve the top 2 semantically closest
// DocumentChunk nodes.

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)

// =============================================================================
// RUN VECTOR SIMILARITY SEARCH FOR EACH REQUEST
// =============================================================================
// db.index.vector.queryNodes() searches a Neo4j vector index for nodes whose
// stored embedding vectors are closest to the provided query vector.
//
// In simple terms:
//
//     "Find the DocumentChunk nodes that are semantically closest to the current
//      request's queryVector."
//
// Because this CALL comes after UNWIND, it runs once per request.
//
// So this one query performs vector search for:
//
//     login
//     payment
//     app_crash
//
// -----------------------------------------------------------------------------
// 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// This is the vector index name.
//
// It is expected to be created on DocumentChunk nodes, usually on an embedding
// property.
//
// The vector index makes semantic similarity search efficient.
//
// -----------------------------------------------------------------------------
// request.topK
// -----------------------------------------------------------------------------
// request.topK tells Neo4j how many nearest matching chunks to return for the
// current request.
//
// Since topK is 2 for each request, Neo4j returns up to two chunks per query
// type.
//
// -----------------------------------------------------------------------------
// request.queryVector
// -----------------------------------------------------------------------------
// request.queryVector is the embedding vector for the current user question.
//
// Important beginner note:
//
//     Neo4j is not directly comparing English sentences here.
//
// It is comparing numeric vectors that represent meaning.

YIELD node AS dc, score

// =============================================================================
// CAPTURE VECTOR SEARCH OUTPUT
// =============================================================================
// YIELD extracts the values returned by db.index.vector.queryNodes().
//
// The procedure returns:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the matching graph node returned by the vector index.
//
// We rename it to:
//
//     dc
//
// because this node represents a DocumentChunk.
//
// This makes the rest of the query easier to read.
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the similarity score between the query vector and the retrieved
// chunk's stored embedding.
//
// A higher score generally means stronger semantic similarity.
//
// This score is later used as:
//
//     vectorScore
//
// and also contributes to:
//
//     retrievalRankScore

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// CONNECT RETRIEVED CHUNK TO KNOWLEDGE ARTICLE AND ISSUE
// =============================================================================
// This MATCH clause enriches each retrieved DocumentChunk with graph context.
//
// The path is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// Meaning:
//
//     "The retrieved chunk belongs to a knowledge article,
//      and that knowledge article solves a specific issue."
//
// -----------------------------------------------------------------------------
// Why this matters
// -----------------------------------------------------------------------------
// Vector search alone tells us:
//
//     "This chunk is semantically similar."
//
// This graph match tells us:
//
//     "This chunk is semantically similar,
//      it came from this knowledge article,
//      and that article solves this issue."
//
// This makes the result explainable and traceable.
//
// -----------------------------------------------------------------------------
// Production relevance
// -----------------------------------------------------------------------------
// This is a key graph-powered RAG pattern.
//
// It connects semantic retrieval to structured business meaning.

OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

// =============================================================================
// OPTIONALLY FIND TICKETS RELATED TO THE ISSUE
// =============================================================================
// This OPTIONAL MATCH finds Ticket nodes connected to the retrieved Issue.
//
// The pattern means:
//
//     "Find tickets that have this issue."
//
// OPTIONAL MATCH is used because not every Issue may have related tickets.
//
// If we used a normal MATCH, then issues without tickets would disappear from
// the result.
//
// That would be risky because a retrieved knowledge article can still be valid
// even if there is no operational ticket evidence attached to it.

OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)

// =============================================================================
// OPTIONALLY FIND CUSTOMERS WHO RAISED RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds customers connected to related tickets.
//
// The pattern means:
//
//     "Find customers who raised these tickets."
//
// Customer context helps measure impact.
//
// It helps answer:
//
//     "Which customers are affected by this issue?"

OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)

// =============================================================================
// OPTIONALLY FIND PRODUCTS RELATED TO THE TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Product nodes connected to related tickets.
//
// The pattern means:
//
//     "Find the product this ticket is about."
//
// Product context helps identify the affected product, application, or service.

OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

// =============================================================================
// OPTIONALLY FIND AGENTS ASSIGNED TO RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Agent nodes assigned to related tickets.
//
// The pattern means:
//
//     "Find the support agent assigned to this ticket."
//
// Agent context helps identify ownership, follow-up responsibility, and possible
// escalation paths.

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

// =============================================================================
// AGGREGATE OPERATIONAL CONTEXT AND CALCULATE RANKING BOOSTS
// =============================================================================
// This WITH clause performs two major jobs:
//
//     1. It aggregates related operational context into lists.
//     2. It calculates graph-based ranking boost values.
//
// Before this point, OPTIONAL MATCH clauses may create multiple rows because one
// issue can connect to many tickets, customers, products, and agents.
//
// This WITH clause compresses those rows into one grouped result per retrieved
// chunk/article/issue combination.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT t.ticketId) AS relatedTicketIds
// -----------------------------------------------------------------------------
// Collects all related ticket IDs into a clean list.
//
// DISTINCT removes duplicates caused by graph expansion.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT c.name) AS relatedCustomers
// -----------------------------------------------------------------------------
// Collects customer names connected through related tickets.
//
// This gives customer impact context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT p.name) AS relatedProducts
// -----------------------------------------------------------------------------
// Collects product names connected through related tickets.
//
// This gives product impact context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT a.name) AS assignedAgents
// -----------------------------------------------------------------------------
// Collects assigned agent names.
//
// This gives support ownership context.
//
// -----------------------------------------------------------------------------
// count(t) AS ticketCount
// -----------------------------------------------------------------------------
// Counts related ticket rows.
//
// This value is later used to decide whether the retrieved issue has operational
// ticket context.
//
// -----------------------------------------------------------------------------
// pageRankBoost
// -----------------------------------------------------------------------------
// pageRankBoost uses the maximum PageRank score among related tickets.
//
// PageRank usually represents graph importance or influence.
//
// coalesce(..., 0.0) ensures that if no ticket or score exists, the boost is
// safely treated as 0.0 instead of null.
//
// -----------------------------------------------------------------------------
// degreeBoost
// -----------------------------------------------------------------------------
// degreeBoost uses the maximum fullDegreeScore and scales it by 0.01.
//
// Degree usually represents how connected a ticket is.
//
// The 0.01 multiplier prevents raw degree values from overpowering vector
// similarity.
//
// -----------------------------------------------------------------------------
// betweennessBoost
// -----------------------------------------------------------------------------
// betweennessBoost uses the maximum betweennessScore and scales it by 0.01.
//
// Betweenness usually identifies bridge-like nodes that connect different parts
// of the graph.
//
// -----------------------------------------------------------------------------
// Production note
// -----------------------------------------------------------------------------
// These boost formulas are good for a teaching/demo workflow.
//
// In production, the boost values should usually be normalized, tuned, and
// validated against expected ranking behavior.

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

// =============================================================================
// BUILD A STRUCTURED CONTEXT ITEM FOR EACH RETRIEVED CHUNK
// =============================================================================
// This WITH clause packages each retrieved chunk into one structured map called:
//
//     contextItem
//
// A map is a key-value object.
//
// In simple terms, this section says:
//
//     "For this retrieved chunk, create one clean object containing the chunk
//      evidence, article source, issue context, operational context, and ranking
//      signals."
//
// -----------------------------------------------------------------------------
// chunkId
// -----------------------------------------------------------------------------
// Stores the unique ID of the retrieved DocumentChunk.
//
// This supports traceability.
//
// -----------------------------------------------------------------------------
// text
// -----------------------------------------------------------------------------
// Stores the actual retrieved chunk text.
//
// This is the evidence that may later be passed to an LLM.
//
// -----------------------------------------------------------------------------
// vectorScore
// -----------------------------------------------------------------------------
// Stores the original semantic similarity score from vector search.
//
// -----------------------------------------------------------------------------
// retrievalRankScore
// -----------------------------------------------------------------------------
// Stores the hybrid ranking score.
//
// The formula is:
//
//     vectorScore + pageRankBoost + degreeBoost + betweennessBoost
//
// This combines semantic similarity with graph-based importance.
//
// -----------------------------------------------------------------------------
// article
// -----------------------------------------------------------------------------
// article is a nested map containing:
//
//     articleId
//     title
//
// This identifies the source KnowledgeArticle.
//
// -----------------------------------------------------------------------------
// issue
// -----------------------------------------------------------------------------
// issue is a nested map containing:
//
//     issueId
//     name
//     severity
//
// This explains the operational issue solved by the article.
//
// -----------------------------------------------------------------------------
// operationalContext
// -----------------------------------------------------------------------------
// operationalContext is a nested map containing:
//
//     status
//     ticketIds
//     customers
//     products
//     assignedAgents
//
// This describes whether ticket context exists and who/what is affected.
//
// -----------------------------------------------------------------------------
// rankingSignals
// -----------------------------------------------------------------------------
// rankingSignals is a nested map containing the score breakdown.
//
// This makes the final retrieval ranking explainable.
//
// Instead of only showing the final retrievalRankScore, we also show:
//
//     vectorScore
//     pageRankBoost
//     degreeBoost
//     betweennessBoost

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

// =============================================================================
// GROUP RETRIEVED CONTEXT AND DEDUPLICATE REPEATED CONTEXT SECTIONS
// =============================================================================
// This WITH clause groups all contextItem maps per request and also creates
// deduplicated context lists.
//
// This is an important LLM-preparation step.
//
// The goal is to avoid sending repeated issue and operational context to the
// LLM unnecessarily.
//
// -----------------------------------------------------------------------------
// collect(contextItem) AS retrievedContext
// -----------------------------------------------------------------------------
// retrievedContext stores all retrieved chunk-level context items for the
// current request.
//
// This keeps the full evidence list.
//
// For example:
//
//     login -> [
//       chunk 1 context,
//       chunk 2 context
//     ]
//
// -----------------------------------------------------------------------------
// collect(DISTINCT contextItem.issue) AS dedupedIssueContext
// -----------------------------------------------------------------------------
// This collects unique issue maps only.
//
// Why deduplicate issue context?
//
// Two retrieved chunks may point to the same issue.
//
// If both chunks solve the same issue, repeating the same issue object multiple
// times would waste space and make the LLM prompt noisier.
//
// DISTINCT keeps only unique issue objects.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT contextItem.operationalContext) AS dedupedOperationalContext
// -----------------------------------------------------------------------------
// This collects unique operational context maps only.
//
// Operational context includes:
//
//     status
//     ticketIds
//     customers
//     products
//     assignedAgents
//
// Deduplication is useful because two retrieved chunks may connect to the same
// operational context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT CASE ... END) AS dedupedWarnings
// -----------------------------------------------------------------------------
// This collects unique warning messages.
//
// The warning condition checks:
//
//     contextItem.operationalContext.status =
//       "No related ticket found for this issue"
//
// If that condition is true, the warning text is collected.
//
// If not, null is returned.
//
// DISTINCT prevents the same warning from appearing multiple times.
//
// -----------------------------------------------------------------------------
// Important note
// -----------------------------------------------------------------------------
// This warning list is intentionally simple.
//
// It stores the warning status text rather than a chunk-specific warning.
//
// That means the final warning tells us:
//
//     "At least one retrieved item has no related ticket context."
//
// If you want chunk-specific warnings, include chunkId inside the warning map or
// warning string.

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

// =============================================================================
// RETURN FINAL DEDUPED LLM-READY CONTEXT PACKAGE
// =============================================================================
// This RETURN clause creates the final output.
//
// The result is one row per request.
//
// Each row contains:
//
//     queryType
//     userQuestion
//     llmReadyContext
//     retrievedEvidenceCount
//     issueContextCount
//     operationalContextCount
//     warningCount
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// Returns the request category.
//
// Example values:
//
//     login
//     payment
//     app_crash
//
// This helps identify which retrieval scenario the row belongs to.
//
// -----------------------------------------------------------------------------
// userQuestion
// -----------------------------------------------------------------------------
// Returns the original natural-language question.
//
// This keeps the final result connected to the user's original intent.
//
// -----------------------------------------------------------------------------
// llmReadyContext
// -----------------------------------------------------------------------------
// llmReadyContext is a structured map designed for downstream LLM usage.
//
// It contains:
//
//     question
//     retrievedEvidence
//     issueContext
//     operationalContext
//     rankingSignals
//     warnings
//     instructionForDay5LLM
//
// This is much cleaner than passing raw database rows to an LLM.
//
// -----------------------------------------------------------------------------
// question
// -----------------------------------------------------------------------------
// Stores the original user question inside the llmReadyContext object.
//
// This makes the context package self-contained.
//
// -----------------------------------------------------------------------------
// retrievedEvidence
// -----------------------------------------------------------------------------
// retrievedEvidence is a list of chunk-level evidence objects.
//
// Each object contains:
//
//     chunkId
//     text
//     vectorScore
//     retrievalRankScore
//
// This is the main evidence the LLM should use when generating an answer.
//
// -----------------------------------------------------------------------------
// issueContext
// -----------------------------------------------------------------------------
// issueContext contains deduplicated issue objects.
//
// Each issue object contains:
//
//     issueId
//     name
//     severity
//
// This gives the LLM structured understanding of the operational issue behind
// the retrieved evidence.
//
// -----------------------------------------------------------------------------
// operationalContext
// -----------------------------------------------------------------------------
// operationalContext contains deduplicated support context.
//
// It includes:
//
//     status
//     ticketIds
//     customers
//     products
//     assignedAgents
//
// This helps the LLM understand whether real ticket context exists and what
// customers/products/agents are involved.
//
// -----------------------------------------------------------------------------
// rankingSignals
// -----------------------------------------------------------------------------
// rankingSignals contains the ranking score breakdown for each retrieved chunk.
//
// This supports explainability because it shows how each retrieval result was
// scored.
//
// It includes:
//
//     vectorScore
//     pageRankBoost
//     degreeBoost
//     betweennessBoost
//     retrievalRankScore
//
// -----------------------------------------------------------------------------
// warnings
// -----------------------------------------------------------------------------
// warnings contains deduplicated warning messages.
//
// In this query, warnings are produced when retrieved context has no related
// ticket evidence.
//
// This helps the LLM avoid overstating operational certainty.
//
// -----------------------------------------------------------------------------
// instructionForDay5LLM
// -----------------------------------------------------------------------------
// This is a guardrail instruction for the downstream LLM.
//
// It says:
//
//     "Use only the provided retrieved evidence and graph context.
//      If warnings exist, mention the data-quality limitation."
//
// This is important because it tells the LLM not to invent unsupported facts.
//
// -----------------------------------------------------------------------------
// retrievedEvidenceCount
// -----------------------------------------------------------------------------
// Counts how many retrieved context items exist.
//
// This should usually match topK if every retrieved chunk has the expected graph
// relationships.
//
// -----------------------------------------------------------------------------
// issueContextCount
// -----------------------------------------------------------------------------
// Counts how many unique issues are present after deduplication.
//
// If retrievedEvidenceCount is 2 but issueContextCount is 1, that means both
// retrieved chunks point to the same issue.
//
// -----------------------------------------------------------------------------
// operationalContextCount
// -----------------------------------------------------------------------------
// Counts how many unique operational context objects exist.
//
// This helps detect whether multiple chunks share the same ticket/customer/
// product/agent context.
//
// -----------------------------------------------------------------------------
// warningCount
// -----------------------------------------------------------------------------
// Counts how many unique warning entries exist.
//
// This quickly tells us whether there are data-quality limitations in the
// returned context.

ORDER BY
  queryType;

// =============================================================================
// ORDER FINAL RESULTS BY QUERY TYPE
// =============================================================================
// ORDER BY controls the order of the final result rows.
//
// Here the query orders by:
//
//     queryType
//
// This makes the output predictable and easy to scan.
//
// Example alphabetical ordering may be:
//
//     app_crash
//     login
//     payment
//
// -----------------------------------------------------------------------------
// Important note about internal ordering
// -----------------------------------------------------------------------------
// This ORDER BY only sorts the final rows.
//
// It does not guarantee the order of items inside:
//
//     retrievedEvidence
//     rankingSignals
//     retrievedContext
//
// If you need the evidence list to be ordered by retrievalRankScore, add an
// ORDER BY before the collect(contextItem) step.
//
// For example, you could sort by:
//
//     retrievalRankScore DESC
//
// before collecting the context items.
//
// =============================================================================
// FINAL TAKEAWAY
// =============================================================================
// This query prepares a deduplicated, explainable, LLM-ready retrieval package.
//
// The main purpose is:
//
//     "Retrieve semantically relevant chunks,
//      enrich them with graph context,
//      calculate ranking signals,
//      deduplicate repeated issue and operational context,
//      and return a clean object that an LLM can use safely."
//
// This is useful for:
// - Day 5 LLM/RAG integration
// - Explainable graph-powered retrieval
// - Batch retrieval testing
// - Prompt context preparation
// - Reducing duplicate context
// - Warning the LLM about missing operational evidence
// - Keeping retrieval evidence and graph context separate
//
// This is a strong production-style design because it supports both:
//
//     1. Better answer generation
//     2. Better traceability and auditability
//
// The LLM receives not just text chunks, but also structured issue context,
// operational context, ranking signals, and explicit safety instructions.

```

# Step 7 — Final Day 4 retrieval validation summary

```cypher
// =============================================================================
// DAY 4 RETRIEVAL VALIDATION SUMMARY FOR BATCH EXPLAINABLE RAG
// =============================================================================
// This query validates whether the Day 4 graph-enriched vector retrieval flow is
// working correctly across multiple user-question scenarios.
//
// In simple terms, it tells Neo4j:
//
//     "Run multiple test questions through vector search,
//      enrich the retrieved chunks with graph context,
//      calculate retrieval quality indicators,
//      summarize each question,
//      and finally return one validation report for the whole retrieval test."
//
// This query is different from earlier detailed retrieval queries.
//
// Earlier queries focused on returning the actual retrieved evidence and context
// for each question.
//
// This query focuses on validation.
//
// It answers questions like:
//
// - How many questions were tested?
// - How many questions retrieved evidence?
// - How many questions had issue context?
// - How many questions had operational ticket context?
// - Which questions produced warnings?
// - What was the maximum retrieval rank score per question?
//
// This is very useful at the end of Day 4 because it gives a clean checkpoint
// before moving into Day 5 LLM integration.
//
// In a production-style workflow, this kind of validation query helps confirm
// that retrieval is not only returning chunks, but also returning enough graph
// context to support explainable answers.

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

// =============================================================================
// DEFINE DAY 4 TEST QUESTIONS USING UNWIND
// =============================================================================
// UNWIND expands a list into individual rows.
//
// Here, the list contains three test requests:
//
//     1. login
//     2. payment
//     3. app_crash
//
// Each request is represented as a map containing:
//
//     queryType
//     userQuestion
//     queryVector
//     topK
//
// After UNWIND runs, Neo4j processes the rest of the query once for each
// request.
//
// This means the same retrieval-validation pipeline is applied to all three
// scenarios.
//
// -----------------------------------------------------------------------------
// Why this is useful
// -----------------------------------------------------------------------------
// This is better than writing three separate queries.
//
// If we want to add more validation scenarios later, such as:
//
//     password_reset
//     refund
//     account_locked
//     performance_issue
//
// we only need to add another map to this list.
//
// The rest of the query does not need to change.
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// queryType is a short category name for the test case.
//
// Examples:
//
//     login
//     payment
//     app_crash
//
// This helps identify which question passed or failed validation later.
//
// -----------------------------------------------------------------------------
// userQuestion
// -----------------------------------------------------------------------------
// userQuestion stores the natural-language question being tested.
//
// This makes the validation summary readable for humans.
//
// -----------------------------------------------------------------------------
// queryVector
// -----------------------------------------------------------------------------
// queryVector is the embedding used for vector search.
//
// In this lab/demo, the vectors are hardcoded so the results are predictable.
//
// In production, these vectors would usually be generated from userQuestion by
// an embedding model.
//
// -----------------------------------------------------------------------------
// topK
// -----------------------------------------------------------------------------
// topK tells Neo4j how many nearest chunks to retrieve for each question.
//
// Here, every request uses:
//
//     topK: 2
//
// So each question asks for the top 2 matching DocumentChunk nodes.

CALL db.index.vector.queryNodes(
  'documentChunk_embedding_vector',
  request.topK,
  request.queryVector
)

// =============================================================================
// RUN VECTOR SEARCH FOR EACH TEST QUESTION
// =============================================================================
// db.index.vector.queryNodes() searches the vector index for nodes whose stored
// embeddings are closest to the request query vector.
//
// In simple terms, this says:
//
//     "For the current test question,
//      find the topK document chunks that are semantically closest."
//
// -----------------------------------------------------------------------------
// 'documentChunk_embedding_vector'
// -----------------------------------------------------------------------------
// This is the name of the Neo4j vector index.
//
// It is expected to be created on DocumentChunk nodes, most likely on their
// embedding property.
//
// -----------------------------------------------------------------------------
// request.topK
// -----------------------------------------------------------------------------
// This controls how many matching chunks should be returned.
//
// Since request.topK is 2, Neo4j returns up to two matching chunks for each
// query type.
//
// -----------------------------------------------------------------------------
// request.queryVector
// -----------------------------------------------------------------------------
// This is the numeric embedding for the current question.
//
// Important beginner note:
//
//     Neo4j is not comparing the English question directly here.
//
// It is comparing numeric vectors that represent meaning.

YIELD node AS dc, score

// =============================================================================
// CAPTURE VECTOR SEARCH OUTPUT
// =============================================================================
// YIELD extracts values returned by db.index.vector.queryNodes().
//
// The procedure returns:
//
//     node
//     score
//
// -----------------------------------------------------------------------------
// node AS dc
// -----------------------------------------------------------------------------
// node is the retrieved graph node.
//
// We rename it to:
//
//     dc
//
// because the retrieved node is expected to be a DocumentChunk.
//
// -----------------------------------------------------------------------------
// score
// -----------------------------------------------------------------------------
// score is the vector similarity score.
//
// A higher score usually means the retrieved chunk is more semantically similar
// to the query vector.
//
// This score later contributes to the final retrievalRankScore.

MATCH (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)

// =============================================================================
// CONNECT RETRIEVED CHUNK TO ARTICLE AND ISSUE CONTEXT
// =============================================================================
// This MATCH clause connects the retrieved DocumentChunk to its parent
// KnowledgeArticle and then to the Issue that the article solves.
//
// The pattern is:
//
//     (dc)-[:PART_OF]->(ka:KnowledgeArticle)-[:SOLVES]->(i:Issue)
//
// Meaning:
//
//     "The retrieved chunk belongs to a knowledge article,
//      and that knowledge article solves an issue."
//
// -----------------------------------------------------------------------------
// Why this matters
// -----------------------------------------------------------------------------
// Vector search alone only proves that a chunk is semantically similar.
//
// This graph match proves that the chunk has structured explanation context.
//
// It allows the system to say:
//
//     "This chunk was retrieved,
//      it came from this article,
//      and that article solves this issue."
//
// For Day 4 validation, this is important because we want to confirm that
// retrieved chunks are not isolated text fragments. They must connect back to
// business/domain context.

OPTIONAL MATCH (t:Ticket)-[:HAS_ISSUE]->(i)

// =============================================================================
// OPTIONALLY FIND TICKETS RELATED TO THE ISSUE
// =============================================================================
// This OPTIONAL MATCH finds Ticket nodes connected to the Issue.
//
// The pattern means:
//
//     "Find tickets that have this issue."
//
// OPTIONAL MATCH is used because some issues may not have tickets.
//
// If a normal MATCH were used, issues without ticket context would disappear
// from the result.
//
// That would hide useful validation information.
//
// For example, if a retrieved issue has no related tickets, we want to know
// that and report it as a warning, not silently remove the result.

OPTIONAL MATCH (c:Customer)-[:RAISED]->(t)

// =============================================================================
// OPTIONALLY FIND CUSTOMERS WHO RAISED RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds customers connected to the related tickets.
//
// The pattern means:
//
//     "Find customers who raised these tickets."
//
// Customer context helps validate whether the retrieved issue has real customer
// impact data.
//
// This is optional because demo data or partially loaded data may not always
// have customer relationships.

OPTIONAL MATCH (t)-[:ABOUT]->(p:Product)

// =============================================================================
// OPTIONALLY FIND PRODUCTS ASSOCIATED WITH RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds Product nodes connected to the tickets.
//
// The pattern means:
//
//     "Find the product that this ticket is about."
//
// Product context helps validate whether the retrieved issue can be connected to
// an affected product or service.

OPTIONAL MATCH (t)-[:ASSIGNED_TO]->(a:Agent)

// =============================================================================
// OPTIONALLY FIND AGENTS ASSIGNED TO RELATED TICKETS
// =============================================================================
// This OPTIONAL MATCH finds support agents assigned to the tickets.
//
// The pattern means:
//
//     "Find the agent assigned to this ticket."
//
// Agent context helps validate whether the retrieved issue has operational
// ownership information.

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

// =============================================================================
// AGGREGATE CONTEXT AND CALCULATE GRAPH-BASED BOOST VALUES
// =============================================================================
// This WITH clause aggregates operational context and calculates ranking boost
// signals.
//
// Before this point, OPTIONAL MATCH clauses may create multiple rows because one
// issue can be connected to many tickets, customers, products, and agents.
//
// This WITH clause compresses those rows into one grouped result per retrieved
// chunk/article/issue combination.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT t.ticketId) AS relatedTicketIds
// -----------------------------------------------------------------------------
// Collects all related ticket IDs into a clean list.
//
// DISTINCT removes duplicates caused by graph expansion.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT c.name) AS relatedCustomers
// -----------------------------------------------------------------------------
// Collects all customer names connected through related tickets.
//
// This helps validate customer impact context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT p.name) AS relatedProducts
// -----------------------------------------------------------------------------
// Collects product names connected through related tickets.
//
// This helps validate product impact context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT a.name) AS assignedAgents
// -----------------------------------------------------------------------------
// Collects agent names assigned to related tickets.
//
// This helps validate support ownership context.
//
// -----------------------------------------------------------------------------
// count(t) AS ticketCount
// -----------------------------------------------------------------------------
// Counts how many ticket rows were found.
//
// This value is later used to decide whether operational ticket context exists.
//
// If ticketCount is 0, the query creates a warning condition.
//
// -----------------------------------------------------------------------------
// pageRankBoost
// -----------------------------------------------------------------------------
// Uses the maximum PageRank score among related tickets.
//
// PageRank usually represents graph importance or influence.
//
// coalesce(..., 0.0) protects the calculation from null values.
//
// If there are no tickets, the boost becomes 0.0.
//
// -----------------------------------------------------------------------------
// degreeBoost
// -----------------------------------------------------------------------------
// Uses the maximum fullDegreeScore and scales it by 0.01.
//
// Degree usually represents how connected a ticket is.
//
// Scaling prevents raw degree values from dominating the final score.
//
// -----------------------------------------------------------------------------
// betweennessBoost
// -----------------------------------------------------------------------------
// Uses the maximum betweennessScore and scales it by 0.01.
//
// Betweenness usually identifies bridge-like nodes that connect different parts
// of the graph.
//
// -----------------------------------------------------------------------------
// Why these boosts exist
// -----------------------------------------------------------------------------
// These boosts help create a hybrid retrieval rank score.
//
// Instead of relying only on vector similarity, the query also considers whether
// the retrieved issue is connected to operationally important tickets.

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

// =============================================================================
// BUILD ONE CONTEXT ITEM PER RETRIEVED CHUNK
// =============================================================================
// This WITH clause packages the retrieval result into a structured map called:
//
//     contextItem
//
// In simple terms, it says:
//
//     "For this retrieved chunk, create one object containing evidence,
//      issue context, operational context, and ranking signals."
//
// -----------------------------------------------------------------------------
// chunkId
// -----------------------------------------------------------------------------
// Stores the retrieved DocumentChunk identifier.
//
// This supports traceability.
//
// -----------------------------------------------------------------------------
// text
// -----------------------------------------------------------------------------
// Stores the retrieved chunk text.
//
// This is the evidence that could later be passed to an LLM.
//
// -----------------------------------------------------------------------------
// vectorScore
// -----------------------------------------------------------------------------
// Stores the raw semantic similarity score from vector search.
//
// -----------------------------------------------------------------------------
// retrievalRankScore
// -----------------------------------------------------------------------------
// Stores the final hybrid ranking score.
//
// The formula is:
//
//     vectorScore + pageRankBoost + degreeBoost + betweennessBoost
//
// -----------------------------------------------------------------------------
// issue
// -----------------------------------------------------------------------------
// Stores structured issue context:
//
//     issueId
//     name
//     severity
//
// This helps validate whether retrieved chunks map to meaningful issues.
//
// -----------------------------------------------------------------------------
// operationalContext
// -----------------------------------------------------------------------------
// Stores related operational context:
//
//     status
//     ticketIds
//     customers
//     products
//     assignedAgents
//
// This is used later to determine whether the result has operational support.
//
// -----------------------------------------------------------------------------
// rankingSignals
// -----------------------------------------------------------------------------
// Stores the score breakdown:
//
//     vectorScore
//     pageRankBoost
//     degreeBoost
//     betweennessBoost
//     retrievalRankScore
//
// This is useful for explainability and debugging ranking behavior.

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

// =============================================================================
// GROUP RETRIEVED ITEMS AND DEDUPLICATE CONTEXT
// =============================================================================
// This WITH clause groups all retrieved context items per request and creates
// deduplicated validation context.
//
// -----------------------------------------------------------------------------
// collect(contextItem) AS retrievedContext
// -----------------------------------------------------------------------------
// Collects all retrieved chunk-level context items for the current request.
//
// Since topK is 2, this will usually contain two items per request, assuming
// both vector results successfully matched article and issue context.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT contextItem.issue) AS dedupedIssueContext
// -----------------------------------------------------------------------------
// Collects unique issue objects.
//
// This prevents duplicate issue context when multiple retrieved chunks point to
// the same issue.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT contextItem.operationalContext) AS dedupedOperationalContext
// -----------------------------------------------------------------------------
// Collects unique operational context objects.
//
// This prevents repeated ticket/customer/product/agent context from being
// counted multiple times unnecessarily.
//
// -----------------------------------------------------------------------------
// collect(DISTINCT CASE ... END) AS dedupedWarnings
// -----------------------------------------------------------------------------
// Creates a deduplicated warning list.
//
// If a retrieved item has no related ticket context, the warning text:
//
//     "No related ticket found for this issue"
//
// is collected.
//
// If operational context exists, null is returned and ignored by collect().
//
// -----------------------------------------------------------------------------
// Why warnings matter
// -----------------------------------------------------------------------------
// Warnings are important because they indicate data-quality or graph-completeness
// limitations.
//
// A retrieved chunk may still be semantically valid, but if it has no related
// tickets, the system should not claim strong operational evidence.

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

// =============================================================================
// DERIVE WHETHER EACH QUESTION HAS OPERATIONAL CONTEXT
// =============================================================================
// This WITH clause creates a boolean flag:
//
//     hasOperationalContext
//
// The logic is:
//
//     If warning count > 0:
//       hasOperationalContext = false
//
//     Otherwise:
//       hasOperationalContext = true
//
// -----------------------------------------------------------------------------
// Why this flag exists
// -----------------------------------------------------------------------------
// This makes the final validation summary easier to read.
//
// Instead of forcing the reader to inspect warnings manually, the query creates
// a direct true/false indicator.
//
// -----------------------------------------------------------------------------
// Important interpretation
// -----------------------------------------------------------------------------
// In this query, hasOperationalContext is based on warnings.
//
// A warning means at least one retrieved item had:
//
//     "No related ticket found for this issue"
//
// So if warnings exist, the question is treated as not fully supported by
// operational ticket context.
//
// This is a useful Day 4 validation rule because Day 5 LLM answers should be
// careful when operational context is incomplete.

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

// =============================================================================
// BUILD QUESTION-LEVEL VALIDATION SUMMARIES
// =============================================================================
// This WITH clause creates one summary object per test question and collects all
// summaries into a list called:
//
//     questionSummaries
//
// Each summary object contains validation metrics for one query type.
//
// -----------------------------------------------------------------------------
// queryType
// -----------------------------------------------------------------------------
// Stores the question category.
//
// Example:
//
//     login
//     payment
//     app_crash
//
// -----------------------------------------------------------------------------
// userQuestion
// -----------------------------------------------------------------------------
// Stores the original natural-language question.
//
// This keeps the validation summary readable.
//
// -----------------------------------------------------------------------------
// retrievedEvidenceCount
// -----------------------------------------------------------------------------
// Counts how many retrieved context items exist for this question.
//
// This helps validate whether vector retrieval returned evidence.
//
// If this count is 0, the retrieval pipeline failed to produce usable evidence
// for that question.
//
// -----------------------------------------------------------------------------
// issueContextCount
// -----------------------------------------------------------------------------
// Counts how many unique issues were found for the retrieved evidence.
//
// If this count is 0, it means retrieved chunks did not connect to issue
// context, which would be a serious explainability gap.
//
// -----------------------------------------------------------------------------
// operationalContextCount
// -----------------------------------------------------------------------------
// Counts how many unique operational context objects were found.
//
// This helps validate whether ticket/customer/product/agent context exists.
//
// -----------------------------------------------------------------------------
// warningCount
// -----------------------------------------------------------------------------
// Counts how many deduplicated warnings exist for the question.
//
// A warning indicates missing related ticket context.
//
// -----------------------------------------------------------------------------
// hasOperationalContext
// -----------------------------------------------------------------------------
// Stores the boolean true/false flag created earlier.
//
// This gives a simple pass/fail style indicator for operational context.
//
// -----------------------------------------------------------------------------
// warnings
// -----------------------------------------------------------------------------
// Stores the warning messages for this question.
//
// This helps identify data-quality limitations.
//
// -----------------------------------------------------------------------------
// maxRetrievalRankScore
// -----------------------------------------------------------------------------
// Calculates the highest retrievalRankScore among all retrieved context items
// for the question.
//
// The reduce() function walks through the retrievedContext list and keeps the
// largest score.
//
// In simple terms:
//
//     "Look at all retrieved items for this question
//      and return the highest hybrid rank score."
//
// -----------------------------------------------------------------------------
// reduce(maxScore = 0.0, item IN retrievedContext | ...)
// -----------------------------------------------------------------------------
// reduce() is used here like a loop.
//
// It starts with:
//
//     maxScore = 0.0
//
// Then for each item in retrievedContext:
//
//     If item.retrievalRankScore is greater than maxScore,
//     replace maxScore with item.retrievalRankScore.
//
// Otherwise, keep the existing maxScore.
//
// The final value becomes maxRetrievalRankScore.
//
// This is useful because it summarizes the strongest retrieval result for each
// question.

RETURN
  size(questionSummaries) AS totalQuestionsTested,

// =============================================================================
// RETURN TOTAL NUMBER OF QUESTIONS TESTED
// =============================================================================
// size(questionSummaries) counts how many question summary objects were created.
//
// Since the input list contains three requests, this should normally return:
//
//     3
//
// This field answers:
//
//     "How many retrieval test questions were included in this validation run?"

  size([
    q IN questionSummaries
    WHERE q.retrievedEvidenceCount > 0
  ]) AS questionsWithRetrievedEvidence,

// =============================================================================
// RETURN COUNT OF QUESTIONS WITH RETRIEVED EVIDENCE
// =============================================================================
// This list comprehension filters questionSummaries to only questions where:
//
//     retrievedEvidenceCount > 0
//
// Then size() counts how many questions passed that condition.
//
// This field answers:
//
//     "How many questions successfully retrieved at least one evidence chunk?"
//
// If this number is lower than totalQuestionsTested, it means at least one test
// question failed to retrieve usable evidence.

  size([
    q IN questionSummaries
    WHERE q.issueContextCount > 0
  ]) AS questionsWithIssueContext,

// =============================================================================
// RETURN COUNT OF QUESTIONS WITH ISSUE CONTEXT
// =============================================================================
// This counts how many questions have at least one deduplicated issue context.
//
// The condition is:
//
//     issueContextCount > 0
//
// This field answers:
//
//     "How many questions retrieved evidence that could be connected to an
//      Issue node?"
//
// This is very important for explainable RAG.
//
// If evidence exists but issue context is missing, the retrieval result is less
// explainable.

  size([
    q IN questionSummaries
    WHERE q.hasOperationalContext = true
  ]) AS questionsWithOperationalContext,

// =============================================================================
// RETURN COUNT OF QUESTIONS WITH OPERATIONAL CONTEXT
// =============================================================================
// This counts how many questions have:
//
//     hasOperationalContext = true
//
// In this query, that means the question has no warnings about missing related
// ticket context.
//
// This field answers:
//
//     "How many test questions have operational ticket context available?"
//
// This is important before Day 5 because LLM answers should be more confident
// when retrieved evidence is supported by operational graph data.

  size([
    q IN questionSummaries
    WHERE q.warningCount > 0
  ]) AS questionsWithWarnings,

// =============================================================================
// RETURN COUNT OF QUESTIONS WITH WARNINGS
// =============================================================================
// This counts how many question summaries have one or more warnings.
//
// The condition is:
//
//     warningCount > 0
//
// This field answers:
//
//     "How many questions have data-quality or missing-context warnings?"
//
// A warning does not necessarily mean the retrieval result is wrong.
//
// It means the result has a limitation that should be mentioned or handled
// carefully.

  [q IN questionSummaries WHERE q.warningCount > 0 | q.queryType] AS queryTypesWithWarnings,

// =============================================================================
// RETURN QUERY TYPES THAT HAVE WARNINGS
// =============================================================================
// This list comprehension returns only the queryType values for questions that
// have warnings.
//
// For example, it may return:
//
//     ["payment"]
//
// or:
//
//     ["login", "app_crash"]
//
// depending on the data.
//
// This field is useful because it immediately tells us which retrieval scenarios
// need attention.
//
// Instead of inspecting the full questionSummaries list manually, we can quickly
// see which query types have missing operational context.

  questionSummaries AS day4RetrievalValidationSummary;

// =============================================================================
// RETURN FULL DAY 4 RETRIEVAL VALIDATION SUMMARY
```
