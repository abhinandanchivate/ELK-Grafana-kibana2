To achieve the goal of registering **three instances** of both `product-service` and `order-service` with **Consul**, and accessing them via the API Gateway with load balancing, follow these steps:

---

## **Solution Overview**

1. **Modify the services to support multiple instances.**
2. **Dynamically generate `SERVICE_ID` using environment variables or UUIDs.**
3. **Update `docker-compose.yml` to spin up multiple instances.**
4. **Configure API Gateway to load balance and identify service instances.**
5. **Verify in Consul UI and logs.**

---

## **Step 1: Modify Services to Support Multiple Instances**

### **1.1 Update `Application.kt` to Auto-generate Service ID**

Modify `product-service/src/main/kotlin/com/ecommerce/Application.kt` (Same applies for `order-service`).

```kotlin
import io.ktor.server.application.*
import io.ktor.server.engine.embeddedServer
import io.ktor.server.netty.Netty
import io.ktor.server.routing.*
import io.ktor.server.response.*
import java.util.*

fun Application.module() {
    routing {
        get("/products") {
            val serviceId = System.getenv("SERVICE_ID") ?: "product-service-${UUID.randomUUID()}"
            call.respond(mapOf("serviceId" to serviceId, "products" to getProductList()))
        }
    }
}

fun getProductList() = listOf(
    mapOf("id" to 1, "name" to "Laptop", "price" to 1000.0),
    mapOf("id" to 2, "name" to "Smartphone", "price" to 500.0)
)

fun main() {
    registerWithConsul()
    embeddedServer(Netty, port = System.getenv("PORT")?.toInt() ?: 8081, module = Application::module).start(wait = true)
}
```

---

### **1.2 Update `ConsulRegistration.kt` to Use Dynamic IDs and Ports**

```kotlin
import okhttp3.*
import java.util.*

fun registerWithConsul() {
    val client = OkHttpClient()
    val serviceId = System.getenv("SERVICE_ID") ?: "product-service-${UUID.randomUUID()}"
    val port = System.getenv("PORT") ?: "8081"
    val requestBody = """
        {
            "ID": "$serviceId",
            "Name": "product-service",
            "Tags": ["ktor", "product"],
            "Address": "localhost",
            "Port": $port
        }
    """.trimIndent()

    val request = Request.Builder()
        .url("http://consul:8500/v1/agent/service/register")
        .put(RequestBody.create("application/json".toMediaTypeOrNull(), requestBody))
        .build()

    client.newCall(request).execute()
}
```

---

## **Step 2: Update `docker-compose.yml` to Run Multiple Instances**

Update the `docker-compose.yml` to start **three instances** of `product-service` and `order-service`, each on different ports.

```yaml
version: '3.7'

services:
  consul:
    image: consul:latest
    ports:
      - "8500:8500"
      - "8600:8600/udp"
    command: agent -dev -client=0.0.0.0
    networks:
      - ecommerce-network

  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      POSTGRES_DB: ecommerce_db
    ports:
      - "5432:5432"
    networks:
      - ecommerce-network

  product-service-1:
    build: ./product-service
    environment:
      - PORT=8081
      - SERVICE_ID=product-service-1
    ports:
      - "8081:8081"
    networks:
      - ecommerce-network

  product-service-2:
    build: ./product-service
    environment:
      - PORT=8082
      - SERVICE_ID=product-service-2
    ports:
      - "8082:8082"
    networks:
      - ecommerce-network

  product-service-3:
    build: ./product-service
    environment:
      - PORT=8083
      - SERVICE_ID=product-service-3
    ports:
      - "8083:8083"
    networks:
      - ecommerce-network

  order-service-1:
    build: ./order-service
    environment:
      - PORT=8084
      - SERVICE_ID=order-service-1
    ports:
      - "8084:8084"
    networks:
      - ecommerce-network

  order-service-2:
    build: ./order-service
    environment:
      - PORT=8085
      - SERVICE_ID=order-service-2
    ports:
      - "8085:8085"
    networks:
      - ecommerce-network

  order-service-3:
    build: ./order-service
    environment:
      - PORT=8086
      - SERVICE_ID=order-service-3
    ports:
      - "8086:8086"
    networks:
      - ecommerce-network

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - consul
    networks:
      - ecommerce-network

networks:
  ecommerce-network:
    driver: bridge
```

---

## **Step 3: Modify API Gateway for Load Balancing and Logging**

Update `GatewayRoutes.kt` in API Gateway to include logging of the instance.

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
val serviceDiscovery = ServiceDiscovery()
val loadBalancer = RoundRobinLoadBalancer()

fun Routing.configureGatewayRoutes() {
    get("/api/products") {
        val instances = serviceDiscovery.getServiceInstances("product-service")
        val targetUrl = loadBalancer.selectInstance(instances)
        val response: HttpResponse = httpClient.get("$targetUrl/products")
        call.respondText("Response from $targetUrl: ${response.bodyAsText()}")
    }
}
```

---

## **Step 4: Build and Deploy the Solution**

1. **Build all services:**
   ```sh
   mvn clean package -DskipTests
   ```

2. **Start the system using Docker Compose:**
   ```sh
   docker-compose up --build
   ```

---

## **Step 5: Verify the Solution**

### **1. Check registered services in Consul UI**

Open **[http://localhost:8500](http://localhost:8500)** in your browser, and you should see:

```
Services:
- product-service (with 3 instances)
- order-service (with 3 instances)
- api-gateway
```

### **2. Fetch Products via API Gateway**

Run the following command to hit the API Gateway:

```sh
curl http://localhost:8080/api/products
```

**Expected Response:**
```json
Response from http://localhost:8081: {
  "serviceId": "product-service-1",
  "products": [
    { "id": 1, "name": "Laptop", "price": 1000.0 },
    { "id": 2, "name": "Smartphone", "price": 500.0 }
  ]
}
```

Repeat the command multiple times to verify **load balancing**, and you should see different instance responses.

---

## **Step 6: Confirm Load Balancing**

To confirm load balancing:

1. Run the API call multiple times:
   ```sh
   curl http://localhost:8080/api/products
   ```
2. Observe different `serviceId` values, confirming load balancing across instances.

---

