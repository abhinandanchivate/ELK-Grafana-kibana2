### Steps to Set Up Apache APISIX with Ktor Microservices

Follow these steps to deploy a dynamic microservices architecture with Apache APISIX, Ktor-based services, and service discovery using Consul.

---

### **1. Project Structure**

```
project-root/
│-- config/
│   └-- apisix.yaml
│-- ktor-service-1/
│   ├-- Dockerfile
│   └-- src/main/kotlin/Main.kt
│-- ktor-service-2/
│   ├-- Dockerfile
│   └-- src/main/kotlin/Main.kt
│-- ktor-service-3/
│   ├-- Dockerfile
│   └-- src/main/kotlin/Main.kt
│-- docker-compose.yml
```

---

### **2. Docker Compose Configuration**

Your `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  apisix:
    image: apache/apisix:latest
    ports:
      - "9080:9080"  # HTTP port
      - "9443:9443"  # HTTPS port
    volumes:
      - ./config/apisix.yaml:/usr/local/apisix/conf/config.yaml
    depends_on:
      - etcd
    networks:
      - apisix-net

  etcd:
    image: bitnami/etcd:latest
    environment:
      - ALLOW_NONE_AUTHENTICATION=yes
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd:2379
    networks:
      - apisix-net

  ktor-service-1:
    build:
      context: ./ktor-service-1
      dockerfile: Dockerfile
    deploy:
      replicas: 3
    ports:
      - "8081:8080"
    networks:
      - apisix-net

  ktor-service-2:
    build:
      context: ./ktor-service-2
      dockerfile: Dockerfile
    deploy:
      replicas: 3
    ports:
      - "8082:8080"
    networks:
      - apisix-net

  ktor-service-3:
    build:
      context: ./ktor-service-3
      dockerfile: Dockerfile
    deploy:
      replicas: 3
    ports:
      - "8083:8080"
    networks:
      - apisix-net

networks:
  apisix-net:
    driver: bridge
```

---

### **3. APISIX Configuration**

Create a `config/apisix.yaml` file:

```yaml
apisix:
  discovery:
    consul:
      servers:
        - "http://etcd:2379"
  routes:
    - uri: /service1/*
      upstream:
        service_name: ktor-service-1
        discovery_type: consul
    - uri: /service2/*
      upstream:
        service_name: ktor-service-2
        discovery_type: consul
    - uri: /service3/*
      upstream:
        service_name: ktor-service-3
        discovery_type: consul
```

---

### **4. Ktor Microservice Setup**

#### **Dockerfile for Each Ktor Service**

Create `ktor-service-1/Dockerfile` (similar for `ktor-service-2` and `ktor-service-3`):

```dockerfile
FROM openjdk:17
WORKDIR /app
COPY build/libs/ktor-service-1-all.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

#### **Ktor Microservice Code**

Example Ktor service at `ktor-service-1/src/main/kotlin/Main.kt`:

```kotlin
package ktor.service

import io.ktor.application.*
import io.ktor.features.*
import io.ktor.response.*
import io.ktor.routing.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*

fun main() {
    embeddedServer(Netty, port = System.getenv("PORT")?.toInt() ?: 8080) {
        install(CallLogging)
        routing {
            get("/hello") {
                call.respondText("Hello from Ktor Service 1")
            }
        }
    }.start(wait = true)
}
```

Repeat similar code for `ktor-service-2` and `ktor-service-3`.

---

### **5. Build and Run the Services**

Run the following commands to build and deploy the services:

```bash
docker-compose up --build -d
```

---

### **6. Testing the Services**

You can now test your services using curl:

```bash
curl http://localhost:9080/service1/hello
curl http://localhost:9080/service2/hello
curl http://localhost:9080/service3/hello
```

If everything is set up correctly, you should receive:

```
Hello from Ktor Service 1
Hello from Ktor Service 2
Hello from Ktor Service 3
```

---

### **7. Scaling Up Services Dynamically**

If you want to scale up the number of instances dynamically, you can run:

```bash
docker-compose up --scale ktor-service-1=5
```

APISIX will automatically route traffic across the available instances.

---

### **8. Observability and Logs**

For monitoring, you can enable logging in APISIX by checking container logs:

```bash
docker logs <container_id>
```

---

This setup provides a fully functional microservices architecture with dynamic service discovery, load balancing, and routing using Apache APISIX and Ktor.

Let me know if you need further guidance!
