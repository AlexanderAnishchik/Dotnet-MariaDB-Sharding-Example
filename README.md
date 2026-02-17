# High-Scale .NET & MariaDB Solution

A production-ready boilerplate demonstrating horizontal database scaling with .NET 6, MariaDB sharding, and master-slave replication. Designed for high-throughput applications requiring distributed data storage and read scalability.

## 🎯 Purpose

This repository provides a complete reference implementation for building high-scale web applications with:
- **Horizontal Database Sharding**: Data is distributed across multiple database shards based on category hashing
- **Master-Slave Replication**: Each shard has a master for writes and a slave replica for reads
- **Load Balancing**: NGINX distributes traffic across multiple application instances
- **Load Testing**: JMeter configuration for performance validation
- **Containerization**: Everything runs in Docker containers for easy deployment

## 🏗️ Architecture

### System Components

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
   ┌───▼────┐
   │ NGINX  │ (Load Balancer on port 5001)
   └───┬────┘
       │
   ┌───┴────────────────┐
   │                    │
┌──▼──────┐      ┌──────▼──┐
│ Web App │      │ Web App │ (2 replicas)
│Instance1│      │Instance2│
└──┬───┬──┘      └──┬───┬──┘
   │   │            │   │
   │   └────────────┼───┘
   │                │
   ├────────────────┼──────────┐
   │                │          │
   ├────────────────┤      ├───┴────────────┤
   │  Shard 1       │      │  Shard 2       │
   │  ┌──────────┐  │      │  ┌──────────┐  │
   │  │ Master   │◄─┼─────►│  │ Master   │  │
   │  │ (Write)  │  │      │  │ (Write)  │  │
   │  └────┬─────┘  │      │  └────┬─────┘  │
   │       │        │      │       │        │
   │       ▼        │      │       ▼        │
   │  ┌──────────┐  │      │  ┌──────────┐  │
   │  │ Slave    │  │      │  │ Slave    │  │
   │  │ (Read)   │  │      │  │ (Read)   │  │
   │  └──────────┘  │      │  └──────────┘  │
   └────────────────┘      └────────────────┘
```

### Database Sharding Strategy

- **Sharding Key**: Category ID
- **Hash Function**: MD5 hash of category → modulo number of shards
- **Write Operations**: Routed to the master node of the determined shard
- **Read Operations**: Routed to the slave replica of the determined shard for load distribution

### Database Configuration

**Shard 1:**
- Master: `mariadb-shard1-master` (port 3311)
- Slave: `mariadb-shard1-slave` (port 3312)

**Shard 2:**
- Master: `mariadb-shard2-master` (port 3411)
- Slave: `mariadb-shard2-slave` (port 3412)

## 🛠️ Technology Stack

- **Application**: .NET 6.0 Web API with ASP.NET Core
- **Database**: MariaDB 10.9 (Bitnami images)
- **ORM**: Entity Framework Core 6.0 with Pomelo.EntityFrameworkCore.MySql
- **API Documentation**: Swagger/OpenAPI
- **Load Balancer**: NGINX
- **Containerization**: Docker & Docker Compose
- **Load Testing**: Apache JMeter

## 📦 Project Structure

```
High-Scale-Dotnet-MariaDB/
├── ShardingProj/               # .NET Web API Application
│   ├── Controllers/
│   │   └── PostController.cs   # REST API endpoints
│   ├── Services/
│   │   └── DataAccess.cs       # Sharding & replication logic
│   ├── Entities/
│   │   ├── Post.cs             # Post entity
│   │   ├── User.cs             # User entity
│   │   ├── Category.cs         # Category entity
│   │   └── PostServiceContext.cs # EF Core DbContext
│   ├── Dockerfile              # Application container image
│   └── ShardingProj.csproj     # .NET project file
├── DockerCompose/
│   ├── docker-compose.yml      # Multi-container orchestration
│   ├── nginx.conf              # NGINX load balancer config
│   └── HTTP Request.jmx        # JMeter load test template
└── README.md                   # This file
```

## 🚀 Getting Started

### Prerequisites

- Docker Desktop or Docker Engine with Docker Compose
- (Optional) Apache JMeter for load testing

### Running the Application

1. **Clone the repository:**
   ```bash
   git clone https://github.com/AlexanderAnishchik/High-Scale-Dotnet-MariaDB.git
   cd High-Scale-Dotnet-MariaDB
   ```

2. **Start all services:**
   ```bash
   cd DockerCompose
   docker-compose up -d
   ```

3. **Wait for services to be healthy:**
   The MariaDB containers have health checks. Wait ~30-60 seconds for all databases to initialize.

4. **Initialize the databases:**
   ```bash
   curl "http://localhost:5001/Post/InitDatabase?countUsers=100&countCategories=10"
   ```

5. **Access the application:**
   - API: http://localhost:5001
   - Swagger UI: http://localhost:5001/swagger

### Stopping the Application

```bash
cd DockerCompose
docker-compose down
```

To remove volumes (data will be lost):
```bash
docker-compose down -v
```

## 📡 API Endpoints

### Initialize Database
```http
GET /Post/InitDatabase?countUsers={count}&countCategories={count}
```
Creates users and categories in both database shards.

### Create Post
```http
POST /Post
Content-Type: application/json

{
  "title": "My Post Title",
  "content": "Post content here",
  "userId": 1,
  "categoryId": "Category1"
}
```
Creates a post. The post is stored in the shard determined by hashing the `categoryId`.

### Get Latest Posts
```http
GET /Post?category={categoryId}&count={number}
```
Retrieves the latest N posts from a specific category. Reads from the slave replica of the appropriate shard.

## 🧪 Load Testing

A JMeter test plan is included for performance testing.

### Running JMeter Tests

1. **Install Apache JMeter** (if not already installed)

2. **Open the test plan:**
   ```bash
   jmeter -t "DockerCompose/HTTP Request.jmx"
   ```

3. **Configure test parameters:**
   - Default: 300 threads (concurrent users)
   - Infinite loop for continuous load
   - Random user IDs (1-100) and category IDs (1-10)

4. **Run the test** and monitor performance metrics

The test simulates POST requests creating posts with randomized user and category data.

## 🔍 How It Works

### Data Distribution

1. When a post is created, the `categoryId` is hashed using MD5
2. The hash determines which shard stores the data (hash % number_of_shards)
3. Writes go to the master node of that shard
4. Reads come from the slave replica of that shard

### Benefits of This Architecture

- **Scalability**: Add more shards to distribute load further
- **High Availability**: Slave replicas provide read redundancy
- **Performance**: Read/write separation reduces database load
- **Load Balancing**: Multiple app instances handle more requests
- **Isolation**: Shard failures don't affect other shards

### Limitations

- Cross-shard queries are not supported
- Rebalancing shards requires manual data migration
- No automatic failover (master failure stops writes to that shard)

## 📝 Configuration

Database connection strings are configured via environment variables in `docker-compose.yml`:

- `MasterPostDbConnectionStrings__Shard1`: Write connection for shard 1
- `MasterPostDbConnectionStrings__Shard2`: Write connection for shard 2
- `ReplicaPostDbConnectionStrings__Shard1`: Read connection for shard 1
- `ReplicaPostDbConnectionStrings__Shard2`: Read connection for shard 2

## 🤝 Contributing

This is a boilerplate/reference implementation. Feel free to fork and adapt to your needs.

## 📄 License

This project is provided as-is for educational and reference purposes.
