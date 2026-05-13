# Especificación Formal — Sistema de Préstamo de Libros

> **Autor:** Daner Alejandro Salazar Colorado, Jaime Andrés Cardona Díaz
> **Fecha:** 2026-05-12
> **Versión:** 1.0
> **Brief de origen:** Correo de Diana Restrepo, Coordinadora de Biblioteca

> Lo que estaba entre corchetes `[...]` en la plantilla original ha sido completado con especificaciones técnicas precisas. Se ha respetado la estructura de tablas y nombres de campos originales.

---

## 1. Propósito del sistema

Gestionar el préstamo y la devolución de ejemplares de la Biblioteca Universidad de Caldas mediante una API REST que permita: consulta de catálogo y disponibilidad, crear préstamos, registrar devoluciones, gestionar renovaciones, administrar colas de solicitud (holds), calcular y registrar multas por retraso, y exponer historial de préstamos por estudiante. Esta versión mantiene los datos en memoria (implementación temporal) para entrega rápida.

---

## 2. Alcance

**Incluido en esta versión:**

- Listado y filtrado del catálogo de libros (paginación y filtros por disponibilidad y alta_demanda).
- Gestión administrativa de libros y ejemplares (endpoints CRUD para pruebas y administración).
- Creación de préstamos para ejemplares disponibles y marcación de ejemplar como prestado.
- Registro de devoluciones con cálculo automático de multas si aplica.
- Renovaciones de préstamos con restricciones (renovación única por préstamo salvo excepción administrativa).
- Sistema de solicitudes/colas de espera (holds) por libro y asignación automática al devolver.
- Consultas de préstamos activos y historial completo por estudiante.
- Registro y gestión de multas (crear, listar, marcar como pagada).


**Explícitamente fuera del alcance:**

- Integración con pasarelas de pago para multas (el pago se marca manualmente en esta versión).
- Flujos avanzados para profesores o investigadores (solo se cubren pregrado y posgrado según el brief).
- Autenticación y autorización (sin login en esta versión).
- Persistencia en base de datos; todos los datos serán temporales en memoria.

---

## 3. Modelo de datos

### Entidad: Libro

| Campo     | Tipo     | Obligatorio | Descripción   |
| `id` | `int` | sí      | Identificador único interno del libro (PK, auto-incremental en in-memory store). |
| `codigo_inventario` | `string` | sí      | Código de inventario único asignado al título (puede agrupar varios ejemplares). |
| `name` | `string` | sí      | Título del libro. |
| `author` | `string` | sí      | Autor(es) del libro, como cadena simple. |
| `location` | `string` | no      | Ubicación física (p.ej. "Sala A", "Estantería 3"). |
| `alta_demanda` | `boolean` | no (false) | Indica si el libro es de alta demanda (reserva, plazo corto). |


### Entidad: Ejemplar

Cada libro puede tener varios ejemplares físicos. Un ejemplar representa una copia identificable con su propio código.

| Campo     | Tipo     | Obligatorio | Descripción   |
| `id` | `int` | sí      | Identificador único del ejemplar (PK). |
| `code` | `string` | sí      | Código de barras o código físico del ejemplar (debe ser único). |
| `book_id` | `int` | sí      | FK -> Libro.id. Relación al título al que pertenece el ejemplar. |
| `state` | `string` | sí      | Estado actual: `disponible`, `prestado`, `reservado`, `perdido`. |
| `notas` | `string` | no      | Observaciones del ejemplar (dañado, etc.). |

### Entidad: Estudiante

Representa al usuario (estudiante) que puede solicitar préstamos.

| Campo     | Tipo     | Obligatorio | Descripción   |
| `id` | `int` | sí      | Identificador único del estudiante (PK). |
| `code` | `string` | sí      | Código único del estudiante (carné). |
| `name` | `string` | sí      | Nombre completo. |
| `program_id` | `int` | sí      | FK -> Programa.id. Programa académico del estudiante. |
| `tipo` | `string` | sí      | Tipo: `pregrado` o `posgrado`. |
| `multas_pendientes` | `int` | sí (0) | Suma en COP de multas pendientes (derivable). |

### Entidad: Programa

