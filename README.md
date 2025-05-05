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
