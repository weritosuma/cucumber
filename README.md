```
https://cucumber.netlify.app/docs/guides/overview/

BDD - ответвление от TDD, тесты пишутся на естеснвтенно языке

BDD состоит из:
1 Title - описание бизнес цели
2 Narrative - описание кто заинтересован в истории, ее состав и ценность для бизнеса
3 Scenarios - сам тест? сохранение данных, запрос, сверка результата

Cucumber - использует естественный язык Gherkin для BDD, его структура создается с помощью отступов, каждая строка начинается с ключевого слова

Cucumber решает проблемы: 1) executable specification - шаги выполнения, 2) Automated testing, 3) Documentation в виде тестов 

тесты проверяют - ответы в json и данные в бд

как писать тест
1 в файле feature
2 данные такие как min/max для шаблона теста берутся из pplication.properties
3 в каталоге resources - файлы json для использования в тестах
	- все файлы указанные в тестах должны быть созданы в resources
	- sql для получения результатов из бд тоже тут в /queries/...
4 в kibana можно посмотреть примеры запроса и ответа, напр токен или шифрованные данные
5 код привязанный к тесту который заполняет базу и проверяет результат в классах ...Steps
	- создается ожидаемый результат и фактический
6 при сравнении результатов в тесты можно задать список ignore полей т.к. не все поля мы можем создать
7 в тестах могут быть уникальные для конкретного теста аннотации
8 классе (напр Hooks) лежат методы хуков с аннотациями @Before/@After для авто тестов, напр заполнение базы
9 внутри теста
	1 @Given - заполняем базу
		DataTable - передается в тест массив данных
	2 @When - выполняем вызов метода сервиса (e.g. запрос rest)
	3 @Then - сравниваем результат (данные в базе)
		expectedTotal или DataTable - передаем для сравнения
```

### Полный гайд по Cucumber, Gherkin, Spring Boot 3, Testcontainers и WireMock (Gradle + Java 17)

---

#### **1. Настройка проекта (Gradle)**
```kotlin
// build.gradle.kts
plugins {
    java
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
}

dependencies {
    // Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")

    // Test
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.cucumber:cucumber-java:7.14.0")
    testImplementation("io.cucumber:cucumber-spring:7.14.0")
    testImplementation("io.cucumber:cucumber-junit-platform-engine:7.14.0")
    
    // Testcontainers
    testImplementation("org.testcontainers:testcontainers:1.19.3")
    testImplementation("org.testcontainers:postgresql:1.19.3")
    testImplementation("org.testcontainers:junit-jupiter:1.19.3")
    
    // WireMock
    testImplementation("com.github.tomakehurst:wiremock-jre8:2.35.0")
    
    // Postgres
    runtimeOnly("org.postgresql:postgresql")
}

tasks.test {
    useJUnitPlatform()
    systemProperty("cucumber.junit-platform.naming-strategy", "long")
    systemProperty("cucumber.plugin", "html:build/reports/cucumber.html")
}
```

---

#### **2. Профиль `itest` и Testcontainers**
**`src/test/resources/application-itest.properties`**:
```properties
spring.datasource.url=jdbc:tc:postgresql:15-alpine:///testdb
spring.datasource.driver-class-name=org.testcontainers.jdbc.ContainerDatabaseDriver
spring.jpa.hibernate.ddl-auto=create-drop
```

**Базовый класс для тестов**:
```java
// BaseIntegrationTest.java
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
@ActiveProfiles("itest")
public abstract class BaseIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

---

#### **3. Конфигурация WireMock**
```java
// WireMockConfig.java
import com.github.tomakehurst.wiremock.WireMockServer;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;

@TestConfiguration
public class WireMockConfig {

    @Bean(initMethod = "start", destroyMethod = "stop")
    public WireMockServer wireMockServer() {
        return new WireMockServer(8089); // Порт для моков
    }
}
```

---

#### **4. Интеграция Cucumber с Spring Boot**
```java
// CucumberSpringConfig.java
import io.cucumber.spring.CucumberContextConfiguration;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ContextConfiguration;

