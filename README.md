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
	Note. код java с логикой и шаги feature файла связаны через regexp выражения в аннотация ниже
	1 @Given - заполняем базу
		DataTable - передается в тест массив данных
	1.2 @And - как @Given, для доп условий
	2 @When - выполняем вызов метода сервиса (e.g. запрос rest)
	3 @Then - сравниваем результат (данные в базе)
		expectedTotal или DataTable - передаем для сравнения
	3.2 @And - как @Then, часто с verify проверками в тестах
	5 Examples: - тут в делаем табл со значением, потом значения из нее можно использовать в @Given, @When, @Then, @And
		(для табл с столбцами username, password, result)
		Given I enter "<username>"
		And I enter "<password>"
		When I click login
		Then I see "<result>"
```

Приведу структурированный самоучитель с примерами:

# Самоучитель по Cucumber и Gherkin для Spring Boot 3

## 1. Настройка проекта (build.gradle)

```groovy
dependencies {
    testImplementation 'io.cucumber:cucumber-java:7.15.0'
    testImplementation 'io.cucumber:cucumber-spring:7.15.0'
    testImplementation 'io.cucumber:cucumber-junit-platform-engine:7.15.0'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.testcontainers:testcontainers:1.19.3'
    testImplementation 'org.testcontainers:postgresql:1.19.3'
    testImplementation 'org.testcontainers:junit-jupiter:1.19.3'
    testImplementation 'com.github.tomakehurst:wiremock-jre8:2.35.0'
}
```

## 2. Конфигурация Testcontainers и WireMock

`src/test/java/com/example/config/IntegrationTestConfig.java`:
```java
@ActiveProfiles("itest")
@Testcontainers
public class IntegrationTestConfig {
    
    @Container
    public static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void postgresProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Bean
    public WireMockServer wireMockServer() {
        return new WireMockServer(wireMockConfig().dynamicPort());
    }
}
```

## 3. Пример feature-файла с различными сценариями

`src/test/resources/features/order-processing.feature`:
```gherkin
Feature: Order Processing
  @Smoke @Task123
  Scenario: Successful order creation
    Given The following products exist in database:
      | id | name    | price |
      | 1  | Laptop  | 999   |
      | 2  | Phone   | 699   |
    And Payment service is available
    When I create order with items:
      | productId | quantity |
      | 1         | 2        |
    Then Response status should be 201
    And Response should contain:
      | field       | value          |
      | totalPrice  | 1998           |
      | orderStatus | PROCESSING     |
    
  @Validation
  Scenario: Order with invalid data
    When I create order with invalid data
    Then Response status should be 400
    And Response should contain error "Invalid order data"
```
With **Examples** block:
```
Scenario Outline: Login
  Given I enter "<username>"
  And I enter "<password>"
  Then I see "<result>"
  
  Examples:
    | username | password | result  |
    | user1    | pass123  | Welcome |
    | invalid  | wrong    | Error   |

//the same as:
Scenario: Valid login
  Given I enter "user1"
  And I enter "pass123"
  Then I see "Welcome"

Scenario: Invalid login
  Given I enter "invalid"
  And I enter "wrong"
  Then I see "Error"
```
## 4. Step Definitions с примерами

`OrderStepDefinitions.java`:
```java
public class OrderStepDefinitions {

    @Autowired
    private TestRestTemplate restTemplate;
    
    @Given("The following products exist in database:")
    public void setupProducts(DataTable dataTable) {
        List<Map<String, String>> products = dataTable.asMaps();
        products.forEach(product -> 
            jdbcTemplate.update(
                "INSERT INTO products (id, name, price) VALUES (?, ?, ?)",
                product.get("id"), product.get("name"), product.get("price")
            )
        );
    }

    @When("I create order with items:")
    public void createOrder(DataTable dataTable) {
        List<Map<String, String>> items = dataTable.asMaps();
        OrderRequest request = new OrderRequest(items);
        response = restTemplate.postForEntity("/orders", request, Map.class);
    }

