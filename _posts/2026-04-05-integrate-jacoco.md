---
title: 'JaCoCo'
author: edison
date: 2025-12-10 21:52:00 +0800
categories: [ 'Java', 'Spring Boot', 'Testing' ]
tags: ['Test']
image:
  path: /assets/images/java/jacoco/jacoco-logo.webp
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: JaCoCo
---

Veremos como identificar el porcentaje de test realizados nuestro código:

- **Java 21** ☕
- **JaCoCo**

El **Code Coverage** es una métrica de cuantas líneas se testean cuando se ejecutan los tests que realizamos sobre nuestro código.

Con esto garantizamos

- Pruebas de **integración**, **sistema**, **regresión** y **unitarias** ejecutadas correctamente.
- Aplicamos un **estándar mínimo de testing** (Garantizamos fiabilidad del sistema)
- Detectamos bugs y código olvidado.
- Es más fácil aplicar cualquier cambio posterior, ya que podemos identificar claramente donde afectan dichos cambios.

Explicación de cada bloque de configuración

```xml
<execution>
	<id>jacoco-initialize</id>
	<goals>
		<goal>prepare-agent</goal>
	</goals>
</execution>
```
> Prepara un agente que se engancha a la **JVM** el cual registra todas las clases y líneas que se ejecutan en los tests.

---

```xml
<execution>
	<id>jacoco-site</id>
	<phase>test</phase>
	<goals>
		<goal>report</goal>
	</goals>
</execution>
```
> Generará el reporte de **test** cuando se ejecute la fase **`mvn test`**. Por defecto lo genera en la ruta _target/site/jacoco/index.html_

---

```xml
<execution>
	<id>check-coverage</id>
	<phase>install</phase>
	<goals>
		<goal>check</goal>
	</goals>
	...
</execution>
```
> Valida que la cobertura mínima configurada se cumpla en la fase de **`mvn install`**.

---

```xml
<execution>
	...
	<configuration>
		<excludes>
			<exclude>**/dto/**</exclude>
			<exclude>**/request/**</exclude>
			<exclude>**/response/**</exclude>
		</excludes>
		...
	</configuration>
</execution>		
```
> Indicamos los archivos que no queremos excluir al momento de ejecutar los test.

---

```xml
<executions>
	...
	
	<execution>
		...
		<configuration>
			...
			<rules>
				<rule>
					<element>BUNDLE</element>
					<limits>
						<limit>
							<counter>INSTRUCTION</counter>
							<value>COVEREDRATIO</value>
							<minimum>0.50</minimum>
						</limit>
						
						<limit>
							<counter>BRANCH</counter>
							<value>COVEREDRATIO</value>
							<minimum>0.50</minimum>
						</limit>
						
						<limit>
							<counter>LINE</counter>
							<value>COVEREDRATIO</value>
							<minimum>0.50</minimum>
						</limit>
					</limits>
				</rule>
			</rules>
		</configuration>
	</execution>
</executions>
```

- **BUNDLE**: Indica que se aplicará a todo el proyecto (paquetes, clases, métodos)

##### **COVEREDRATIO**

- **INSTRUCTION**: Contabiliza todas las instrucciones a nivel de **bytecode**

```java
public int sum(int a, int b) {
    return a + b;
}
```

Este código se puede traducir a instrucciones en la JVM

+ Carga `a`
+ Carga `b`
+ Suma `a` + `b`
+ Retorna el resultado

**JaCoCo** se encarga de contabilizar todas las instrucciones


- **BRANCH**: Contabiliza todas las decisiones lógicas (ifs, ternarios, if-else, switch)
- **LINE**: Contabiliza todas las líneas ejecutadas.

### **Ejemplo de configuración completa**

```xml
<plugin>
	<groupId>org.jacoco</groupId>
	<artifactId>jacoco-maven-plugin</artifactId>
	<version>0.8.12</version>
	
	<executions>
		<execution>
			<id>jacoco-initialize</id>
			<goals>
				<goal>prepare-agent</goal>
			</goals>
		</execution>
		
		<execution>
			<id>jacoco-site</id>
			<phase>test</phase>
			<goals>
				<goal>report</goal>
			</goals>
		</execution>
		
		<execution>
			<id>check-coverage</id>
			<phase>install</phase>
			<goals>
				<goal>check</goal>
			</goals>
			<configuration>
				<excludes>
					<exclude>**/dto/**</exclude>
					<exclude>**/request/**</exclude>
					<exclude>**/response/**</exclude>
				</excludes>
				<rules>
					<rule>
						<element>BUNDLE</element>
						
						<limits>
							<limit>
								<counter>INSTRUCTION</counter>
								<value>COVEREDRATIO</value>
								<minimum>0.50</minimum>
							</limit>
							
							<limit>
								<counter>BRANCH</counter>
								<value>COVEREDRATIO</value>
								<minimum>0.50</minimum>
							</limit>
							
							<limit>
								<counter>LINE</counter>
								<value>COVEREDRATIO</value>
								<minimum>0.50</minimum>
							</limit>
						</limits>
					</rule>
				</rules>
			</configuration>
		</execution>
	</executions>
</plugin>
```

Al ejecutar un **`mvn clean install`** veremos que si no cumplimos con los umbrales mínimos configurados obtendremos una excepción

```
[WARNING] Rule violated for bundle api-expenses-quotas: instructions covered ratio is 0.02, but expected minimum is 0.50
[WARNING] Rule violated for bundle api-expenses-quotas: branches covered ratio is 0.00, but expected minimum is 0.50
[WARNING] Rule violated for bundle api-expenses-quotas: lines covered ratio is 0.01, but expected minimum is 0.50
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  7.993 s
[INFO] Finished at: 2026-04-05T11:54:26-05:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.jacoco:jacoco-maven-plugin:0.8.12:check (check-coverage) on project api-expenses-quotas: Coverage checks have not been met. See log for details. -> [Help 1]
```

El reporte nos indicará que clases se han testeado, qué código en específico y el porcentaje de cada uno

![JacoCo_Report.png](/assets/images/java/jacoco/report.png)

**JaCoCo** se puede integrar con herramientas de CI/CD y análisis de código (Sonar), con esto podemos configurar que si el código no cumple con los umbrales mínimos configurados no permita el despliegue automático 
y con esto cumplir con un estándar mínimo de testing en todos nuestros proyectos.
