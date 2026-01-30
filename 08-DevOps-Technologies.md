# DevOps Technologies - Surface Level

> Docker, Kubernetes, Elasticsearch, Redis basics

---

## Docker

### What is Docker?
Docker is a platform for developing, shipping, and running applications in **containers**. Containers are lightweight, isolated environments that package an application with all its dependencies.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Image** | Read-only template with instructions for creating a container (like a class) |
| **Container** | Running instance of an image (like an object) |
| **Dockerfile** | Script that defines how to build an image |
| **Registry** | Storage for images (Docker Hub, Azure Container Registry) |
| **Volume** | Persistent storage that survives container restarts |
| **Network** | Virtual network for container communication |

### Why Use Docker?

- **Consistency**: "Works on my machine" problem solved
- **Isolation**: Dependencies don't conflict
- **Portability**: Run anywhere Docker runs
- **Scalability**: Easy to spin up multiple instances
- **DevOps**: Simplifies CI/CD pipelines

### Basic Dockerfile for .NET

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj and restore
COPY ["MyApp.Api/MyApp.Api.csproj", "MyApp.Api/"]
RUN dotnet restore "MyApp.Api/MyApp.Api.csproj"

# Copy everything and build
COPY . .
WORKDIR "/src/MyApp.Api"
RUN dotnet build -c Release -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Set environment
ENV ASPNETCORE_URLS=http://+:80
EXPOSE 80

ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### Basic Docker Commands

```bash
# Build an image
docker build -t myapp:1.0 .

# List images
docker images

# Run a container
docker run -d -p 8080:80 --name myapp myapp:1.0
# -d: detached (background)
# -p 8080:80: map host port 8080 to container port 80
# --name: container name

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# View container logs
docker logs myapp
docker logs -f myapp  # Follow logs

# Execute command in container
docker exec -it myapp bash

# Stop container
docker stop myapp

# Remove container
docker rm myapp

# Remove image
docker rmi myapp:1.0

# Docker Compose (multi-container)
docker-compose up -d
docker-compose down
docker-compose logs
```

### Docker Compose Example

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__Default=Server=db;Database=myapp;User=sa;Password=Pass@word123
    depends_on:
      - db
      - redis

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    ports:
      - "1433:1433"
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Pass@word123
    volumes:
      - sqldata:/var/opt/mssql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  sqldata:
```

### Interview Q&A

**Q: What's the difference between a Docker image and a container?**
> An **image** is a read-only template with the application and its dependencies—like a class definition. A **container** is a running instance of an image—like an object. You can create multiple containers from the same image.

**Q: What is a multi-stage build?**
> Multi-stage builds use multiple FROM statements in a Dockerfile. Each stage can use a different base image. We typically use an SDK image for building (larger, has tools) and a runtime image for the final container (smaller, just what's needed to run). This reduces final image size significantly.

---

## Kubernetes (K8s)

### What is Kubernetes?
Kubernetes is a container orchestration platform that automates deployment, scaling, and management of containerized applications.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Cluster** | Set of nodes running containerized applications |
| **Node** | Worker machine (VM or physical) that runs containers |
| **Pod** | Smallest deployable unit; one or more containers |
| **Deployment** | Declares desired state for pods (replicas, updates) |
| **Service** | Stable network endpoint to access pods |
| **ConfigMap** | Configuration data (non-sensitive) |
| **Secret** | Sensitive data (passwords, API keys) |
| **Namespace** | Virtual cluster for resource isolation |
| **Ingress** | External HTTP/HTTPS access to services |

### Why Use Kubernetes?

- **Auto-scaling**: Scale up/down based on load
- **Self-healing**: Restart failed containers automatically
- **Rolling updates**: Deploy without downtime
- **Load balancing**: Distribute traffic across pods
- **Service discovery**: Pods find each other by name
- **Secret management**: Secure handling of credentials

### Basic Kubernetes YAML

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: api
        image: myregistry/myapp:1.0
        ports:
        - containerPort: 80
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: Production
        - name: ConnectionString
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-connection
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP  # Internal only
---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

### Basic kubectl Commands

```bash
# Get resources
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get nodes

# Detailed info
kubectl describe pod myapp-api-xxx
kubectl describe deployment myapp-api

# View logs
kubectl logs myapp-api-xxx
kubectl logs -f myapp-api-xxx  # Follow

# Apply configuration
kubectl apply -f deployment.yaml

# Scale deployment
kubectl scale deployment myapp-api --replicas=5

# Rolling update
kubectl set image deployment/myapp-api api=myregistry/myapp:2.0

# Rollback
kubectl rollout undo deployment/myapp-api

# Port forward (for debugging)
kubectl port-forward pod/myapp-api-xxx 8080:80

# Execute command in pod
kubectl exec -it myapp-api-xxx -- /bin/bash

