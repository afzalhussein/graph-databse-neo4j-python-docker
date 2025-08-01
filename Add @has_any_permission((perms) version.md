## A clean and production-ready `@has_any_permission()` decorator to complement `@has_permission()`. It will allow access if **any** of the given permissions match.

---

## ✅ `@has_any_permission` Decorator

```python
from fastapi import HTTPException, Request
from functools import wraps

def has_any_permission(*required_permissions: str):
    def decorator(route_handler):
        @wraps(route_handler)
        async def wrapper(*args, request: Request, **kwargs):
            token = request.headers.get("Authorization")
            if not token or not token.startswith("Bearer "):
                raise HTTPException(status_code=401, detail="Unauthorized")

            token = token.removeprefix("Bearer ").strip()
            username = decode_token(token)
            user_permissions = get_user_permissions(username)

            if not any(perm in user_permissions for perm in required_permissions):
                raise HTTPException(status_code=403, detail="Forbidden: Insufficient permissions")

            return await route_handler(*args, username=username, **kwargs)
        return wrapper
    return decorator
```

---

## 🧪 Example Usage

```python
@app.get("/admin-tools")
@has_any_permission("admin", "superuser", "system-maintainer")
async def admin_tools(username: str):
    return {"message": f"{username} is authorized to use admin tools."}
```

This will allow any user with **at least one** of `"admin"`, `"superuser"`, or `"system-maintainer"` permissions to access the endpoint.

---

## 📎 Comparison

| Decorator                       | Access if...                      |
| ------------------------------- | --------------------------------- |
| `@has_permission("X")`          | User has permission `X`           |
| `@has_any_permission("X", "Y")` | User has at least one of `X`, `Y` |

