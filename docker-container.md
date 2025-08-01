## 📄 `Dockerfile`

```Dockerfile
# Use official lightweight Python image
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Set work directory
WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

# Copy the application code
COPY . .

# Run the FastAPI server
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 🛠️ Update `docker-compose.yml`

Now, link this Dockerfile with your FastAPI service.

```yaml
  app:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - neo4j
```

Make sure your project directory looks like:

```
social-graph-api/
├── main.py
├── Dockerfile
├── requirements.txt
└── docker-compose.yml
```

---

## 🚀 Build & Run

From the project root directory:

```bash
docker-compose up --build
```

---

## ✅ Done!

Your FastAPI app is now containerized and runs alongside Neo4j.

You can access:

* **Swagger Docs**: [http://localhost:8000/docs](http://localhost:8000/docs)
* **Neo4j Browser**: [http://localhost:7474](http://localhost:7474)