# Delete resources
kubectl delete deployment myapp-api
kubectl delete -f deployment.yaml
```

### Interview Q&A

**Q: What's the difference between a Pod and a Deployment?**
> A **Pod** is the smallest deployable unit—one or more containers running together. A **Deployment** manages Pods declaratively—it ensures the desired number of replicas are running, handles rolling updates, and recreates failed pods. You almost never create pods directly; you use Deployments.

**Q: How does Kubernetes handle application updates?**
> Kubernetes uses **rolling updates** by default. When you update a Deployment's image, it gradually replaces old pods with new ones while maintaining availability. You can configure the update strategy (e.g., max unavailable, max surge) and rollback if something goes wrong.

**Q: What are liveness and readiness probes?**
> **Liveness probes** check if a container is alive. If it fails, Kubernetes restarts the container. **Readiness probes** check if a container is ready to serve traffic. If it fails, the pod is removed from the service's endpoints until it passes again. Both are crucial for production reliability.

---

## Elasticsearch

### What is Elasticsearch?
Elasticsearch is a distributed, RESTful search and analytics engine built on Apache Lucene. It's commonly used for:
- Full-text search
- Log analysis
- Real-time analytics
- Application performance monitoring

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Index** | Collection of documents (like a database) |
| **Document** | JSON data stored in an index (like a row) |
| **Field** | Key-value pair in a document (like a column) |
| **Mapping** | Schema definition for an index |
| **Shard** | Horizontal partition of an index |
| **Replica** | Copy of a shard for redundancy |
| **Cluster** | Collection of nodes |
| **Node** | Single Elasticsearch instance |

### Common Use Cases

1. **Application Search**: Site search, product catalog search
2. **Logging & Metrics**: ELK Stack (Elasticsearch, Logstash, Kibana)
3. **Security Analytics**: SIEM solutions
4. **Business Analytics**: Real-time dashboards

### Basic Operations

```bash
# Create an index
PUT /products
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "price": { "type": "float" },
      "category": { "type": "keyword" },
      "description": { "type": "text" },
      "created_at": { "type": "date" }
    }
  }
}

# Index a document
POST /products/_doc
{
  "name": "Laptop",
  "price": 15000,
  "category": "Electronics",
  "description": "High-performance laptop for developers",
  "created_at": "2024-01-15"
}

# Search
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "laptop" } }
      ],
      "filter": [
        { "range": { "price": { "gte": 10000, "lte": 20000 } } },
        { "term": { "category": "Electronics" } }
      ]
    }
  },
  "sort": [
    { "price": "asc" }
  ]
}

# Aggregation
GET /products/_search
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": { "field": "category" }
    },
    "avg_price": {
      "avg": { "field": "price" }
    }
  }
}
```

### C# with Elasticsearch (NEST)

```csharp
// Install: NEST

var settings = new ConnectionSettings(new Uri("http://localhost:9200"))
    .DefaultIndex("products");

var client = new ElasticClient(settings);

// Index document
var product = new Product { Name = "Laptop", Price = 15000 };
var response = await client.IndexDocumentAsync(product);

// Search
var searchResponse = await client.SearchAsync<Product>(s => s
    .Query(q => q
        .Bool(b => b
            .Must(m => m.Match(ma => ma.Field(f => f.Description).Query("laptop")))
            .Filter(f => f.Range(r => r.Field(p => p.Price).GreaterThanOrEquals(10000)))
        )
    )
    .Sort(so => so.Ascending(p => p.Price))
    .Size(10)
);

var products = searchResponse.Documents;
```

### Interview Q&A

**Q: When would you use Elasticsearch over a traditional database?**
> Elasticsearch excels at **full-text search**, **log analysis**, and **real-time analytics**. I'd use it when I need fast text searching with relevance scoring, need to search across large volumes of logs, or require real-time aggregations. For transactional data with ACID requirements, I'd still use a traditional database as the primary store.

**Q: What's the ELK stack?**
> **ELK** stands for Elasticsearch, Logstash, and Kibana. Logstash collects and transforms logs from various sources. Elasticsearch stores and indexes the logs. Kibana provides visualization dashboards. It's a popular stack for centralized logging and monitoring.

---

## Redis

### What is Redis?
Redis (Remote Dictionary Server) is an in-memory data structure store used as:
- Cache
- Message broker
- Session store
- Real-time leaderboards
- Rate limiting

### Key Features

- **In-memory**: Extremely fast (sub-millisecond latency)
- **Data structures**: Strings, lists, sets, sorted sets, hashes
- **Persistence**: Optional disk persistence (RDB, AOF)
- **Pub/Sub**: Message broadcasting
- **Clustering**: Horizontal scaling
- **Lua scripting**: Atomic operations

### Use Cases

| Use Case | Data Type | Example |
|----------|-----------|---------|
| Caching | String | API response cache |
| Session storage | Hash | User session data |
| Rate limiting | String (with TTL) | API rate limits |
| Leaderboards | Sorted Set | Game scores |
| Queues | List | Background job queue |
| Real-time analytics | HyperLogLog | Unique visitor counts |
| Pub/Sub | - | Real-time notifications |

### Basic Commands

```bash
# Strings
SET user:1:name "John"
GET user:1:name
SETEX session:abc123 3600 "user_data"  # Set with expiry (1 hour)
INCR pageviews  # Atomic increment

