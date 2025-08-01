## **Cypher schema setup** for both user types — *social* and *admin/maintenance* — designed for Neo4j.

---

## 🔧 1. **Constraints and Indexes (Optional but Recommended)**

```cypher
CREATE CONSTRAINT user_id_unique IF NOT EXISTS
ON (u:User)
ASSERT u.id IS UNIQUE;

CREATE CONSTRAINT role_name_unique IF NOT EXISTS
ON (r:Role)
ASSERT r.name IS UNIQUE;

CREATE CONSTRAINT permission_name_unique IF NOT EXISTS
ON (p:Permission)
ASSERT p.name IS UNIQUE;
```

---

## 👤 2. **Create Example Users**

```cypher
// Social Users
CREATE (alice:User {id: 'u1', name: 'Alice', type: 'social', email: 'alice@example.com'});
CREATE (bob:User {id: 'u2', name: 'Bob', type: 'social', email: 'bob@example.com'});

// Admin User
CREATE (admin:User {id: 'u3', name: 'AdminUser', type: 'admin', email: 'admin@example.com'});
```

---

## 🤝 3. **Mutual Friendships for Social Users**

```cypher
MATCH (a:User {id: 'u1'}), (b:User {id: 'u2'})
MERGE (a)-[:FRIENDS_WITH]->(b)
MERGE (b)-[:FRIENDS_WITH]->(a);
```

> In undirected graphs, Neo4j doesn’t enforce symmetry, so we create both directions explicitly for consistency.

---

## 🔐 4. **Create Roles and Permissions for Admin Users**

```cypher
CREATE (moderatorRole:Role {name: 'moderator'});
CREATE (adminRole:Role {name: 'super_admin'});

CREATE (permDelete:Permission {name: 'delete_user'});
CREATE (permBan:Permission {name: 'ban_user'});
CREATE (permEdit:Permission {name: 'edit_post'});
```

---

## 🧩 5. **Connect Roles to Permissions**

```cypher
MATCH (r:Role {name: 'moderator'}), (p:Permission {name: 'edit_post'})
MERGE (r)-[:GRANTS]->(p);

MATCH (r:Role {name: 'super_admin'}), (p1:Permission {name: 'delete_user'}), (p2:Permission {name: 'ban_user'}), (p3:Permission {name: 'edit_post'})
MERGE (r)-[:GRANTS]->(p1)
MERGE (r)-[:GRANTS]->(p2)
MERGE (r)-[:GRANTS]->(p3);
```

---

## 🧑‍💼 6. **Assign Role to Admin User**

```cypher
MATCH (u:User {id: 'u3'}), (r:Role {name: 'super_admin'})
MERGE (u)-[:HAS_ROLE]->(r);
```

---

## 📊 Full Schema Summary

```plaintext
(:User {type: "social"}) -[:FRIENDS_WITH]- (:User {type: "social"})
(:User {type: "admin"}) -[:HAS_ROLE]-> (:Role) -[:GRANTS]-> (:Permission)
```