| Campo     | Tipo     | Obligatorio | Descripción   |
| `id` | `int` | sí      | Identificador único del programa (PK). |
| `code` | `string` | sí      | Código corto del programa. |
| `name` | `string` | sí      | Nombre legible del programa. |


### Entidad: Préstamo

Registra la acción de prestar un ejemplar a un estudiante.

| Campo     | Tipo     | Obligatorio | Descripción   |
| `id` | `int` | sí      | Identificador único del préstamo (PK). |
| `student_id` | `int` | sí      | FK -> Estudiante.id. |
| `copy_id` | `int` | sí      | FK -> Ejemplar.id (ejemplar prestado). |
| `fecha_prestamo` | `date` | sí      | Fecha de inicio del préstamo (YYYY-MM-DD). |
| `fecha_devolucion_esperada` | `date` | sí      | Fecha calculada de devolución esperada. |
| `fecha_devolucion_real` | `date` | no      | Fecha en que se devolvió realmente (si aplica). |
| `estado` | `string` | sí      | `activo`, `devuelto`, `vencido`, `cancelado`. |
| `renovaciones` | `int` | sí (0) | Contador de renovaciones aplicadas. |

### Entidad: Multa

| Campo     | Tipo     | Obligatorio | Descripción   |
| `id` | `int` | sí      | Identificador único de la multa (PK). |
| `prestamo_id` | `int` | sí      | FK -> Préstamo.id. |
| `student_id` | `int` | sí      | FK -> Estudiante.id. |
| `dias_atraso` | `int` | sí      | Días calendario de retraso. |
| `monto` | `int` | sí      | Monto en COP (dias_atraso * 2000). |
| `pagada` | `boolean` | sí (false) | Indica si la multa fue pagada/conciliada. |
| `fecha_generada` | `date` | sí      | Fecha en que se creó la multa. |

### Diagrama de relaciones

```text
Libro 1 --- N Ejemplar
Ejemplar 1 --- N Prestamo (a lo largo del tiempo)
Estudiante 1 --- N Prestamo
Prestamo 0..1 --- 1 Multa
Libro 1 --- N Solicitud
Programa 1 --- N Estudiante

# Notas:
- "Libro" representa el título; "Ejemplar" es la copia física.
- Una "Solicitud" (hold) se relaciona con Libro (no con ejemplar) y tiene una posición en cola.
```

---

## 4. Endpoints REST

| Método | Ruta | Propósito | Body / Query | Respuesta éxito | Códigos error posibles |
|---|---|---|---|---|---|
| `GET` | `/libros` | Listar catálogo (filtros: q, disponible, alta_demanda, page, per_page) | query | `200` con lista y metadatos | `400` |
| `GET` | `/libros/:id` | Detalle de un libro con sus ejemplares | - | `200` con objeto | `404` |
| `POST` | `/libros` | Crear un nuevo libro (admin) | `{ codigo_inventario, name, author, sala, alta_demanda }` | `201` con libro | `400` |
| `POST` | `/ejemplares` | Crear ejemplar para un libro | `{ book_id, code }` | `201` con ejemplar | `400`, `404` |
| `GET` | `/ejemplares/:id` | Detalle del ejemplar | - | `200` | `404` |
| `POST` | `/prestamos` | Crear préstamo (prestar ejemplar) | `{ student_id, copy_id, fecha_prestamo? }` | `201` con préstamo | `400`, `404`, `409` |
| `POST` | `/prestamos/:id/renovar` | Renovar préstamo | - | `200` con préstamo actualizado | `400`, `404`, `409` |
| `POST` | `/devoluciones` | Registrar devolución y calcular multa si aplica | `{ prestamo_id, fecha_devolucion? }` | `200` con { prestamo, multa? } | `400`, `404` |
| `GET` | `/estudiantes/:id/prestamos` | Préstamos activos del estudiante | - | `200` | `404` |
| `GET` | `/estudiantes/:id/historial` | Historial completo de préstamos del estudiante | - | `200` | `404` |
| `POST` | `/solicitudes` | Crear solicitud/hold por libro | `{ student_id, book_id }` | `201` con solicitud | `400`, `404`, `409` |
| `GET` | `/prestamos/vencidos` | Listar préstamos vencidos (para notificaciones) | - | `200` con lista | `400` |

