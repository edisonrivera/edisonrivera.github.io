---
title: 'TestContainers en Spring Boot'
author: edison
date: 2025-11-11 18:51:00 +0800
categories: [ 'Java', 'Spring Boot' ]
tags: ['Testcontainer', 'Test' ]
image:
  path: /assets/images/java/testcontainers/logo.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Test Containers
---

Veremos como testear nuestros servicios con Testcontainers, permitiendo un entorno más real, rápido y eliminando las 
limitaciones de **Mocks** o bases de datos en memoria **H2**.

---
Para este ejemplo usaremos:
- **Java 21** ☕
- **@SpringBootTest**
- **WebFlux**
- **TestContainer MSSQL Server**

---


1. Definimos una **Entity**

```java
@Entity
@Table(name = "quota", schema = "dbo", catalog = "expense")
@Data
public class QuotaEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "cod_quota")
    private Long codQuota;

    private String description;

    @Column(name = "begin_date")
    private LocalDate beginDate;

    private Short months;

    private BigDecimal total;

    private BigDecimal amortization;

    private boolean finished;
}
```

2. Definimos un **Repository**

```java
@Repository
public interface QuotaRepository extends ListCrudRepository<QuotaEntity, Long> {
    @Query("SELECT months FROM QuotaEntity WHERE codQuota = :codQuota")
    Optional<Short> getMonthsByCodQuota(Long codQuota);

    @Modifying
    @Transactional
    @Query("UPDATE QuotaEntity SET finished = :finished WHERE codQuota = :codQuota")
    void setFinishedStatus(Long codQuota, Boolean finished);
}
```

3. Realizaremos el test de un **CRUD**

```java
@RestController
@RequestMapping("/v1/api/quotas")
@RequiredArgsConstructor
public class QuotaController {
  ...

  @PostMapping("/all")
  public Mono<QuotaResponseDto> getAll(@RequestBody @Valid final QuotaFilterRequestDto quotaFilterRequestDto) {
    return this.quotaService.getAll(quotaFilterRequestDto);
  }

  @PostMapping
  @ResponseStatus(HttpStatus.CREATED)
  public Mono<Void> create(@RequestBody @Valid final QuotaRequestDto quotaRequestDto) {
    return this.quotaService.create(quotaRequestDto);
  }

  @DeleteMapping
  @ResponseStatus(HttpStatus.NO_CONTENT)
  public Mono<Void> delete(@RequestBody @Valid final QuotaCodRequestDto quotaCodRequestDto) {
    return this.quotaService.delete(quotaCodRequestDto);
  }
}
```

4. Agregamos las dependencias para usar **Testcontainers**

```xml
  <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
  </dependency>
  <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>testcontainers-mssqlserver</artifactId>
      <version>2.0.1</version>
      <scope>test</scope>
  </dependency>
```

5. Definimos la siguiente clase para usar un contenedor que ejecute SQL Server

```java
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.mssqlserver.MSSQLServerContainer;
import org.testcontainers.utility.DockerImageName;

@Testcontainers(disabledWithoutDocker = true)
public abstract class MssqlServerTestContainer {
    @Container
    private static final MSSQLServerContainer MSSQL_CONTAINER =
            new MSSQLServerContainer(DockerImageName.parse("mcr.microsoft.com/mssql/server:2022-latest"))
                    .acceptLicense()
                    .withPassword("Testcontainers2024!")
                    .withReuse(true);

    @DynamicPropertySource
    static void registerDatabaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", MSSQL_CONTAINER::getJdbcUrl);
        registry.add("spring.datasource.username", MSSQL_CONTAINER::getUsername);
        registry.add("spring.datasource.password", MSSQL_CONTAINER::getPassword);
        registry.add("spring.datasource.driver-class-name", () -> "com.microsoft.sqlserver.jdbc.SQLServerDriver");
    }
}
```

**Observaciones**
- **`@Testcontainers(disabledWithoutDocker = true)`**: Indicamos que en caso de que en el entorno donde ejecutemos las pruebas no tenga **Docker activado** las pruebas no se ejecuten.
- **`@Container`**: Con esto indicamos que el contenedor se inicie y detenga automáticamente antes y después de finalizar todos los tests.
- **`.withReuse(true)`**: Nos permite poder reutilizar el mismo **test container** en múltiples test, evitando parar el contenedor y volver a crear otro entre cada test.
- Usamos `@DynamicPropertySource` para cargar las **application.properties** de forma dinámica (En tiempo de ejecución).

6. En este ejemplo se usa la base de datos **expense**, como el testcontainer inica vacío debemos indicarle que cuando inicie cree esta base de datos

- **application.properties**
```
spring.jpa.hibernate.ddl-auto=create-drop
spring.sql.init.mode=always
spring.sql.init.data-locations=classpath:init-expense.sql
```

