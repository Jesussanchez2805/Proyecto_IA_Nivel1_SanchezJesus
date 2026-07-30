# AgendaBot Services — Documentación Técnica

Bot conversacional en Telegram con máquina de estados, orquestado en n8n y persistido en Google Sheets.

## 1. Archivos entregados

| Archivo | Qué es |
|---|---|
| `AgendaBot_Workflow_Principal.json` | Workflow n8n importable: recibe mensajes de Telegram, ejecuta la máquina de estados y responde |
| `AgendaBot_Workflow_Cron.json` | Workflow n8n importable: resumen diario automático (Fase 7) |
| `AgendaBot_Documentacion.md` | Este documento |

**Cómo importar:** en n8n → menú `⋮` (arriba a la derecha) → *Import from File* → selecciona cada `.json`. Se importan como workflows nuevos con todos los nodos y el código ya cargado.

## 2. Antes de activar: cosas que DEBES configurar

1. **Spreadsheet ID**: ya está cargado en todos los nodos (`1ZBBzCiGk-GMBf9eKGtDiDrYhdqvqk6HcrN6Vd9nU1ts`), no hace falta tocarlo salvo que cambies de hoja de cálculo.
2. **Credenciales**: los nodos referencian credenciales llamadas `AgendaBot_Telegram` (Telegram API) y `AgendaBot_GoogleSheets` (Google Sheets OAuth2). Al importar, n8n te pedirá que las re-vincules a tus credenciales reales (los IDs `"1"` y `"2"` del JSON son solo placeholders).
3. **Zona horaria del nodo Schedule Trigger** (workflow Cron): por defecto usa la zona horaria configurada en tu instancia de n8n. Verifica que sea la tuya antes de activarlo, o el resumen de las 8:00 AM llegará a otra hora.

## 3. Arquitectura del workflow principal

```
Telegram Trigger
   → Normalizar Input (Code)
   → Get Session (Sheets: SESSIONS, Always Output Data)
   → Get Usuario (Sheets: USUARIOS, Always Output Data)
   → Get Citas Activas (Sheets: CITAS, Always Output Data)
   → Preparar Contexto (Code, reconstruye el item original desde Normalizar Input)
   → AgendaBot Brain (Code) ── el "cerebro": toda la máquina de estados
        ├─→ Guardar Sesion (Sheets upsert: SESSIONS)
        ├─→ IF Escribir Citas   → Escribir Citas (appendOrUpdate)
        ├─→ IF Escribir Tareas  → Escribir Tareas (appendOrUpdate)
        ├─→ IF Escribir Habitos → Escribir Habitos (appendOrUpdate)
        ├─→ IF Escribir Listas  → Escribir Listas (appendOrUpdate)
        ├─→ IF Escribir Items_Lista → Escribir Items_Lista (appendOrUpdate)
        ├─→ IF Usuario Nuevo    → Registrar Usuario Nuevo (appendOrUpdate)
        ├─→ Registrar Log (Sheets append: LOGS)
        └─→ Enviar Respuesta (Telegram Send Message)
```

## 4. Cobertura por fase

| Fase | Estado | Dónde está |
|---|---|---|
| 1. Base de datos | Hecha por ti previamente | — |
| 2. Webhook + normalización | Implementada | Nodo `Normalizar Input` |
| 3. Sesión y máquina de estados | Implementada | `Get Session` + lógica de sesión en `Brain` + `Guardar Sesion` |
| 4. Enrutador principal | Implementada | `switch(pantalla)` dentro de `Brain` |
| 5. Wizard de citas (6 pasos) | Implementada completa: validación de fecha/hora, solapamiento, confirmar/editar/cancelar | `case 'WIZARD_CITA'` en `Brain` |
| 6. Módulos secundarios (Tareas, Hábitos, Listas) + permisos | Implementada: creación y marcado de completado para los 3 módulos; control de rol ADMIN para la opción 8 | `MENU_TAREAS`, `MENU_HABITOS`, `MENU_LISTAS` y sus wizards, más `esAdmin` en `Brain` |
| 7. Cron resumen diario | Implementada | `AgendaBot_Workflow_Cron.json` |
| 8. Logs + exportación | Logs implementados (se registra cada interacción); exportación es el propio archivo `.json` que ya tienes | `Registrar Log` |


