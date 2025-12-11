---
title: 'TestContainers 2: R2DBC + Mockserver Spring Boot'
author: edison
date: 2025-12-10 21:52:00 +0800
categories: [ 'Java', 'Spring Boot' ]
tags: ['Testcontainer', 'Test', 'Mockserver', 'R2DBC' ]
image:
  path: /assets/images/java/testcontainers/logo-2.jpg
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Test Containers
---

Veremos como testear nuestros servicios de Spring Boot con Testcontainers:

- **Java 21** ☕
- **R2DBC**
- **Mockserver**

> **R2DBC** facilita la integración asincrónica con bases de datos relacionales. 🤖
{: .prompt-info }

> **Mockserver** levanta un servidor "falso", permitiéndonos simular llamadas a servicios reales. 🤖
{: .prompt-info }

La parte de configuración de entidades, repositorios y servicios la usaremos de <https://edisonrivera.github.io/posts/test-containers-springboot/>

+ **application.properties** principal.

```
# R2DBC
spring.r2dbc.url=r2dbc:sqlserver://localhost:1443/bank?encrypt=true&trustServerCertificate=true
spring.r2dbc.username=
spring.r2dbc.password=

# External Services
api.url=http://localhost:8081
```

**`api.url`**: Es la URL base del servicio externo que vamos a mockear.


+ Agregamos las siguientes dependencias al pom.

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-webflux-test</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-testcontainers</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>testcontainers-junit-jupiter</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>testcontainers-mssqlserver</artifactId>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>mockserver</artifactId>
  <version>1.21.3</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.mock-server</groupId>
  <artifactId>mockserver-netty</artifactId>
  <version>5.15.0</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.testcontainers</groupId>
  <artifactId>testcontainers-r2dbc</artifactId>
  <scope>test</scope>
</dependency>
```

+ Creamos un archivo **container-license-acceptance.txt** en la ruta `/test/resources`, necesario para aceptar la licencia de uso de SQL Server y permitir inicializar el testcontainer

```text
mcr.microsoft.com/mssql/server:2017-CU12
```

+ Creamos una clase base en la que configuraremos el **MSSQL Testcontainer** y **Mockserver Testcontainer**.

```java
import org.junit.jupiter.api.BeforeAll;
import org.mockserver.client.MockServerClient;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MockServerContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.mssqlserver.MSSQLServerContainer;
import org.testcontainers.utility.DockerImageName;

@Testcontainers(disabledWithoutDocker = true)
public abstract class BaseContainerTest {
    @Container
    static final MSSQLServerContainer MSSQL_CONTAINER =
            new MSSQLServerContainer(DockerImageName.parse("mcr.microsoft.com/mssql/server:2022-latest"))
                    .acceptLicense()
                    .withPassword("testcontainerssqlserver123!$")
                    .withReuse(true);

    @Container
    static final MockServerContainer MOCK_SERVER_CONTAINER = new MockServerContainer(
            DockerImageName.parse("mockserver/mockserver:5.15.0")
    );

    protected static MockServerClient mockServerClient;

    @DynamicPropertySource
    static void registerDatabaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.r2dbc.url", () -> "r2dbc:tc:sqlserver:///?TC_IMAGE_TAG=2017-CU12");
        registry.add("spring.r2dbc.username", MSSQL_CONTAINER::getUsername);
        registry.add("spring.r2dbc.password", MSSQL_CONTAINER::getPassword);

        registry.add("api.url", MOCK_SERVER_CONTAINER::getEndpoint);
    }

    @BeforeAll
    static void beforeAll() {
        mockServerClient = new MockServerClient(
                MOCK_SERVER_CONTAINER.getHost(), MOCK_SERVER_CONTAINER.getServerPort()
        );
    }
}
```

**`registry.add("spring.r2dbc.url", () -> "r2dbc:tc:sqlserver:///?TC_IMAGE_TAG=2017-CU12")`**: Definimos la cadena de conexión a la base de datos, en formato reactivo.

```java
@BeforeAll
static void beforeAll() {
    mockServerClient = new MockServerClient(
            MOCK_SERVER_CONTAINER.getHost(), MOCK_SERVER_CONTAINER.getServerPort()
    );
}
```

Con esto, inicializamos **MockServerClient** para agregar los mocks a los distintos servicios externos que necesitemos.

```java
registry.add("api.url", MOCK_SERVER_CONTAINER::getEndpoint);
```

Indicamos como **URL Base** la url del **MockerServer**

+ Usamos los testcontainers en un test y configuramos el mock de un servicio.

```java
import org.mockserver.model.HttpRequest;
import org.mockserver.model.HttpResponse;


@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class AccountControllerTest extends BaseContainerTest {
  @BeforeEach
  void setUp() {
    this.webTestClient = WebTestClient.bindToServer()
      .baseUrl("http://localhost:%s/api/v1/accounts".formatted(this.port))
      .build();


    mockServerClient.when(
        HttpRequest.request()
          .withMethod("GET")
          .withPath("/service-api-customer/api/v1/customers")
      )
      .respond(
        HttpResponse.response()
          .withStatusCode(200)
          .withHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
      );
  }
}
```

Configuramos un mock con: 
- **Servicio Simulado**: [GET] `/service-api-customer/api/v1/customers`
- **HTTP Status**: `200`

+ Inicializamos la base de datos con información previa

```text
# Spring JPA
spring.sql.init.mode=always
spring.jpa.defer-datasource-initialization=false
spring.sql.init.schema-locations=classpath:init.sql
```

Archivo **init.sql**

```sql
CREATE DATABASE bank;

create table genre
(
    id_genre    tinyint identity primary key,
    description varchar(20) not null unique
);

INSERT INTO genre (description) VALUES (N'Masculino');
INSERT INTO genre (description) VALUES (N'Femenino');
INSERT INTO genre (description) VALUES (N'Prefiero no decirlo');
```

Con esto, cuando ejecutemos algún test de integración de un controlador que llame al servicio mockeado, se simulará este request.

```java
[INFO] Running ec.bank.api.transaction.testcontainer.controller.AccountControllerTest
21:31:00.225 [main] INFO tc.mockserver/mockserver:5.15.0 -- Creating container for image: mockserver/mockserver:5.15.0
21:31:00.303 [main] INFO tc.mockserver/mockserver:5.15.0 -- Container mockserver/mockserver:5.15.0 is starting: ab76d1059e12d07b723025563f1de8842e80cb4812fbd8f1176d1af2f6a7187d
21:31:02.338 [main] INFO tc.mockserver/mockserver:5.15.0 -- Container mockserver/mockserver:5.15.0 started in PT2.1122864S

[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 63.87 s -- in ec.bank.api.transaction.testcontainer.controller.AccountControllerTest
[INFO] Running ec.bank.api.transaction.testcontainer.service.movement.MovementTypeServiceTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.403 s -- in ec.bank.api.transaction.testcontainer.service.movement.MovementTypeServiceTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
  ...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
  [INFO] Total time:  01:17 min
[INFO] Finished at: 2025-12-10T21:32:09-05:00
[INFO] ------------------------------------------------------------------------
```

![Project-Structure-2.png](/assets/images/java/testcontainers/project-structure-2.png)
> Estructura del proyecto


#### **Ventajas** ✅
- Al "falsear" llamadas a servicios externos, logramos realizar de forma completa tests de integración.
- Se puede usar **R2DBC** con **testcontainers** y evitarnos configuraciones largas y limitantes con **H2**
