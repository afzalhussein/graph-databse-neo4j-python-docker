## Adding a **custom decorator** makes applying RBAC checks more elegant and reusable in your FastAPI app.

---

## ✅ Goal

Instead of writing:

```python
@app.get("/dashboard")
def dashboard(username: str = Depends(require_permission("read"))):
```

You'll write:

```python
@app.get("/dashboard")
@has_permission("read")
def dashboard(username: str):
```

---

## 🧩 1. Decorator Implementation

```python
from functools import wraps
from fastapi import Request

def has_permission(permission: str):
    def decorator(route_handler):
        @wraps(route_handler)
        async def wrapper(*args, request: Request, **kwargs):
            token = request.headers.get("Authorization")
            if not token or not token.startswith("Bearer "):
                raise HTTPException(status_code=401, detail="Unauthorized")

            token = token.removeprefix("Bearer ").strip()
            username = decode_token(token)
            permissions = get_user_permissions(username)

            if permission not in permissions:
                raise HTTPException(status_code=403, detail="Forbidden")

            return await route_handler(*args, username=username, **kwargs)
        return wrapper
    return decorator
```

> ⚠️ Important:
>
> * This version assumes the route accepts `username` as a keyword argument.
> * Works with async endpoints only. For sync functions, use `def` instead of `async def wrapper(...)` and call `route_handler(...)` directly.

---

## 🧪 2. Decorated API Endpoints

```python
@app.get("/dashboard")
@has_permission("read")
async def dashboard(username: str):
    return {"message": f"Welcome {username}, you have read access."}

@app.post("/content")
@has_permission("write")
async def create_content(username: str):
    return {"message": f"{username} is allowed to write content."}

@app.delete("/content")
@has_permission("delete")
async def delete_content(username: str):
    return {"message": f"{username} is allowed to delete content."}
```

---

## 🧼 Optional Enhancements

* ✅ Add a `@has_any_permission(*perms)` version
* ✅ Cache permissions in Redis for performance
* ✅ Combine with `@router` if modularized


