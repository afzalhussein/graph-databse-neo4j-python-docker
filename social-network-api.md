## **minimal social network API**

---

## ✅ New Features:

* ✅ `POST /user` → create a user
* ✅ `POST /friendship` → create a friendship between users
* ✅ `GET /friends/{name}` → get direct friends
* ✅ `GET /fof/{name}` → get friends-of-friends

---

## 🧱 Full Project Structure

```
social-graph-api/
├── main.py
├── requirements.txt
└── docker-compose.yml
```

---

## 📄 `requirements.txt`

```txt
fastapi
uvicorn
neo4j
```

---

## 🐳 `docker-compose.yml`

```yaml
version: '3.8'

services:
  neo4j:
    image: neo4j:5
    container_name: neo4j-fastapi
    environment:
      - NEO4J_AUTH=neo4j/password
    ports:
      - "7474:7474"
      - "7687:7687"

  app:
    build: .
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    command: uvicorn main:app --host 0.0.0.0 --reload
    depends_on:
      - neo4j
```

> (You can add a Dockerfile later for production; for now we'll run FastAPI from host)

---

## 🧩 `main.py` (Full Version)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from neo4j import GraphDatabase

app = FastAPI()

# Connect to Neo4j
NEO4J_URI = "bolt://neo4j:7687"  # Use "localhost" if running without Docker Compose
NEO4J_USER = "neo4j"
NEO4J_PASS = "password"
driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASS))

# -------- Data Models --------
class UserCreate(BaseModel):
    name: str

class FriendshipCreate(BaseModel):
    user1: str
    user2: str

# -------- API Endpoints --------

@app.post("/user")
def create_user(user: UserCreate):
    with driver.session() as session:
        result = session.run("MATCH (u:User {name: $name}) RETURN u", name=user.name)
        if result.peek():
            raise HTTPException(status_code=400, detail="User already exists")

        session.run("CREATE (:User {name: $name})", name=user.name)
        return {"message": f"User {user.name} created"}

@app.post("/friendship")
def create_friendship(f: FriendshipCreate):
    with driver.session() as session:
        # Ensure both users exist
        for name in [f.user1, f.user2]:
            result = session.run("MATCH (u:User {name: $name}) RETURN u", name=name)
            if not result.peek():
                raise HTTPException(status_code=404, detail=f"User '{name}' not found")

        # Create FRIEND relation
        session.run("""
            MATCH (a:User {name: $name1}), (b:User {name: $name2})
            MERGE (a)-[:FRIEND]->(b)
        """, name1=f.user1, name2=f.user2)

        return {"message": f"{f.user1} is now friends with {f.user2}"}

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

@app.get("/users")
def list_users():
    with driver.session() as session:
        result = session.run("MATCH (u:User) RETURN u.name AS name ORDER BY name")
        return {"users": [record["name"] for record in result]}
```

---

## 🚀 Run Instructions

1. **Start services**:

   ```bash
   docker-compose up
   ```

2. **Open Swagger UI**:
   [http://localhost:8000/docs](http://localhost:8000/docs)

---

## 🧪 Example Usage

* `POST /user` with `{"name": "Alice"}`
* `POST /user` with `{"name": "Bob"}`
* `POST /friendship` with `{"user1": "Alice", "user2": "Bob"}`
* `GET /friends/Alice` → returns `["Bob"]`
* `GET /fof/Alice` → returns friends-of-friends

---

## 🧠 Extensions You Can Add Later

* Bi-directional friendships (e.g., auto-create reverse `FRIEND` edge)
* Add `age`, `location` to users
* Recommend friends (mutual connections)
* Delete users or relationships
* Containerize with a `Dockerfile`