Nota: Los endpoints administrativos (`POST /libros`, `POST /ejemplares`) son opcionales para pruebas; en producción requerirían autenticación.

---

## 5. Reglas de negocio

### RN1 — Límite de préstamos por tipo de estudiante

- **Trigger:** al recibir `POST /prestamos`.
- **Condición:**
  - Estudiante `pregrado`: máximo 3 préstamos con `estado = "activo"`.
  - Estudiante `posgrado`: máximo 5 préstamos con `estado = "activo"`.
- **Acción si cumple:** crear el préstamo y marcar `ejemplar.state = "prestado"`.
- **Acción si no cumple:** retornar `409 Conflict` con `{ error: "limite_prestamos_alcanzado", limite: N, actuales: M }`.

### RN2 — Bloqueo por multas o préstamos vencidos

- **Trigger:** al recibir `POST /prestamos`.
- **Condición:** el estudiante no debe tener multas con `pagada = false` ni préstamos con `estado = "vencido"`.
- **Acción si cumple:** continuar con la creación del préstamo.
- **Acción si no cumple:** retornar `409 Conflict` con `{ error: "bloqueado_por_multas_o_vencidos" }`.

### RN3 — Duración del préstamo según tipo de libro

- **Trigger:** al crear un préstamo (`POST /prestamos`).
- **Condición:** verificar `libro.alta_demanda`.
- **Acción si cumple:** si `alta_demanda = true` => `plazo = 3` días; si no => `plazo = 15` días. Fijar `fecha_devolucion_esperada = fecha_prestamo + plazo`.
- **Acción si no cumple:** (no aplica; regla obligatoria para cálculo de fechas).

### RN4 — Disponibilidad del ejemplar

- **Trigger:** al crear préstamo.
- **Condición:** `ejemplar.state == "disponible"`.
- **Acción si cumple:** marcar ejemplar como `prestado` y crear préstamo.
- **Acción si no cumple:** retornar `409 Conflict` con `{ error: "ejemplar_no_disponible" }`.

### RN5 — Renovación de préstamo

- **Trigger:** `POST /prestamos/:id/renovar`.
- **Condición:**
  - El préstamo está en `estado = "activo"` y no está `vencido`.
  - No existen solicitudes activas (`Solicitud.estado = "activa"`) para el `book_id` del préstamo.
  - `renovaciones < MAX_RENOVACIONES` (decisión técnica: `MAX_RENOVACIONES = 1`).
- **Acción si cumple:** aumentar `fecha_devolucion_esperada` por el mismo plazo original y `renovaciones++`.
- **Acción si no cumple:** retornar `409 Conflict` con `{ error: "renovacion_no_permitida", motivo: "..." }`.

### RN6 — Asignación desde cola de solicitudes al devolver

- **Trigger:** `POST /devoluciones`.
- **Condición:** al devolver, existen solicitudes activas para el `book_id` del ejemplar devuelto.
- **Acción si cumple:** asignar ejemplar al primer solicitante: crear notificación (registro) y marcar `solicitud.estado = "satisfecha"`; marcar ejemplar `reservado` por 48 horas.
- **Acción si no cumple:** marcar ejemplar `disponible`.

### RN7 — Cálculo de multa por atraso

- **Trigger:** `POST /devoluciones`.
- **Condición:** `fecha_devolucion_real > fecha_devolucion_esperada`.
- **Acción si cumple:** `dias_atraso = fecha_devolucion_real - fecha_devolucion_esperada` (días calendario). `monto = dias_atraso * 2000`. Crear `Multa` con `pagada = false` y sumar al `multas_pendientes` del estudiante. Retornar detalle de multa en la respuesta.
- **Acción si no cumple:** no crear multa.

### RN8 — Un ejemplar solo puede tener un préstamo activo

- **Trigger:** `POST /prestamos`.
- **Condición:** el `ejemplar.state` no debe ser `prestado`.
- **Acción si cumple:** crear préstamo.
- **Acción si no cumple:** `409 Conflict` `{ error: "ejemplar_en_prestamo" }`.

### RN2 — [...]

[...]

### RN3 — [...]

