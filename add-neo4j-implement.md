## the `neo4j` Python driver isn't installed in the current environment, so the code can't run yet. But here's what you can do to set it up properly in your local system with Docker and FastAPI:

---

### ✅ 1. **Install Required Packages**

Create a `requirements.txt` file with:

```txt
fastapi
uvicorn
neo4j
```

Then run:

```bash
pip install -r requirements.txt
```

---

### ✅ 2. **Docker Setup**

#### `docker-compose.yml`

```yaml
version: '3.8'

services:
  neo4j:
    image: neo4j:5.18
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      NEO4J_AUTH: neo4j/test
```

Start Neo4j:

```bash
docker-compose up -d
```

---

### ✅ 3. **FastAPI App with Neo4j Schema Setup**

Save this as `main.py`:

```python
from fastapi import FastAPI
from neo4j import GraphDatabase
import uvicorn

app = FastAPI()

NEO4J_URI = "bolt://localhost:7687"
NEO4J_USER = "neo4j"
NEO4J_PASSWORD = "test"

driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))

@app.on_event("startup")
def init_neo4j():
    schema = [
        # Constraints
        "CREATE CONSTRAINT user_id_unique IF NOT EXISTS ON (u:User) ASSERT u.id IS UNIQUE",
        "CREATE CONSTRAINT role_name_unique IF NOT EXISTS ON (r:Role) ASSERT r.name IS UNIQUE",
        "CREATE CONSTRAINT permission_name_unique IF NOT EXISTS ON (p:Permission) ASSERT p.name IS UNIQUE",

        # Social Users
        "MERGE (alice:User {id: 'u1', name: 'Alice', type: 'social'})",
        "MERGE (bob:User {id: 'u2', name: 'Bob', type: 'social'})",
        "MATCH (a:User {id: 'u1'}), (b:User {id: 'u2'}) "
        "MERGE (a)-[:FRIENDS_WITH]->(b) "
        "MERGE (b)-[:FRIENDS_WITH]->(a)",

        # Admin Users
        "MERGE (admin:User {id: 'u3', name: 'AdminUser', type: 'admin'})",
        "MERGE (r1:Role {name: 'moderator'}), (r2:Role {name: 'super_admin'})",
        "MERGE (p1:Permission {name: 'edit_post'})",
        "MERGE (p2:Permission {name: 'delete_user'})",
        "MERGE (p3:Permission {name: 'ban_user'})",
        "MERGE (r1)-[:GRANTS]->(p1)",
        "MERGE (r2)-[:GRANTS]->(p1)",
        "MERGE (r2)-[:GRANTS]->(p2)",
        "MERGE (r2)-[:GRANTS]->(p3)",
        "MATCH (admin:User {id: 'u3'}), (r2:Role {name: 'super_admin'}) MERGE (admin)-[:HAS_ROLE]->(r2)"
    ]
    with driver.session() as session:
        for stmt in schema:
            session.run(stmt)

@app.get("/")
def root():
    return {"message": "Neo4j schema initialized successfully"}

if __name__ == "__main__":
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

---

### 🧪 Test

Once everything is running:

* Access FastAPI at: [http://localhost:8000](http://localhost:8000)
* Neo4j browser: [http://localhost:7474](http://localhost:7474)
  Username: `neo4j`, Password: `test`