## Actualización: Comunicación Humanizada (v2)

El `Brain` fue reescrito para seguir los Artículos 6 a 10: saludo cercano, explicación breve, opciones numeradas, sugerencia y forma de continuar/salir en cada mensaje. Cambios concretos:

- **Menú principal ampliado a 9 opciones (0-8):** 0 Ayuda, 1 Agenda, 2 Tareas, 3 Recordatorios, 4 Hábitos, 5 Listas, 6 Reportes, 7 Configuración, 8 Administrador. Los números de Hábitos y Listas se corrieron (antes 3 y 4, ahora 4 y 5) para dejar espacio a Recordatorios.
- **Bienvenida (Artículo 6):** se muestra una sola vez, cuando `Get Session` no devuelve fila para ese `telegram_user` (usuario/sesión nuevos). El texto es el que me pasaste, literal.
- **Menú principal recurrente (Artículo 8):** se muestra cada vez que se vuelve a `MAIN_MENU` desde cualquier punto (opción 9, fin de un flujo, etc.), con la sugerencia "te recomiendo la opción 1".
- **Opción inválida (Artículo 7):** ahora es dinámico — cada pantalla tiene un nombre legible y un rango de opciones válidas (tabla `PANTALLAS_INFO` al inicio de `brain.js`), y el mensaje arma automáticamente "Estás en: X / Opciones disponibles: Y".
- **Menú de Agenda (Artículo 9)**, con las 5 opciones nuevas: Agendar, Consultar, **Reprogramar** (nuevo), Cancelar, **Marcar como completada** (nuevo).
- **Wizard de citas (Artículo 10):** los 6 pasos y el mensaje de éxito usan tu texto exacto. Después de agendar/cancelar/reprogramar/completar una cita, el bot ahora pregunta "1. Volver a Agenda / 2. Ir al menú principal" (pantalla `POST_ACCION_AGENDA`) en vez de mandarte directo al menú principal como en la v1. Apliqué el mismo patrón de "post-acción" a Tareas, Hábitos y Listas para mantener la comunicación consistente en todo el bot.

### Puntos que quedan abiertos con este cambio

- **Recordatorios (opción 3) no tiene hoja propia**: las 8 pestañas de la Fase 1 no incluían `RECORDATORIOS`. El menú ya existe y explica esto al usuario en vez de fallar silenciosamente. Hay que decidir: ¿reutilizamos `HABITOS` (ya tiene `hora_recordatorio`) o creamos una hoja `RECORDATORIOS` nueva?
- **Reportes (opción 6)** hoy solo puede mostrar datos de `CITAS` (confirmadas/canceladas, citas de hoy) porque es la única hoja que el workflow trae completa en cada mensaje. Reportes de Tareas/Hábitos/Listas necesitan que agreguemos esas lecturas.
- **Configuración (opción 7)** es un placeholder: solo muestra el nombre/rol del usuario; "cambiar nombre visible" todavía no escribe a ningún lado.
- Los textos de Tareas, Hábitos, Listas, Reportes, Configuración, Administrador y Ayuda **no venían especificados** en tu mensaje (solo Agenda y Bienvenida/Menú/Opción inválida lo estaban) — los redacté siguiendo la misma estructura humanizada. Son 100% editables, están cada uno en su propia función al inicio de `brain.js`.

## 4bis. Actualización: Comunicación Humanizada (Artículos 6-10)

Se reescribió por completo el texto de todos los mensajes del bot siguiendo la regla de Comunicación Humanizada (saludo/contexto breve + qué puede hacer + opciones numeradas + sugerencia + cómo continuar/salir), y se adoptó el nuevo Menú Principal de 9 opciones (0 Ayuda, 1 Agenda, 2 Tareas, 3 Recordatorios, 4 Hábitos, 5 Listas, 6 Reportes, 7 Configuración, 8 Administrador).

Cambios de comportamiento nuevos respecto a la versión anterior:

