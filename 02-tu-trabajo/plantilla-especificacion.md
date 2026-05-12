# Especificación Formal — Sistema de Préstamo de Libros

> **Autor:** Juan Fernando García Restrepo - Juan Sebastian Molina Escobar
> **Fecha:** 5 de mayo de 2026
> **Versión:** 1.0
> **Brief de origen:** Correo de Diana Restrepo, Coordinadora de Biblioteca

---

## 1. Propósito del sistema

Desarrollar una API para gestionar el ciclo completo de préstamos de libros en biblioteca (consulta de catálogo, préstamo, devolución, renovación, control de vencidos y multas), con datos en memoria y reglas de negocio claras para pregrado y posgrado.

---

## 2. Alcance

**Incluido en esta versión:**

- Consulta de catálogo de libros y disponibilidad por ejemplar.
- Creación de préstamos sobre ejemplares disponibles.
- Registro de devolución y cálculo automático de multa por mora.
- Renovación de préstamos bajo condición de no existir solicitudes de espera.
- Consulta de préstamos vigentes por estudiante.
- Consulta de préstamos vencidos.
- Consulta de historial de préstamos por estudiante.
- Consulta de multas pendientes por estudiante.
- Gestión básica de solicitudes de espera (cola) para soportar la regla de renovación.

**Explícitamente fuera del alcance:**

- Lógica de préstamos para profesores investigadores.
- Integración con app móvil o portal web.
- Persistencia en base de datos (solo memoria en esta versión).
- Autenticación y autorización.
- Flujo de pago de multas (solo registro y consulta de estado pendiente/pagada).
- Notificaciones por correo, SMS o push.

---

## 3. Modelo de datos

### Entidad: Libro

| Campo             | Tipo                          | Obligatorio | Descripción                                     |
| ----------------- | ----------------------------- | ----------- | ----------------------------------------------- |
| id                | string                        | sí          | Identificador interno único del libro.          |
| codigo_inventario | string                        | sí          | Código único de catálogo del libro.             |
| titulo            | string                        | sí          | Título del libro.                               |
| autor             | string                        | sí          | Autor principal del libro.                      |
| sala              | string                        | sí          | Sala/ubicación física en biblioteca.            |
| tipo_prestamo     | enum(`normal`,`alta_demanda`) | sí          | Define plazo base del préstamo.                 |
| activo            | boolean                       | sí          | Indica si el libro está disponible en catálogo. |

### Entidad: Ejemplar

| Campo             | Tipo                                                 | Obligatorio | Descripción                               |
| ----------------- | ---------------------------------------------------- | ----------- | ----------------------------------------- |
| id                | string                                               | sí          | Identificador interno único del ejemplar. |
| libro_id          | string                                               | sí          | Referencia al libro al que pertenece.     |
| estado            | enum(`disponible`,`prestado`,`mantenimiento`,`baja`) | sí          | Estado operativo del ejemplar.            |
| ubicacion_detalle | string                                               | no          | Estante o referencia interna.             |

### Entidad: Usuario

| Campo              | Tipo                                                               | Obligatorio | Descripción                        |
| ------------------ | ------------------------------------------------------------------ | ----------- | ---------------------------------- |
| id                 | string                                                             | sí          | Identificador interno del usuario. |
| codigo_estudiante? | string                                                             | sí          | Código único institucional.        |
| nombre             | string                                                             | sí          | Nombre completo del usuario.       |
| rol                | enum(`estudiante_pregrado`,`estudiante_posgrado`, `bibliotecario`) | sí          | Define límites de préstamo.        |
| estado             | enum(`activo`,`bloqueado`, `inactivo`)                             | sí          | Estado general para préstamos.     |

### Entidad: Préstamo

