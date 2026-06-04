# Análisis Técnico - Backend Spring Boot
### Proyecto: `Backend de Repo` · Programación Web

---

## Índice

1. [Estructura General del Proyecto](#1-estructura-general-del-proyecto)
2. [Función de las Capas](#2-función-de-las-capas)
3. [Configuración de Seguridad y CORS](#4-configuración-de-seguridad-y-cors)
4. [Identificación de Mejoras](#5-identificación-de-mejoras)

---

## 1. Estructura General del Proyecto

El proyecto es una API REST construida con **Spring Boot 4.0.6** y **Java 21**. Su propósito es exponer métricas de desarrollo de software (commits, bugs resueltos, tareas completadas, story points) a un frontend React.

### Árbol de directorios

```
backFinalProgramacionWeb-master/
└── demo/
    ├── pom.xml
    └── src/
        └── main/
            ├── java/com/exampleback/demo/
            │   ├── DemoApplication.java          ← Punto de entrada
            │   ├── config/
            │   │   ├── CorsConfig.java            ← Configuración CORS
            │   │   └── SecurityConfig.java        ← Configuración de seguridad
            │   ├── controller/
            │   │   └── MetricsController.java     ← Capa de presentación
            │   ├── service/
            │   │   └── MetricsService.java        ← Lógica de negocio
            │   ├── repository/
            │   │   └── DeveloperMetricRepository.java  ← Acceso a datos
            │   ├── dto/
            │   │   └── MetricResponseDTO.java     ← DTO de salida
            │   └── model/
            │       └── DeveloperMetric.java       ← Entidad de dominio
            └── resources/
                └── application.properties        ← Configuración mínima
```

> **Nota:** Se eliminaron dos archivos de código muerto que existían en el proyecto original:
> - `dto/MetricRequestDTO.java` — clase sin getters, setters ni anotaciones, no referenciada en ningún lugar.
> - `repository/MetricResponseDTO.java` — duplicado exacto de `dto/MetricResponseDTO.java` ubicado en el paquete incorrecto.

### Dependencias principales (pom.xml)

| Dependencia | Versión | Uso declarado | Uso real |
|---|---|---|---|
| `spring-boot-starter-web` | 4.0.6 | API REST | Utilizada |
| `spring-boot-starter-security` | 4.0.6 | Seguridad | Solo deshabilita todo |
| `spring-boot-starter-data-jpa` | 4.0.6 | Persistencia ORM | No utilizada |
| `h2` | runtime | Base de datos en memoria | No utilizada |
| `firebase-admin` | 9.1.1 | Autenticación/DB Firebase | No utilizada |
| `lombok` | — | Reducción de boilerplate | Parcialmente |

---

## 2. Función de las Capas

### 2.1 Controller — `MetricsController.java`

```java
@RestController
@RequestMapping("/metrics")
@RequiredArgsConstructor
public class MetricsController {

    private final MetricsService service;

    @GetMapping("/{metric}")
    public List<MetricResponseDTO> getMetricData(@PathVariable String metric) {
        return service.getMetricData(metric);
    }
}
```

**Función:** Recibe las peticiones HTTP GET en la ruta `/metrics/{metric}`, donde `{metric}` es el tipo de dato solicitado (`commits`, `bugs`, `tasks`, `storyPoints`). Delega el procesamiento al Service y retorna la lista de DTOs de respuesta. Usa `@RequiredArgsConstructor` de Lombok para inyección de dependencias.

---

### 2.2 Service — `MetricsService.java`

```java
@Service
@RequiredArgsConstructor
public class MetricsService {

    private final DeveloperMetricRepository repository;

    public List<MetricResponseDTO> getMetricData(String metric) {
        List<DeveloperMetric> metrics = repository.findAll();

        return metrics.stream()
            .map(m -> {
                MetricResponseDTO dto = new MetricResponseDTO();
                dto.setLabel(m.getMetricDate().toString());

                switch (metric) {
                    case "commits":       dto.setValue(m.getCommits()); break;
                    case "bugs":          dto.setValue(m.getBugsFixed()); break;
                    case "tasks":         dto.setValue(m.getTasksCompleted()); break;
                    case "storyPoints":   dto.setValue(m.getStoryPoints()); break;
                    default:              dto.setValue(0);
                }
                return dto;
            }).toList();
    }
}
```

**Función:** Contiene la lógica de negocio. Obtiene todos los registros del Repository, los transforma en DTOs de respuesta y aplica un filtro por tipo de métrica mediante un `switch`. Es el único componente que conoce tanto las entidades del dominio como los DTOs de presentación.

---

### 2.3 Repository — `DeveloperMetricRepository.java`

```java
@Repository
public class DeveloperMetricRepository {

    public List<DeveloperMetric> findAll() {
        return List.of(
            new DeveloperMetric("Francisco", LocalDate.of(2026, 5, 1), 12, 2, 5, 8),
            new DeveloperMetric("Francisco", LocalDate.of(2026, 5, 2), 18, 1, 7, 13),
            new DeveloperMetric("Francisco", LocalDate.of(2026, 5, 3), 15, 3, 6, 10),
            new DeveloperMetric("Francisco", LocalDate.of(2026, 5, 4), 22, 0, 8, 15)
        );
    }
}
```

**Función:** Simula la capa de acceso a datos. En lugar de conectarse a una base de datos real, retorna una lista estática de objetos `DeveloperMetric` codificados directamente en el código (_hardcoded_). Solo implementa `findAll()` y no extiende ninguna interfaz de Spring Data JPA.

---

### 2.4 DTO — Data Transfer Objects

Tras eliminar los archivos de código muerto, el proyecto cuenta con un único DTO activo:

#### `dto/MetricResponseDTO.java`
```java
@Data
public class MetricResponseDTO {
    private String label;
    private Integer value;
}
```

**Función:** Transportar únicamente los datos necesarios al cliente, evitando exponer la estructura interna de las entidades. Lleva el `label` (fecha como texto) y el `value` (valor numérico de la métrica seleccionada).

---

### 2.5 Model/Entity — `DeveloperMetric.java`

```java
public class DeveloperMetric {
    private String developerName;
    private LocalDate metricDate;
    private Integer commits;
    private Integer bugsFixed;
    private Integer tasksCompleted;
    private Integer storyPoints;

    // Constructor con todos los campos
    // Getters manuales (sin Lombok, sin setters)
}
```

**Función:** Representa la estructura de datos del dominio: las métricas diarias de un desarrollador. No está anotada como entidad JPA (`@Entity`, `@Table`), por lo que no se mapea a ninguna tabla en base de datos.

---

## 3. Configuración de Seguridad y CORS

### 3.1 SecurityConfig

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(Customizer.withDefaults())
            .authorizeHttpRequests(auth -> auth
                    .anyRequest()
                    .permitAll()
            );
        return http.build();
    }
}
```

**Análisis:**

| Configuración | Estado | Impacto |
|---|---|---|
| `csrf.disable()` | Deshabilitado | Elimina protección contra ataques CSRF. Aceptable solo si no hay sesiones con cookies. |
| `cors(Customizer.withDefaults())` | Correcto | Delega la config CORS al bean `CorsConfigurationSource` declarado en `CorsConfig`. |
| `anyRequest().permitAll()` | Crítico | **Todos los endpoints son públicos**. No existe ningún tipo de autenticación ni autorización. |

### 3.2 CorsConfig

```java
configuration.setAllowedOrigins(List.of("http://localhost:5173"));
configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
configuration.setAllowedHeaders(List.of("*"));
configuration.setAllowCredentials(true);
source.registerCorsConfiguration("/**", configuration);
```

**Análisis:**

| Configuración | Estado | Detalle |
|---|---|---|
| `AllowedOrigins` | Solo localhost | Funciona en desarrollo, pero no en producción |
| `AllowedMethods` | Excesivo | Declara PUT/DELETE/POST aunque solo existe un endpoint GET |
| `AllowedHeaders("*")` | Muy permisivo | Acepta cualquier header, incluyendo potencialmente peligrosos |
| `AllowCredentials(true)` | Innecesario | Se habilita para cookies/tokens, pero no hay autenticación implementada |

---

## 4. Identificación de Mejoras

A continuación se detallan todos los problemas identificados en el proyecto, organizados por severidad.

---

### Errores Críticos

#### [ERR-01] El Repository no es un Repository real

**Archivo:** `repository/DeveloperMetricRepository.java`

**Problema:** La clase tiene la anotación `@Repository` pero no extiende `JpaRepository` ni ninguna interfaz de Spring Data. Los datos están completamente _hardcodeados_ en memoria como una lista estática. No existe ninguna conexión a base de datos.

**Impacto:** La aplicación no puede persistir, consultar ni actualizar datos reales. Cualquier cambio en los datos requiere modificar el código fuente y recompilar.

**Corrección sugerida:**
```java
// Convertir el modelo a entidad JPA
@Entity
@Table(name = "developer_metrics")
public class DeveloperMetric {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    // ... resto de campos con @Column
}

// Usar Spring Data JPA
public interface DeveloperMetricRepository
        extends JpaRepository<DeveloperMetric, Long> {
    List<DeveloperMetric> findByDeveloperName(String name);
}
```

---

#### [ERR-02] El modelo `DeveloperMetric` no está anotado como entidad JPA

**Archivo:** `model/DeveloperMetric.java`

**Problema:** La clase no tiene ninguna anotación JPA (`@Entity`, `@Id`, `@Table`). A pesar de que `spring-boot-starter-data-jpa` está declarado en el `pom.xml`, esta entidad no puede ser mapeada a una tabla de base de datos.

**Corrección sugerida:** Agregar anotaciones JPA y usar Lombok para eliminar el código manual de getters/constructor:
---

### Malas Prácticas Significativas

#### [ERR-03] Dependencias declaradas en `pom.xml` que no se utilizan

**Archivo:** `pom.xml`

**Problema:** El proyecto incluye dependencias pesadas que no están siendo usadas:

- `firebase-admin 9.1.1` — No hay ninguna referencia a Firebase en el código. Agrega ~50 MB innecesarios al classpath.
- `spring-boot-starter-data-jpa` — Importado pero el Repository no lo usa.
- `h2` — Base de datos en memoria declarada pero no configurada ni utilizada.

**Corrección sugerida:** Eliminar las dependencias no utilizadas. Si se planea conectar a una base de datos real, configurar correctamente `application.properties`:
```properties
spring.datasource.url=jdbc:h2:mem:metricsdb
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
spring.h2.console.enabled=true
```

---

#### [ERR-04] Lógica de filtrado mal ubicada en el Service

**Archivo:** `service/MetricsService.java`

**Problema:** El Service carga TODOS los registros con `repository.findAll()` y luego filtra por tipo de métrica mediante un `switch` en Java. Esto es ineficiente y viola la separación de responsabilidades: el filtrado debería hacerse a nivel de base de datos.

Adicionalmente, el caso `default` del switch retorna `0` silenciosamente sin notificar al cliente que recibió un parámetro inválido.

**Corrección sugerida:**
```java
public List<MetricResponseDTO> getMetricData(String metric) {
    List<String> validMetrics = List.of("commits", "bugs", "tasks", "storyPoints");
    if (!validMetrics.contains(metric)) {
        throw new IllegalArgumentException("Métrica no válida: " + metric);
    }
    // ... resto de la lógica
}
```

---

#### [ERR-05] Datos hardcodeados en el Repository

**Archivo:** `repository/DeveloperMetricRepository.java`

**Problema:** Los 4 registros de métricas están escritos literalmente en el código Java, con fechas y valores fijos. La aplicación es completamente estática: no puede recibir nuevos datos sin modificar el código fuente. Además, todos los registros pertenecen al mismo desarrollador ("Francisco"), sin posibilidad de multi-usuario.

**Corrección sugerida:** Conectar a una base de datos real (H2 en memoria para desarrollo, PostgreSQL/MySQL para producción) y poblarla con un archivo `data.sql` o mediante un `CommandLineRunner`.

---

#### [ERR-06] El modelo usa getters manuales en lugar de Lombok

**Archivo:** `model/DeveloperMetric.java`

**Problema:** El proyecto ya tiene Lombok como dependencia y lo usa en el DTO (`@Data`), pero el modelo `DeveloperMetric` declara todos los getters manualmente (6 métodos de ~3 líneas cada uno). Esto es inconsistente y genera código boilerplate innecesario.

**Corrección sugerida:** Reemplazar todos los getters manuales con `@Data` o `@Getter` de Lombok.

---

### Mejoras de Calidad y Buenas Prácticas

#### [ERR-07] Ausencia total de manejo de excepciones

**Problema:** No existe ningún `@ControllerAdvice`, `@ExceptionHandler`, ni manejo de errores en ninguna capa. Si el service recibe un parámetro inválido, retorna `value: 0` sin código de error. Si ocurriera una excepción real, el cliente recibiría un error 500 sin estructura.

**Corrección sugerida:**
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> handleBadRequest(IllegalArgumentException e) {
        return Map.of("error", e.getMessage());
    }
}
```

---

#### [ERR-08] Seguridad completamente deshabilitada

**Archivo:** `config/SecurityConfig.java`

**Problema:** Aunque se importa Spring Security, la configuración actual deshabilita CSRF y hace todos los endpoints públicos (`permitAll()`). La dependencia `firebase-admin` sugería la intención de implementar autenticación con Firebase JWT, pero nunca se implementó.

**Corrección sugerida:** Si no se va a usar autenticación en esta fase, eliminar la dependencia de Spring Security para no dar una falsa sensación de seguridad. Si se quiere implementar, agregar un filtro JWT
---

#### [ERR-09] `application.properties` prácticamente vacío

**Archivo:** `src/main/resources/application.properties`

**Problema:** El archivo de configuración solo contiene `spring.application.name=demo`. Faltan configuraciones críticas: puerto del servidor, configuración de base de datos, nivel de logging y perfiles de entorno (dev/prod).

**Corrección sugerida:**
```properties
spring.application.name=metrics-backend
server.port=8080

# Base de datos H2 (desarrollo)
spring.datasource.url=jdbc:h2:mem:metricsdb
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=create-drop
spring.h2.console.enabled=true

# Logging
logging.level.com.exampleback=DEBUG
```

---

#### [ERR-10] Nombre del proyecto y artefacto genérico

**Archivo:** `pom.xml`

**Problema:** El `artifactId` es `demo`, y los campos `<name/>` y `<description/>` están vacíos. Esto es el template generado por Spring Initializr sin personalizar.

**Corrección sugerida:**
```xml
<artifactId>metrics-backend</artifactId>
<name>Developer Metrics API</name>
<description>API REST para métricas de rendimiento de desarrolladores</description>
```

---

#### [ERR-11] El Controller no valida el parámetro de entrada

**Archivo:** `controller/MetricsController.java`

**Problema:** El `@PathVariable String metric` acepta cualquier valor sin validación. Una petición a `/metrics/cualquierCosa` no produce error, sino una respuesta con `value: 0` para todos los registros, lo cual es un comportamiento silencioso e incorrecto.

**Corrección sugerida:** Validar con una lista de valores permitidos o usar un `enum`. El error puede lanzarse en el Service y ser capturado por un `GlobalExceptionHandler`:
---

#### [ERR-12] No hay pruebas unitarias reales

**Archivo:** `DemoApplicationTests.java`

**Problema:** El único test existente es el generado por Spring Initializr por defecto (`contextLoads()`), que únicamente verifica que el contexto de Spring arranca. No hay pruebas para el Service, Repository ni Controller.

**Corrección sugerida:** Implementar al menos pruebas unitarias del Service con Mockito
---

## Resumen de Hallazgos

| ID | Categoría | Descripción | Severidad |
|---|---|---|---|
| ERR-01 | Arquitectura | Repository sin conexión real a BD, datos hardcodeados | Crítico |
| ERR-02 | JPA | Modelo sin anotaciones `@Entity` | Crítico |
| ERR-03 | Dependencias | Firebase, JPA y H2 declarados pero no usados | Significativo |
| ERR-04 | Lógica | Filtrado en Java en lugar de en consulta a BD | Significativo |
| ERR-05 | Datos | 4 registros estáticos para un solo desarrollador | Significativo |
| ERR-06 | Consistencia | Getters manuales en Model a pesar de tener Lombok | Significativo |
| ERR-07 | Robustez | Sin manejo global de excepciones | Mejora |
| ERR-08 | Seguridad | Spring Security importado pero completamente deshabilitado | Mejora |
| ERR-09 | Configuración | `application.properties` sin configuración real | Mejora |
| ERR-10 | Proyecto | Metadatos del pom.xml vacíos (nombre, descripción) | Mejora |
| ERR-11 | Validación | `@PathVariable` sin validación, falla silenciosamente | Mejora |
| ERR-12 | Testing | Sin pruebas unitarias reales | Mejora |