@CucumberContextConfiguration
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ContextConfiguration(classes = { WireMockConfig.class })
@ActiveProfiles("itest")
public class CucumberSpringConfig {
}
```

---

#### **5. Примеры feature-файлов**
**`user_registration.feature`**:
```gherkin
Feature: User Registration API
  Scenario: Create user with valid data (DataTable example)
    Given the user registration endpoint is available
    And external service is available
    When a POST request is sent with:
      | username | email          |
      | testuser | test@email.com |
    Then the response status is 201
    And the response contains:
      | username | email          |
      | testuser | test@email.com |

  Scenario Outline: Registration with different data
    Given the user registration endpoint is available
    When a POST request is sent with <username> and <email>
    Then the response status is <status>

    Examples:
      | username | email          | status |
      | user1    | user1@test.com | 201    |
      | invalid  | bad-email      | 400    |
```

---

#### **6. Step Definitions (с обработкой DataTable)**
```java
// UserRegistrationSteps.java
import io.cucumber.java.en.*;
import io.cucumber.datatable.DataTable;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.ResponseEntity;
import com.github.tomakehurst.wiremock.WireMockServer;
import static org.junit.jupiter.api.Assertions.*;

public class UserRegistrationSteps extends BaseIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private WireMockServer wireMockServer;

    @Autowired
    private UserRepository userRepository;

    private ResponseEntity<User> response;

    // Пример с DataTable
    @When("a POST request is sent with:")
    public void sendPostRequestWithDataTable(DataTable dataTable) {
        Map<String, String> data = dataTable.asMap();
        User user = new User(data.get("username"), data.get("email"));
        response = restTemplate.postForEntity("/api/users", user, User.class);
    }

    @Then("the response contains:")
    public void verifyResponseBody(DataTable dataTable) {
        Map<String, String> expected = dataTable.asMap();
        User responseBody = response.getBody();
        
        assertEquals(expected.get("username"), responseBody.getUsername());
        assertEquals(expected.get("email"), responseBody.getEmail());
    }

    // Пример с Scenario Outline
    @When("a POST request is sent with {string} and {string}")
    public void sendPostRequest(String username, String email) {
        User user = new User(username, email);
        response = restTemplate.postForEntity("/api/users", user, User.class);
    }

    @Then("the response status is {int}")
    public void verifyStatus(int status) {
        assertEquals(status, response.getStatusCode().value());
    }

    // WireMock пример
    @Given("external service is available")
    public void mockExternalService() {
        wireMockServer.stubFor(
            post(urlEqualTo("/api/external"))
                .willReturn(aResponse().withStatus(200))
        );
    }
}
```

---

#### **7. Сущность и контроллер (для контекста)**
```java
// User.java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String email;

    // Геттеры и сеттеры
}

// UserController.java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        // Логика сохранения (например, userRepository.save(user))
        return ResponseEntity.status(201).body(user);
    }
}
```

---

#### **8. Запуск и дебаг**
**Запуск тестов**:
```bash
./gradlew test -Pitest
```

**Дебаг**:
1. Добавьте в `gradle.properties`:
```properties
org.gradle.jvmargs=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
```
2. В IntelliJ: `Run -> Edit Configurations -> Remote JVM Debug` (порт 5005).

---

#### **9. Лучшие практики**
1. **Очистка данных**:
```java
@Before("@Registration")
public void clearDatabase() {
    userRepository.deleteAll();
}
```

2. **Мокирование через WireMock**:
```java
wireMockServer.stubFor(
    get(urlPathEqualTo("/api/validate"))
        .willReturn(aResponse()
            .withHeader("Content-Type", "application/json")
            .withBody("{ \"valid\": true }"))
);
```

3. **Параметризация тестов**:
```gherkin
Scenario Outline: Data-driven testing
  When I enter <value> in the field
  Then the result should be <result>

  Examples:
    | value | result |
    | 5     | 25     |
    | 10    | 100    |
