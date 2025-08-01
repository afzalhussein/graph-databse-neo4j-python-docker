## Let’s **extend the graph schema** with **RBAC (Role-Based Access Control)** nodes, then write **Pytest test cases** to validate it.

---

## 🧠 Graph Schema: Roles & Permissions

We'll model:

* `(:User)`
* `(:Role)`
* `(:Permission)`
* `(:User)-[:HAS_ROLE]->(:Role)`
* `(:Role)-[:HAS_PERMISSION]->(:Permission)`

This allows reusable roles (e.g. "admin", "editor") and fine-grained permissions.

---

## ⚙️ Cypher: Schema Setup

You can run this in your setup or directly via a test:

```cypher
CREATE (admin:Role {name: "admin"})
CREATE (editor:Role {name: "editor"})

CREATE (read:Permission {name: "read"})
CREATE (write:Permission {name: "write"})
CREATE (delete:Permission {name: "delete"})

MERGE (admin)-[:HAS_PERMISSION]->(read)
MERGE (admin)-[:HAS_PERMISSION]->(write)
MERGE (admin)-[:HAS_PERMISSION]->(delete)

MERGE (editor)-[:HAS_PERMISSION]->(read)
MERGE (editor)-[:HAS_PERMISSION]->(write)

MERGE (u:User {username: "rbacuser"})
MERGE (u)-[:HAS_ROLE]->(admin)
```

---

## 🧪 Pytest: RBAC Test Cases

### 1. ✅ Validate user has correct role and permissions

```python
def test_user_has_admin_role():
    with driver.session() as session:
        result = session.run("""
            MATCH (u:User {username: $username})-[:HAS_ROLE]->(r:Role)
            RETURN r.name AS role_name
        """, username="rbacuser")
        
        role = result.single()["role_name"]
        assert role == "admin", f"Expected role 'admin', got '{role}'"

def test_admin_has_permissions():
    with driver.session() as session:
        result = session.run("""
            MATCH (u:User {username: $username})-[:HAS_ROLE]->(:Role)-[:HAS_PERMISSION]->(p:Permission)
            RETURN COLLECT(p.name) AS permissions
        """, username="rbacuser")
        
        permissions = result.single()["permissions"]
        assert set(permissions) == {"read", "write", "delete"}, f"Unexpected permissions: {permissions}"
```

---

## 🧹 Optional Teardown (Clean Up Test Data)

```python
def teardown_module(module):
    with driver.session() as session:
        session.run("""
            MATCH (u:User {username: 'rbacuser'}) DETACH DELETE u
        """)
        session.run("""
            MATCH (r:Role) WHERE r.name IN ['admin', 'editor'] DETACH DELETE r
        """)
        session.run("""
            MATCH (p:Permission) WHERE p.name IN ['read', 'write', 'delete'] DETACH DELETE p
        """)
```

---

## ✅ Result

You now have:

* **Role/Permission graph modeling**
* **Neo4j Cypher setup**
* **Tested with `pytest` and validated via Cypher**

