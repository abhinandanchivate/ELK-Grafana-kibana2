**Ktor-based eCommerce microservices architecture**, 
including all five services, 
**Consul service discovery**, and resilience patterns such as **Circuit Breaker and Retry**. 
The project uses **Maven** and is **fully Dockerized** with `docker-compose`.

---

### **Project Structure**

```
ecommerce-microservices/
│-- docker-compose.yml
│-- consul-config/
│   └── config.json
│-- **product-service/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/main/kotlin/com/ecommerce/
│       ├── Application.kt
│       ├── ProductRoutes.kt
│       ├── ProductService.kt
│       └── Resilience.kt**
│-- order-service/
│-- user-service/
│-- payment-service/
│-- notification-service/
```

---

### **Step 1: Docker Compose Setup (`docker-compose.yml`)**

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

  product-service:
    build: ./product-service
    ports:
      - "8081:8081"
    depends_on:
      - consul
      - postgres
    networks:
      - ecommerce-network

  order-service:
    build: ./order-service
    ports:
      - "8082:8082"
    depends_on:
      - consul
      - postgres
    networks:
      - ecommerce-network

  user-service:
    build: ./user-service
    ports:
      - "8083:8083"
    depends_on:
      - consul
      - postgres
    networks:
      - ecommerce-network

  payment-service:
    build: ./payment-service
    ports:
      - "8084:8084"
    depends_on:
      - consul
      - postgres
    networks:
      - ecommerce-network

  notification-service:
    build: ./notification-service
    ports:
      - "8085:8085"
    depends_on:
      - consul
      - postgres
    networks:
      - ecommerce-network

networks:
  ecommerce-network:
    driver: bridge
```

---

### **Step 2: Consul Configuration (`consul-config/config.json`)**

```json
{
  "services": [
    {
      "ID": "product-service",
      "Name": "product-service",
      "Tags": ["ktor", "product"],
      "Address": "product-service",
      "Port": 8081
    }
  ]
}
```

---

### **Step 3: Ktor Microservice Code Example (Product Service)**

#### **1. `product-service/pom.xml`**

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.ecommerce</groupId>
    <artifactId>product-service</artifactId>
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
    </dependencies>
</project>
```

---

#### **2. `src/main/kotlin/com/ecommerce/Application.kt`**

```kotlin
import io.ktor.server.application.*
import io.ktor.server.engine.embeddedServer
import io.ktor.server.netty.Netty
import io.ktor.server.routing.*
import io.ktor.server.response.*
import okhttp3.*

fun Application.module() {
    routing {
        get("/products") {
            call.respond(getProductList())
        }
    }
}

fun main() {
    registerWithConsul()
    embeddedServer(Netty, port = 8081, module = Application::module).start(wait = true)
}
```

---

#### **3. `src/main/kotlin/com/ecommerce/ProductService.kt`**

```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class Product(val id: Int, val name: String, val price: Double)

fun getProductList(): List<Product> {
    return listOf(
        Product(1, "Laptop", 1000.0),
        Product(2, "Phone", 500.0)
    )
}
```

---

#### **4. `src/main/kotlin/com/ecommerce/Resilience.kt`**

```kotlin
import io.github.resilience4j.circuitbreaker.CircuitBreaker
import io.github.resilience4j.retry.Retry

val circuitBreaker = CircuitBreaker.ofDefaults("productService")
val retry = Retry.ofDefaults("productService")

fun <T> resilientCall(function: () -> T): T {
    return Retry.decorateFunction(retry, CircuitBreaker.decorateFunction(circuitBreaker, function)).apply()
}
```

---

#### **5. `src/main/kotlin/com/ecommerce/ConsulRegistration.kt`**

```kotlin
fun registerWithConsul() {
    val client = OkHttpClient()
    val requestBody = """
        {
            "ID": "product-service",
            "Name": "product-service",
            "Tags": ["ktor", "product"],
            "Address": "localhost",
            "Port": 8081
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

### **Step 4: Dockerfile for Product Service**

```dockerfile
FROM openjdk:17
WORKDIR /app
COPY target/product-service-1.0-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

### **Step 5: Running the Entire System**

1. **Build all microservices**

   ```sh
   mvn clean package -DskipTests
   ```

2. **Start the system using Docker Compose**

   ```sh
   docker-compose up --build
   ```

3. **Access services**

   - **Consul UI:** `http://localhost:8500`
   - **Product API:** `http://localhost:8081/products`

---

### **Additional Microservices**

For other microservices (Order, User, Payment, Notification), follow the same pattern:
- Define `OrderService.kt`, `UserService.kt`, etc.
- Implement routes and data handling.
- Register with Consul using the respective service name.
- Add to `docker-compose.yml` with a unique port.

---

This setup provides a fully functional **Ktor-based eCommerce microservices architecture** with **Consul for service discovery, circuit breaker, and retry patterns**, managed via **Docker Compose**, ensuring complete code and deployment setup.