```

---

#### **10. Полная структура проекта**
```
src/
├── main/
│   └── java/
│       └── com/example/
│           ├── User.java
│           ├── UserController.java
│           └── UserRepository.java
└── test/
    ├── java/
    │   ├── config/
    │   │   ├── BaseIntegrationTest.java
    │   │   └── WireMockConfig.java
    │   ├── steps/
    │   │   └── UserRegistrationSteps.java
    │   └── CucumberSpringConfig.java
    └── resources/
        ├── features/
        │   └── user_registration.feature
        ├── application-itest.properties
        └── junit-platform.properties
```

---

Теперь у вас есть:
- Полная интеграция с **Testcontainers** (PostgreSQL)
- Мокирование внешних API через **WireMock**
- Параметризованные тесты с **DataTable** и **Scenario Outline**
- Поддержка Spring Boot 3 и Java 17
- Генерация HTML-отчетов
- Возможность дебага через порт 5005

Для запуска: 
```bash
./gradlew test -Pitest --debug-jvm # Для дебага
```

### Пример обработки заказа магазина на Spring Boot 3 (пошагово)

---

#### **1. Сущности (Entity)**
```java
// Product.java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private double price;
    private int stock;

    // Геттеры и сеттеры
}

// Order.java
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    private User user;
    
    @ElementCollection
    private Map<Long, Integer> items; // productId -> quantity
    
    private double totalAmount;
    private OrderStatus status;

    // Геттеры и сеттеры
}

enum OrderStatus {
    CREATED, PAID, CANCELLED
}
```

---

#### **2. Репозитории**
```java
// ProductRepository.java
public interface ProductRepository extends JpaRepository<Product, Long> {
}

// OrderRepository.java
public interface OrderRepository extends JpaRepository<Order, Long> {
}
```

---

#### **3. Сервис обработки заказов**
```java
// OrderService.java
@Service
@Transactional
public class OrderService {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private OrderRepository orderRepository;

    public Order createOrder(Long userId, Map<Long, Integer> items) {
        // Проверка наличия товаров
        items.forEach((productId, quantity) -> {
            Product product = productRepository.findById(productId)
                .orElseThrow(() -> new RuntimeException("Product not found"));
            
            if (product.getStock() < quantity) {
                throw new RuntimeException("Not enough stock for product " + productId);
            }
        });

        // Расчет суммы
        double total = items.entrySet().stream()
            .mapToDouble(entry -> {
                Product product = productRepository.findById(entry.getKey()).get();
                return product.getPrice() * entry.getValue();
            })
            .sum();

        // Создание заказа
        Order order = new Order();
        order.setUser(new User(userId)); // Предполагается, что User уже существует
        order.setItems(items);
        order.setTotalAmount(total);
        order.setStatus(OrderStatus.CREATED);

        // Обновление остатков
        items.forEach((productId, quantity) -> {
            Product product = productRepository.findById(productId).get();
            product.setStock(product.getStock() - quantity);
            productRepository.save(product);
        });

        return orderRepository.save(order);
    }
}
```

---

#### **4. REST-контроллер**
```java
// OrderController.java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping
    public ResponseEntity<?> createOrder(@RequestBody OrderRequest request) {
        try {
            Order order = orderService.createOrder(request.getUserId(), request.getItems());
            return ResponseEntity.status(201).body(order);
        } catch (RuntimeException e) {
            return ResponseEntity.status(409).body(Map.of("error", e.getMessage()));
        }
    }
}

// OrderRequest.java
public class OrderRequest {
    private Long userId;
    private Map<Long, Integer> items;

    // Геттеры и сеттеры
}
```

---

#### **5. Тесты с Cucumber и Testcontainers**

**Feature-файл (`order_processing.feature`)**:
```gherkin
Feature: Order Processing
  Scenario: Successful order creation
    Given the following products exist:
      | name   | price | stock |
      | Laptop | 1000  | 5     |
      | Mouse  | 50    | 10    |
    When a POST request is sent to /api/orders with:
      | userId | 1   |
      | items  | { "1": 2, "2": 3 } |
    Then the response status is 201
    And the order total amount is 2150.0
    And product stock is updated:
      | productId | stock |
      | 1         | 3     |
      | 2         | 7     |

  Scenario: Order with insufficient stock
    Given the following products exist:
      | name  | price | stock |
      | Book  | 20    | 1     |
    When a POST request is sent to /api/orders with:
      | userId | 1   |
      | items  | { "1": 3 } |
    Then the response status is 409
    And the error message is "Not enough stock for product 1"