- **init-expense.sql**
```sql
CREATE DATABASE expense;
```

![Project-Structure.png](/assets/images/java/testcontainers/project-structure.png)

> Ejemplo de estructura del proyecto

6. Definimos los test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@Slf4j
class QuotaControllerTest extends MssqlServerTestContainer {
  @Autowired
  private WebTestClient webTestClient;

  @LocalServerPort
  private int port;

  @BeforeEach
  void setup() {
    this.webTestClient = WebTestClient.bindToServer()
      .baseUrl("http://localhost:%s/v1/api/quotas".formatted(this.port))
      .build();
  }

  @Test
  @Order(1)
  void create_isCreated() {
    final QuotaRequestDto quotaRequestDto = new QuotaRequestDto();
    quotaRequestDto.setDescription("Microondas Electrolux");
    quotaRequestDto.setBeginDate(LocalDate.now());
    quotaRequestDto.setMonths((short) 10);
    quotaRequestDto.setTotal(BigDecimal.valueOf(100));

    this.webTestClient.post()
      .uri("")
      .contentType(MediaType.APPLICATION_JSON)
      .bodyValue(quotaRequestDto)
      .exchange()
      .expectStatus().isCreated();
  }

  @Test
  @Order(2)
  void getFiltered_isOk() {
    final QuotaFilterRequestDto quotaFilterRequestDto = new QuotaFilterRequestDto(null, null, 0, 10);

    this.webTestClient.post()
      .uri("/all")
      .contentType(MediaType.APPLICATION_JSON)
      .bodyValue(quotaFilterRequestDto)
      .exchange()
      .expectStatus().isOk()
      .expectBody()
      .consumeWith(response -> {
        final String body = new String(Objects.requireNonNull(response.getResponseBody()));
        log.info("[getFiltered_isOk] Response body: {}",  body);
      })
      .jsonPath("$.quotas.length()").isEqualTo("1")
      .jsonPath("$.totalQuotas").isEqualTo("1");
  }

  @Test
  @Order(3)
  void delete_isNoContent() {
    final QuotaCodRequestDto quotaChangeRequestDto = new QuotaCodRequestDto(Collections.singleton(1L));

    this.webTestClient.method(HttpMethod.DELETE)
      .uri("")
      .bodyValue(quotaChangeRequestDto)
      .exchange()
      .expectStatus().isNoContent();
  }
}
```

7. Al ejecutar **`mvn clean install`** veremos que el tipo de base de datos para los tests es **SQL Server**.

```log
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
19:15:51.442 [main] INFO org.testcontainers.DockerClientFactory -- Testcontainers version: 1.21.3
19:15:51.977 [main] INFO org.testcontainers.dockerclient.DockerClientProviderStrategy -- Loaded org.testcontainers.dockerclient.NpipeSocketClientProviderStrategy from ~/.testcontainers.properties, will try it first
19:15:52.477 [main] INFO org.testcontainers.dockerclient.DockerClientProviderStrategy -- Found Docker environment with local Npipe socket (npipe:////./pipe/docker_engine)
19:15:52.479 [main] INFO org.testcontainers.DockerClientFactory -- Docker host IP address is localhost
19:15:52.505 [main] INFO org.testcontainers.DockerClientFactory -- Connected to docker: 
  Server Version: 28.5.1
  API Version: 1.51
  Operating System: Docker Desktop
  Total Memory: 7895 MB
  Labels: 
    com.docker.desktop.address=npipe://\\.\pipe\docker_cli
19:16:08.526 [main] INFO tc.mcr.microsoft.com/mssql/server:2022-latest -- Container mcr.microsoft.com/mssql/server:2022-latest started in PT13.5149946S
19:16:08.526 [main] INFO tc.mcr.microsoft.com/mssql/server:2022-latest -- Container is started (JDBC URL: jdbc:sqlserver://localhost:61136;encrypt=false)
...
[ERROR] Surefire is going to kill self fork JVM. The exit has elapsed 30 seconds after System.exit(0).
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:05 min
[INFO] Finished at: 2025-11-15T19:16:51-05:00
[INFO] ------------------------------------------------------------------------

Process finished with exit code 0
```

#### **Ventajas** ✅
- Reducción en el tiempo de escribir tests, ya que si usamos alguna característica especial de una base de datos (SQL Server, Postgres, Oracle, etc.) para hacerlo funcionar con **H2** nos podría llevar más tiempo configurarla o directamente no se podría lograr.
- Ejecutamos pruebas realistas, confiables y eliminando bugs (Ej. Funciona con **H2**, pero con **SQL Server** no), ya que usamos el mismo software que en el entorno de producción.



