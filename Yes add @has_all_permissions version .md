## The `@has_all_permissions()` decorator, which ensures that **all** specified permissions are present for the user:

---

## ✅ `@has_all_permissions` Decorator

```python
from fastapi import HTTPException, Request
from functools import wraps

def has_all_permissions(*required_permissions: str):
    def decorator(route_handler):
        @wraps(route_handler)
        async def wrapper(*args, request: Request, **kwargs):
            token = request.headers.get("Authorization")
            if not token or not token.startswith("Bearer "):
                raise HTTPException(status_code=401, detail="Unauthorized")

            token = token.removeprefix("Bearer ").strip()
            username = decode_token(token)
            user_permissions = get_user_permissions(username)

            if not all(perm in user_permissions for perm in required_permissions):
                raise HTTPException(status_code=403, detail="Forbidden: All permissions required")

            return await route_handler(*args, username=username, **kwargs)
        return wrapper
    return decorator
```

---

## 🧪 Example Usage

```python
@app.get("/critical-ops")
@has_all_permissions("admin", "system-maintainer")
async def critical_ops(username: str):
    return {"message": f"{username} has full clearance."}
```

✅ This endpoint will only allow users who have **both** `"admin"` and `"system-maintainer"` permissions.

---

## 🔁 Summary of Decorators

| Decorator                        | Requirement                                  |
| -------------------------------- | -------------------------------------------- |
| `@has_permission("p")`           | Has permission `p`                           |
| `@has_any_permission("p", "q")`  | Has at least one of `p`, `q`                 |
| `@has_all_permissions("p", "q")` | Must have **all** permissions `p`, `q`, etc. |

