## **basic token-based authentication** using **OAuth2 with Password (Bearer token)**—the standard approach in FastAPI.

---

## 🔐 Overview of What We’ll Add:

1. ✅ `/register` – Create a user with a password
2. ✅ `/token` – Login endpoint that returns a Bearer token
3. ✅ Secure all existing endpoints with token auth
4. ✅ Passwords are **hashed** using `bcrypt`
5. ✅ Auth tokens are **JWTs** signed with a secret

---

## 📦 Install Additional Dependencies

Add to your `requirements.txt`:

```txt
python-jose[cryptography]
passlib[bcrypt]
```

Then update:

```bash
pip install -r requirements.txt
```

---

## 🧩 Updated `main.py` with Auth Support

Here's a **secure and complete version**:

### 🔁 Replace your `main.py` with this:

```python
from fastapi import FastAPI, HTTPException, Depends, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel
from passlib.context import CryptContext
from jose import JWTError, jwt
from neo4j import GraphDatabase
from typing import List
from datetime import datetime, timedelta

app = FastAPI()

# Neo4j connection
NEO4J_URI = "bolt://neo4j:7687"
driver = GraphDatabase.driver(NEO4J_URI, auth=("neo4j", "password"))

# Auth config
SECRET_KEY = "your-super-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


# ---------------- Models ----------------
class Token(BaseModel):
    access_token: str
    token_type: str

class UserCreate(BaseModel):
    username: str
    password: str

class FriendshipCreate(BaseModel):
    user1: str
    user2: str


# ---------------- Auth Helpers ----------------
def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    return pwd_context.hash(password)

def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Invalid token",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if not username:
            raise credentials_exception
        return username
    except JWTError:
        raise credentials_exception

# ---------------- Auth Endpoints ----------------

@app.post("/register", status_code=201)
def register(user: UserCreate):
    with driver.session() as session:
        result = session.run("MATCH (u:User {name: $name}) RETURN u", name=user.username)
        if result.peek():
            raise HTTPException(status_code=400, detail="User already exists")
        hashed_pw = get_password_hash(user.password)
        session.run("CREATE (:User {name: $name, password: $password})",
                    name=user.username, password=hashed_pw)
        return {"message": f"User '{user.username}' registered successfully"}

@app.post("/token", response_model=Token)
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    with driver.session() as session:
        result = session.run("MATCH (u:User {name: $name}) RETURN u.password AS password",
                             name=form_data.username)
        record = result.single()
        if not record or not verify_password(form_data.password, record["password"]):
            raise HTTPException(status_code=401, detail="Invalid credentials")

    access_token = create_access_token(data={"sub": form_data.username},
                                       expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    return {"access_token": access_token, "token_type": "bearer"}


# ---------------- Protected Endpoints ----------------

@app.get("/users", dependencies=[Depends(get_current_user)])
def list_users():
    with driver.session() as session:
        result = session.run("MATCH (u:User) RETURN u.name AS name ORDER BY name")
        return {"users": [record["name"] for record in result]}

@app.post("/friendship", dependencies=[Depends(get_current_user)])
def create_friendship(f: FriendshipCreate):
    with driver.session() as session:
        for name in [f.user1, f.user2]:
            res = session.run("MATCH (u:User {name: $name}) RETURN u", name=name)
            if not res.peek():
                raise HTTPException(status_code=404, detail=f"User '{name}' not found")
        session.run("""
            MATCH (a:User {name: $name1}), (b:User {name: $name2})
            MERGE (a)-[:FRIEND]->(b)
        """, name1=f.user1, name2=f.user2)
        return {"message": f"{f.user1} is now friends with {f.user2}"}

@app.get("/friends/{name}", dependencies=[Depends(get_current_user)])
def get_friends(name: str):
    with driver.session() as session:
        result = session.run("""
            MATCH (u:User {name: $name})-[:FRIEND]->(f:User)
            RETURN f.name AS friend
        """, name=name)
        return {"friends": [record["friend"] for record in result]}

@app.get("/fof/{name}", dependencies=[Depends(get_current_user)])
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

## 🔐 How to Use Auth

### 🔹 Register a user

`POST /register`

```json
{
  "username": "Alice",
  "password": "secret"
}
```

### 🔹 Login to get token

`POST /token`
Content-Type: `application/x-www-form-urlencoded`

```
username=Alice
password=secret
```

➡️ Returns:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6...",
  "token_type": "bearer"
}
```

### 🔹 Authenticated requests

Use the token in `Authorization` header:

```
Authorization: Bearer <token>
```

In Swagger UI: click **Authorize** (🔒 icon) and paste `Bearer <token>`.

---

## ✅ Summary

| Feature          | Implementation                   |
| ---------------- | -------------------------------- |
| Auth type        | OAuth2 with Bearer token (JWT)   |
| Password storage | Secure bcrypt hashes             |
| Token expiration | 30 minutes                       |
| Protected routes | All friendship & query endpoints |

