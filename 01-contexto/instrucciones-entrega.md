# Instrucciones de Entrega Final

Lo que vas a entregar son **6 archivos** y una carpeta de código. 

---

## 1. `especificacion.md`

Esto **no es un copy-paste del correo**. Es la traducción del correo a un documento técnico que cualquier desarrollador podría usar para construir el sistema sin hablar con la cliente.

Mínimo debe contener:

- **Modelo de datos:** entidades (libro, ejemplar, estudiante, préstamo, multa) con sus campos, tipos y relaciones.
- **Endpoints:** lista exacta de rutas REST con su método HTTP, parámetros, body esperado y respuesta esperada.
- **Reglas de negocio numeradas:** cada regla del correo convertida a una regla técnica con identificador (RN1, RN2...) y condiciones precisas.
- **Códigos HTTP** que vas a usar para cada caso (éxito, validación fallida, recurso no encontrado, conflicto).
- **Decisiones tomadas:** lista de decisiones que tomaste tú porque el correo no las especificaba. Por ejemplo: "El correo no dice cómo manejar la fecha; decidí usar ISO 8601 en UTC."
- **Preguntas pendientes para la cliente:** mínimo 3 preguntas que le harías.

Plantilla en `02-tu-trabajo/plantilla-especificacion.md`.

---

## 2. Carpeta `prompts/`

Cada vez que le pides algo a la IA, guardas el prompt en un archivo Markdown numerado:

```
prompts/
├── 01-generacion-inicial.md
├── 02-correccion-validacion-fechas.md
├── 03-tests-reglas-negocio.md
├── 04-correccion-bug-cupo.md
└── ...
```

Cada archivo debe tener:

- **Fecha y hora** del prompt.
- **Propósito** en una línea.
- **Prompt completo** que enviaste.
- **Resumen de la respuesta** de la IA (no la respuesta completa, solo qué hizo).
- **Tu evaluación:** ¿la respuesta fue útil? ¿la aceptaste tal cual? ¿la modificaste?

**El docente va a leer estos prompts.** Son la evidencia más clara de que pensaste antes de pedir.

Plantilla en `02-tu-trabajo/plantilla-prompts.md`.

---

## 3. Carpeta `proyecto/`

El código generado por la IA, organizado como cualquier proyecto profesional:

```
proyecto/
├── package.json (o requirements.txt, o pom.xml según tu stack)
├── app.js (o main.py, o equivalente)
├── modelos/
├── rutas/
└── ...
```

**Importante:** la carpeta `node_modules/` o equivalente no se entrega. Solo el código fuente y el archivo de dependencias.

El proyecto debe poder arrancarse con un comando estándar (`npm start`, `uvicorn main:app`, etc.) y debe estar **funcionando** al momento de la entrega.

---

## 4. Carpeta `tests/`

Los tests generados por la IA, ya ejecutados al menos una vez. Cuando los entregues, deben estar verdes (todos pasando contra tu versión final del código).

Si algún test queda rojo a propósito (porque documenta un bug que no alcanzaste a corregir), lo declaras en la `bitacora.md`.

---

## 5. `bitacora.md`

Tu cuaderno de campo durante el taller. Mínimo debe contener:

- **Hallazgos de la auditoría:** bugs detectados, omisiones, decisiones de la IA que cuestionaste. Cada hallazgo con archivo, línea aproximada, severidad y descripción.
- **Bugs corregidos:** cuáles bugs corregiste y cómo (con IA o a mano).
- **Bugs documentados pero no corregidos:** si quedó algo sin arreglar por tiempo, declaralo.
- **Aprendizajes:** 3 a 5 cosas que descubriste durante el ejercicio que no sabías al empezar.

Plantilla en `02-tu-trabajo/plantilla-bitacora.md`.

---

## 6. `diff-de-correcciones.md`

Una tabla mínima donde declaras qué cambios fueron tuyos a mano y cuáles le pediste a la IA:

| # | Archivo y línea | Cambio | ¿Por IA o por mí? | Razón |
|---|---|---|---|---|
| 1 | rutas/prestamos.js línea 45 | Cambio de `>=` a `>` en validación de fecha | Por mí | La IA no entendía la sutileza del boundary inclusivo |
| 2 | modelos/prestamo.js línea 80 | Agregar validación de tipo de libro | Por IA | Le pedí explícitamente con un prompt |

Esta tabla muestra que **no estás vibe-codeando ciegamente**: hay decisiones humanas conscientes en el flujo.

---

## 7. `reflexion-final.md`

Tres preguntas para responder en máximo 1 página:

1. **¿Cuál fue la decisión más difícil que tuviste que tomar al traducir el brief a la especificación?** Explica la decisión y por qué la tomaste así.

2. **Describe un momento en el que la IA generó algo plausible pero incorrecto.** ¿Cómo te diste cuenta? ¿Qué hiciste?

3. **Si mañana tu jefe te dice "no necesitamos QA, la IA ya genera tests", ¿qué le contestas?** Argumenta usando lo que aprendiste hoy. No respondas con generalidades; usa un ejemplo concreto del taller.

---

## Defensa oral (3 minutos al final)

Cuando entregues, el docente va a:

- Abrir tu proyecto y arrancarlo en su computador (no en el tuyo).
- Hacer 1 o 2 llamados a la API para confirmar que funciona.
- Señalar **cualquier línea del código** y pedirte que la expliques.
- Hacerte 1 pregunta de la `reflexion-final.md` y escuchar tu respuesta.

Esta defensa es donde se distingue al estudiante que entendió del que solo ejecutó. Si no puedes explicar una línea, esa línea cuenta como código que no es tuyo.

---

## Resumen visual de la entrega

```
mi-entrega/
├── especificacion.md
├── prompts/
│   ├── 01-...
│   ├── 02-...
│   └── ...
├── proyecto/
│   └── (todos los archivos del código)
├── tests/
│   └── (archivos de test ejecutables)
├── bitacora.md
├── diff-de-correcciones.md
└── reflexion-final.md
```

Si falta alguno de los 6 archivos obligatorios o las dos carpetas, la entrega se considera incompleta.
