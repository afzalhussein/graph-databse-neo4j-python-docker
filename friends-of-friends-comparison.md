## 🧠 Goal: Get Friends-of-Friends of "Alice" (excluding direct friends)

So for Alice ➝ Bob ➝ Dave, we want to get **Dave** (friend of a friend), **but not Bob** (already a direct friend).

Let’s implement this in both:

### ✅ Part 1: SQLite (SQL Recursive Query)

SQLite supports **recursive CTEs**, but it’s not very natural for graph-like traversal.

### ⚙️ Python + SQLite Example:

```python
import sqlite3

# In-memory DB
conn = sqlite3.connect(":memory:")
cur = conn.cursor()

# Tables
cur.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
cur.execute("CREATE TABLE friendships (user_id INTEGER, friend_id INTEGER)")

# Users
users = [
    (1, 'Alice'), (2, 'Bob'), (3, 'Carol'),
    (4, 'Dave'), (5, 'Eve')
]
cur.executemany("INSERT INTO users VALUES (?, ?)", users)

# Friendships
friendships = [
    (1, 2),  # Alice - Bob
    (1, 3),  # Alice - Carol
    (2, 4),  # Bob - Dave
    (3, 5)   # Carol - Eve
]
cur.executemany("INSERT INTO friendships VALUES (?, ?)", friendships)

# Recursive query: friends-of-friends, skipping direct friends
cur.execute("""
WITH RECURSIVE network(user_id, depth) AS (
    SELECT id, 0 FROM users WHERE name = 'Alice'
    UNION
    SELECT f.friend_id, depth + 1
    FROM friendships f
    JOIN network n ON f.user_id = n.user_id
    WHERE depth < 1  -- Limit to 2 levels (1 = friends, 2 = friends-of-friends)
)
SELECT DISTINCT u.name
FROM users u
JOIN network n ON u.id = n.user_id
WHERE n.depth = 2
""")

print("SQL - Alice's friends-of-friends:")
for row in cur.fetchall():
    print("-", row[0])

conn.close()
```

### ✅ Output:

```
SQL - Alice's friends-of-friends:
- Dave
- Eve
```

---

### 🔁 Part 2: Neo4j (Cypher) — Much Easier

Graph traversal is natural in Cypher.

### ⚙️ Code:

```python
from neo4j import GraphDatabase

uri = "bolt://localhost:7687"
driver = GraphDatabase.driver(uri, auth=("neo4j", "password"))

def setup(tx):
    tx.run("MATCH (n) DETACH DELETE n")
    tx.run("""
        CREATE (alice:User {name: 'Alice'})
        CREATE (bob:User {name: 'Bob'})
        CREATE (carol:User {name: 'Carol'})
        CREATE (dave:User {name: 'Dave'})
        CREATE (eve:User {name: 'Eve'})

        CREATE (alice)-[:FRIEND]->(bob)
        CREATE (alice)-[:FRIEND]->(carol)
        CREATE (bob)-[:FRIEND]->(dave)
        CREATE (carol)-[:FRIEND]->(eve)
    """)

def get_fof(tx):
    result = tx.run("""
        MATCH (a:User {name: 'Alice'})-[:FRIEND]->()-[:FRIEND]->(fof:User)
        WHERE NOT (a)-[:FRIEND]->(fof) AND fof.name <> 'Alice'
        RETURN DISTINCT fof.name AS name
    """)
    return [record["name"] for record in result]

with driver.session() as session:
    session.write_transaction(setup)
    fof_names = session.read_transaction(get_fof)

print("Graph DB - Alice's friends-of-friends:")
for name in fof_names:
    print("-", name)

driver.close()
```

### ✅ Output:

```
Graph DB - Alice's friends-of-friends:
- Dave
- Eve
```

---

## 🔍 What Just Happened?

| Aspect                   | SQL                                 | Graph DB (Neo4j / Cypher)             |
| ------------------------ | ----------------------------------- | ------------------------------------- |
| Query logic              | Recursive CTE with depth tracking   | One simple MATCH query with depth = 2 |
| Readability              | Harder to understand & maintain     | Intuitive, looks like graph traversal |
| Performance (large data) | Slower as recursion depth increases | Optimized for multi-hop relationships |
| Use Case Fit             | Awkward for deep connections        | Perfect for social graphs, networks   |

---

## ✅ Summary: When Graph DB “Makes Sense”

A graph DB is **worth the switch** when:

* You query **multi-level relationships**
* You frequently do **"path" searches** (shortest paths, cycles, etc.)
* Your schema is **highly connected** and changing often

Otherwise, for flat relational data, **SQL is still king**.