| Campo                     | Tipo                                           | Obligatorio | Descripción                                  |
| ------------------------- | ---------------------------------------------- | ----------- | -------------------------------------------- |
| id                        | string                                         | sí          | Identificador único del préstamo.            |
| usuario_id                | string                                         | sí          | Usuario que realiza el préstamo.             |
| ejemplar_id               | string                                         | sí          | Ejemplar prestado.                           |
| fecha_prestamo            | string (ISO 8601)                              | sí          | Fecha y hora de creación del préstamo.       |
| fecha_devolucion_esperada | string (ISO 8601)                              | sí          | Fecha límite calculada por regla de negocio. |
| fecha_devolucion_real     | string (ISO 8601)                              | no          | Fecha real de devolución.                    |
| estado                    | enum(`activo`,`vencido`,`devuelto`,`renovado`) | sí          | Estado actual del préstamo.                  |
| renovaciones              | number                                         | sí          | Cantidad de renovaciones aplicadas.          |

### Entidad: Multa

| Campo            | Tipo                       | Obligatorio | Descripción                      |
| ---------------- | -------------------------- | ----------- | -------------------------------- |
| id               | string                     | sí          | Identificador único de la multa. |
| prestamo_id      | string                     | sí          | Préstamo que originó la multa.   |
| usuario_id       | string                     | sí          | Usuario responsable de la multa. |
| dias_retraso     | number                     | sí          | Días de retraso al devolver.     |
| valor_por_dia    | number                     | sí          | Valor fijo por día (2000).       |
| valor_total      | number                     | sí          | Total calculado de multa.        |
| estado           | enum(`pendiente`,`pagada`) | sí          | Estado de pago de la multa.      |
| fecha_generacion | string                     | sí          | Fecha de creación de la multa.   |

### Entidad: SolicitudEspera

| Campo           | Tipo                                  | Obligatorio | Descripción                        |
| --------------- | ------------------------------------- | ----------- | ---------------------------------- |
| id              | string                                | sí          | Identificador único de solicitud.  |
| libro_id        | string                                | sí          | Libro que el estudiante desea.     |
| usuario_id      | string                                | sí          | Usuario solicitante.               |
| fecha_solicitud | string (ISO 8601)                     | sí          | Fecha de creación de la solicitud. |
| estado          | enum(`activa`,`atendida`,`cancelada`) | sí          | Estado de la solicitud.            |

### Diagrama de relaciones

| Entidad Origen | Relación   | Entidad Destino | Notas                                                            |
| -------------- | ---------- | --------------- | ---------------------------------------------------------------- |
| Libro          | 1 --- N    | Ejemplar        | Un libro puede tener múltiples copias físicas.                   |
| Estudiante     | 1 --- N    | Préstamo        | Un estudiante puede tener varios préstamos activos o históricos. |
| Ejemplar       | 1 --- N    | Préstamo        | Historial de préstamos por cada copia física (no simultáneos).   |
| Préstamo       | 1 --- 0..1 | Multa           | Una multa se origina solo si hay mora en un préstamo.            |
| Libro          | 1 --- N    | SolicitudEspera | Un libro puede tener varios estudiantes en cola.                 |
| Estudiante     | 1 --- N    | SolicitudEspera | Un estudiante puede esperar por varios libros.                   |

---

## 4. Endpoints REST

| Método  | Ruta                                   | Propósito                                          | Body / Query                                            | Respuesta éxito                        | Códigos error posibles |
| ------- | -------------------------------------- | -------------------------------------------------- | ------------------------------------------------------- | -------------------------------------- | ---------------------- |
| `GET`   | `/libros`                              | Listar catálogo de libros                          | Query opcional: `titulo`, `autor`, `sala`, `disponible` | `200` con lista                        | `400`                  |
| `GET`   | `/libros/:id`                          | Consultar detalle de libro                         | -                                                       | `200` con objeto                       | `404`                  |
| `GET`   | `/libros/:id/ejemplares`               | Consultar ejemplares y estado por libro            | -                                                       | `200` con lista                        | `404`                  |
| `GET`   | `/estudiantes/:id/prestamos-activos`   | Ver préstamos vigentes de estudiante               | -                                                       | `200` con lista                        | `404`                  |
| `GET`   | `/estudiantes/:id/historial-prestamos` | Ver historial completo de préstamos                | Query opcional: `estado`, `desde`, `hasta`              | `200` con lista                        | `404`                  |
| `GET`   | `/estudiantes/:id/multas`              | Ver multas de estudiante                           | Query opcional: `estado`                                | `200` con lista                        | `404`                  |
| `POST`  | `/prestamos`                           | Crear préstamo de ejemplar                         | `{estudiante_id, ejemplar_id}`                          | `201` con préstamo                     | `400`, `404`, `409`    |
| `PATCH` | `/prestamos/:id/renovar`               | Renovar préstamo activo                            | `{motivo}` opcional                                     | `200` con préstamo actualizado         | `400`, `404`, `409`    |
| `PATCH` | `/prestamos/:id/devolver`              | Registrar devolución y calcular multa si aplica    | `{fecha_devolucion_real}` opcional                      | `200` con préstamo y multa (si aplica) | `400`, `404`, `409`    |
| `GET`   | `/prestamos/vencidos`                  | Consultar préstamos vencidos                       | Query opcional: `estudiante_id`                         | `200` con lista                        | `400`                  |
| `POST`  | `/solicitudes-espera`                  | Registrar estudiante en cola de espera de un libro | `{estudiante_id, libro_id}`                             | `201` con solicitud                    | `400`, `404`, `409`    |

