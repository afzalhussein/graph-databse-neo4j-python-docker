## **pytest test cases** to verify:

* Adding friends
* Listing friends
* Preventing duplicate friendships

These assume you're using the FastAPI routes with Neo4j where `/add_friend/{friend_username}` adds a friendship and `/friends/{username}` fetches them.

---

### 🔧 Assumptions

* You already have the `/add_friend/{friend_username}` and `/friends/{username}` endpoints implemented.
* JWT authentication is required on these endpoints.
* The user must exist before being added as a friend.

---

### ✅ Add to your `test_main.py`

```python
def test_create_second_user():
    # Create a second user to be added as a friend
    response = client.post("/register", json={"username": "frienduser", "password": "friendpass"})
    if response.status_code != 200:
        assert response.status_code == 400  # Already exists

def test_friend_login():
    # Login friend user and return token
    response = client.post(
        "/token",
        data={"username": "frienduser", "password": "friendpass"},
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    assert response.status_code == 200
    token = response.json()["access_token"]
    assert token
    global friend_token
    friend_token = token

def test_add_friend(auth_token):
    # Add frienduser as a friend to testuser
    response = client.post("/add_friend/frienduser", headers={"Authorization": f"Bearer {auth_token}"})
    assert response.status_code == 200
    assert "added as a friend" in response.json()["msg"]

def test_add_same_friend_again(auth_token):
    # Adding the same friend again should be handled gracefully
    response = client.post("/add_friend/frienduser", headers={"Authorization": f"Bearer {auth_token}"})
    assert response.status_code == 400 or response.status_code == 409
    # Optional: assert specific error message
    assert "already friends" in response.json()["detail"].lower()

def test_list_friends(auth_token):
    # Get friends of testuser
    response = client.get("/friends/testuser", headers={"Authorization": f"Bearer {auth_token}"})
    assert response.status_code == 200
    friends = response.json()["friends"]
    assert isinstance(friends, list)
    assert "frienduser" in friends
```

---

### 🧪 Run tests

```bash
pytest test_main.py -v
```

---

### 🚀 Bonus Suggestion

You can enhance tests further by adding:

* **Bi-directional check**: If `A` adds `B`, does `B` see `A` as a friend?
* **Graph integrity check**: Query Neo4j directly in a test to validate data.

