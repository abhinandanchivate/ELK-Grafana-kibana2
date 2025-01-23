Integrating **Apache APISIX**, **Ktor-based microservices**, and **Consul** involves several steps to achieve service discovery, load balancing, routing, and resilience patterns. Below is a comprehensive guide to setting up the integration.

---

### **Step 1: Install and Configure Consul**

**1.1 Install Consul (Docker-based setup)**
```bash
docker pull hashicorp/consul
docker run -d --name=consul -p 8500:8500 consul
```
- Access the Consul UI at `http://localhost:8500`.

**1.2 Configure Consul for Service Discovery**
- Create a configuration file `consul-config.json`:
```json
{
  "service": {
    "name": "ktor-service",
    "tags": ["ktor", "backend"],
    "port": 8080,
    "check": {
      "http": "http://localhost:8080/health",
      "interval": "10s"
    }
  }
}
```
- Register the service:
```bash
consul agent -dev -config-file=consul-config.json
```

---

### **Step 2: Set Up Ktor Microservices**

**2.1 Create a Ktor Microservice Project**
Use the Ktor CLI or IntelliJ plugin to create a new project with dependencies:

```kotlin
dependencies {
    implementation("io.ktor:ktor-server-core:3.x.x")
    implementation("io.ktor:ktor-server-netty:3.x.x")
    implementation("io.ktor:ktor-server-consul:3.x.x") // For Consul integration
    implementation("org.jetbrains.exposed:exposed-core:0.42.0") // For database access
}
```

**2.2 Define Ktor Service Configuration**
- Configure `application.conf`:
```hocon
ktor {
    deployment {
        port = 8080
    }
    application {
        modules = [ com.example.ApplicationKt.module ]
    }
}
```

**2.3 Create Main Application File**
```kotlin
fun Application.module() {
    routing {
        get("/") {
            call.respondText("Ktor service is running", ContentType.Text.Plain)
        }
        get("/health") {
            call.respond(HttpStatusCode.OK, "Healthy")
        }
    }
}
```

**2.4 Register Ktor Service with Consul**
```kotlin
install(Consul) {
    agent {
        address = "http://localhost:8500"
    }
    service {
        name = "ktor-service"
        port = environment.config.property("ktor.deployment.port").getString().toInt()
        check {
            http = "http://localhost:8080/health"
            interval = "10s"
        }
    }
}
```

**2.5 Build and Run the Application**
```bash
./gradlew run
```

---

### **Step 3: Configure Apache APISIX**

**3.1 Install Apache APISIX (Docker-based setup)**
```bash
docker run -d --name=apisix \
  -p 9080:9080 -p 9180:9180 \
  apache/apisix
```

**3.2 Configure Consul Integration in APISIX**
- Edit the `config.yaml` file of APISIX to include Consul as a discovery service:
```yaml
discovery:
  consul:
    host:
      - "127.0.0.1:8500" # Consul address
    prefix: "services/"
    skip_keys: ""
    timeout: 3
```

**3.3 Register Ktor Service in APISIX**
- Use APISIX Admin API to add the upstream:
```bash
curl http://localhost:9180/apisix/admin/upstreams/1 -X PUT -d '
{
    "nodes": {
        "127.0.0.1:8080": 1
    },
    "type": "roundrobin",
    "discovery_type": "consul",
    "service_name": "ktor-service"
}'
```

**3.4 Define Routing in APISIX**
```bash
curl http://localhost:9180/apisix/admin/routes/1 -X PUT -d '
{
    "uri": "/ktor",
    "upstream_id": 1
}'
```

- Now, access the service via APISIX:
  ```
  http://localhost:9080/ktor
  ```

---

### **Step 4: Implement Resilience Patterns**

**4.1 Circuit Breaker and Retry in Ktor**
Install `ktor-features` for resilience:
```kotlin
install(HttpTimeout) {
    requestTimeoutMillis = 5000
}
install(CallLogging)
```

**4.2 Configure Circuit Breaker in APISIX**
- Add a plugin to enable the circuit breaker:
```bash
curl http://localhost:9180/apisix/admin/plugins/circuit-breaker -X PUT -d '
{
    "breakers": [
        {
            "name": "breaker1",
            "max_fails": 3,
            "fail_timeout": 30,
            "unhealthy": {
                "http_failures": 5,
                "timeouts": 3,
                "http_statuses": [500, 502]
            }
        }
    ]
}'
```

---

### **Step 5: Testing the Integration**

1. **Check Consul Registration**
   - Go to `http://localhost:8500` and verify that the `ktor-service` is registered.

2. **Check APISIX Routing**
   - Access your Ktor service via APISIX using `http://localhost:9080/ktor`.

3. **Failover Testing**
   - Stop the Ktor service and observe APISIX response handling (e.g., 502 Bad Gateway).

---

### **Step 6: Deploying the Solution Using Docker Compose**

Create a `docker-compose.yml` file for a production-ready setup:

```yaml
version: '3.7'
services:
  consul:
    image: hashicorp/consul
    ports:
      - "8500:8500"
  
  ktor-service:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - consul

  apisix:
    image: apache/apisix
    ports:
      - "9080:9080"
      - "9180:9180"
    depends_on:
      - consul
```

Run the stack:
```bash
docker-compose up -d
```

---

### **Step 7: Monitoring and Logging**

- **Enable APISIX Observability**
  - Enable metrics and logs in `config.yaml` of APISIX for Prometheus and Elasticsearch.

- **Monitor with Consul**
  - Use the Consul UI or CLI to check the health and availability of services.

---

### **Conclusion**

You have successfully integrated:

- **Ktor microservices** registered with **Consul** for service discovery.
- **Apache APISIX** for routing, load balancing, and resilience.
- **Docker Compose** for deployment.

By following these steps, you can build a scalable and resilient eCommerce microservices architecture.