---

## 5. Reglas de negocio

### RN1 — Límite de préstamos por tipo de estudiante

- **Trigger:** al recibir `POST /prestamos`.
- **Condición:**
  - Estudiante de pregrado: máximo 3 préstamos con `estado = "activo"`.
  - Estudiante de posgrado: máximo 5 préstamos con `estado = "activo"`.
- **Acción si cumple:** continuar con el flujo de creación.
- **Acción si no cumple:** retornar `409 Conflict` con `{error: "limite_prestamos_alcanzado", limite: N, actuales: M}`.

### RN2 — Bloqueo por préstamos vencidos

- **Trigger:** al recibir `POST /prestamos`.
- **Condición:** el estudiante no debe tener préstamos con `estado = "vencido"` o activos con `fecha_devolucion_esperada < ahora`.
- **Acción si cumple:** continuar con el flujo.
- **Acción si no cumple:** retornar `409 Conflict` con `{error: "estudiante_con_prestamo_vencido"}`.

### RN3 — Bloqueo por multas pendientes

- **Trigger:** al recibir `POST /prestamos`.
- **Condición:** el estudiante no debe tener multas con `estado = "pendiente"`.
- **Acción si cumple:** continuar con el flujo.
- **Acción si no cumple:** retornar `409 Conflict` con `{error: "multas_pendientes"}`.

### RN4 — Disponibilidad de ejemplar

- **Trigger:** al recibir `POST /prestamos`.
- **Condición:** el ejemplar debe existir y estar en estado `disponible`.
- **Acción si cumple:** crear préstamo y cambiar estado del ejemplar a `prestado`.
- **Acción si no cumple:** retornar `409 Conflict` con `{error: "ejemplar_no_disponible"}`.

### RN5 — Plazo del préstamo según tipo de libro

- **Trigger:** al crear préstamo o renovar (`POST /prestamos`, `PATCH /prestamos/:id/renovar`).
- **Condición:**
  - Si `libro.tipo_prestamo = "normal"`: plazo de 15 días calendario.
  - Si `libro.tipo_prestamo = "alta_demanda"`: plazo de 3 días calendario.
- **Acción si cumple:** calcular `fecha_devolucion_esperada` según regla.
- **Acción si no cumple:** retornar `400 Bad Request` con `{error: "tipo_prestamo_invalido"}`.

### RN6 — Renovación condicionada por solicitudes de espera

- **Trigger:** al recibir `PATCH /prestamos/:id/renovar`.
- **Condición:** no debe existir ninguna `SolicitudEspera` activa para el libro asociado al préstamo.
- **Acción si cumple:** extender `fecha_devolucion_esperada` según RN5 e incrementar `renovaciones`.
- **Acción si no cumple:** retornar `409 Conflict` con `{error: "libro_con_solicitudes_en_espera"}`.

### RN7 — Devolución y cierre de préstamo