- **Primer contacto**: ahora cualquier usuario sin sesión previa recibe el Mensaje de Bienvenida completo (Artículo 6) en vez de ir directo al menú.
- **Opción 0 cambió de significado**: antes "0" repetía el menú principal; ahora "0" abre la Ayuda (Artículo 6/8). Para volver a ver el menú principal desde la Ayuda, hay que usar el número de la sección deseada o escribir 9 desde cualquier submenú.
- **Mensaje de opción inválida** ahora sigue el formato exacto del Artículo 7 ("Estás en: ... / Opciones disponibles: ..."), con el menú completo debajo para no obligar al usuario a desplazarse hacia arriba.
- **Agenda (Menú Artículo 9)**: se agregaron dos flujos nuevos que no existían antes — **Reprogramar cita** (`WIZARD_REPROGRAMAR_CITA`: pide ID, nueva fecha, nueva hora, valida solapamiento) y **Marcar cita como completada** (`WIZARD_COMPLETAR_CITA`).
- **Tras agendar una cita** se agregó un estado intermedio nuevo, `POST_CITA_EXITO` (mensaje de éxito del Artículo 10), que pregunta "1. Volver a Agenda / 2. Ir al menú principal" en vez de mandar directo al menú principal como antes.
- **Tres módulos nuevos en el menú principal** que la tabla de Google Sheets original (Fase 1) no contemplaba: **Recordatorios**, **Reportes**, **Configuración**. Los dejé funcionando con el formato humanizado pero como placeholders honestos:
  - *Recordatorios*: aviso de que el módulo está en construcción (no hay hoja `RECORDATORIOS` en el modelo de datos original; hoy esa función vive parcialmente en `HABITOS.hora_recordatorio`).
  - *Reportes*: implementé de una vez la opción 1 (resumen de citas de hoy), reutilizando `Get Citas Activas`.
  - *Configuración*: muestra nombre, rol y estado de acceso del usuario (datos que ya tenemos de `Get Usuario`), sin opciones editables todavía.

**Para la charla final:** decidir si Recordatorios necesita su propia hoja en `AgendaBot_DB` (Fase 1 tendría que ampliarse) o si reutilizamos `HABITOS`, y qué otras opciones quieres en Reportes/Configuración.

- **Tareas → "Consultar tareas pendientes"**: lista las tareas con `estado != COMPLETADA`, ordenadas por prioridad.
- **Hábitos → "Consultar mis hábitos"**: lista los hábitos con `estado != INACTIVO`, mostrando frecuencia y hora de recordatorio si la tienen.
- **Listas → "Consultar mis listas"**: lista `id_lista`, `nombre_lista` y `tipo`.
- **Reportes**: ahora suma también tareas pendientes, hábitos activos y cantidad de listas al resumen (antes solo mostraba citas).

## 6. Plan de pruebas sugerido (Fase 8)

1. **Sesión nueva**: escribe cualquier cosa desde un usuario nunca antes visto → debe crear fila en `USUARIOS` y `SESSIONS`, y mostrar el menú principal.
2. **Navegación básica**: `1` → `9` desde cada submenú → debe volver siempre a `MAIN_MENU`.
3. **Wizard de citas feliz**: completar los 6 pasos con datos válidos → verificar fila nueva en `CITAS` con `estado = CONFIRMADA`.
4. **Wizard de citas — validaciones**: fecha pasada, fecha con formato incorrecto, hora inválida → debe repetir el mismo paso con el mensaje de error, sin avanzar `paso_actual`.
5. **Solapamiento**: agenda dos citas con la misma fecha/hora → la segunda debe rechazarse y reiniciar el wizard en el paso 1.
6. **Permisos**: con un usuario `rol = USER`, opción `8` desde el menú principal → debe responder "acceso restringido" y quedarse en `MAIN_MENU`.
7. **Logs**: revisar la hoja `LOGS` después de cada prueba anterior → debe haber una fila por interacción con el `resultado` correcto (`OK`, `VALIDACION_FALLIDA`, `OPCION_INVALIDA`, `ACCESO_DENEGADO`, etc.).
8. **Cron**: ejecutar manualmente el workflow de cron (botón "Execute Workflow" en n8n) con al menos una cita y una tarea con fecha de hoy → verificar que cada usuario en `USUARIOS` con `permitido = TRUE` reciba el mensaje.
