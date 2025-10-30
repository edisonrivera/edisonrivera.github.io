---
title: 'Validar requests con Spring Validation'
author: edison
date: 2025-10-28 18:51:00 +0800
categories: [ 'Java', 'Spring Boot' ]
tags: [ 'Validación' ]
image:
  path: /assets/images/java/validation/logo.webp
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Spring Validation
---

Veremos varias anotaciones de **spring-boot-starter-validation** y como validar la información que llega a nuestros
servicios antes de ejecutar procesos 👽

| Anotación             | Descripción                                                                                           |
|:----------------------|:------------------------------------------------------------------------------------------------------|
| **`@NotNull`**        | Verifica que un objeto no sea **nulo**                                                                |
| **`@NotEmpty`**       | Verifica que el objeto no sea **nulo** y no esté **vacío**                                            |
| **`@NotBlank`**       | Verifica que un objeto no sea **nulo** y no contenga solo caracteres en blanco                        |
| **`@Size`**           | Verifica que un **String** o **Iterable** tenga tamaños límite                                        |
| **`@Max`**            | Verifica que el número sea `>=` (Mayor igual) al indicado                                             |
| **`@Min`**            | Verifica que el número sea `=<` (Menor igual) al indicado                                             |
| **`@Pattern`**        | Verifica que el objeto coincida con un regex                                                          |
| **`@Past`**           | Verifica que la fecha sea menor a la actual                                                           |
| **`@Future`**         | Verifica que la fecha sea mayor a la actual                                                           |
| **`@Digits`**         | Asegura que el valor numérico del campo tenga un número específico de dígitos enteros y fraccionarios |
| **`@DecimalMin`**     | Asegura que el valor decimal sea `>=` (Mayor igual) al especificado.                                  |
| **`@DecimalMax`**     | Asegura que el valor decimal sea `=<` (Menor igual) al especificado.                                  |
| **`@AssertTrue`**     | Asegura que el valor sea **true**.                                                                    |
| **`@AssertFalse`**    | Asegura que el valor sea siempre **false**.                                                           |
| **`@Positive`**       | Asegura el que valor sea positivo.                                                                    |
| **`@PositiveOrZero`** | 	Asegura el que valor sea positivo o cero.                                                            |
| **`@Negative`**       | Asegura el que valor sea negativo.                                                                    |
| **`@NegativeOrZero`** | Asegura el que valor sea negativo o cero.                                                             |
| **`@URL`**            | Asegura que el valor sea un URL válida.                                                               |
| **`@Negative`**       | Asegura el que valor sea negativo.                                                                    |

---

Para usar estas anotaciones y validar la información, seguiremos los siguientes pasos:

- Agregar la dependencia **spring-boot-starter-validation** en nuestro pom.xml

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

- Debemos anotar con **`@Valid`** el dto.

```java
@DeleteMapping
@ResponseStatus(HttpStatus.NO_CONTENT)
public Mono<Void> delete(@RequestBody @Valid final QuotaCodRequestDto quotaCodRequestDto) {
    return this.quotaService.delete(quotaCodRequestDto);
}
```

- Si queremos validar **`@PathVariable`** o **`@RequestParam`**, debemos agregar la anotación **`@Validated`** en nuestro controlador

```java
@Validated
public class QuotaController {
  ...
}
```

Para cada anotación podemos indicar un mensaje de error en caso de que la validación no se cumpla

```java
public record QuotaFilterRequestDto(
        @Size(max = 50, message = "El campo description tiene un m\u00e1ximo de {max} caracteres")
        String description,

        Boolean finished,

        @NotNull(message = "El campo pago es obligatorio")
        @Min(value = 0, message = "El campo pageNo tiene un mínimo de {value}")
        Integer pageNo,

        @NotNull(message = "El campo pageSize es obligatorio")
        @Min(value = 1, message = "El campo pageSize tiene un mínimo de {value}")
        @Max(value = 10, message = "El campo pageSize tiene un máximo de {value}")
        Integer pageSize
) {
}
```

