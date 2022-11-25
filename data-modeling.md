## Data modeling process

- Export historical transaction data into a single CSV file
- Create account nodes after removing duplicate addresses
- Configure APOC if using the library: https://community.neo4j.com/t5/neo4j-graph-platform/setting-apoc-import-file-enabled-true-in-your-neo4j-conf/m-p/46405

```sh
//Create account node
:auto USING PERIODIC COMMIT 1000 LOAD CSV WITH HEADERS FROM 'file:///transfers.csv' AS row
WITH row.from_address AS fromAddress, row.to_address AS toAddress
WITH [fromAddress] AS list1, [toAddress] AS list2
UNWIND list1 + list2 AS combinedList
WITH DISTINCT combinedList AS address
MERGE (a:Account {address: address})
RETURN a;
```

If the above query doesn't work in latest Neo4j versions, this query would work
```sh
//Create account node
:auto LOAD CSV WITH HEADERS FROM 'file:///transfers.csv' AS row
CALL {
  WITH row
  UNWIND [row.from_address] + [row.to_address] AS address
  WITH DISTINCT address
  MERGE (a:Accoutn {address: address})
  RETURN a
}
IN TRANSACTIONS OF 1000 ROWS
WITH a
RETURN a
```

Alternative query
```sh
CALL apoc.periodic.iterate('
    CALL apoc.load.csv("file:///transfers.csv") YIELD map AS row RETURN row
','
    WITH row
    UNWIND [row.from_address] + [row.to_address] AS address
    WITH DISTINCT address
    MERGE (a:Accoutn {address: address})
', {batchSize:10000, iterateList:true, parallel:true})
```

- Create NFT nodes

```sh
//Create NFT node
:auto USING PERIODIC COMMIT 1000 LOAD CSV WITH HEADERS FROM 'file:///transfers.csv' AS row
WITH row.nft_address AS address, row.token_id AS token
WITH DISTINCT address AS address, token AS token
MERGE (n:NFT {address: address, token: token})
RETURN n;
```

If the above query doesn't work in latest Neo4j versions, this query would work
```sh
//Create account node
:auto LOAD CSV WITH HEADERS FROM 'file:///transfers.csv' AS row
CALL {
  WITH row
  WITH DISTINCT row.nft_address AS address, row.token_id AS token
  MERGE (n:NFT {address: address, token: token})
  RETURN n
}
IN TRANSACTIONS OF 1000 ROWS
WITH n
RETURN n
```

Alternative query
```sh
CALL apoc.periodic.iterate('
    CALL apoc.load.csv("file:///transfers.csv") YIELD map AS row RETURN row
','
    WITH row.nft_address AS address, row.token_id AS token
    WITH DISTINCT address AS address, token AS token
    MERGE (n:NFT {address: address, token: token})
', {batchSize:10000, iterateList:true, parallel:true})
```

- Create transaction nodes

```sh
//Create transaction node
:auto USING PERIODIC COMMIT 1000 LOAD CSV WITH HEADERS FROM 'file:///transfers.csv' AS row
WITH row.event_id AS eventId, row.transaction_value AS trxValue, row.transaction_hash AS trxHash, row.timestamp AS timestamp
MERGE (t:Transaction {id: eventId, hash: trxHash, value:trxValue, timestamp: timestamp})
RETURN t;
```

If the above query doesn't work in latest Neo4j versions, this query would work
```sh
//Create account node
:auto LOAD CSV WITH HEADERS FROM 'file:///transfers.csv' AS row
CALL {
  WITH row
  MERGE (t:Transaction {id: row.event_id, hash: row.transaction_hash, value:row.transaction_value, timestamp: row.timestamp})
  RETURN t
}
IN TRANSACTIONS OF 1000 ROWS
WITH t
RETURN t
```

Alternative query
```sh
CALL apoc.periodic.iterate('
    CALL apoc.load.csv("file:///transfers.csv") YIELD map AS row RETURN row
','
    WITH row
    MERGE (t:Transaction {id: row.event_id, hash: row.transaction_hash, value:row.transaction_value, timestamp: row.timestamp})
    RETURN t
', {batchSize:10000, iterateList:true, parallel:true})
```

- Create transaction graph

```sh
//Create graph
:auto USING PERIODIC COMMIT 1000 LOAD CSV WITH HEADERS FROM 'file:///transfers.csv' AS row
WITH row.event_id AS eventId, row.from_address AS fromAddress, row.to_address AS toAddress, row.nft_address AS NFTAddress, row.token_id AS tokenId, row.transaction_hash AS trxHash, row.transaction_value AS trxValue, row.timestamp AS timestamp
MATCH (a1:Account), (a2:Account), (t:Transaction), (n:NFT)
WHERE a1.address = fromAddress AND a2.address = toAddress AND t.id = eventId AND t.hash = trxHash AND t.value = trxValue AND t.timestamp = timestamp AND n.address = NFTAddress AND n.token = tokenId
CREATE (a1)-[s:STARTED]->(t)-[c:CONTAINING]->(n)-[e:SENT_TO]->(a2)
RETURN a1, s, t, c, n, e, a2;
```