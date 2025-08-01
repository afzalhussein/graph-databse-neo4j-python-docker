### 1. **SQLite (SQL database)**

### 2. **Neo4j (Graph database using Cypher)**

We’ll simulate both versions of:
🧠 *"Get the list of Alice’s friends"*

---

## ✅ Part 1: SQL (SQLite with `sqlite3` in Python)

### ⚙️ Step-by-Step Code

```python
import sqlite3

# Create in-memory SQLite DB
conn = sqlite3.connect(":memory:")
cur = conn.cursor()

# Create tables
cur.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
cur.execute("CREATE TABLE friendships (user_id INTEGER, friend_id INTEGER)")

# Insert users
users = [(1, 'Alice'), (2, 'Bob'), (3, 'Carol')]
cur.executemany("INSERT INTO users VALUES (?, ?)", users)

# Create friendships: Alice -> Bob, Alice -> Carol
friendships = [(1, 2), (1, 3)]
cur.executemany("INSERT INTO friendships VALUES (?, ?)", friendships)

# Query: Get friends of Alice
cur.execute("""
    SELECT u2.name AS friend_name
    FROM users u1
    JOIN friendships f ON u1.id = f.user_id
    JOIN users u2 ON f.friend_id = u2.id
    WHERE u1.name = 'Alice'
""")

# Print results
print("SQL - Alice's friends:")
for row in cur.fetchall():
    print("-", row[0])

conn.close()
```

### ✅ Output:

```
SQL - Alice's friends:
- Bob
- Carol
```

---

## 🔁 Part 2: Neo4j (Graph DB with Cypher via `neo4j` Python driver)

### 🧱 Requirements:

Install the Neo4j driver:

```bash
pip install neo4j
```

Ensure Neo4j is running locally on port `7687`, with `neo4j` / `password` as credentials.

### ⚙️ Code:

```python
from neo4j import GraphDatabase

# Connect to local Neo4j
uri = "bolt://localhost:7687"
driver = GraphDatabase.driver(uri, auth=("neo4j", "password"))

def setup_graph(tx):
    tx.run("MATCH (n) DETACH DELETE n")  # Clear DB

    tx.run("""
        CREATE (a:User {name: 'Alice'})
        CREATE (b:User {name: 'Bob'})
        CREATE (c:User {name: 'Carol'})
        CREATE (a)-[:FRIEND]->(b)
        CREATE (a)-[:FRIEND]->(c)
    """)

def get_friends_of_alice(tx):
    result = tx.run("""
        MATCH (a:User {name: 'Alice'})-[:FRIEND]->(friend:User)
        RETURN friend.name AS friend_name
    """)
    return [record["friend_name"] for record in result]

with driver.session() as session:
    session.write_transaction(setup_graph)
    friends = session.read_transaction(get_friends_of_alice)

print("Graph DB - Alice's friends:")
for name in friends:
    print("-", name)

driver.close()
```

### ✅ Output:

```
Graph DB - Alice's friends:
- Bob
- Carol
```

---

## 🧠 Summary

| Feature        | SQLite (SQL)           | Neo4j (Graph DB)                   |
| -------------- | ---------------------- | ---------------------------------- |
| Setup          | Lightweight, in-memory | Requires Neo4j server running      |
| Syntax         | Familiar SQL JOINs     | Cypher with intuitive traversal    |
| Performance    | Slower with many JOINs | Faster with deep and complex links |
| Ideal Use Case | Tabular/flat data      | Networked or relational data       |