> Podemos usar placeholders `{}` en los mensajes de error, esto facilita el que si tenemos que cambiar
> algún valor de la validación, no nos preocupemos por cambiar en el mensaje también. 🤖
{: .prompt-info }

---

Si queremos capturar el mensaje de error para devolverlo como respuesta en el api, lo haremos con un **`@RestControllerAdvice`**

```java
public record ErrorMessageResponseDto(String message, Instant timestamp) {
}
```
> Usaremos este dto para devolver información sobre los errores de validación.


- Capturar errores de **`@RequestBody`** + **`@Valid`**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<ErrorMessageResponseDto> handleMethodValidationExceptions(final MethodArgumentNotValidException ex) {
    final String errorMessage = ex.getBindingResult().getAllErrors().stream().findFirst()
      .map(DefaultMessageSourceResolvable::getDefaultMessage).orElse("Validation error unexpected");

    return new ResponseEntity<>(new ErrorMessageResponseDto(errorMessage, Instant.now()), HttpStatus.UNPROCESSABLE_ENTITY);
  }
}
```

- Capturar errores de **`@RequestParam`**, **`@PathVariable`** + **`@Validated`**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorMessageResponseDto> handleConstraintViolationExceptions(final ConstraintViolationException ex) {
        final String errorMessage = ex.getConstraintViolations().stream().findFirst()
                .map(ConstraintViolation::getMessage).orElse("Validation error unexpected");

        return new ResponseEntity<>(new ErrorMessageResponseDto(errorMessage, Instant.now()), HttpStatus.UNPROCESSABLE_ENTITY);
    }
}
```

> Al validar **`@RequestBody`** + **`@Valid`** la excepción es del tipo **MethodArgumentNotValidException**.
> Y si validamos **`@RequestParam`**, **`@PathVariable`** + **`@Validated`** la excepción es **ConstraintViolationException** 💡
{: .prompt-info }

---

Por defecto **`@Valid`** únicamente verifica y valida los campos de primer nivel del dto.

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class AdminRegisterRequestDto implements Serializable {
  @NotBlank(message = "El nombre es obligatorio")
  @Pattern(regexp = "^[a-zA-Z\\s]{5,100}$", message = "El nombre solo permite letras")
  private String name;

  @NotBlank(message = "El apellido es obligatorio")
  @Pattern(regexp = "^[a-zA-Z\\s]{5,100}$", message = "El apellido solo permite letras")
  private String lastName;

  @NotNull(message = "Los datos de usuario son obligatorios")
  private UsuarioRegisterDto user;
}
```

Por ejemplo, aquí únicamente se validarán los campos **name** y **lastName**, si el dto **UsuarioRegisterDto** también tiene
validaciones no se tomarán en cuenta.

Para validar dto's anidados debemos anotarlos con **`@Valid`**, esto garantiza que se validen los campos de ese dto.

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class AdminRegisterRequestDto implements Serializable {
  @NotBlank(message = "El nombre es obligatorio")
  @Pattern(regexp = "^[a-zA-Z\\s]{5,100}$", message = "El nombre solo permite letras")
  private String name;

  @NotBlank(message = "El apellido es obligatorio")
  @Pattern(regexp = "^[a-zA-Z\\s]{5,100}$", message = "El apellido solo permite letras")
  private String lastName;

  @Valid
  @NotNull(message = "Los datos de usuario son obligatorios")
  private UsuarioRegisterDto user;
}
```

### **¿Por qué validar la información de nuestros servicios?** 🤔

- Garantiza que se trabaje con información limpia, evitando errores inesperados.
- Optimiza el uso de recursos, ya que si los datos no pasan la validación nunca llega a ejecutar lógica interna (Consultar base de datos, llamar a servicios terceros, etc.)
- Informamos los errores de manera detallada siguiendo estándares y buenas prácticas.


#### **Observaciones**

- No todas las anotaciones se pueden usar con todos los tipos de datos. Por ejemplo si tratamos de validar con **`@Past`** un Integer obtendremos una excepción

```
jakarta.validation.UnexpectedTypeException: HV000030: No validator could be found for constraint 'jakarta.validation.constraints.Past' validating type 'java.lang.Integer'. Check configuration for 'pageNo'
```
