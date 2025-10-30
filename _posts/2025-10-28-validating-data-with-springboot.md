---
title: 'Validar requests con Spring Validation'
author: edison
date: 2025-10-28 18:51:00 +0800
categories: [ 'Java', 'Spring Boot' ]
tags: [ 'Validación' ]
image:
  path: /assets/images/java/validation/logo.png
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

Para cada anotación podemos indicar un mensaje en caso de que la validación no se cumpla

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
> algún valor de la validación, no nos preocupemos por cambiar en el mensaje también.  🤖
{: .prompt-info }

#### ✅ **Ventajas**

- Validar la información que llega los servicios garantiza que se trabaje con información limpia, evitando errores inesperados.
- Optimiza el uso de recursos, ya que si los datos no pasan la validación nunca llega a ejecutar lógica intern (Consulta base de datos, llamar a servicios terceros, etc)
- Informamos los errores de manera detallada siguiendo éstandares y buenas prácticas.

#### **Observaciones**

- No todas las anotaciones se pueden usar con todos los tipos de datos. Por ejemplo si tratamos de validar con **`@Past`** un Integer obtendremos una excepción

```java
jakarta.validation.UnexpectedTypeException: HV000030: No validator could be found for constraint 'jakarta.validation.constraints.Past' validating type 'java.lang.Integer'. Check configuration for 'pageNo'
```
