## **Graph Integrity Check** in your `pytest` test suite, you can **directly connect to Neo4j** from your test and query the graph to confirm expected relationships exist.

---

## ✅ Step-by-step: Graph Integrity Check with Neo4j

### 1. 📦 Install the Python Neo4j Driver (if not already)

```bash
pip install neo4j
```

---

### 2. 🧪 Add to `test_main.py`

At the top of your file:

```python
from neo4j import GraphDatabase
import os

NEO4J_URI = os.getenv("NEO4J_URI", "bolt://localhost:7687")
NEO4J_USER = os.getenv("NEO4J_USER", "neo4j")
NEO4J_PASSWORD = os.getenv("NEO4J_PASSWORD", "test")

driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))
```

---

### 3. 📋 Test: Verify Friendship Exists in Graph

```python
def test_neo4j_friendship_exists():
    with driver.session() as session:
        result = session.run("""
            MATCH (a:User {username: $user1})-[r:FRIENDS_WITH]-(b:User {username: $user2})
            RETURN COUNT(r) AS count
        """, user1="testuser", user2="frienduser")
        
        count = result.single()["count"]
        assert count == 1, "Friendship relationship not found in graph"
```

---

### 4. 🧹 Optionally: Clean-Up Test Data

To avoid polluting your dev database, you can delete test nodes:

```python
def teardown_module(module):
    with driver.session() as session:
        session.run("""
            MATCH (u:User)
            WHERE u.username IN ['testuser', 'frienduser']
            DETACH DELETE u
        """)
```

---

### ✅ Full Test Flow Now Includes

* User creation
* JWT token auth
* Friendship creation
* REST validation
* **Graph-level Cypher validation**
