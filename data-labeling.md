## Data labeling process
- Export transaction data grouped by NFT object into a single CSV file
- Create account nodes after removing the duplicate addresses
```sh
//Removing duplicate addresses & create nodes
LOAD CSV WITH HEADERS FROM 'file:///filename.csv' AS row
WITH row.event_id AS eventId, row.from_address AS fromAddress, row.to_address AS toAddress, row.transaction_value AS trxValue
WITH [fromAddress] AS list1, [toAddress] AS list2
UNWIND list1 + list2 AS combinedList
WITH DISTINCT combinedList AS address
MERGE (a:Account {address: address})
RETURN a;
```
- Create relationships among nodes
```sh
//Create relationships
LOAD CSV WITH HEADERS FROM 'file:///filename.csv' AS row
WITH row.event_id AS eventId, row.from_address AS fromAddress, row.to_address AS toAddress, row.transaction_value AS trxValue, row.timestamp AS timestamp
MATCH (a1:Account), (a2:Account)
WHERE a1.address = fromAddress AND a2.address = toAddress
CREATE (a1)-[t:TRANSFERRED_TO {trxId: eventId, trxValue: trxValue, timestamp:t imestamp}]->(a2)
RETURN a1, t, a2;
```
- Apply graph algorithms to the graph model
 