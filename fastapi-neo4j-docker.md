## 🧱 Overview

We'll:

1. 🐳 **Run Neo4j in Docker**
2. ⚙️ Set up **FastAPI** with a Neo4j driver
3. 🧪 Create two endpoints:

   * `/friends/{name}` → direct friends
   * `/fof/{name}` → friends-of-friends

---

## 🐳 Step 1: Run Neo4j via Docker

```bash
docker run \
  --name neo4j-fastapi \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password \
  -d neo4j:5
```

* Admin panel: [http://localhost:7474](http://localhost:7474)
* Bolt port: `7687`
* Credentials: `neo4j / password`

---

## ⚙️ Step 2: Python Environment Setup

```bash
pip install fastapi uvicorn neo4j
```

---

## 🧩 Step 3: Create `main.py`

```python
from fastapi import FastAPI
from neo4j import GraphDatabase

app = FastAPI()

# Neo4j driver
NEO4J_URI = "bolt://localhost:7687"
NEO4J_USER = "neo4j"
NEO4J_PASS = "password"

driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASS))


@app.on_event("startup")
def seed_graph():
    with driver.session() as session:
        session.run("MATCH (n) DETACH DELETE n")
        session.run("""
            CREATE (a:User {name: 'Alice'})
            CREATE (b:User {name: 'Bob'})
            CREATE (c:User {name: 'Carol'})
            CREATE (d:User {name: 'Dave'})
            CREATE (e:User {name: 'Eve'})
            CREATE (a)-[:FRIEND]->(b)
            CREATE (a)-[:FRIEND]->(c)
            CREATE (b)-[:FRIEND]->(d)
            CREATE (c)-[:FRIEND]->(e)
        """)


@app.get("/friends/{name}")
def get_friends(name: str):
    with driver.session() as session:
        result = session.run("""
            MATCH (u:User {name: $name})-[:FRIEND]->(f:User)
            RETURN f.name AS friend
        """, name=name)
        return {"friends": [record["friend"] for record in result]}


@app.get("/fof/{name}")
def get_friends_of_friends(name: str):
    with driver.session() as session:
        result = session.run("""
            MATCH (u:User {name: $name})-[:FRIEND]->()-[:FRIEND]->(fof:User)
            WHERE NOT (u)-[:FRIEND]->(fof) AND fof.name <> $name
            RETURN DISTINCT fof.name AS friend_of_friend
        """, name=name)
        return {"friends_of_friends": [record["friend_of_friend"] for record in result]}
```

---

## 🚀 Step 4: Run FastAPI

```bash
uvicorn main:app --reload
```

Visit in browser:

* 🔹 `http://localhost:8000/friends/Alice` → `["Bob", "Carol"]`
* 🔹 `http://localhost:8000/fof/Alice` → `["Dave", "Eve"]`
* 🔹 `http://localhost:8000/docs` → Swagger UI

---

## 🧠 Summary

| Part   | Tech                                        | Purpose                    |
| ------ | ------------------------------------------- | -------------------------- |
| DB     | Neo4j in Docker                             | Graph database with Cypher |
| API    | FastAPI                                     | REST endpoints             |
| Driver | `neo4j` Python package                      | Talk to Neo4j              |
| Result | `/friends/Alice` and `/fof/Alice` endpoints | Graph-based social queries |

