## **Cypher schema setup** (social `FRIENDS_WITH` + admin `HAS_ROLE`/`GRANTS`) into your **FastAPI + Neo4j + Docker** project, here's a full setup:

---

### ✅ 1. **Project Structure**

```
project/
├── app/
│   ├── main.py
│   ├── db.py
│   ├── models.py
│   ├── schema_init.py   <-- Cypher schema runner
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
```

---

### 📦 2. `requirements.txt`

```txt
fastapi
uvicorn
neo4j
```

---

### 🐳 3. `docker-compose.yml`

```yaml
version: '3'
services:
  neo4j:
    image: neo4j:5
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      NEO4J_AUTH: neo4j/test123
    volumes:
      - neo4j-data:/data

  fastapi:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - neo4j
    environment:
      NEO4J_URI: bolt://neo4j:7687
      NEO4J_USER: neo4j
      NEO4J_PASSWORD: test123

volumes:
  neo4j-data:
```

---

### 🐍 4. `db.py` – Neo4j Driver

```python
from neo4j import GraphDatabase
import os

NEO4J_URI = os.getenv("NEO4J_URI", "bolt://localhost:7687")
NEO4J_USER = os.getenv("NEO4J_USER", "neo4j")
NEO4J_PASSWORD = os.getenv("NEO4J_PASSWORD", "test123")

driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))

def get_driver():
    return driver
```

---

### 🧱 5. `schema_init.py` – Cypher Bootstrap

```python
from db import get_driver

def setup_schema():
    cypher_queries = [

        # Constraints
        "CREATE CONSTRAINT user_id_unique IF NOT EXISTS ON (u:User) ASSERT u.id IS UNIQUE",
        "CREATE CONSTRAINT role_name_unique IF NOT EXISTS ON (r:Role) ASSERT r.name IS UNIQUE",
        "CREATE CONSTRAINT permission_name_unique IF NOT EXISTS ON (p:Permission) ASSERT p.name IS UNIQUE",

        # Sample Users
        "MERGE (:User {id: 'u1', name: 'Alice', type: 'social', email: 'alice@example.com'})",
        "MERGE (:User {id: 'u2', name: 'Bob', type: 'social', email: 'bob@example.com'})",
        "MERGE (:User {id: 'u3', name: 'AdminUser', type: 'admin', email: 'admin@example.com'})",

        # Friendships
        """MATCH (a:User {id: 'u1'}), (b:User {id: 'u2'})
           MERGE (a)-[:FRIENDS_WITH]->(b)
           MERGE (b)-[:FRIENDS_WITH]->(a)""",

        # Roles
        "MERGE (:Role {name: 'moderator'})",
        "MERGE (:Role {name: 'super_admin'})",

        # Permissions
        "MERGE (:Permission {name: 'delete_user'})",
        "MERGE (:Permission {name: 'ban_user'})",
        "MERGE (:Permission {name: 'edit_post'})",

        # GRANTS
        """MATCH (r:Role {name: 'moderator'}), (p:Permission {name: 'edit_post'})
           MERGE (r)-[:GRANTS]->(p)""",

        """MATCH (r:Role {name: 'super_admin'}),
                     (p1:Permission {name: 'delete_user'}),
                     (p2:Permission {name: 'ban_user'}),
                     (p3:Permission {name: 'edit_post'})
           MERGE (r)-[:GRANTS]->(p1)
           MERGE (r)-[:GRANTS]->(p2)
           MERGE (r)-[:GRANTS]->(p3)""",

        # Role Assignment
        """MATCH (u:User {id: 'u3'}), (r:Role {name: 'super_admin'})
           MERGE (u)-[:HAS_ROLE]->(r)"""
    ]

    with get_driver().session() as session:
        for query in cypher_queries:
            session.run(query)
        print("✔ Schema initialized")
```

---

### 🚀 6. `main.py` – FastAPI App

```python
from fastapi import FastAPI
from schema_init import setup_schema

app = FastAPI()

@app.on_event("startup")
def startup_event():
    setup_schema()

@app.get("/")
def root():
    return {"message": "Neo4j + FastAPI ready"}
```

---

### 🛠 7. `Dockerfile`

```dockerfile
FROM python:3.10-slim

WORKDIR /app
COPY ./app /app
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### ✅ Run it all

```bash
docker-compose up --build
```

Once it starts:

* Open [http://localhost:7474](http://localhost:7474) → Login: `neo4j/test123`
* Open [http://localhost:8000](http://localhost:8000) → FastAPI is live
* Query the Neo4j browser to confirm the graph schema

