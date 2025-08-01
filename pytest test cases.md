## A straightforward set of **pytest** test cases for your FastAPI app covering:

* User registration
* Token login
* Accessing protected `/users/me`
* Checking admin permission on `/admin`

---

### 1. Install pytest and test client dependencies

```bash
pip install pytest httpx
```

---

### 2. Save this as `test_main.py`

```python
import pytest
from fastapi.testclient import TestClient
from main import app  # your FastAPI app

client = TestClient(app)

@pytest.fixture(scope="module")
def registered_user():
    username = "testuser"
    password = "testpass"
    # Register user (ignore if already exists)
    client.post("/register", json={"username": username, "password": password})
    return {"username": username, "password": password}

@pytest.fixture(scope="module")
def auth_token(registered_user):
    response = client.post(
        "/token",
        data={"username": registered_user["username"], "password": registered_user["password"]},
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    assert response.status_code == 200
    return response.json()["access_token"]

def test_register_duplicate():
    # Attempt to register same user again should fail
    response = client.post("/register", json={"username": "testuser", "password": "testpass"})
    assert response.status_code == 400
    assert response.json()["detail"] == "Username already registered"

def test_login_fail():
    response = client.post(
        "/token",
        data={"username": "wrong", "password": "wrong"},
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    assert response.status_code == 401

def test_get_me(auth_token):
    response = client.get("/users/me", headers={"Authorization": f"Bearer {auth_token}"})
    assert response.status_code == 200
    assert response.json()["username"] == "testuser"

def test_get_me_no_token():
    response = client.get("/users/me")
    assert response.status_code == 401

def test_admin_access_forbidden(auth_token):
    # Regular user should NOT access admin route
    response = client.get("/admin", headers={"Authorization": f"Bearer {auth_token}"})
    assert response.status_code == 403

def test_admin_access_allowed():
    # Assuming you have an admin user seeded with username "AdminUser" and password "test"
    response = client.post(
        "/token",
        data={"username": "AdminUser", "password": "test"},
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    assert response.status_code == 200
    token = response.json()["access_token"]
    response = client.get("/admin", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 200
    assert "Welcome admin" in response.json()["msg"]
```

---

### 3. Run tests

```bash
pytest test_main.py -v
```

---

This gives you a solid automated test base for your authentication flow.

