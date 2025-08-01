## **integrate RBAC access control into FastAPI endpoints** using roles and permissions stored in **Neo4j**, enforced via **JWT authentication**.

---

## ✅ Overview

We’ll:

1. Use JWT to authenticate users.
2. Look up the user’s roles & permissions in Neo4j.
3. Enforce RBAC with a dependency on endpoints.

---

## 🛠️ 1. JWT & User Setup (recap)

Assume your JWT payload has:

```json
{
  "sub": "rbacuser"
}
```

We’ll use `sub` as the username to fetch permissions from the graph.

---

## 🧱 2. RBAC Dependency (FastAPI)

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

SECRET_KEY = "supersecret"  # For real apps, use a secure env var
ALGORITHM = "HS256"

def decode_token(token: str):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return username
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

def get_user_permissions(username: str):
    with driver.session() as session:
        result = session.run("""
            MATCH (u:User {username: $username})-[:HAS_ROLE]->(:Role)-[:HAS_PERMISSION]->(p:Permission)
            RETURN COLLECT(DISTINCT p.name) AS permissions
        """, username=username)
        record = result.single()
        return set(record["permissions"] or [])

def require_permission(permission: str):
    def permission_dependency(token: str = Depends(oauth2_scheme)):
        username = decode_token(token)
        permissions = get_user_permissions(username)
        if permission not in permissions:
            raise HTTPException(status_code=403, detail="Forbidden")
        return username  # Return user for use in route
    return permission_dependency
```

---

## 🚀 3. Example API Endpoints

```python
@app.get("/dashboard")
def dashboard(username: str = Depends(require_permission("read"))):
    return {"message": f"Welcome {username} — you can read the dashboard."}

@app.post("/content")
def create_content(username: str = Depends(require_permission("write"))):
    return {"message": f"{username} can create content."}

@app.delete("/content")
def delete_content(username: str = Depends(require_permission("delete"))):
    return {"message": f"{username} can delete content."}
```

---

## 🔐 4. Token Generation for Testing (dev-only)

You can use this for development or testing:

```python
from datetime import datetime, timedelta

def create_test_token(username: str):
    to_encode = {"sub": username, "exp": datetime.utcnow() + timedelta(hours=1)}
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
```

---

## 🧪 5. How to Test

```bash
TOKEN=$(curl -X POST ... | jq -r '.access_token')  # or use the test token
curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/dashboard
```

---

## ✅ Summary

* RBAC roles & permissions live in Neo4j.
* JWT token carries the `username`.
* Middleware (`require_permission`) enforces access on routes.

