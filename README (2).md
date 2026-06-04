# Análisis Técnico — Proyecto Final Programación Web

> **Repositorio Frontend:** [gitFinalFrontProgramacionWeb](https://github.com/franciscogomez2722/gitFinalFrontProgramacionWeb)
> **Repositorio Backend:** `backFinalProgramacionWeb-master`
> **Stack:** React + Vite (Frontend) · Spring Boot 4.0.6 / Java 21 (Backend)

---

## Índice

1. [Análisis del Frontend](#parte-1-análisis-del-frontend)
   - [Estructura de Carpetas](#1-estructura-de-carpetas)
   - [Componentes Principales](#2-componentes-principales)
   - [Manejo del Estado con Hooks](#3-manejo-del-estado-con-hooks)
   - [Consumo de APIs](#4-consumo-de-apis)
   - [Flujo de Datos entre Componentes](#5-flujo-de-datos-entre-componentes)
   - [Implementación de Gráficas](#6-implementación-de-gráficas-y-visualizaciones)
   - [Áreas de Mejora — Frontend](#7-áreas-de-mejora--frontend)
2. [Análisis del Backend](#parte-2-análisis-del-backend-spring-boot)
   - [Estructura General del Proyecto](#1-estructura-general-del-proyecto)
   - [Función de las Capas](#2-función-de-las-capas)
   - [Configuración de Seguridad y CORS](#3-configuración-de-seguridad-y-cors)
   - [Identificación de Mejoras — Backend](#4-identificación-de-mejoras--backend)

---

# Parte 1: Análisis del Frontend

## 1. Estructura de Carpetas

El proyecto Frontend está construido con **React + Vite** y su puerto de desarrollo predeterminado es `http://localhost:5173`. La estructura sigue las convenciones estándar de un proyecto Vite con separación por responsabilidad:

```
gitFinalFrontProgramacionWeb/
├── productivity-dashboard/          ← Proyecto Vite/React principal
│   ├── src/
│   │   ├── components/
│   │   │   └── Dashboard.jsx        ← Componente principal de visualización
│   │   ├── services/
│   │   │   └── metricsService.js    ← Capa de acceso a la API REST
│   │   ├── App.jsx                  ← Componente raíz, enrutamiento
│   │   ├── main.jsx                 ← Punto de entrada de la aplicación
│   │   ├── App.css                  ← Estilos globales de la aplicación
│   │   └── index.css                ← Estilos base / reset
│   ├── index.html                   ← HTML raíz (plantilla Vite)
│   ├── vite.config.js               ← Configuración del bundler
│   └── package.json                 ← Dependencias y scripts
├── node_modules/                    ← Dependencias instaladas (NO debería estar en git)
├── package.json                     ← package.json raíz
└── package-lock.json
```

> **Nota:** `node_modules/` está incluida en el repositorio, lo cual es una mala práctica. Debe agregarse al `.gitignore`.

### Distribución de lenguajes

| Lenguaje | Porcentaje | Uso principal |
|---|---|---|
| CSS | 50.8 % | Estilos de componentes y modo oscuro |
| JavaScript | 45.4 % | Lógica React, hooks, servicios |
| HTML | 3.8 % | Plantilla `index.html` de Vite |

---

## 2. Componentes Principales

### Diagrama de Componentes

```
┌────────────────────────────────────────────────────┐
│                     App.jsx                        │
│  (Componente raíz — routing y layout global)       │
│                         │                          │
│                         ▼                          │
│              ┌──────────────────┐                  │
│              │   Dashboard.jsx  │                  │
│              │  ┌────────────┐  │                  │
│              │  │  useState  │  │  ← estado local  │
│              │  │  useEffect │  │  ← carga datos   │
│              │  └────────────┘  │                  │
│              │         │        │                  │
│              │    metricsService│                  │
│              │    .getMetrics() │                  │
│              └──────────────────┘                  │
│                         │                          │
│              ┌──────────┴──────────┐               │
│              ▼                     ▼               │
│        [Tarjetas de           [Gráfica de          │
│         métricas]              líneas/barras]       │
└────────────────────────────────────────────────────┘
```

### `App.jsx` — Componente raíz

Punto de entrada visual de la aplicación. Contiene el layout general y renderiza `Dashboard.jsx`. No tiene estado propio ni lógica de negocio.

### `Dashboard.jsx` — Componente central

Es el componente más relevante de la aplicación. Sus responsabilidades incluidas:

- Declarar el estado de los datos de métricas (`useState`)
- Disparar la llamada a la API al montarse (`useEffect`)
- Calcular el valor promedio con `.toFixed(1)` (retorna `string`)
- Renderizar las tarjetas de resumen y la gráfica
- Manejar (parcialmente) los estados de carga y error

### `metricsService.js` — Capa de servicio

Abstrae las llamadas HTTP al backend. Centraliza la `API_URL` (actualmente hardcodeada a `http://localhost:8080`) y expone funciones `async/await` que `Dashboard.jsx` consume.

---

## 3. Manejo del Estado con Hooks

El proyecto utiliza los hooks nativos de React sin ninguna librería de gestión de estado global (sin Redux, sin Zustand, sin Context).

### `useState`

```jsx
// En Dashboard.jsx
const [metricsData, setMetricsData] = useState([]);
const [loading, setLoading]         = useState(true);
const [selectedMetric, setSelectedMetric] = useState('commits');
```

| Estado | Tipo inicial | Propósito |
|---|---|---|
| `metricsData` | `[]` | Almacena el array de `{label, value}` devuelto por la API |
| `loading` | `true` | Controla el indicador de carga mientras se obtienen datos |
| `selectedMetric` | `'commits'` | Define qué tipo de métrica se visualiza actualmente |

### `useEffect`

```jsx
useEffect(() => {
  const fetchData = async () => {
    try {
      const data = await getMetrics(selectedMetric);
      setMetricsData(data);
    } catch (error) {
      console.error(error); // ← Sin estado de error para el usuario
    } finally {
      setLoading(false);
    }
  };
  fetchData();
}, [selectedMetric]);
```

El efecto se re-ejecuta cada vez que `selectedMetric` cambia, lo que permite al usuario cambiar entre tipos de métricas (commits, bugs, tasks, storyPoints) y ver los datos actualizados.

### Diagrama de ciclo de vida del estado

```
Montaje del componente
        │
        ▼
useState inicializa: metricsData=[], loading=true
        │
        ▼
useEffect dispara fetchData()
        │
   ┌────┴──────────────┐
   ▼                   ▼
 éxito                error
   │                   │
setMetricsData(data)  console.error()  ← sin UI de error
   │
   ▼
setLoading(false)
   │
   ▼
Re-render: gráfica + tarjetas con datos
```

---

## 4. Consumo de APIs

### `metricsService.js`

```javascript
// URL hardcodeada — debería venir de .env
const API_URL = 'http://localhost:8080';

export const getMetrics = async (metric) => {
  const response = await fetch(`${API_URL}/metrics/${metric}`);
  return response.json();
};
```

### Endpoint consumido

| Método | Ruta | Parámetro | Respuesta |
|---|---|---|---|
| `GET` | `/metrics/{metric}` | `commits` \| `bugs` \| `tasks` \| `storyPoints` | `[{ label: "2026-05-01", value: 12 }, ...]` |

### Flujo de la petición HTTP

```
Usuario selecciona métrica
        │
        ▼
Dashboard.jsx actualiza selectedMetric (useState)
        │
        ▼
useEffect detecta cambio → llama getMetrics(selectedMetric)
        │
        ▼
metricsService.js → fetch('http://localhost:8080/metrics/commits')
        │
        ▼
Backend Spring Boot responde con JSON
        │
        ▼
setMetricsData(data) → Re-render de gráfica
```

---

## 5. Flujo de Datos entre Componentes

El flujo de datos es **unidireccional** (top-down), siguiendo el patrón React estándar. Toda la lógica reside en `Dashboard.jsx`, que actúa como contenedor inteligente (_smart component_).

```
metricsService.js
     │  retorna Promise<Array<{label, value}>>
     ▼
Dashboard.jsx  (estado + lógica)
     │
     ├──── metricsData ────────────────────► <LineChart /> o <BarChart />
     │                                         (recibe data como prop)
     │
     ├──── total / promedio ──────────────► <TarjetaResumen />
     │     (calculado inline)                  (recibe valores como props)
     │
     └──── selectedMetric ────────────────► <SelectorDeMétrica />
           (controlado por useState)           (onChange actualiza estado)
```

No existe comunicación entre componentes hermanos (sibling-to-sibling); todo pasa por `Dashboard.jsx` como intermediario.

---

## 6. Implementación de Gráficas y Visualizaciones

La aplicación utiliza una librería de gráficas de React (compatible con el ecosistema Recharts o Chart.js vía react-chartjs-2) para renderizar una visualización de líneas o barras con los datos históricos de métricas.

### Datos pasados a la gráfica

```javascript
// Transformación en Dashboard.jsx
const chartData = metricsData.map(item => ({
  name: item.label,    // fecha formateada desde el backend
  value: item.value,   // valor numérico de la métrica
}));
```

### Cálculo de totales

```javascript
// Ejemplo del cálculo de promedio (retorna string, no number)
const promedio = (total / metricsData.length).toFixed(1);
//                                              ↑
//                  .toFixed() convierte a string — posible NaN si length = 0
```

### Visualización actual

La gráfica muestra la evolución temporal de una métrica seleccionada (commits, bugs resueltos, tareas completadas o story points) para el desarrollador "Francisco" durante 4 días de Mayo 2026. No tiene leyenda vertical, ni histograma, ni filtro por usuario.

---

## 7. Áreas de Mejora — Frontend

A continuación se listan los problemas y mejoras identificados en el proyecto frontend, organizados por impacto.

---

### [FRONT-01] Ausencia de barra de navegación principal

**Descripción:** La aplicación no tiene navegación entre secciones. Se requiere una navbar con acceso directo a las vistas: **Books**, **Commits**, **Tasks** y **Entrypoints**.

**Corrección sugerida:**
```jsx
// NavBar.jsx
const NavBar = () => (
  <nav>
    <Link to="/books">Books</Link>
    <Link to="/commits">Commits</Link>
    <Link to="/tasks">Tasks</Link>
    <Link to="/entrypoints">Entrypoints</Link>
  </nav>
);
```
Requiere instalar `react-router-dom` si no está incluido.

---

### [FRONT-02] No hay filtro por usuario

**Descripción:** La página solo muestra métricas para el usuario "Francisco" (hardcodeado en el backend). No existe ningún selector o filtro para cambiar de desarrollador. Cuando el backend soporte múltiples usuarios, el frontend debe exponer este control.

**Corrección sugerida:** Agregar un `<select>` controlado con `useState` que pase el nombre del desarrollador como parámetro a la llamada de API.

```jsx
const [developer, setDeveloper] = useState('Francisco');

// En la llamada al servicio:
const data = await getMetrics(selectedMetric, developer);
```

---

### [FRONT-03] Modo oscuro — texto ilegible

**Descripción:** En modo oscuro, el texto no contrasta con el fondo, haciendo la interfaz completamente ilegible. Los colores de texto y fondo no están adaptados para ambos temas.

**Corrección sugerida:**
```css
/* index.css */
@media (prefers-color-scheme: dark) {
  :root {
    --color-text: #e2e8f0;
    --color-bg: #1a202c;
    --color-card: #2d3748;
  }
}
```
O implementar un toggle de tema con CSS variables y `data-theme` en el `<html>`.

---

### [FRONT-04] Gráfica sin leyenda vertical — usar histograma

**Descripción:** La gráfica actual no tiene etiqueta en el eje Y (leyenda vertical), lo que impide saber la unidad de la métrica. Además, para datos de conteo por fecha, un **histograma (gráfica de barras)** es más apropiado que una gráfica de líneas.

**Corrección sugerida con Recharts:**
```jsx
<BarChart data={chartData}>
  <XAxis dataKey="name" />
  <YAxis label={{ value: 'Cantidad', angle: -90, position: 'insideLeft' }} />
  <Tooltip />
  <Legend />
  <Bar dataKey="value" fill="#4A90E2" />
</BarChart>
```

---

### [FRONT-05] Títulos poco descriptivos

**Descripción:** Los títulos de las tarjetas y la gráfica son genéricos (ej. "Commits", "Total"). No comunican el período analizado, el desarrollador, ni la unidad de medida.

**Corrección sugerida:**
- Antes: `"Commits"`
- Después: `"Commits — Francisco · Mayo 2026"`
- Antes: `"Total"`
- Después: `"Total de commits en el período"`

---

### [FRONT-06] `API_URL` hardcodeada

**Archivo:** `services/metricsService.js`

**Descripción:** La URL del backend está fija como `http://localhost:8080`. Esto impide desplegar el frontend apuntando a otro servidor sin modificar el código fuente.

**Corrección sugerida:**
```javascript
// .env
VITE_API_URL=http://localhost:8080

// metricsService.js
const API_URL = import.meta.env.VITE_API_URL;
```

---

### [FRONT-07] Promedio calculado como `string`

**Archivo:** `Dashboard.jsx`

**Descripción:** `.toFixed(1)` convierte el número a `string`, lo que puede generar comportamientos inesperados si ese valor se usa en operaciones posteriores. Además, si `metricsData.length` es `0`, el resultado es `NaN`.

**Corrección sugerida:**
```javascript
// Proteger contra división por cero y mantener tipo número
const promedio = metricsData.length > 0
  ? parseFloat((total / metricsData.length).toFixed(1))
  : 0;
```

---

### [FRONT-08] Sin estado de error visible para el usuario

**Archivo:** `Dashboard.jsx`

**Descripción:** Cuando la llamada a la API falla, solo se ejecuta `console.error(error)`. El usuario no recibe ninguna indicación visual de que algo salió mal.

**Corrección sugerida:**
```jsx
const [error, setError] = useState(null);

// En el catch:
} catch (err) {
  setError('No se pudieron cargar las métricas. Intenta nuevamente.');
}

// En el render:
{error && <div className="error-banner">{error}</div>}
```

---

### [FRONT-09] `node_modules` incluido en el repositorio

**Descripción:** La carpeta `node_modules/` está versionada en Git. Esto incrementa el tamaño del repositorio innecesariamente y puede causar conflictos entre sistemas operativos.

**Corrección sugerida:**
```
# .gitignore
node_modules/
dist/
.env
.env.local
```

---

### [FRONT-10] Ausencia de ESLint y Prettier

**Descripción:** No hay configuración de linting ni formateo de código. Esto genera inconsistencias de estilo y dificulta detectar errores en tiempo de desarrollo.

**Corrección sugerida:**
```bash
npm install -D eslint prettier eslint-plugin-react eslint-config-prettier
npx eslint --init
```

---

### Resumen de Mejoras — Frontend

| ID | Área | Descripción | Prioridad |
|---|---|---|---|
| FRONT-01 | UX / Navegación | Agregar navbar con Books, Commits, Tasks, Entrypoints | Alta |
| FRONT-02 | Funcionalidad | Filtro por desarrollador/usuario | Alta |
| FRONT-03 | Accesibilidad | Corregir modo oscuro (texto ilegible) | Alta |
| FRONT-04 | Visualización | Cambiar a histograma con leyenda vertical en eje Y | Media |
| FRONT-05 | UX / Claridad | Títulos más descriptivos (usuario + período + unidad) | Media |
| FRONT-06 | Configuración | Mover `API_URL` a variable de entorno `.env` | Media |
| FRONT-07 | Robustez | Proteger cálculo de promedio contra NaN y tipo string | Media |
| FRONT-08 | UX | Mostrar estado de error visible al usuario | Media |
| FRONT-09 | Repositorio | Agregar `node_modules/` al `.gitignore` | Baja |
| FRONT-10 | Calidad | Configurar ESLint + Prettier | Baja |

---
---

# Parte 2: Análisis del Backend Spring Boot

### Proyecto: `Backend de Repo` · Programación Web

---

## Índice

1. [Estructura General del Proyecto](#1-estructura-general-del-proyecto)
2. [Función de las Capas](#2-función-de-las-capas)
3. [Configuración de Seguridad y CORS](#3-configuración-de-seguridad-y-cors)
4. [Identificación de Mejoras — Backend](#4-identificación-de-mejoras--backend)

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

## 4. Identificación de Mejoras — Backend

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

**Corrección sugerida:** Agregar anotaciones JPA y usar Lombok para eliminar el código manual de getters/constructor.

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

**Corrección sugerida:** Si no se va a usar autenticación en esta fase, eliminar la dependencia de Spring Security para no dar una falsa sensación de seguridad. Si se quiere implementar, agregar un filtro JWT.

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

**Corrección sugerida:** Validar con una lista de valores permitidos o usar un `enum`. El error puede lanzarse en el Service y ser capturado por un `GlobalExceptionHandler`.

---

#### [ERR-12] No hay pruebas unitarias reales

**Archivo:** `DemoApplicationTests.java`

**Problema:** El único test existente es el generado por Spring Initializr por defecto (`contextLoads()`), que únicamente verifica que el contexto de Spring arranca. No hay pruebas para el Service, Repository ni Controller.

**Corrección sugerida:** Implementar al menos pruebas unitarias del Service con Mockito.

---

## Resumen de Hallazgos — Backend

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

---

*Documento generado como entregable de análisis técnico — Programación Web*
