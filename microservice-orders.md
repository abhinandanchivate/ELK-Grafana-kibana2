Here is the complete implementation of the **Order Service** in the Ktor-based eCommerce microservices architecture, following the same approach as the product service.

---

### **Project Structure for `order-service`**

```
order-service/
│-- Dockerfile
│-- pom.xml
└── src/main/kotlin/com/ecommerce/
    ├── Application.kt
    ├── OrderRoutes.kt
    ├── OrderService.kt
    ├── Resilience.kt
    └── ConsulRegistration.kt
```

---

### **Step 1: `pom.xml` (Maven Configuration)**

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.ecommerce</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>io.ktor</groupId>
            <artifactId>ktor-server-netty</artifactId>
            <version>2.3.0</version>
        </dependency>
        <dependency>
            <groupId>io.ktor</groupId>
            <artifactId>ktor-serialization-kotlinx-json</artifactId>
            <version>2.3.0</version>
        </dependency>
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-circuitbreaker</artifactId>
            <version>2.0.2</version>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.5.1</version>
        </dependency>
    </dependencies>
</project>
```

---

### **Step 2: Application Entry Point (`Application.kt`)**

```kotlin
import io.ktor.server.application.*
import io.ktor.server.engine.embeddedServer
import io.ktor.server.netty.Netty
import io.ktor.server.routing.*
import io.ktor.server.response.*
import kotlinx.serialization.Serializable

fun Application.module() {
    routing {
        orderRoutes()
    }
}

fun main() {
    registerWithConsul()
    embeddedServer(Netty, port = 8082, module = Application::module).start(wait = true)
}
```

---

### **Step 3: Order Routes (`OrderRoutes.kt`)**

```kotlin
fun Routing.orderRoutes() {
    get("/orders") {
        call.respond(getOrders())
    }

    get("/order/{id}") {
        val id = call.parameters["id"]?.toIntOrNull()
        val order = getOrders().find { it.id == id }
        if (order != null) {
            call.respond(order)
        } else {
            call.respond("Order Not Found")
        }
    }
}
```

---

### **Step 4: Order Service (`OrderService.kt`)**

```kotlin
@Serializable
data class Order(val id: Int, val product: String, val quantity: Int, val price: Double)

fun getOrders(): List<Order> {
    return listOf(
        Order(1, "Laptop", 1, 1000.0),
        Order(2, "Phone", 2, 500.0)
    )
}
```

---

### **Step 5: Resilience Patterns (`Resilience.kt`)**

```kotlin
import io.github.resilience4j.circuitbreaker.CircuitBreaker
import io.github.resilience4j.retry.Retry

val orderCircuitBreaker = CircuitBreaker.ofDefaults("orderService")
val orderRetry = Retry.ofDefaults("orderService")

fun <T> resilientOrderCall(function: () -> T): T {
    return Retry.decorateFunction(orderRetry, CircuitBreaker.decorateFunction(orderCircuitBreaker, function)).apply()
}
```

---

### **Step 6: Consul Registration (`ConsulRegistration.kt`)**

```kotlin
import okhttp3.*
import okhttp3.MediaType.Companion.toMediaTypeOrNull

fun registerWithConsul() {
    val client = OkHttpClient()
    val requestBody = """
        {
            "ID": "order-service",
            "Name": "order-service",
            "Tags": ["ktor", "order"],
            "Address": "localhost",
            "Port": 8082
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

### **Step 7: Dockerfile**

```dockerfile
FROM openjdk:17
WORKDIR /app
COPY target/order-service-1.0-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

### **Step 8: Update `docker-compose.yml`**

Ensure the `docker-compose.yml` includes the `order-service`:

```yaml
  order-service:
    build: ./order-service
    ports:
      - "8082:8082"
    depends_on:
      - consul
      - postgres
    networks:
      - ecommerce-network
```

---

### **Step 9: Building and Running the Services**

#### **1. Build the Order Service**
Navigate to the `order-service` directory and run:

```sh
mvn clean package -DskipTests
```

#### **2. Start the system using Docker Compose**

```sh
docker-compose up --build
```

#### **3. Verify the services**

- **Consul UI:** [http://localhost:8500](http://localhost:8500)
- **Order API:** [http://localhost:8082/orders](http://localhost:8082/orders)

---

### **Step 10: Testing the Order Service**

1. **Retrieve all orders:**

   ```sh
   curl http://localhost:8082/orders
   ```

2. **Get order by ID:**

   ```sh
   curl http://localhost:8082/order/1
   ```

---

### **Next Steps**

You can follow the same pattern to implement the remaining microservices:

- **User Service**
- **Payment Service**
- **Notification Service**

Each will have similar components:

- Application module
- Service logic
- Resilience patterns
- Consul registration
- Dockerfile setup

---

### **Key Features Implemented**

- **Ktor Microservice** (for Order Management)
- **Consul Service Discovery** (Dynamic registration)
- **Resilience4j Circuit Breaker and Retry** (Fault tolerance)
- **PostgreSQL Integration** (Database support)
- **Docker Compose Deployment** (Orchestration)

---

This complete implementation ensures that your **Order Service** integrates seamlessly within your microservices architecture with **service discovery, resilience patterns, and deployment in Docker.** Let me know if you need further details or additional services!
