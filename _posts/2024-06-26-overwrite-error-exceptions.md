---
title: 'Configuración de errores en validaciones en Spring Boot'
author: edison
date: 2024-06-26 18:51:00 +0800
categories: ['Java', 'Spring Boot']
tags: ['Configuración']
---

Veremos una forma de sobreescribir los errores al momento de validar la información en Spring Boot 🍃


1. Creamos una clase la cual se encargará de esto

```java
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.HttpStatusCode;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@ControllerAdvice
public class ValidationHandler extends ResponseEntityExceptionHandler {

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex,
                                                                  HttpHeaders headers, HttpStatusCode status, WebRequest request) {

        Map<String, List<String>> errors = new HashMap<>();
        List<String> errorMessages = new ArrayList<>();

        ex.getBindingResult().getAllErrors().forEach(error -> {
            String message = error.getDefaultMessage();
            errorMessages.add(message);
        });

        errors.put("errors", errorMessages);

        return new ResponseEntity<>(errors, HttpStatus.UNPROCESSABLE_ENTITY);
    }
}
```

En este caso, haremos que todos los errores que se lleguen a mapear se coloquen en un **`HashMap`** logrando una respuesta similar a:

```json
{
  "errors": [
    "Error 1",
    "Error 2",
    "Error 3",
    ...
  ]
}
```

Y además indicamos un código de error `422 Unprocessable Entity` para indicar un error en los datos que hemos recibido

> Con esto ya tenemos gran parte del manejo de excepciones que podamos llegar a tener, Podemos devolver cualquie tipo de dato. Por ejemplo, en lugar de devolver una lista de errores indicar únicamente un error a la vez o cosas por el estilo. 🤖
{: .prompt-info }

2. Ahora, creamos un **endpoint** en cual recibiremos dos datos por ejemplo y le añadiremos validaciones.

```java
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import lombok.Data;

@Data
public class CategoryCreateRequest {
    @NotBlank(message = "Title is required")
    @Size(max = 30, min = 3, message = "Title must not exceed 30 characters")
    private String title;

    @NotBlank(message = "TypeId is required")
    private Integer typeId;
}
```

> **`@NotBlank`** Validamos que el campo no sea nulo y que no esté vacío después de quitar los espacios en blanco.
{: .prompt-info }

> **`@Size`** Validamos una longitud máxima y mínima del campo.
{: .prompt-info }


```java
@PostMapping
public ResponseEntity<ResponseData<CategoryCreateResponse>> createCategory(@Valid @RequestBody CategoryCreateRequest request) {
    return categoryControllerService.createCategory(request);
}
```

> **`@Valid`** Esta anotación es **importante** ya que si no lo indicamos no se validaran los datos y no veremos los mensajes de error que hayamos indicado.
{: .prompt-info }

3. Ahora, nos queda comprobar

```json
{
  "title": "Este es un título super largo, al menos más de 30 caracteres",
  "typeId": 0
}
```

![01-Response.png](/assets/images/java/errorhandling/01-response.png)


Eso sería todo 🌟, podemos agregar aún más validaciones con **anotaciones** de Spring Boot y podemos modificar la estructura de la respuesta en caso de que existan errores al momento de validar la información.