[...]


---

## 6. Decisiones tomadas (lo que el correo no dice)

### D1 — Días para cálculo de multa

- **Contexto:** el brief no precisa si los días de retraso son hábiles o calendario.
- **Decisión:** usar días calendario para el cálculo de multas.
- **Justificación:** implementación simple, evita dependencias de calendarios laborales y coincide con la práctica común.

### D2 — Formato de fechas

- **Contexto:** no se especifica formato de fecha en el brief.
- **Decisión:** usar ISO 8601 (`YYYY-MM-DD`) para todas las fechas; para timestamps usar `YYYY-MM-DDTHH:MM:SSZ` (UTC) si se requiere.
- **Justificación:** interoperabilidad y facilidad de parsing.

### D3 — Identificadores y unicidad

- **Contexto:** la plantilla original tenía campos `id`, `code`, etc., sin detalles.
- **Decisión:** IDs enteros auto-incrementales en el store en memoria; `code`/`codigo` para ejemplares y `codigo_inventario` para libros deben ser únicos.
- **Justificación:** simplifica implementación en memoria y facilita pruebas.

### D4 — Renovaciones permitidas

- **Contexto:** el brief menciona renovación sin límites.
- **Decisión:** permitir 1 renovación por préstamo por defecto (`MAX_RENOVACIONES = 1`). Renovaciones adicionales requieren intervención administrativa.
- **Justificación:** evita préstamos indefinidos en la primera entrega y reduce casos límite en la versión rápida.

### D5 — Ventana para reclamar reservas

- **Contexto:** no hay una regla en el brief sobre cuánto tiempo un solicitante puede reclamar una copia reservada.
- **Decisión:** 48 horas para reclamar una copia reservada; si no reclama, pasar al siguiente solicitante.
- **Justificación:** equilibrio entre equidad y operativa diaria del mostrador.

### D6 — Multas y pagos

- **Contexto:** el brief solicita cálculo de multas pero no integración de pagos.
- **Decisión:** la API solo registrará multas y permitirá marcarlas como pagadas mediante `POST /multas/:id/pagar`. La conciliación real será manual en el mostrador.
- **Justificación:** cumplir la funcionalidad requerida sin retrasar la entrega por integración de pagos.

---

## 7. Códigos HTTP usados

| Código | Significado | Cuándo se usa |
|---|---|---|
| 200 | OK | GET exitosos |
| 201 | Created | POST exitosos que crean recursos |
| 400 | Bad Request | Body malformado o validación fallida |
| 404 | Not Found | Recurso no existe |
| 409 | Conflict | Reglas de negocio violadas (límite alcanzado, duplicado, etc.) |
| 500 | Internal Server Error | Error no controlado del servidor |

[Si se necesitan otros códigos se documentarán aquí (por ejemplo 422 para validaciones semánticas).]

---

## 8. Restricciones técnicas
- **Stack:** [Node.js + Express]
- **Persistencia:** datos en memoria. No usar base de datos.
- **TypeScript** (según tu stack).
- **Sin autenticación** en esta versión.
- **Sin frontend** en esta versión. Solo API REST.
- 
- **Arquitectura de Software:** Implementación obligatoria de Arquitectura Hexagonal (Ports and Adapters) y Arquitectura Limpia (Clean Architecture).
- **Principios de Diseño:** El código debe adherirse estrictamente a los principios SOLID y al principio KISS (Keep It Simple, Stupid), garantizando código modular, mantenible y sin sobreingeniería.
- **Separación de Responsabilidades:** El código debe estar desacoplado en capas claras:
  - `infrastructure/` (routes, controllers): Adaptadores de entrada para manejar peticiones HTTP y respuestas.
  - `application/` (services o use cases): Lógica de orquestación y validación de las reglas de negocio (RN1 a RN8).
  - `domain/`: Modelos, entidades y reglas puras.
  - `infrastructure/repositories/`: Adaptadores de salida para el manejo de los datos en memoria.
- **Inyección de Dependencias:** Obligatorio inyectar los repositorios (puertos) en los servicios para garantizar el desacoplamiento.
- **Manejo de Errores:** Centralizado a través de un middleware de Express en la capa de infraestructura.