- **Trigger:** al recibir `PATCH /prestamos/:id/devolver`.
- **Condición:** el préstamo debe existir y estar en estado `activo` o `vencido`.
- **Acción si cumple:** registrar `fecha_devolucion_real`, cambiar estado del préstamo a `devuelto` y estado del ejemplar a `disponible`.
- **Acción si no cumple:** retornar `409 Conflict` con `{error: "prestamo_no_devoluble"}`.

### RN8 — Cálculo automático de multa por mora

- **Trigger:** durante `PATCH /prestamos/:id/devolver`.
- **Condición:** si `fecha_devolucion_real > fecha_devolucion_esperada`, hay mora.
- **Acción si cumple:** crear multa con `dias_retraso = floor(diferencia_dias)` y `valor_total = dias_retraso * 2000`.
- **Acción si no cumple:** no crear multa.

### RN9 — Préstamos vencidos

- **Trigger:** al consultar `GET /prestamos/vencidos` o al listar préstamos activos.
- **Condición:** un préstamo está vencido si no tiene `fecha_devolucion_real` y `fecha_devolucion_esperada < ahora`.
- **Acción si cumple:** marcar/retornar como `vencido`.
- **Acción si no cumple:** mantener estado actual.

---

## 6. Decisiones tomadas (lo que el correo no dice)

### D1 — Cálculo de días para multa

- **Contexto:** el correo no precisa si los días de retraso son calendario o hábiles.
- **Decisión:** usar días calendario.
- **Justificación:** es la interpretación más simple y se alinea con lo que la mayoría de bibliotecas hacen.

### D2 — Prioridad de regla de límite por tipo de estudiante

- **Contexto:** el correo primero menciona máximo 3 libros, pero luego especifica 3 para pregrado y 5 para posgrado.
- **Decisión:** aplicar límite por tipo (`pregrado=3`, `posgrado=5`) como fuente final.
- **Justificación:** la sección detallada de tipos de estudiante aporta mayor precisión operativa.

### D3 — Manejo de solicitudes de espera para renovación

- **Contexto:** se exige negar renovación cuando otro estudiante está esperando, pero no se define estructura para representar esa espera.
- **Decisión:** crear entidad `SolicitudEspera` en cola por libro.
- **Justificación:** permite validar renovación de forma determinística y auditable.

### D4 — Gestión del estado de renovación

- **Contexto:** el correo no define cómo distinguir un préstamo renovado en el historial.
- **Decisión:** incluir campo `renovaciones` y permitir estado `renovado` como traza de operación.
- **Justificación:** facilita reportes sin duplicar registros de préstamo.

### D5 — Actualización automática a vencido

- **Contexto:** no se especifica si el estado `vencido` se actualiza por proceso batch o en consulta.
- **Decisión:** calcular vencimiento en lectura (consulta) y reflejar estado vencido sin job programado.
- **Justificación:** simplifica una versión en memoria y evita dependencias de scheduler.

### D6 — Alcance del manejo de multas

- **Contexto:** el correo habla de multas pendientes sin detallar su pago.
- **Decisión:** exponer consulta de multas y estado (`pendiente`/`pagada`), pero no implementar pasarela o proceso de recaudo.
- **Justificación:** mantiene coherencia con una primera versión enfocada en préstamos.

---

## 7. Códigos HTTP usados

| Código | Significado           | Cuándo se usa                                                                          |
| ------ | --------------------- | -------------------------------------------------------------------------------------- |
| 200    | OK                    | GET exitosos y PATCH exitosos                                                          |
| 201    | Created               | POST exitosos que crean recursos                                                       |
| 400    | Bad Request           | Body malformado o validación de formato fallida                                        |
| 404    | Not Found             | Recurso no existe                                                                      |
| 409    | Conflict              | Reglas de negocio violadas (límite alcanzado, mora pendiente, no disponibilidad, etc.) |
| 422    | Unprocessable Entity  | Formato válido pero semántica inválida (opcional en validación avanzada)               |
| 500    | Internal Server Error | Error no controlado del servidor                                                       |

---

## 8. Restricciones técnicas

- **Stack:** Node.js + Express
- **Persistencia:** datos en memoria. No usar base de datos.
- **TypeScript**
- **Sin autenticación** en esta versión.
- **Sin frontend** en esta versión. Solo API REST.
