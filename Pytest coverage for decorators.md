## Write **Pytest unit tests** to cover the permission decorators: `@has_permission`, `@has_any_permission`, and `@has_all_permissions`.

---

## 🧪 Setup: `test_decorators.py`

```python
import pytest
from fastapi import Request, HTTPException
from starlette.testclient import TestClient
from fastapi import FastAPI

from myauth import has_permission, has_any_permission, has_all_permissions

# A mock token decoder and permission fetcher you can patch
from myauth import decode_token, get_user_permissions

app = FastAPI()

@app.get("/only-admin")
@has_permission("admin")
async def only_admin(username: str):
    return {"ok": True, "user": username}

@app.get("/any-read-or-write")
@has_any_permission("read", "write")
async def any_read_write(username: str):
    return {"ok": True, "user": username}

@app.get("/must-have-both")
@has_all_permissions("admin", "write")
async def both_required(username: str):
    return {"ok": True, "user": username}

client = TestClient(app)
```

---

## 🧪 Fixtures & Mocks: `conftest.py`

```python
import pytest
from unittest.mock import patch

@pytest.fixture
def mock_user_permissions():
    with patch("myauth.decode_token", return_value="john"), \
         patch("myauth.get_user_permissions") as mocked:
        yield mocked
```

---

## ✅ Tests for Each Decorator

```python
def test_has_permission_allowed(mock_user_permissions):
    mock_user_permissions.return_value = ["admin", "read"]
    headers = {"Authorization": "Bearer valid.token"}
    response = client.get("/only-admin", headers=headers)
    assert response.status_code == 200
    assert response.json()["user"] == "john"

def test_has_permission_denied(mock_user_permissions):
    mock_user_permissions.return_value = ["read"]
    headers = {"Authorization": "Bearer valid.token"}
    response = client.get("/only-admin", headers=headers)
    assert response.status_code == 403

def test_has_any_permission_pass(mock_user_permissions):
    mock_user_permissions.return_value = ["write"]
    headers = {"Authorization": "Bearer valid.token"}
    response = client.get("/any-read-or-write", headers=headers)
    assert response.status_code == 200

def test_has_any_permission_fail(mock_user_permissions):
    mock_user_permissions.return_value = ["admin"]
    headers = {"Authorization": "Bearer valid.token"}
    response = client.get("/any-read-or-write", headers=headers)
    assert response.status_code == 403

def test_has_all_permissions_pass(mock_user_permissions):
    mock_user_permissions.return_value = ["admin", "write"]
    headers = {"Authorization": "Bearer valid.token"}
    response = client.get("/must-have-both", headers=headers)
    assert response.status_code == 200

def test_has_all_permissions_fail(mock_user_permissions):
    mock_user_permissions.return_value = ["admin"]
    headers = {"Authorization": "Bearer valid.token"}
    response = client.get("/must-have-both", headers=headers)
    assert response.status_code == 403
```

---

## 📁 Directory Layout

```
.
├── app.py
├── myauth.py        # contains decorators, decode_token, get_user_permissions
├── test_decorators.py
└── conftest.py
```
