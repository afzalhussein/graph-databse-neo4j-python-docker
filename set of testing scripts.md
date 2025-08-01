## A simple set of **testing scripts** using **Python `requests`** to cover:

* Register user
* Get JWT token (login)
* Access protected endpoint (`/users/me`)
* Try admin endpoint (which requires role)

---

### Save this as `test_auth.py`:

```python
import requests

BASE_URL = "http://localhost:8000"

def register(username, password):
    resp = requests.post(f"{BASE_URL}/register", json={"username": username, "password": password})
    print("Register:", resp.status_code, resp.json())

def login(username, password):
    resp = requests.post(f"{BASE_URL}/token",
                         data={"username": username, "password": password},
                         headers={"Content-Type": "application/x-www-form-urlencoded"})
    print("Login:", resp.status_code, resp.json())
    if resp.status_code == 200:
        return resp.json()["access_token"]
    return None

def get_me(token):
    headers = {"Authorization": f"Bearer {token}"}
    resp = requests.get(f"{BASE_URL}/users/me", headers=headers)
    print("Get /users/me:", resp.status_code, resp.json())

def admin_check(token):
    headers = {"Authorization": f"Bearer {token}"}
    resp = requests.get(f"{BASE_URL}/admin", headers=headers)
    print("Get /admin:", resp.status_code, resp.json())

if __name__ == "__main__":
    # Register new user (social)
    register("alice", "secret123")

    # Login user
    token = login("alice", "secret123")
    if token:
        # Access protected endpoint
        get_me(token)

        # Try admin endpoint - expected 403 for social user
        admin_check(token)

    # For admin user (already created in DB with super_admin role):
    admin_token = login("AdminUser", "test")  # Password needs to match actual hashed pwd
    if admin_token:
        admin_check(admin_token)
```

---

### Notes

* Replace `"AdminUser"` password with whatever you set or seed for admin user.
* You can run:

```bash
python test_auth.py
```

* The script prints HTTP status codes and responses so you can verify behavior.

---

**Postman collections** or **pytest test cases** next?