    @Then("Response status should be {int}")
    public void verifyResponseStatus(int status) {
        assertThat(response.getStatusCodeValue()).isEqualTo(status);
    }

    @And("Response should contain:")
    public void verifyResponseFields(DataTable dataTable) {
        Map<String, Object> responseBody = response.getBody();
        dataTable.asMaps().forEach(row -> 
            assertThat(responseBody.get(row.get("field")))
                .isEqualTo(parseValue(row.get("value")))
        );
    }
}
```

## 5. Работа с WireMock

`PaymentServiceSteps.java`:
```java
public class PaymentServiceSteps {

    @Autowired
    private WireMockServer wireMock;

    @Given("Payment service is available")
    public void setupPaymentService() {
        wireMock.stubFor(post(urlEqualTo("/payments"))
            .willReturn(aResponse()
                .withHeader("Content-Type", "application/json")
                .withBody("{\"status\": \"SUCCESS\"}")));
    }
}
```

## 6. Использование application.properties

`application-itest.properties`:
```properties
order.min-items=1
order.max-items=10
```

В тестах:
```java
@Value("${order.min-items}")
private int minItems;

@Then("Validate order items count")
public void validateItemsCount() {
    assertThat(order.getItems().size()).isBetween(minItems, maxItems);
}
```

## 7. Аннотации для тестов

### Кастомные аннотации:
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@CucumberContextConfiguration
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Import(IntegrationTestConfig.class)
public @interface CucumberIntegrationTest {}
```

### Маркировка тестов:
```java
@CucumberIntegrationTest
@CucumberOptions(features = "classpath:features")
@Tag("Integration")
public class RunCucumberTest {}
```

## 8. Запуск и дебаг

Запуск через Gradle:
```bash
./gradlew test -Pprofile=itest
```

Запуск с дебагом:
```bash
./gradlew test -Dorg.gradle.jvmargs="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005"
```

## 9. Работа с SQL в @Given

`src/test/resources/data/insert_products.sql`:
```sql
INSERT INTO products (id, name, price) VALUES
(1, 'Laptop', 999),
(2, 'Phone', 699);
```

В step-определении:
```java
@Given("Preloaded products")
@Sql(scripts = "/data/insert_products.sql")
public void preloadProducts() {}
```

## 10. Пример с DataTable для сравнения

```java
@Then("The response should contain following items:")
public void verifyResponseItems(DataTable dataTable) {
    List<Map<String, String>> expected = dataTable.asMaps();
    List<Map<String, Object>> actual = (List) response.getBody().get("items");
    
    assertThat(actual).hasSameSizeAs(expected);
    
    IntStream.range(0, expected.size()).forEach(i -> {
        assertThat(actual.get(i).get("productId"))
            .isEqualTo(Integer.parseInt(expected.get(i).get("productId")));
        assertThat(actual.get(i).get("quantity"))
            .isEqualTo(Integer.parseInt(expected.get(i).get("quantity")));
    });
}
```

## 11. Группировка тестов

```java
@Smoke
@Feature("Order Processing")
@Story("Basic Order Operations")
@Task("PROJ-123")
public class OrderFeatureTest {}
```

Запуск тестов по категориям:
```bash
./gradlew test -Dcucumber.filter.tags="@Smoke and not @Wip"
```

## 12. Структура проекта

```
src/test/
├── java/
│   ├── config/
│   ├── steps/
│   └── RunCucumberTest.java
├── resources/
│   ├── application-itest.properties
│   ├── features/
│   └── data/
```

Все примеры адаптированы для:
- Java 17
- Spring Boot 3
- JUnit 5
- Testcontainers
- Gradle

Тесты можно запускать как целиком, так и по тегам, с возможностью дебага через IDE. Конфигурация позволяет работать с реальными сервисами в изолированном окружении.
