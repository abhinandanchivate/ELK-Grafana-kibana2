Here's a complete implementation of an **API Gateway** for your Ktor-based microservices architecture, incorporating **load balancing** and **service discovery using Consul**, all packaged with **Maven** and **Docker Compose**.

---

## **Architecture Overview**

The API Gateway will act as a single entry point for clients and will:

- Route requests to the appropriate microservices.
- Load balance across multiple instances of the services.
- Perform service discovery using **Consul**.
- Implement basic request logging.

---

## **Project Structure with API Gateway**

```
ecommerce-microservices/
│-- docker-compose.yml
│-- consul-config/
│   └── config.json
│-- api-gateway/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/main/kotlin/com/ecommerce/
│       ├── Application.kt
│       ├── GatewayRoutes.kt
│       ├── ServiceDiscovery.kt
│       └── LoadBalancer.kt
│-- product-service/
│-- order-service/
│-- user-service/
│-- payment-service/
│-- notification-service/
```

---

## **Step 1: Add API Gateway Service to `docker-compose.yml`**

```yaml
services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - consul
      - product-service
      - order-service
    networks:
      - ecommerce-network
```

---

## **Step 2: Create the API Gateway Microservice**

### **1. `api-gateway/pom.xml` (Maven Configuration)**

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.ecommerce</groupId>
    <artifactId>api-gateway</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>io.ktor</groupId>
            <artifactId>ktor-server-netty</artifactId>
            <version>2.3.0</version>
        </dependency>
        <dependency>
            <groupId>io.ktor</groupId>
            <artifactId>ktor-client-core</artifactId>
            <version>2.3.0</version>
        </dependency>
        <dependency>
            <groupId>io.ktor</groupId>
            <artifactId>ktor-client-okhttp</artifactId>
            <version>2.3.0</version>
        </dependency>
    </dependencies>
</project>
```

---

### **2. `src/main/kotlin/com/ecommerce/Application.kt` (Entry Point)**

```kotlin
import io.ktor.server.application.*
import io.ktor.server.engine.embeddedServer
import io.ktor.server.netty.Netty
import io.ktor.server.routing.*
import io.ktor.server.response.*
import com.ecommerce.routes.configureGatewayRoutes

fun Application.module() {
    routing {
        configureGatewayRoutes()
    }
}

fun main() {
    embeddedServer(Netty, port = 8080, module = Application::module).start(wait = true)
}
```

---

### **3. `src/main/kotlin/com/ecommerce/GatewayRoutes.kt` (API Gateway Routes)**

```kotlin
package com.ecommerce.routes

import io.ktor.client.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.server.application.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import com.ecommerce.servicediscovery.ServiceDiscovery
import com.ecommerce.loadbalancer.RoundRobinLoadBalancer

val httpClient = HttpClient()

fun Routing.configureGatewayRoutes() {
    val serviceDiscovery = ServiceDiscovery()
    val loadBalancer = RoundRobinLoadBalancer()

    get("/api/products") {
        val instances = serviceDiscovery.getServiceInstances("product-service")
        val targetUrl = loadBalancer.selectInstance(instances)
        val response: HttpResponse = httpClient.get("$targetUrl/products")
        call.respondText(response.bodyAsText())
    }

    get("/api/orders") {
        val instances = serviceDiscovery.getServiceInstances("order-service")
        val targetUrl = loadBalancer.selectInstance(instances)
        val response: HttpResponse = httpClient.get("$targetUrl/orders")
        call.respondText(response.bodyAsText())
    }
}
```

---

### **4. `src/main/kotlin/com/ecommerce/ServiceDiscovery.kt` (Consul Service Discovery)**

```kotlin
package com.ecommerce.servicediscovery

import okhttp3.OkHttpClient
import okhttp3.Request
import org.json.JSONObject

class ServiceDiscovery {
    private val client = OkHttpClient()
    private val consulUrl = "http://consul:8500/v1/catalog/service/"

    fun getServiceInstances(serviceName: String): List<String> {
        val request = Request.Builder().url("$consulUrl$serviceName").build()
        val response = client.newCall(request).execute()
        val jsonArray = JSONObject(response.body!!.string())

        return jsonArray.getJSONArray(serviceName).map {
            val service = it as JSONObject
            "http://${service.getString("ServiceAddress")}:${service.getInt("ServicePort")}"
        }
    }
}
```

---

### **5. `src/main/kotlin/com/ecommerce/LoadBalancer.kt` (Round-Robin Load Balancer)**

```kotlin
package com.ecommerce.loadbalancer

class RoundRobinLoadBalancer {
    private var index = 0

    fun selectInstance(instances: List<String>): String {
        if (instances.isEmpty()) throw IllegalStateException("No instances available")
        val instance = instances[index % instances.size]
        index++
        return instance
    }
}
```

---

### **6. `Dockerfile` for API Gateway**

```dockerfile
FROM openjdk:17
WORKDIR /app
COPY target/api-gateway-1.0-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## **Step 3: Build and Deploy the API Gateway**

1. **Build the API Gateway service:**

   ```sh
   mvn clean package -DskipTests
   ```

2. **Start the entire system with Docker Compose:**

   ```sh
   docker-compose up --build
   ```

3. **Verify the services:**

   - Consul UI: [http://localhost:8500](http://localhost:8500)
   - API Gateway: [http://localhost:8080/api/products](http://localhost:8080/api/products)
   - API Gateway: [http://localhost:8080/api/orders](http://localhost:8080/api/orders)

---

## **Step 4: Testing the API Gateway**

#### **1. Fetch Products via Gateway**

```sh
curl http://localhost:8080/api/products
```

**Expected Response:**
```json
[
  {
    "id": 1,
    "name": "Laptop",
    "price": 1000.0
  },
  {
    "id": 2,
    "name": "Smartphone",
    "price": 500.0
  }
]
```

---

#### **2. Fetch Orders via Gateway**

```sh
curl http://localhost:8080/api/orders
```

**Expected Response:**
```json
[
  {
    "id": 1,
    "product": "Laptop",
    "quantity": 1,
    "price": 1000.0
  },
  {
    "id": 2,
    "product": "Phone",
    "quantity": 2,
    "price": 500.0
  }
]
```

---