```

---

#### **6. Step Definitions**
```java
// OrderSteps.java
public class OrderSteps extends BaseIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private OrderRepository orderRepository;

    private ResponseEntity<Map> response;

    @Given("the following products exist:")
    public void createProducts(DataTable dataTable) {
        List<Map<String, String>> products = dataTable.asMaps();
        products.forEach(p -> {
            Product product = new Product();
            product.setName(p.get("name"));
            product.setPrice(Double.parseDouble(p.get("price")));
            product.setStock(Integer.parseInt(p.get("stock")));
            productRepository.save(product);
        });
    }

    @When("a POST request is sent to /api/orders with:")
    public void sendOrderRequest(DataTable dataTable) {
        Map<String, Object> request = dataTable.asMap(String.class, Object.class);
        // Преобразование items из строки в Map<Long, Integer>
        String itemsJson = (String) request.get("items");
        Map<Long, Integer> items = parseJsonToMap(itemsJson);
        
        request.put("items", items);
        response = restTemplate.postForEntity("/api/orders", request, Map.class);
    }

    @Then("the order total amount is {double}")
    public void verifyTotal(double expectedTotal) {
        Order order = orderRepository.findAll().get(0);
        assertEquals(expectedTotal, order.getTotalAmount());
    }

    @Then("product stock is updated:")
    public void verifyStock(DataTable dataTable) {
        List<Map<String, String>> expected = dataTable.asMaps();
        expected.forEach(e -> {
            Product product = productRepository.findById(Long.parseLong(e.get("productId"))).get();
            assertEquals(Integer.parseInt(e.get("stock")), product.getStock());
        });
    }

    private Map<Long, Integer> parseJsonToMap(String json) {
        // Реализация парсинга JSON (можно использовать Jackson)
    }
}
```

---

#### **7. Конфигурация Testcontainers**
```java
// BaseIntegrationTest.java (как в предыдущих примерах)
@Testcontainers
@ActiveProfiles("itest")
public abstract class BaseIntegrationTest {
    // Конфигурация PostgreSQL
}
```

---

#### **8. Запуск тестов**
```bash
./gradlew test -Pitest
```

---

#### **9. Как это работает**
1. **Инициализация данных**: 
   - Шаг `Given` создает продукты через ProductRepository.
   - Testcontainers автоматически поднимает PostgreSQL.

2. **Создание заказа**: 
   - POST-запрос отправляется через TestRestTemplate.
   - Сервис проверяет остатки, рассчитывает сумму, сохраняет заказ.

3. **Проверки**:
   - Статус ответа (201 или 409).
   - Общая сумма заказа.
   - Обновленные остатки товаров.

---

#### **10. Обработка ошибок**
- Если товара нет в наличии, возвращается статус `409 Conflict`.
- Для обработки исключений добавлен `try-catch` в контроллере.

---

#### **11. Полная структура проекта**
```
src/
├── main/
│   └── java/
│       └── com/shop/
│           ├── model/
│           │   ├── Product.java
│           │   ├── Order.java
│           │   └── User.java
│           ├── repository/
│           │   ├── ProductRepository.java
│           │   └── OrderRepository.java
│           ├── service/
│           │   └── OrderService.java
│           └── controller/
│               └── OrderController.java
└── test/
    ├── java/
    │   ├── steps/
    │   │   └── OrderSteps.java
    │   └── config/
    │       ├── BaseIntegrationTest.java
    │       └── CucumberSpringConfig.java
    └── resources/
        ├── features/
        │   └── order_processing.feature
        └── application-itest.properties
```

Теперь вы можете: 
- Создавать заказы через REST API
- Тестировать бизнес-логику с Cucumber
- Использовать реальную БД через Testcontainers
- Обрабатывать ошибки недостатка товаров