# Hashes (like objects)
HSET user:1 name "John" email "john@example.com" age 30
HGET user:1 name
HGETALL user:1
HINCRBY user:1 age 1

# Lists (queue/stack)
LPUSH queue:jobs "job1" "job2"  # Push left
RPOP queue:jobs                  # Pop right (queue)
LRANGE queue:jobs 0 -1          # Get all

# Sets (unique values)
SADD tags:post:1 "tech" "dotnet" "csharp"
SMEMBERS tags:post:1
SISMEMBER tags:post:1 "tech"

# Sorted Sets (with scores)
ZADD leaderboard 100 "player1" 200 "player2" 150 "player3"
ZRANK leaderboard "player1"        # Rank by score
ZREVRANGE leaderboard 0 2          # Top 3 (highest first)

# Keys
KEYS user:*       # Find keys (don't use in production!)
SCAN 0 MATCH user:*  # Iterate keys (safe)
TTL key           # Time to live
EXPIRE key 3600   # Set expiry
DEL key           # Delete
```

### C# with Redis (StackExchange.Redis)

```csharp
// Install: StackExchange.Redis

// Connection
var redis = ConnectionMultiplexer.Connect("localhost:6379");
var db = redis.GetDatabase();

// String operations
await db.StringSetAsync("user:1:name", "John");
var name = await db.StringGetAsync("user:1:name");

// With expiry
await db.StringSetAsync("session:abc", "data", TimeSpan.FromHours(1));

// Increment
await db.StringIncrementAsync("pageviews");

// Hash operations
await db.HashSetAsync("user:1", new HashEntry[]
{
    new("name", "John"),
    new("email", "john@example.com")
});
var user = await db.HashGetAllAsync("user:1");

// Sorted set (leaderboard)
await db.SortedSetAddAsync("leaderboard", "player1", 100);
var topPlayers = await db.SortedSetRangeByRankAsync("leaderboard", 0, 9, Order.Descending);

// Pub/Sub
var sub = redis.GetSubscriber();
await sub.SubscribeAsync("notifications", (channel, message) =>
{
    Console.WriteLine($"Received: {message}");
});
await sub.PublishAsync("notifications", "Hello!");
```

### Caching Pattern with Redis

```csharp
public class CachedProductService
{
    private readonly IDatabase _redis;
    private readonly IProductRepository _repository;
    private readonly TimeSpan _cacheExpiry = TimeSpan.FromMinutes(10);

    public async Task<Product?> GetProductAsync(int id)
    {
        var cacheKey = $"product:{id}";

        // Try cache first
        var cached = await _redis.StringGetAsync(cacheKey);
        if (cached.HasValue)
        {
            return JsonSerializer.Deserialize<Product>(cached!);
        }

        // Cache miss - get from database
        var product = await _repository.GetByIdAsync(id);

        if (product != null)
        {
            // Store in cache
            await _redis.StringSetAsync(
                cacheKey,
                JsonSerializer.Serialize(product),
                _cacheExpiry);
        }

        return product;
    }

    public async Task InvalidateCacheAsync(int id)
    {
        await _redis.KeyDeleteAsync($"product:{id}");
    }
}
```

### Interview Q&A

**Q: When would you use Redis vs in-memory caching?**
> **In-memory** (like `IMemoryCache`) is local to one server—fast but not shared. **Redis** is a distributed cache—shared across all servers. I'd use in-memory for small, local data that doesn't need consistency across instances. I'd use Redis when I need shared cache (multiple servers), session storage, or when cache needs to survive app restarts.

**Q: What are the persistence options in Redis?**
> Redis offers two persistence options:
> 1. **RDB** (Redis Database): Point-in-time snapshots at intervals. Good for backups, but you can lose data between snapshots.
> 2. **AOF** (Append Only File): Logs every write operation. More durable but slower.
> You can use both together. For pure caching where data can be regenerated, you might not need persistence at all.

**Q: How does Redis handle memory when it's full?**
> Redis uses **eviction policies** when maxmemory is reached:
> - `noeviction`: Return errors (default)
> - `allkeys-lru`: Evict least recently used keys
> - `volatile-lru`: Evict LRU keys with expiry set
> - `allkeys-random`: Random eviction
> For caching, `allkeys-lru` is typically the best choice.

---

## Quick Reference

### Docker
- **Image**: Template for containers
- **Container**: Running instance of image
- **Dockerfile**: Build instructions
- **docker-compose**: Multi-container orchestration

### Kubernetes
- **Pod**: Smallest unit (containers)
- **Deployment**: Manages pod replicas
- **Service**: Stable endpoint for pods
- **Ingress**: External HTTP access

### Elasticsearch
- **Index**: Document collection
- **Document**: JSON data
- **Use for**: Full-text search, logs, analytics
- **ELK**: Elasticsearch + Logstash + Kibana

### Redis
- **In-memory**: Sub-millisecond latency
- **Data types**: String, Hash, List, Set, Sorted Set
- **Use for**: Caching, sessions, queues, leaderboards
- **Persistence**: RDB (snapshots), AOF (log)
