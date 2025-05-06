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
	4 @And - как @Then, часто с verify проверками в тестах
```

Приведу структурированный самоучитель с примерами для Spring Boot 3, Java 17 и Gradle:

# Самоучитель по Cucumber и Gherkin с Spring Boot 3

## 1. Настройка проекта (build.gradle)

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
}

dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'io.cucumber:cucumber-spring:7.15.0'
    testImplementation 'io.cucumber:cucumber-java:7.15.0'
    testImplementation 'io.cucumber:cucumber-junit-platform-engine:7.15.0'
    testImplementation 'org.testcontainers:testcontainers:1.19.3'
    testImplementation 'org.testcontainers:postgresql:1.19.3'
    testImplementation 'org.testcontainers:junit-jupiter:1.19.3'
    testImplementation 'com.github.tomakehurst:wiremock:3.0.1'
}
```

## 2. Конфигурация тестового окружения

`src/test/resources/application-itest.properties`:
```properties
spring.datasource.url=jdbc:tc:postgresql:15-alpine:///orderdb
spring.datasource.driver-class-name=org.testcontainers.jdbc.ContainerDatabaseDriver
wiremock.server.port=8089
order.validation.min-items=1
order.validation.max-items=10
```

## 3. Примеры тестов с Cucumber

### 3.1 Пример с DataTable и сравнением

`order.feature`:
```gherkin
Feature: Order processing
  @OrderProcessing @TASK-123
  Scenario: Create order with multiple items
    Given The following order items:
      | productId | quantity |
      | 101       | 2        |
      | 102       | 1        |
    When I create a new order
    Then The response should contain:
      | field       | value           |
      | status      | CREATED         |
      | totalItems  | 3               |
```

`OrderStepDefinitions.java`:
```java
public class OrderStepDefinitions {

    @Autowired
    private OrderRepository repository;
    
    private ResponseEntity<OrderResponse> response;

    @Given("The following order items:")
    public void createOrderItems(List<Map<String, Integer>> items) {
        // Преобразование DataTable в объект заказа
    }
    
    @Then("The response should contain:")
    public void verifyResponse(List<Map<String, String>> expected) {
        expected.forEach(entry -> {
            String actualValue = switch(entry.get("field")) {
                case "status" -> response.getBody().getStatus();
                case "totalItems" -> String.valueOf(response.getBody().getTotalItems());
                default -> throw new IllegalArgumentException();
            };
            assertEquals(entry.get("value"), actualValue);
        });
    }
}
```

### 3.2 Пример с одиночным значением
```gherkin
Scenario: Get order by id
  When I get order with id "123"
  Then The response status should be 200
  And The order status should be "COMPLETED"
```

```java
@Then("The order status should be {string}")
public void verifyOrderStatus(String expectedStatus) {
    assertEquals(expectedStatus, response.getBody().getStatus());
}
```

### 3.3 Пример с использованием значений из properties
```java
@Value("${order.validation.max-items}")
private int maxItems;

@Then("Validate order items count")
public void validateItemsCount() {
    assertTrue(order.getItems().size() <= maxItems);
}
```

## 4. Конфигурация Testcontainers и WireMock

`TestContainersConfig.java`:
```java
@TestConfiguration
public class TestContainersConfig {
    
    @Bean(initMethod = "start", destroyMethod = "stop")
    public WireMockServer wireMockServer() {
        return new WireMockServer(options().port(8089));
    }
}
```

## 5. Аннотации для тестов

### 5.1 Пользовательские аннотации
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@CucumberOptions(tags = "@integration")
public @interface IntegrationTest {}

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Tag("smoke")
public @interface SmokeTest {}
```

### 5.2 Использование аннотаций
```java
@CucumberContextConfiguration
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@ActiveProfiles("itest")
@IntegrationTest
@ContextConfiguration(classes = {TestContainersConfig.class})
public class CucumberSpringConfiguration {}
```

### 5.3 Запуск через командную строку
```bash
# Запуск smoke-тестов
./gradlew test -Dcucumber.filter.tags="@smoke"

# Запуск тестов для конкретной задачи
./gradlew test -Dcucumber.filter.tags="@TASK-123"
```

## 6. Настройка дебага

Для запуска с дебагом:
```bash
./gradlew bootRun --args='--spring.profiles.active=itest --debug'
```

В IntelliJ:
1. Создайте конфигурацию "Remote JVM Debug"
2. Порт: 5005
3. Запустите приложение с профилем `itest`

## 7. Пример использования DataTable для преобразования

```java
@DataTableType
public OrderItem convertOrderItem(Map<String, String> entry) {
    return new OrderItem(
        Integer.parseInt(entry.get("productId")),
        Integer.parseInt(entry.get("quantity"))
    );
}

@Given("The following order items:")
public void setOrderItems(List<OrderItem> items) {
    orderRequest.setItems(items);
}
```

## 8. Полная структура проекта
```
src/
  test/
    java/
      com/example/
        steps/
          OrderStepDefinitions.java
        config/
          TestContainersConfig.java
        CucumberSpringConfiguration.java
    resources/
      features/
        order.feature
      application-itest.properties
```

## 9. Группировка тестов

```gherkin
@Regression @TASK-456
Scenario: Update order status
  # ...

@Smoke @Security
Scenario: Check order access
  # ...
```

Для запуска определенных групп:
```bash
./gradlew test -Dcucumber.filter.tags="@Smoke and not @Security"
```

## 10. Интеграция с JUnit 5

`build.gradle`:
```groovy
tasks.withType(Test).configureEach {
    useJUnitPlatform()
}
```

Этот подход позволяет:
- Использовать Testcontainers для Postgres и WireMock
- Запускать тесты с профилем `itest`
- Дебажить тесты через IDE
- Группировать тесты с помощью аннотаций
- Использовать параметризацию через DataTable
- Интегрироваться со Spring Boot 3
