# FraudDetectionUEBAUsingGraphDatabase
Bank Fraud Detection using Neo4j as UEBA with AI/ML Capabilities -- A Prototyp

========================================================================================
Datasets used for First Party Fraud and Second Party Fraud Use Cases are from: https://github.com/neo4j-graph-examples/fraud-detection/tree/main/data

These are PaySim datasets. 

(I am unable to add the datasets touching almost 200 MB)

=======================================================================================
Cyphers Used for the Use Cases:

------------------------------
First Party Fraud Use Case:
First-party fraud detection identifies potential synthetic identities by analyzing shared personally identifiable information (PII) across client accounts. The process uses graph algorithms to identify clusters of related identities that may represent fraud rings.

Prerequisites
Neo4j database is running with the fraud-detection-40 dump data loaded
Neo4j Graph Data Science (GDS) library is installed
APOC library is installed
Access to Neo4j Browser (http://localhost:7474)

Execution Order
For proper execution, follow these steps in order:

Run sharedIdentifiers.cypher (Step 1)
Run loadWCCNamedGraph.cypher (Step 2)
Run wccWrite.cypher (Step 3)
Generate similarity scores (Step 4)
Run degreeWrite.cypher (Step 5)
Run firstPartyFraudScoreWrite.cypher (Step 6)
Run visualizeSharedIdentifiers.cypher (Step 7)

Process Steps
1. Identifying Shared PII
The system identifies clients who share the same personally identifiable information (PII) like email addresses, phone numbers, and SSNs
Execute the sharedIdentifiers.cypher script to match pairs of clients connecting to the same PII nodes
For each pair of clients, the script counts shared PII elements and creates a SHARED_IDENTIFIERS relationship with a count property
// Run this in Neo4j Browser
MATCH (c1:Client)-[:HAS_EMAIL|:HAS_PHONE|:HAS_SSN]->(info)
<-[:HAS_EMAIL|:HAS_PHONE|:HAS_SSN]-(c2:Client)
WHERE c1.id<>c2.id
WITH c1, c2, count(*) as cnt
MERGE (c1) - [:SHARED_IDENTIFIERS {count: cnt}] - (c2);
Verification:

// Check the number of SHARED_IDENTIFIERS relationships created
MATCH ()-[r:SHARED_IDENTIFIERS]->() RETURN COUNT(r);
2. Creating a Graph Projection
Execute the loadWCCNamedGraph.cypher script to create a projected graph named 'WCC' in Neo4j Graph Data Science
This projection includes only Client nodes and SHARED_IDENTIFIERS relationships with their count properties
The projection allows efficient algorithm execution on this subset of the data
CALL gds.graph.create('WCC', 'Client',
    {
        SHARED_IDENTIFIERS:{
            type: 'SHARED_IDENTIFIERS',
            properties: {
                count: {
                    property: 'count'
                }
            }
        }
    }
) YIELD graphName,nodeCount,relationshipCount,projectMillis;
Verification:

// Verify the graph projection exists
CALL gds.graph.list() YIELD graphName, nodeCount, relationshipCount
WHERE graphName = 'WCC'
RETURN graphName, nodeCount, relationshipCount;
3. Running Weakly Connected Components
Execute the wccWrite.cypher script to run the Weakly Connected Components algorithm on the projected graph
WCC identifies clusters of interconnected clients who share PII information
Only clusters with more than one client are considered (potential fraud rings)
Each client in these clusters gets a firstPartyFraudGroup property set to their component ID
CALL gds.wcc.stream('WCC')
YIELD componentId,nodeId
WITH componentId AS cluster,gds.util.asNode(nodeId) AS client
WITH cluster, collect(client.id) AS clients
WITH *,size(clients) AS clusterSize
WHERE clusterSize>1
UNWIND clients AS client
MATCH(c:Client) 
WHERE c.id=client
SET c.firstPartyFraudGroup=cluster;
Verification:

// Count clients assigned to fraud groups
MATCH (c:Client)
WHERE exists(c.firstPartyFraudGroup)
RETURN COUNT(c);

// View fraud groups and their sizes
MATCH (c:Client)
WHERE exists(c.firstPartyFraudGroup)
WITH c.firstPartyFraudGroup AS fraudGroup, count(*) AS groupSize
RETURN fraudGroup, groupSize
ORDER BY groupSize DESC
LIMIT 10;
4. Computing Node Similarity
The system calculates similarity scores between clients using the Jaccard similarity algorithm
This creates SIMILAR_TO relationships between clients with a jaccardScore property
The jaccardScore indicates how similar two clients are based on their shared identifiers
// Create a similarity graph projection
CALL gds.graph.project('similarity',
    'Client',
    'SHARED_IDENTIFIERS',
    {
        relationshipProperties: 'count'
    }
);

// Run the Node Similarity algorithm and mutate the graph
CALL gds.nodeSimilarity.mutate('similarity',
    {
        topK: 10,
        mutateProperty: 'jaccardScore',
        mutateRelationshipType: 'SIMILAR_TO'
    }
);
Verification:

// Verify SIMILAR_TO relationships in the projection
CALL gds.graph.relationshipTypes('similarity')
YIELD relationshipType
RETURN relationshipType;
5. Calculating Fraud Scores
Execute the degreeWrite.cypher script to assign a firstPartyFraudScore to each client
Uses a weighted degree centrality algorithm
This score is derived from the sum of jaccard similarity scores on incoming SIMILAR_TO relationships
Clients with higher scores have more connections to other suspicious clients, making them potentially higher risk
CALL gds.degree.write('similarity',
    {
        nodeLabels: ['Client'],
        relationshipTypes: ['SIMILAR_TO'],
        relationshipWeightProperty: 'jaccardScore',
        writeProperty: 'firstPartyFraudScore'
    }
);
Verification:

// Check distribution of fraud scores
MATCH (c:Client)
WHERE exists(c.firstPartyFraudScore)
RETURN 
    count(*) AS totalScoredClients,
    min(c.firstPartyFraudScore) AS minScore,
    max(c.firstPartyFraudScore) AS maxScore,
    avg(c.firstPartyFraudScore) AS avgScore,
    percentileCont(c.firstPartyFraudScore, 0.8) AS percentile80;
6. Identifying First-Party Fraudsters
Execute the firstPartyFraudScoreWrite.cypher script to identify clients with fraud scores above a threshold
The threshold is set at the 80th percentile of all fraud scores
Clients exceeding this threshold are labeled as :FirstPartyFraudster
MATCH(c:Client) 
WHERE exists(c.firstPartyFraudScore)
WITH percentileCont(c.firstPartyFraudScore, 0.8)
    AS firstPartyFraudThreshold
MATCH(c:Client)
WHERE c.firstPartyFraudScore>firstPartyFraudThreshold
SET c:FirstPartyFraudster;
Verification:

// Count identified fraudsters
MATCH (c:FirstPartyFraudster)
RETURN COUNT(c);

// View top fraudsters by score
MATCH (c:Client:FirstPartyFraudster)
RETURN c.name, c.firstPartyFraudScore
ORDER BY c.firstPartyFraudScore DESC
LIMIT 25;
7. Visualization
Execute the visualizeSharedIdentifiers.cypher script to visualize relationships between clients
Shows connections between clients who share at least 2 PII elements
MATCH p = (c:Client) - [s:SHARED_IDENTIFIERS] - () WHERE s.count >= 2 RETURN p LIMIT 25;
-------------------------------------------------------------------------------------------------------------------------
Second Party Frause Use Case:

Second-party fraud detection identifies potential money mules by analyzing transfer patterns between accounts. Money mules are individuals who transfer illegally acquired money on behalf of others. The process uses graph algorithms like PageRank to identify suspicious accounts based on transfer patterns.

Prerequisites
Neo4j database is running with the fraud-detection-40 dump data loaded
Neo4j Graph Data Science (GDS) library is installed
APOC library is installed
Access to Neo4j Browser (http://localhost:7474)

Execution Order
For proper execution, follow these steps in order:

Create graph projection (Step 1)
Run PageRank algorithm (Step 2)
Label money mules (Step 3)
Analyze transaction patterns (Step 4)
Run visualization queries (Step 5)

Process Steps
1. Creating a Graph Projection
Execute the following script to create a projected graph named 'SecondPartyFraudNetwork' in Neo4j Graph Data Science
This projection includes Client nodes and TRANSFER_TO relationships with their amount properties
The projection allows efficient algorithm execution on this subset of the data
CALL gds.graph.project('SecondPartyFraudNetwork', 'Client', 'TRANSFER_TO',
    {relationshipProperties: ['amount']}
) YIELD graphName, nodeCount, relationshipCount;
Verification:

// Verify the graph projection exists
CALL gds.graph.list() YIELD graphName, nodeCount, relationshipCount
WHERE graphName = 'SecondPartyFraudNetwork'
RETURN graphName, nodeCount, relationshipCount;
2. Running PageRank Algorithm
Execute the following script to run the PageRank algorithm on the projected graph
PageRank identifies clients who receive many transfers or high-value transfers from multiple sources
Accounts with high PageRank scores may be functioning as money mules
Each client gets a secondPartyFraudScore property set to their PageRank value
CALL gds.pageRank.write('SecondPartyFraudNetwork',
    {
        maxIterations: 20,
        dampingFactor: 0.85,
        writeProperty: 'secondPartyFraudScore',
        relationshipWeightProperty: 'amount'
    }
) YIELD nodePropertiesWritten, ranIterations;
Verification:

// Count clients assigned fraud scores
MATCH (c:Client)
WHERE exists(c.secondPartyFraudScore)
RETURN COUNT(c);

// View distribution of fraud scores
MATCH (c:Client)
WHERE exists(c.secondPartyFraudScore)
RETURN 
    count(*) AS totalScoredClients,
    min(c.secondPartyFraudScore) AS minScore,
    max(c.secondPartyFraudScore) AS maxScore,
    avg(c.secondPartyFraudScore) AS avgScore,
    percentileCont(c.secondPartyFraudScore, 0.8) AS percentile80;
3. Identifying Money Mules
Execute the following script to identify clients with PageRank scores above a threshold
The threshold is set at the 80th percentile of all PageRank scores
Clients exceeding this threshold are labeled as :MoneyMule
MATCH(c:Client) 
WHERE exists(c.secondPartyFraudScore)
WITH percentileCont(c.secondPartyFraudScore, 0.8)
    AS secondPartyFraudThreshold
MATCH(c:Client)
WHERE c.secondPartyFraudScore > secondPartyFraudThreshold
SET c:MoneyMule;
Verification:

// Count identified money mules
MATCH (c:MoneyMule)
RETURN COUNT(c);

// View top money mules by score
MATCH (c:Client:MoneyMule)
RETURN c.name, c.secondPartyFraudScore
ORDER BY c.secondPartyFraudScore DESC
LIMIT 25;
4. Analyzing Transaction Flow Patterns
Execute the following script to analyze the transaction flow patterns of suspected money mules
This identifies clients who receive funds from multiple sources and then transfer those funds elsewhere
The analysis calculates inflow vs. outflow ratios
MATCH (c:Client:MoneyMule)
OPTIONAL MATCH (source:Client)-[inflow:TRANSFER_TO]->(c)
OPTIONAL MATCH (c)-[outflow:TRANSFER_TO]->(target:Client)
WITH c,
    count(distinct source) AS sourcesCount,
    count(distinct target) AS targetsCount,
    sum(inflow.amount) AS totalInflow,
    sum(outflow.amount) AS totalOutflow
WHERE sourcesCount > 3 AND totalOutflow > 0
RETURN 
    c.name,
    c.secondPartyFraudScore,
    sourcesCount,
    targetsCount,
    totalInflow,
    totalOutflow,
    totalInflow/totalOutflow AS flowRatio
ORDER BY c.secondPartyFraudScore DESC
LIMIT 25;
5. Visualization
Execute the following script to visualize suspected money mules and their transaction networks
Shows the flow of funds into and out of suspected money mule accounts
// Visualize top money mules and their transaction patterns
MATCH (c:MoneyMule)
WITH c ORDER BY c.secondPartyFraudScore DESC LIMIT 5
MATCH p = (source:Client)-[:TRANSFER_TO]->(c)-[:TRANSFER_TO]->(target:Client)
RETURN p
LIMIT 75;
Additional Visualization Queries:

// Visualize money mule clusters
MATCH (m1:MoneyMule)
WITH m1 ORDER BY m1.secondPartyFraudScore DESC LIMIT 10
MATCH p = (source:Client)-[:TRANSFER_TO]->(m1)
WHERE source:MoneyMule
RETURN p
LIMIT 50;

// Visualize pattern of high-value transfers
MATCH p = (c1:Client)-[t:TRANSFER_TO]->(c2:Client)
WHERE c2:MoneyMule AND t.amount > 5000
RETURN p
LIMIT 50;

-----------------------------------------------------------------------------------------------------------
There are no code bases required for this prototype. They are not used: only datasets and cyphers are used.


=================================================================================================================================

