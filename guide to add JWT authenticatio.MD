## a concise guide to add **JWT authentication** to your FastAPI + Neo4j app with:

* User registration (with hashed passwords)
* Login endpoint to get JWT token
* Protected routes requiring JWT bearer tokens
* Role checking example

---

### 1. **Add dependencies**

Add these to your `requirements.txt`:

```txt
fastapi
uvicorn
neo4j
python-jose[cryptography]
passlib[bcrypt]
```

Install with:

```bash
pip install -r requirements.txt
```

---

### 2. **Update `main.py`**

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel
from neo4j import GraphDatabase
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional

app = FastAPI()

# Neo4j Setup
NEO4J_URI = "bolt://localhost:7687"
NEO4J_USER = "neo4j"
NEO4J_PASSWORD = "test"
driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))

# JWT Setup
SECRET_KEY = "your-secret-key-here"  # Change to a secure random key in prod
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Pydantic models
class UserCreate(BaseModel):
    username: str
    password: str

class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: Optional[str] = None

# Utility functions
def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def get_user(username: str):
    with driver.session() as session:
        result = session.run("MATCH (u:User {name: $username}) RETURN u.password AS password, u.name AS name", username=username)
        record = result.single()
        if record:
            return {"username": record["name"], "hashed_password": record["password"]}
    return None

def authenticate_user(username: str, password: str):
    user = get_user(username)
    if not user or not verify_password(password, user["hashed_password"]):
        return False
    return user

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username)
    except JWTError:
        raise credentials_exception
    user = get_user(token_data.username)
    if user is None:
        raise credentials_exception
    return user

# Routes

@app.post("/register", status_code=201)
def register(user: UserCreate):
    with driver.session() as session:
        existing = session.run("MATCH (u:User {name: $name}) RETURN u", name=user.username)
        if existing.peek():
            raise HTTPException(status_code=400, detail="Username already registered")
        hashed_pw = get_password_hash(user.password)
        session.run("CREATE (:User {name: $name, password: $password, type: 'social'})", name=user.username, password=hashed_pw)
    return {"msg": "User registered successfully"}

@app.post("/token", response_model=Token)
async def login_for_access_token(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=401, detail="Incorrect username or password")
    access_token = create_access_token(data={"sub": user["username"]}, expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/users/me")
async def read_users_me(current_user: dict = Depends(get_current_user)):
    return {"username": current_user["username"]}

# Example protected route with role check (optional)
@app.get("/admin")
async def admin_route(current_user: dict = Depends(get_current_user)):
    with driver.session() as session:
        result = session.run("""
            MATCH (u:User {name: $username})-[:HAS_ROLE]->(r:Role)
            RETURN r.name AS role
        """, username=current_user["username"])
        roles = [record["role"] for record in result]
    if "super_admin" not in roles:
        raise HTTPException(status_code=403, detail="Not enough permissions")
    return {"msg": f"Welcome admin {current_user['username']}"}
```

---

### 3. **How to test**

* Register a user:

```bash
curl -X POST "http://localhost:8000/register" -H "Content-Type: application/json" -d '{"username":"alice","password":"secret"}'
```

* Get a token:

```bash
curl -X POST "http://localhost:8000/token" -d "username=alice&password=secret" -H "Content-Type: application/x-www-form-urlencoded"
```

* Use the token to call a protected endpoint:

```bash
curl -H "Authorization: Bearer <token>" "http://localhost:8000/users/me"
```

* If your user has admin role (`super_admin`), call `/admin` endpoint.

