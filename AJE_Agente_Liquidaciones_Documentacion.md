# 📋 Documentación — AJE Agente Liquidaciones (Multisesiones)

> Workflow n8n para gestión automatizada de liquidaciones de transportistas vía WhatsApp, con análisis de comprobantes por IA (visión), soporte multisesión y panel administrativo.

---

## 1. Resumen general

| Atributo | Valor |
|---|---|
| **Nombre** | AJE - Agente Liquidaciones (Multisesiones) |
| **Estado** | `active: true` |
| **Total de nodos** | 168 |
| **Canal de entrada** | WhatsApp Business (Meta Graph API v19.0) |
| **Motor IA** | Groq — `llama-3.1-8b-instant` (texto) + modelo de visión vía `$vars.IA_MODEL_VISION` |
| **Persistencia** | Google Sheets (3 spreadsheets) + Google Drive (archivo de imágenes) |
| **Reportería** | Excel (`spreadsheetFile`) + Gmail |

### Composición por tipo de nodo

| Tipo | Cantidad | Uso |
|---|---:|---|
| `googleSheets` | 48 | Lectura/escritura de sesiones, comprobantes, autorizados, maestro |
| `if` | 36 | Bifurcaciones de lógica |
| `whatsApp` | 34 | Envío de mensajes al transportista/admin |
| `code` | 19 | Mapeo, validación, normalización, parsing de IA |
| `httpRequest` | 12 | Groq, descarga de imágenes, mensajes interactivos WA |
| `googleDrive` | 8 | Carpetas por día y guardado de comprobantes |
| `chainLlm` (LangChain) | 3 | Clasificador de intención + agente conversacional + agente admin |
| `switch` | 3 | Enrutamiento por intención / tipo de mensaje |
| `lmChatGroq` | 1 | Modelo Groq para los chains |
| `whatsAppTrigger` | 1 | Disparador de entrada |
| `spreadsheetFile` | 1 | Generación de Excel del reporte |
| `gmail` | 1 | Envío del reporte por correo |
| `outputParserStructured` | 1 | Parseo de salida JSON del agente admin |

---

## 2. Variables de entorno (`$vars`)

El flujo depende de **variables de entorno de n8n** (centralización correcta, buena práctica):

| Variable | Propósito |
|---|---|
| `SHEET_ID_CONTROL` | Spreadsheet de control (sesiones, comprobantes, autorizados) |
| `SHEET_ID_MAESTRO` | Spreadsheet maestro de liquidaciones (`Detalle`) |
| `SHEET_ID_REPORTE` | Spreadsheet de referencias para reporte (bancos) |
| `WA_PHONE_ID` | Phone Number ID de WhatsApp Business |
| `NOM_AGENTE` | Nombre del agente conversacional |
| `IA_MODEL_VISION` | Modelo de visión usado en Groq |
| `TOLERANCIA_BS` | Tolerancia en bolivianos para cierre de liquidación (default 1.00) |
| `DRIVE_FOLDER_COMPR` | Carpeta raíz de comprobantes en Drive |
| `DRIVE_FOLDER_RECHA` | Carpeta de rechazados/no legibles |

---

## 3. Credenciales requeridas

| Servicio | Credencial |
|---|---|
| WhatsApp Trigger | `whatsAppTriggerApi` — "WhatsApp OAuth account" |
| WhatsApp Send | `whatsAppApi` — "WhatsApp account" |
| Google Sheets | `googleSheetsOAuth2Api` — ⚠️ aparecen **dos** nombres distintos: "Google Sheets OAuth2 API" y "Google Sheets account" |
| Google Drive | `googleDriveOAuth2Api` — "Google Drive OAuth2 API" |
| Gmail | `gmailOAuth2` — "Gmail OAuth2 API" |
| Groq | `groqApi` — "Groq account" |

---

## 4. Estructura de datos — Google Sheets

### 4.1 Spreadsheet de CONTROL (`$vars.SHEET_ID_CONTROL`)
ID hardcodeado encontrado en un nodo: `1MaJR0bmUvB5r3iCtJwzMQNcrIgw-M4lz20zL3zDTriA` (⚠️ ver hallazgos).

#### Pestaña `sesiones` (gid=0)
Núcleo del estado. Columnas escritas en el `append` inicial:

| Columna | Descripción |
|---|---|
| `estado` | ACTIVO / PENDIENTE / CONFIRMADO / COMPLETADO |
| `phone` | Teléfono del transportista |
| `guia` | Código de guía normalizado `COM-0000000000XXXXXX` |
| `liquidacion_total` | Monto total a liquidar (del maestro) |
| `total_recaudado` | Acumulado de comprobantes válidos |
| `total_restante` | Saldo pendiente |
| `fecha_inicio` | Inicio de la sesión |
| `reintentos` | Contador de intentos fallidos (límite controlado) |
| `ultimo_movimiento` | Timestamp última actividad |
| `transportista` | Nombre del perfil WA |
| `foco_ant` | Sesión anterior en foco (multisesión) |
| `s_fecha` | Fecha de sesión |
| `sesion_foco` | Marca de sesión activa en foco |

#### Pestaña `comprobantes` (gid=916570564)
Datos extraídos del comprobante por IA + validación:

`referencia`, `monto`, `fecha`, `hora`, `tipo_pago`, `banco_billetera`, `banco_origen`, `cuenta_destino`, `cuenta_origen`, `nombre_depositante`, `nombre_beneficiario`, `agencia`, `legible`, `fecha_registro`, `monto_confianza`, `tipo_pago_confianza`, `referencia_confianza`, `fecha_confianza`, `phone`, `guia`, `link_img`, `estado_user`, `redondeo`, `s_fecha`, `advertencias`, `validacion`.

#### Pestaña `autorizados`
`phone`, `nombre`, rol (ADMIN/OPERATIVO), `contexto`, `ult_mss` — control de acceso y memoria del agente admin.

### 4.2 Spreadsheet MAESTRO (`$vars.SHEET_ID_MAESTRO`)
- **Pestaña `Detalle`** — fuente de verdad de las guías y su monto total a liquidar. Se consulta en "buscamos la guia", "Leer Liquidaciones", "buscamos la guia1".

### 4.3 Spreadsheet REPORTE (`$vars.SHEET_ID_REPORTE`)
- **Pestaña gid=1288535529** ("Leer referencias bancos") — catálogo de bancos/cuentas para enriquecer el reporte.

---

## 5. Arquitectura del flujo

### 5.1 Entrada y normalización

```
WhatsApp Trigger ("User envia mensaje")
   └─> Filtrar eventos validos        (descarta status updates / eventos vacíos)
        └─> Mapeamos datos del mensaje (extrae phone, tipo, texto, imageId, buttonId, comandos FIN/CERRAR)
             └─> Buscar autorizado     (valida phone contra hoja 'autorizados')
                  └─> Es autorizado?
                       ├─ NO  -> NO AUTORIZADO (corta)
                       └─ SÍ  -> Es ADMIN?
```

El nodo **"Mapeamos datos del mensaje"** soporta doble formato de payload (Meta nativo y un formato traducido al español: `mensajes`, `contactos`, `texto.cuerpo`, etc.), detecta comandos de cierre (`FIN`, `CERRAR`, `LISTO`, `TERMINAR`) y extrae `guiaCierre` de botones `COMPLETAR_GUIA_*`.

### 5.2 Bifurcación por rol

```
Es ADMIN?
 ├─ SÍ (Admin) -> Is message int? (reporte) -> Switch2
 │                                              ├─ REPORTE  -> Leer comprobantes del día -> Excel -> Gmail -> WA
 │                                              ├─ INFORME  -> Leer sesiones -> Construir informe
 │                                              └─ GUIA     -> Normalizar Guia2 -> cierre por admin
 │                IS GUIA?2 / Basic LLM Chain1 (agente admin con Structured Output Parser)
 │
 └─ NO -> Es OPERATIVO? -> Is message int? -> Switch (4 ramas de sesión/comprobante)
```

### 5.3 Lógica de sesión del transportista (`Switch`)
Cuatro salidas que enrutan según si tiene/recibe sesión o comprobante pendiente:
- **CON_SESION / REC_SESION** → "Buscamos la sesion PENDIENTE"
- **CON_COMPROBANTE / REC_COMPROBANTE** → "Buscamos comprob PENDIENTE"

Maneja confirmaciones interactivas (`acepta?`, `confirma?`), activación de estado (`ESTADO ACTIVO`), cancelación y limpieza de falsas sesiones (`ELIINAMOS FALSA SESION`).

### 5.4 Clasificación de intención (`Switch1`)
Alimentado por el chain **"Intencion del mensaje"** (Groq) que devuelve una sola palabra:
- **ESTADO** → consulta de saldo → "Buscamos su liquidacion"
- **CIERRE** → "Cierre de caja"
- **GUIA** → "Normalizar Guia" (inicia/activa sesión)
- **false** → mensaje genérico / agente conversacional

### 5.5 Procesamiento de comprobantes (imagen)

```
Es IMG? -> OBTENER ID DE LA IMG -> DESCARGAR IMG (Graph API)
        -> CONVERTIR A BASE64 -> ANALIZAR CON GROQ (visión)
        -> FILTRAR DATOS (parse JSON: monto, referencia, banco, confianza, legible)
        -> Validar datos (cuentas AJE, nombres, fecha futura)
        -> existe comprobante? (anti-duplicado por referencia)
        -> Buscamos/Creamos carpeta del día (Drive) -> Guardamos imagen -> permisos lectura
        -> registrar comprobante -> Actualizar sesión con montos
        -> Validar restante liq (tolerancia Bs) -> liq completa? -> COMPLETAR LIQ
```

Existe una **segunda cadena espejo** (`ANALIZAR CON GROQ1`, `FILTRAR DATOS1`, `DESCARGAR IMG2/IMG3`, `Es IMG?1`) para flujos paralelos (sesión pendiente / cierre). Hay control de reintentos (`reintentos ++`, `reintentos 0`, `Error limite`) para imágenes no legibles.

### 5.6 Multisesión (diferenciador del flujo)
Lógica para que un transportista tenga varias guías y se vaya activando la siguiente automáticamente:
- `Desactivar sesiones en foco anteriores`, `Activar sesion de la guia elegida`
- `Buscar siguiente sesion pendiente` (+ variante "(caja)") → `IF ¿Hay siguiente sesion?` → `Activar siguiente sesion` → `WA aviso sesion activada automaticamente`
- `volvemos a sesion anterior`, `tenia sesiones anteriores?` para rollback.

### 5.7 Reportería admin
`Leer comprobantes del día` + `Leer referencias bancos` → `Transformar para reporte` (limpia notación científica, formatea fechas DD/MM/YYYY, identifica banco por nº de cuenta) → `¿Hay datos hoy?` → `Generar Excel` → `Enviar email con reporte` → `WA confirmación reporte`.

---

## 6. Integraciones externas

| Endpoint | Uso |
|---|---|
| `graph.facebook.com/v19.0/{WA_PHONE_ID}/messages` | Envío de mensajes interactivos (botones), 7 nodos HTTP |
| `api.groq.com/openai/v1/chat/completions` | Análisis de comprobantes por visión (2 nodos) |
| WhatsApp node nativo | Mensajes de texto simples (34 nodos) |

---

## 7. Hallazgos, mejoras y arreglos sugeridos

### 🔴 Críticos

1. **`sheetName` referenciado por `gid` numérico en ~40 nodos** (`gid=0`, `916570564`, `1288535529`). Si alguien reordena o recrea pestañas, el `gid` cambia y **todo el flujo se rompe silenciosamente**. → Usar el **nombre** de pestaña (`sesiones`, `comprobantes`) de forma consistente. Hay además mezcla de formatos (`=gid=0` vs `gid=0`), lo que es frágil.

2. **ID de spreadsheet hardcodeado** en al menos un nodo (`1MaJR0bmUvB5r3iCtJwzMQNcrIgw...`) mientras el resto usa `$vars.SHEET_ID_CONTROL`. → Unificar **todo** a la variable para evitar apuntar a un sheet equivocado.

3. **Dos credenciales distintas de Google Sheets** ("Google Sheets OAuth2 API" y "Google Sheets account"). → Consolidar en una sola; reduce riesgo de fallos de auth parciales y de cuotas dispersas.

### 🟠 Importantes

4. **Concurrencia / condiciones de carrera**: con multisesión y comprobantes simultáneos, las operaciones read→update sobre `sesiones` no son atómicas. Dos mensajes casi simultáneos del mismo transportista pueden recalcular `total_restante` sobre un valor obsoleto. → Considerar bloqueo lógico (campo `procesando`), cola, o `executionOrder` + deduplicación por `messageId`.

5. **Idempotencia del trigger**: WhatsApp puede reenviar el mismo webhook. El anti-duplicado existe a nivel de `referencia` de comprobante, pero no a nivel de `messageId`. → Guardar `messageId` procesados para descartar reentregas.

6. **Parsing de IA sin defensa**: `FILTRAR DATOS` hace `JSON.parse` y accede a `parsed.monto.valor` sin try/catch ni validación de estructura. Si Groq devuelve algo fuera de formato, el nodo lanza excepción y corta el flujo sin avisar al usuario. → Envolver en try/catch y enviar mensaje de "no pude leer el comprobante".

7. **Cadenas de imagen duplicadas** (`ANALIZAR CON GROQ` / `GROQ1`, `FILTRAR DATOS` / `DATOS1`, `DESCARGAR IMG`/`IMG1`/`IMG2`/`IMG3`). Mantener cuatro variantes casi idénticas multiplica el costo de cambios. → Extraer a un **sub-workflow** reutilizable ("Procesar comprobante") invocado por las distintas ramas.

8. **Modelo de texto fijo `llama-3.1-8b-instant`**: para el clasificador de intención está bien por costo, pero es propenso a falsos negativos en frases ambiguas. → Reforzar el prompt con ejemplos few-shot o subir de modelo solo en la rama de clasificación.

### 🟡 Menores / mantenibilidad

9. **Errores tipográficos en nombres de nodos** (`ELIINAMOS FALSA SESION`, espacios sobrantes en `reintentos `, `guia foco1`). No rompen nada, pero dificultan referencias `$('...')`. → Renombrar con convención clara.

10. **Sin sticky notes / documentación interna** (0 encontradas) en un flujo de 168 nodos. → Agregar notas por bloque (Entrada, Roles, Comprobantes, Multisesión, Reportes) para facilitar el handover.

11. **Hardcode de cuentas/nombres AJE** dentro del código de `Validar datos` (`CUENTAS_AJE`, `NOMBRES_AJE`). → Mover a una pestaña de configuración o a `$vars` para no editar código al cambiar una cuenta.

12. **`Switch1` con una rama sin `outputKey`** (la tercera salida aparece vacía). → Etiquetar todas las ramas para claridad.

13. **Manejo de errores global**: no se observa un Error Workflow ni rama de captura general. → Configurar un *Error Trigger* que notifique a un admin si una ejecución falla.

### ✅ Aspectos bien resueltos
- Centralización de configuración en `$vars`.
- Normalización robusta de guías (`padStart(16,'0')`) y de texto (NFD, quita tildes/SA/SRL).
- Anti-duplicado de comprobantes por referencia.
- Tolerancia configurable en bolivianos para cierre de liquidación.
- Soporte de doble formato de payload de WhatsApp.
- Separación clara de roles (ADMIN / OPERATIVO / no autorizado).
- Lógica de multisesión con activación automática de la siguiente guía.

---

## 8. Diagrama lógico resumido

```
WA Trigger
  → Filtrar eventos → Mapear mensaje → Buscar autorizado → ¿Autorizado?
       ├─ No → NO AUTORIZADO
       └─ Sí → ¿ADMIN?
             ├─ ADMIN → Switch2 → {Reporte Excel+Gmail | Informe | Cierre guía} + Agente Admin (LLM+Parser)
             └─ OPERATIVO →
                   ├─ Imagen → Descargar → Groq Visión → Filtrar → Validar → Drive → Registrar comprobante → Actualizar saldos → ¿Completa? → Cerrar liq
                   └─ Texto  → Intención (Groq) → Switch1 → {ESTADO saldo | CIERRE caja | GUIA nueva sesión | conversacional}
                                   └─ Multisesión: desactivar/activar foco, siguiente sesión automática
```

---

## 9. Plan de mejoras — Backlog y seguimiento

Plan acordado, ordenado por prioridad/dependencia. Cada mejora se aplicará entregando el **JSON modificado listo para importar**.

| # | Fase | Mejora | Prioridad | Estado |
|---|---|---|---|---|
| 1 | 1 · Estabilidad | Reemplazar `gid` numérico por nombre de pestaña (~40 nodos Sheets) | 🔴 Crítica | ⏸️ **Pospuesta** (registrada para el futuro) |
| 2 | 1 · Estabilidad | Unificar spreadsheet ID hardcodeado → `$vars.SHEET_ID_CONTROL` | 🔴 Crítica | ✅ **Aplicada** (nodo "Desactivar sesiones en foco anteriores") |
| 3 | 1 · Estabilidad | Consolidar las dos credenciales de Google Sheets en una | 🔴 Crítica | ✅ **Aplicada** (48 nodos → "Google Sheets account") |
| 4 | 2 · Robustez | `try/catch` en el `JSON.parse` de Groq + mensaje al usuario | 🟠 Alta | ✅ **Aplicada** (`FILTRAR DATOS` y `FILTRAR DATOS1`) |
| 5 | 2 · Robustez | Idempotencia por `messageId` (descartar webhooks reenviados) | 🟠 Alta | ✅ **Aplicada (versión mínima)** — `messageId` expuesto en `Mapeamos datos del mensaje` |
| 6 | 2 · Robustez | Error Workflow global que avise a un admin | 🟠 Alta | ✅ **Aplicada** — workflow separado `AJE - Notificador de Errores` (requiere activación manual, ver abajo) |
| 7 | 3 · Concurrencia | Proteger read→update sobre `sesiones` (flag `procesando` / dedup) | 🟠 Alta | ⬜ Pendiente |
| 8 | 4 · Mantenibilidad | Extraer las 4 cadenas de imagen duplicadas a sub-workflow | 🟡 Media | 📐 **Refactor futuro** (diseño documentado abajo) |
| 9 | 4 · Mantenibilidad | Mover cuentas/nombres AJE hardcodeados a config (`$vars`/pestaña) | 🟡 Media | ✅ **Aplicada (opción A: `$vars`)** — nodo `Validar datos` |
| 10 | 4 · Mantenibilidad | Renombrar nodos con erratas + sticky notes por bloque | 🟡 Baja | ✅ **Sticky notes aplicadas (7)** · ⏸️ renombrado de erratas pospuesto |
| 11 | 4 · Mantenibilidad | Etiquetar la rama vacía de `Switch1` | 🟡 Baja | ⬜ Pendiente |

### Detalle de mejoras pospuestas (para retomar)

**#1 — Reemplazar `gid` numérico por nombre de pestaña**
- **Por qué importa:** si se recrea o reordena una pestaña, el `gid` cambia y los nodos dejan de encontrarla *sin lanzar error visible*; el flujo devuelve vacío y la lógica se rompe en silencio.
- **Alcance:** ~40 nodos `googleSheets`. Mezcla de formatos a normalizar (`=gid=0` vs `gid=0`).
- **Mapeo de reemplazo:**
  - `gid=0` → `sesiones`
  - `916570564` → `comprobantes`
  - `1288535529` → pestaña de referencias de bancos (⚠️ confirmar nombre real antes de aplicar)
- **Cómo aplicar cuando se retome:** regenerar el JSON completo del workflow corregido y reimportar (más limpio que editar 40 fragmentos).

> **Nota Mejora #1 (futuro):** el usuario YA tiene creadas las variables de pestaña necesarias para aplicarla sin fricción:
> `SHEET_TAB_SESIONES=0`, `SHEET_TAB_COMPROBANTES=916570564`, `SHEET_TAB_DETALLE=0`, `SHEET_TAB_REFERENCIAS_BANCOS=1288535529`, `SHEET_TAB_REPORTE_LIQUIDACIONES=987654321`. Al retomar, reemplazar el `gid` directo de cada nodo por la `$vars` correspondiente (o mejor, por el nombre de pestaña).

> **Detalle Mejora #2 (aplicada):** solo había **un** nodo con el ID hardcodeado en `documentId.value`: *"Desactivar sesiones en foco anteriores"*. Se cambió a `={{ $vars.SHEET_ID_CONTROL }}` (mismo spreadsheet, confirmado por captura: `SHEET_ID_CONTROL = 1MaJR0bmUvB5r3iCtJwz...`). Las otras apariciones del ID eran solo `cachedResultUrl` (caché visual de n8n, sin efecto en ejecución).

> **Detalle Mejora #3 (aplicada):** 44 nodos usaban "Google Sheets account" (id `FaAiIXMHXGGQkpkt`) y 4 usaban "Google Sheets OAuth2 API" (id `bQsC9SfJCaJhNihd`). Se unificó hacia la mayoritaria → los **48 nodos** de Sheets usan ahora una sola credencial. Confirmado por el usuario que ambas apuntaban a la misma cuenta Google. *Sugerencia opcional: borrar la credencial "Google Sheets OAuth2 API" en n8n una vez verificado que el flujo importado funciona.*

> **Detalle Mejora #4 (aplicada):** ambos nodos de parseo ahora envuelven el acceso a `choices[0].message.content` y el `JSON.parse` en `try/catch`, usan acceso seguro a `.valor`/`.confianza` (no revientan si falta un campo) y, ante cualquier fallo, devuelven `{ error:true, legible:false, motivo_error }` en lugar de lanzar excepción.
> **⚠️ Pendiente de cableado (acción del usuario en n8n):** estos nodos ahora pueden emitir `error:true`, pero el flujo aún NO enruta esa salida. Para aprovecharlo, agregar un nodo IF después de cada `FILTRAR DATOS` que evalúe `{{ $json.error === true }}` y lo conecte a un mensaje WhatsApp tipo *"No pude leer el comprobante, reenvíalo más claro"* (puede reutilizar la rama de reintentos existente: `reintentos ++` / `Error limite`).

> **Detalle Mejora #5 (aplicada — mínima):** se añadió `messageId = message.id` al nodo `Mapeamos datos del mensaje`, expuesto en el objeto resultado. No cambia el comportamiento; deja el identificador único disponible para una futura deduplicación. **Decisión:** no se implementó la versión completa (hoja `procesados` + dedup) porque en 3 semanas de pruebas en producción no se observó ningún caso de duplicidad. Nota: el nodo `Filtrar eventos validos` ya descarta status updates y tipos no válidos, pero NO compara IDs — es un filtro de tipo, no de idempotencia.

> **Detalle Mejora #6 (aplicada):** se generó un workflow SEPARADO `AJE - Notificador de Errores (Error Workflow)` con un nodo `Error Trigger` → `HTTP Request` que notifica por WhatsApp al **59163332108** con: nombre del workflow, nodo que falló, mensaje de error, hora e ID de ejecución. Reutiliza `$vars.WA_PHONE_ID` y la credencial `whatsAppApi`.

#### 🛠️ Pasos MANUALES en n8n para activar la Mejora #6
1. **Importar** el archivo `AJE_-_Notificador_de_Errores.json` como un workflow nuevo (Workflows → Import from File).
2. **Verificar** que la credencial WhatsApp quedó vinculada en el nodo "Notificar Admin por WhatsApp" (si el id no coincide en tu instancia, reasignarla manualmente).
3. **Activar** ese workflow (toggle "Active" arriba a la derecha). *Un Error Workflow debe estar activo para dispararse.*
4. Ir al **workflow principal** "AJE - Agente Liquidaciones" → menú **⋯ → Settings** → campo **"Error Workflow"** → seleccionar `AJE - Notificador de Errores`. Guardar.
5. (Opcional) Repetir el paso 4 en CUALQUIER otro workflow que quieras monitorear con la misma alerta.
6. **Probar:** forzar un error (p.ej. desconectar temporalmente una credencial en un flujo de prueba) y confirmar que llega el WhatsApp.

> ⚠️ **Limitación de WhatsApp (importante):** los mensajes de *texto libre* solo se entregan si el número destino (59163332108) escribió al bot en las últimas 24 h ("ventana de servicio"). Como los errores pueden ocurrir fuera de esa ventana, para alertas 100% confiables conviene crear y usar una **plantilla (template) aprobada** de Meta para el mensaje de error. Mientras tanto, el admin puede mandar un "hola" al bot cada día para mantener la ventana abierta, o usar email como canal de respaldo.

### 📐 Diseño de la Mejora #8 (refactor futuro) — Sub-workflow "Procesar comprobante"

**Objetivo:** eliminar la duplicación de la cadena de visión, que hoy existe en variantes casi idénticas (`DESCARGAR IMG`/`IMG1`/`IMG2`/`IMG3`, `ANALIZAR CON GROQ`/`GROQ1`, `FILTRAR DATOS`/`DATOS1`). Un solo cambio de prompt o de parsing debería tocar **un único lugar**.

**Diseño propuesto:**
1. Crear un workflow nuevo: **`AJE - SUB - Procesar Comprobante`**.
   - Nodo inicial: `Execute Workflow Trigger` (recibe inputs).
   - **Inputs esperados:** `imageId` (o `url`), `phone`, `guia`, `transportista`, y un flag opcional `modo` (`comprobante` | `guia`) para saber qué parser aplicar.
   - Cuerpo: `OBTENER ID DE LA IMG` → `DESCARGAR IMG` → `CONVERTIR A BASE64` → `ANALIZAR CON GROQ` → `FILTRAR DATOS` (con el try/catch ya aplicado en Mejora #4).
   - **Output:** el objeto normalizado (`monto`, `referencia`, `legible`, `error`, etc.).
2. En el workflow principal, reemplazar cada una de las 4 cadenas por **un nodo `Execute Workflow`** que llame al sub-workflow pasando los inputs de esa rama.
3. Borrar los nodos duplicados una vez verificado.

**Riesgos / precauciones al ejecutarlo:**
- Hacerlo en un entorno de **pruebas / copia**, no en producción directa.
- Verificar el paso de binarios (imagen) entre workflows: `Execute Workflow` propaga JSON; para datos binarios puede convenir pasar la `url`/`imageId` y descargar DENTRO del sub-workflow (recomendado) en lugar de pasar el base64.
- Mantener las dos variantes de parser (`comprobante` con monto/banco vs `guia` con codigo_completo/numero) seleccionables por el flag `modo`, ya que hoy `FILTRAR DATOS` y `FILTRAR DATOS1` devuelven estructuras distintas.
- Probar las 4 rutas (entrada normal, sesión pendiente, cierre, cierre admin) antes de eliminar los nodos viejos.

**Beneficio:** ~12-15 nodos menos, un único punto de mantenimiento para el prompt de visión y el parsing.

> **Detalle Mejora #9 (aplicada — opción A):** las listas `NOMBRES_AJE` y `CUENTAS_AJE` del nodo `Validar datos` ahora se leen desde `$vars.NOMBRES_AJE` y `$vars.CUENTAS_AJE` (formato: valores separados por comas). Si la variable no existe, usa **fallback** a los valores actuales, así nada se rompe si aún no las creas.
>
> **🛠️ Paso manual OPCIONAL en n8n** (para empezar a editar desde Settings sin tocar código): crear dos variables en *Settings → Variables*:
> - `NOMBRES_AJE` = `AJE,INDUSTRIAS AJE,INDUSTRIAS AJE BOLIVIA,AJE BOLIVIA,AJE GROUP,AJEBOLIVIA,INDUSTRIAS AJEBOLIVIA`
> - `CUENTAS_AJE` = `7015046986392,1041309287,1311415157`
>
> Mientras no las crees, el flujo sigue funcionando con los valores por defecto del código.

> **Detalle Mejora #10 (parcial):** se añadieron **7 sticky notes** documentando los bloques del flujo (Entrada, Roles, Panel Admin, Clasificación de intención, Procesamiento de comprobantes, Multisesión, Reportería). Son cajas de texto, no afectan la ejecución. Tras importar, puedes reposicionarlas arrastrándolas si tapan algún nodo.
>
> **⏸️ Renombrado de erratas — pospuesto (decisión del usuario):** queda pendiente corregir nombres como `ELIINAMOS FALSA SESION` / `ELIINAMOS FALSA SESION1` (falta una "M") y `reintentos ` (espacio sobrante). Es **riesgoso** porque esos nombres pueden estar referenciados en expresiones `$('...')` de otros nodos; renombrarlos exige buscar y reemplazar TODAS las referencias a la vez. Recomendado hacerlo en el editor visual usando la función de búsqueda, o en una sesión dedicada con verificación de cada `$(...)`.

---

## 10. MEJORA GRANDE — Soporte multisucursal (COM / LP / CBB)

**Objetivo:** registrar guías, sesiones y comprobantes de 3 sucursales (Principal=COM, La Paz=LP, Cochabamba=CBB) sin mezclar datos, manteniendo UN solo workflow.

**Decisiones de diseño (acordadas):**
- Mismos documentos Sheets, **pestañas separadas por sucursal**: `Detalle`/`Detalle_LP`/`Detalle_CBB` (maestro); `sesiones`/`sesiones_LP`/`sesiones_CBB` y `comprobantes`/`comprobantes_LP`/`comprobantes_CBB` (control). Principal sin sufijo.
- Un transportista pertenece a **una sola** sucursal → la sucursal se ancla al transportista en la hoja `autorizados`.
- Prefijos parametrizados en variable mapa: `SUCURSALES = COM:|LP:_LP|CBB:_CBB` (formato `PREFIJO:SUFIJO_HOJA`).
- Las 3 sucursales tienen lógica idéntica (no se duplican workflows).

**Alcance técnico:** 42 nodos operativos (sesiones/comprobantes) + 3 de maestro + 3 de normalización. Verificado que los 42 nodos son alcanzables desde `Buscar autorizado`, por lo que `$('Buscar autorizado')` / `Resolver Sucursal` son referenciables en todos.

**Plan por fases:**
| Fase | Contenido | Estado |
|---|---|---|
| A | Variable `SUCURSALES` + columna `sucursal` en `autorizados` + nodo `Resolver Sucursal` | ✅ Nodo aplicado (faltan pasos manuales) |
| B | Generalizar 3 nodos `Normalizar Guia` (multi-prefijo + validación de coincidencia) | ✅ Aplicada (sin validación cruzada, por decisión) |
| C | 3 nodos de maestro → `sheetName` dinámico (`Detalle`+sufijo) | ✅ Aplicada |
| D | 42 nodos operativos → `sheetName` dinámico | ✅ Aplicada (35 sesiones + 7 comprobantes) |
| E | Crear pestañas nuevas + pruebas por sucursal | ⬜ Manual |

> **Detalle Fase A (aplicada):** se insertó el nodo Code **`Resolver Sucursal`** entre `Buscar autorizado` y `Es autorizado?`. Lee la columna `sucursal` del transportista, la mapea con la variable `SUCURSALES` y expone `sucursal_codigo` y `sucursal_sufijo` (con fallback a principal si el código es desconocido). No modifica el nodo `Buscar autorizado`, así que las referencias existentes `$('Buscar autorizado')` siguen válidas.

#### 🛠️ Pasos MANUALES en n8n para la Fase A
1. **Crear variable** en *Settings → Variables*: `SUCURSALES` = `COM:|LP:_LP|CBB:_CBB`
2. **Añadir columna `sucursal`** en la pestaña `autorizados` del documento de control.
3. **Rellenar** esa columna para cada transportista con su código: `COM`, `LP` o `CBB`. (Vacío o desconocido = se trata como Principal/COM.)
4. Importar el JSON de esta fase y verificar que el flujo sigue corriendo normal (Principal no cambia su comportamiento porque su sufijo es vacío).

> **Detalle Fase B (aplicada):** los 3 nodos `Normalizar Guia` / `Normalizar Guia1` / `Normalizar Guia2` ahora detectan **cualquier** prefijo definido en la variable `SUCURSALES` (no solo COM), mediante una regex construida dinámicamente y ordenada por longitud de prefijo (evita choques tipo C vs CBB). Reconstruyen la guía con el prefijo detectado (`${prefijo}-${numeroPadded}`, 16 dígitos) y exponen dos campos nuevos: **`sucursal_guia`** (prefijo detectado) y **`sufijo_guia`** (sufijo de hoja correspondiente).
> **Decisión:** NO se valida que el prefijo coincida con la sucursal del transportista. Los datos se enrutan según el **prefijo de la guía**, no según la sucursal registrada del usuario. Implicación: si un transportista de LP envía una guía `COM-`, se procesará en las hojas de Principal. → Por eso en la Fase D, el sufijo de hoja para operaciones ligadas a una guía debe tomarse de `sufijo_guia` (de la normalización), no del transportista.

> **Detalle Fase C (aplicada):** los 3 nodos que leen el maestro (`buscamos la guia`, `buscamos la guia1`, `Leer Liquidaciones`) ahora usan `sheetName` dinámico: `={{ "Detalle" + ($('Resolver Sucursal').item.json.sucursal_sufijo || '') }}`. Resuelven a `Detalle` / `Detalle_LP` / `Detalle_CBB` según la sucursal del transportista. **Nota:** estos nodos pasaron de `gid=0` a nombre dinámico (avanza parcialmente la Mejora #1 para estos 3). **Decisión Fase D confirmada:** la fuente única del sufijo es el transportista (`Resolver Sucursal.sucursal_sufijo`); como transportista = 1 sucursal, toda su actividad se guarda en las hojas de su sucursal sin importar el prefijo que escriba.

> **Detalle Fase D (aplicada):** se enrutaron **42 nodos** operativos a `sheetName` dinámico usando `={{ "sesiones" + ($('Resolver Sucursal').item.json.sucursal_sufijo || '') }}` (y equivalente para `comprobantes`): **35** de sesiones + **7** de comprobantes. Verificado: 0 nodos quedaron con `gid` crudo, y los 42 son alcanzables desde `Resolver Sucursal` (referencia siempre válida). **NO se tocaron** (correcto): `Buscar autorizado`, `Update row in sheet` (ambos hoja `autorizados`), `Leer referencias bancos` (referencias), y los 3 de maestro (ya dinámicos en Fase C). Total de nodos con hoja dinámica en el flujo: **45** (35 sesiones + 7 comprobantes + 3 maestro).
>
> **Bonus:** la Fase D también convirtió todos esos `sheetName` de `gid` numérico a nombre dinámico, completando de facto la Mejora #1 (pospuesta) para sesiones, comprobantes y maestro. Quedan con `gid`/número solo los no afectados: `autorizados` (gid 700117647) y `referencias bancos` (1288535529).

#### 🛠️ Pasos MANUALES para probar el flujo completo multisucursal
1. Crear las pestañas nuevas con **encabezados idénticos** a las actuales (recomendado: duplicar pestaña y borrar datos):
   - `sesiones_LP`, `sesiones_CBB` → mismos encabezados que `sesiones`
   - `comprobantes_LP`, `comprobantes_CBB` → mismos encabezados que `comprobantes`
   - `Detalle_LP`, `Detalle_CBB` → mismos encabezados que `Detalle` (incluida `SER_IMPREG`)
2. Verificar variable `SUCURSALES = COM:|LP:_LP|CBB:_CBB` y columna `sucursal` en `autorizados` rellenada.
3. Importar el JSON con Fase D.
4. Probar de punta a punta: 1 transportista COM (no-regresión), 1 LP y 1 CBB (registrar guía → comprobante → cierre), confirmando que cada uno escribe en SUS hojas.

**Encabezados de referencia:**
- `sesiones*`: estado, phone, guia, liquidacion_total, total_recaudado, total_restante, fecha_inicio, reintentos, ultimo_movimiento, transportista, foco_ant, s_fecha, sesion_foco
- `comprobantes*`: referencia, monto, fecha, hora, tipo_pago, banco_billetera, banco_origen, cuenta_destino, cuenta_origen, nombre_depositante, nombre_beneficiario, agencia, legible, fecha_registro, monto_confianza, tipo_pago_confianza, referencia_confianza, fecha_confianza, phone, guia, link_img, estado_user, redondeo, s_fecha, advertencias, validacion

> **Detalle Fase D+ (COM hardcodeado en lógica) — aplicado:** se detectaron y generalizaron **6 nodos** que tenían el prefijo `COM` fijo en su lógica (no en textos de ejemplo):
> - **`IS GUIA?`, `IS GUIA?1`, `IS GUIA?2`** (críticos, detenían el flujo de LP/CBB): la condición `/COM-\d{6,}/` se reemplazó por una expresión que construye la regex desde `$vars.SUCURSALES` (acepta cualquier prefijo, ordenados por longitud, conserva el mínimo de 6 dígitos). Validado con 8 casos.
> - **`Transformar para reporte`, `Transformar para reporte1`, `Preparar informe actual`** (reportes/informes admin): la regex `/COM-0*(\d+)/` pasó a `/([A-Z]+)-0*(\d+)/`, capturando la serie en el grupo 1 y el número en el grupo 2. **Se corrigieron los índices de grupo corridos** (`matchGuia[1]`→`[2]` para el número; `Serie` ahora dinámica), evitando un bug que habría puesto la serie donde iba el número.
> - **`ANALIZAR CON GROQ1`** (extracción de guía por imagen): el prompt ahora indica que la serie puede ser COM/LP/CBB y pide devolver la serie exacta detectada (antes solo decía 'COM').
>
> **⚠️ Punto de mantenimiento:** el prompt de `ANALIZAR CON GROQ1` lista los prefijos COM/LP/CBB en texto plano. Si en el futuro cambias los prefijos en la variable `SUCURSALES`, recuerda actualizar también ese prompt a mano (es el único punto que no lee la variable automáticamente).
>
> **Nota:** los textos de EJEMPLO para el usuario (`MGenerico, Pedir guia`, `Sesión cancelada`, etc.) siguen mostrando `Ejemplo: COM-...`. No rompen nada (son ilustrativos); se pueden generalizar opcionalmente más adelante.

---

## 11. ⚠️ RIESGO CONOCIDO — Anti-duplicado de comprobantes entre sucursales

**Estado:** abierto / pendiente de decisión (a la espera de confirmación del área de Finanzas).

**Descripción del problema:** al dividir el registro de comprobantes por sucursal (`comprobantes`, `comprobantes_LP`, `comprobantes_CBB`), el control anti-duplicado dejó de ser global. Hoy la verificación "¿ya existe esta referencia?" se hace **solo dentro de la hoja de la sucursal activa**. Por tanto, el **mismo comprobante puede registrarse hasta 3 veces**, una por cada sucursal, iniciando sesión como transportista de cada una. Confirmado mediante prueba real (mismo comprobante aceptado en las 3 hojas).

**Probabilidad en la práctica:** baja por accidente (las sucursales son geográficamente distantes y los transportistas distintos). El escenario relevante es el **fraude deliberado** (reusar un comprobante en dos sucursales), no la colisión casual.

**Por qué NO se implementó el bloqueo global todavía:** la empresa opera con **7 bancos distintos**, y no está confirmado si el número de **referencia** de un comprobante es único globalmente o puede repetirse legítimamente entre bancos. Implementar un bloqueo global "solo por referencia" sin esa confirmación arriesga **rechazar comprobantes legítimos** (dos pagos reales de bancos distintos con la misma referencia). Eso sería peor que el problema actual.

**Tarea pendiente (acción del usuario):** consultar a Finanzas → ¿la referencia es única en todo el sistema, o la unicidad real es `referencia + banco` (o `referencia + cuenta_destino`)?

**Plan recomendado cuando se confirme (Opción 3 — hoja índice global):**
- Crear una hoja única `comprobantes_index` (no dividida) que registre solo la **clave de unicidad** confirmada (referencia, o referencia+banco) + sucursal + fecha de cada comprobante aceptado.
- El nodo anti-duplicado consulta esa hoja única (1 sola lectura → mantiene la velocidad actual) en lugar de la hoja por sucursal.
- Los datos completos del comprobante siguen guardándose en la hoja de su sucursal.
- Ventajas: unicidad global, escala a N sucursales sin cambios, registro central para auditoría.
- Alternativas evaluadas y descartadas por ahora: verificar en las 3 hojas antes de registrar (más lecturas, no escala); volver comprobantes a una sola hoja con columna `sucursal` (deshace la Fase D ya probada).

**Mientras tanto:** el flujo queda operativo tal cual (v8). Riesgo aceptado a corto plazo de forma consciente.

> **Detalle — Ejemplos de guía dinámicos en mensajes WhatsApp (aplicado):** se actualizaron los 3 mensajes que mostraban un ejemplo de guía fijo con COM, para que muestren el prefijo de la sucursal del transportista (`$('Resolver Sucursal').item.json.sucursal_codigo`, fallback COM):
> - `Mensaje no se encontro guia` → ejemplo `<PREFIJO>-114320`
> - `MGenerico, Pedir guia` → `Ejemplo: <PREFIJO>-0000000000114320`
> - `Sesión cancelada` → `Ejemplo: <PREFIJO>-0000000000114320`
> Los 3 son alcanzables desde `Resolver Sucursal`, así que la referencia siempre resuelve.
>
> Además, se completó la generalización del prompt de **`ANALIZAR CON GROQ1`**: las instrucciones internas que aún decían "código COM" ahora dicen "código de guía" / "serie detectada (COM, LP o CBB)". Quedan menciones de COM solo en los EJEMPLOS ilustrativos del prompt (correcto, muestran las series posibles). **Recordatorio de mantenimiento:** este prompt sigue listando los prefijos en texto plano; si cambian en `SUCURSALES`, actualizarlo a mano.

---

## 12. MEJORA GRANDE — Reportes admin avanzados (multisucursal + rango temporal)

**Pedidos del área admin:**
1. Fix: el nodo Groq del admin a veces devuelve comillas → rompe el mensaje interactivo WA.
2. Variable global `REPORTES_MULTISUCURSAL` (true/false) para activar reportes de LP/CBB.
3. Tras elegir tipo de reporte, si la variable es true → menú de sucursal (Santa Cruz/COM, La Paz/LP, Cochabamba/CBB); si es false → usar COM por defecto.
4. Tras tipo+sucursal → menú de rango temporal: Día actual / Rango de fechas (escrito) / Historial completo. Todo por correo (el Informe sigue por WA).

**Decisiones de diseño (acordadas):**
- Fix comillas: **sanitizar la salida** (no depender del prompt).
- Encadenado de menús: **codificar todo en el `buttonId`** (sin estado), salvo el rango de fechas escrito.
- Rango de fechas: el admin **escribe** el rango (`dd/mm/aaaa a dd/mm/aaaa`); para recuperar tipo+sucursal al recibir el texto, se guarda **mínimamente** la selección en `autorizados` (columnas `contexto`/`ult_mss` o nuevas).
- Opciones de rango: Día actual, Rango de fechas, Historial completo.

**Plan por fases:**
| Fase | Contenido | Estado |
|---|---|---|
| 1 | Fix comillas: nodo `Sanitizar respuesta admin` | ✅ Aplicada |
| 2 | Variable `REPORTES_MULTISUCURSAL` + menú de sucursal condicional | ✅ Aplicada en JSON (v11) — diseño con lecturas dedicadas |
| 3 | Menú de rango temporal + filtrado por fechas + rango escrito | ✅ Aplicada en JSON (v12) |

> **Detalle Fase 1 (aplicada):** se insertó el nodo Code **`Sanitizar respuesta admin`** entre `Basic LLM Chain1` y `Botón reporte del día`. Limpia `output[0].texto` (quita comillas dobles/simples rectas y tipográficas, normaliza saltos de línea, limita a ~1000 chars, y pone un texto de respaldo si queda vacío) y expone `texto_limpio`. El mensaje interactivo `Botón reporte del día` ahora usa `texto_limpio` en lugar de `output[0].texto`. El `contexto` que guarda `Update row in sheet` sigue leyéndose del chain (sin cambios). Esto elimina el error de formato sin depender de que el modelo respete la regla de no usar comillas.

> **Detalle Fase 2 (plano, no JSON):** al construir la Fase 2 se detectó que las lecturas `Leer comprobantes del día`, `Leer sesiones` y `Leer Liquidaciones` están **compartidas** entre la rama operativa (transportista) y la de reportes (admin), cada una con fuente de sufijo distinta; además, una operación de lectura de Sheets **borra** los campos del item (se pierde `reporte_sufijo`). Para evitar cruces frágiles, el diseño correcto es **lecturas dedicadas** para reportes (duplicar los nodos de lectura) en vez de compartirlas. Por la cantidad de reconexiones y el riesgo de cablear mal a ciegas, se entregó un **plano de implementación detallado** (`Plano_Fase2_Menu_Sucursal.md`) para armar en el editor visual, en lugar de JSON. La base estable sigue siendo `v10` (Fase 1 aplicada). **Fase 3 (rango temporal) queda pendiente hasta completar la 2.**

> **Detalle Fase 2 (aplicada en JSON v11):** se implementó el menú de sucursal condicional. Nodos nuevos: 3 IF `¿Multisuc? COMP/SES/INF` (evalúan `$vars.REPORTES_MULTISUCURSAL`), 3 menús WA `Menú sucursal COMP/SES/INF` (botones REP_<tipo>_<COM/LP/CBB>), 3 Code `Sufijo reporte COMP/SES/INF` (calculan `reporte_sufijo` desde el buttonId), y 4 lecturas **dedicadas** (`Leer comprobantes (reporte)`, `Leer sesiones (reporte)`, `Leer sesiones (informe)`, `Leer Liquidaciones (informe)`) que NO comparten con la rama operativa. `Switch2` se amplió a 7 salidas (REPORTE, INF_ACTUAL, SES_PRIMERO, GUIA, REP_COMP, REP_SES, REP_INF). Las cadenas de transformación (`Transformar para reporte`, `Preparar informe actual`) usan referencias **tolerantes** (try/catch: lectura dedicada si existe, original si no), por lo que sirven a ambas rutas sin romper la operativa. Validado: lecturas dedicadas alcanzables desde sus nodos de sufijo; las 3 cadenas llegan a su generación final. **Total: 190 nodos.**
>
> **🛠️ Manual:** crear la variable `REPORTES_MULTISUCURSAL` (empezar en `false`). Con `false` el comportamiento es idéntico a hoy (Santa Cruz, sin menú). Probar: (a) false → cada reporte como hoy; (b) true → aparece menú de sucursal, elegir LP/CBB lee de sus hojas; (c) transportista operativo → sin afectación (usa lecturas originales).
> **Pendiente:** Fase 3 (rango temporal: día / fechas escritas / historial).

> **Detalle Fase 3 (aplicada en JSON v12):** rango temporal Día / Rango de fechas / Historial completo. Componentes nuevos:
> - Se **quitó el filtro de fecha** del nodo Sheets en las lecturas dedicadas (ahora traen todo) y el filtrado se hace **en código**.
> - 3 menús de rango (`Menú rango COMP/SES/INF`) que aparecen tras elegir sucursal; el botón codifica `RNG_<tipo>_<suc>_<DIA|RANGO|HISTORIAL>`.
> - 3 nodos `Filtrar fecha COMP/SES/INF`: deducen el modo del `buttonId` o (rama escrita) del texto + contexto; filtran por la columna `s_fecha` (`yyyy-MM-dd`). DIA = hoy; RANGO = `dd/mm/aaaa a dd/mm/aaaa`; HISTORIAL = todo.
> - Rama de **fechas escritas**: `¿Es RANGO?` → guarda `PENDIENTE_RANGO|tipo|suc` en `autorizados.contexto` (`Guardar contexto reporte`) y pide las fechas (`Pedir fechas rango`). Cuando el admin escribe, `¿Respuesta de fechas?` (en la rama de texto admin, antes de `Basic LLM Chain1`) detecta formato de fecha + contexto pendiente → `Router reporte por fechas` recupera tipo+suc → `Switch tipo (fechas)` → lectura dedicada → filtro → generación.
> - Las lecturas dedicadas usan referencia **tolerante** al sufijo (prueba `Sufijo reporte X`, luego `Router reporte por fechas`).
> - Validado: todas las rutas (menú y fechas escritas) llegan a Excel/informe, sin huérfanos. **205 nodos.**
>
> **⚠️ PUNTOS DE PRUEBA PRIORITARIOS (riesgo conocido):**
> 1. **Formato de `s_fecha`**: el filtro asume `yyyy-MM-dd`. Verificar que comprobantes Y sesiones tengan esa columna con ese formato; si no, el rango devuelve vacío. Ajustar el filtro si el formato real difiere.
> 2. **Detección de fechas escritas**: toca la rama de entrada de texto del admin. Probar que un texto normal del admin (no-fecha) siga yendo al chat (`Basic LLM Chain1`) y solo las fechas con contexto pendiente se desvíen.
> 3. **Encadenamiento multi-ejecución**: tipo→sucursal→rango son 3 mensajes/ejecuciones; el `contexto` en `autorizados` debe persistir entre ellas (limpiarlo tras generar para no reusarlo).
> 4. **Error de formato de fecha**: el filtro emite `_error_fechas:true` si el texto no parsea; actualmente no hay rama que avise al admin — considerar añadir un mensaje "formato inválido".
>
> **🛠️ Manual:** ninguna variable nueva (reusa `SUCURSALES`, `REPORTES_MULTISUCURSAL`). Asegurar columna `contexto` en `autorizados`.

> **Correcciones Fase 3 (v13, tras pruebas del usuario):**
> 1. **Excel de comprobantes**: el usuario restauró su formato original de encabezados (Transaccion, Serie, Guia, Documento, Importe, Percepcion, Redondeo, Faltante, Robo, Total, Tipo Pago, Banco, Cuenta, Fecha, Comprobante, Deposito, etc.) en `Transformar para reporte`. ✅ resuelto por el usuario.
> 2. **Filtro de rango devolvía vacío**: los nodos `Filtrar fecha *` ahora extraen las fechas con regex global `\d{1,2}/\d{1,2}/\d{4}` (soporta separadores "a", "al", "-", "hasta", y prefijos "del/desde"), fuerzan modo RANGO cuando llega texto con fecha, ordenan desde/hasta si vienen al revés. El formato de `s_fecha` se confirmó como `yyyy-MM-dd` (coincide con la comparación).
> 3. **Mensaje WA del historial decía "no encontró registros" (pese a enviar el correo)**: `Preparar resumen WA` detectaba el tipo solo con `buttonId === 'INFORME_SESIONES'`, que falla con los botones nuevos `REP_*`/`RNG_*`. Detección corregida para reconocer `INFORME_SESIONES`, `REP_SES_*`, `RNG_SES_*` y contexto `PENDIENTE_RANGO|SES|`. Validado con 8 casos.
>
> **Aún pendiente de afinar (no bloqueante):** limpiar `autorizados.contexto` tras generar un reporte por rango (para no reusarlo); añadir mensaje al admin si el formato de fecha es inválido (`_error_fechas`).

> **Corrección informe-actual (v13):**
> 1. **Alcance del informe**: el informe-actual mantiene el menú de **sucursal** (si `REPORTES_MULTISUCURSAL=true`) pero NO el menú de rango temporal ni histórico — esos son exclusivos de los reportes por correo. Se sacó el informe del flujo de rango: `Switch2[INFORME_ACTUAL] → ¿Multisuc? INF → (sucursal o directo) → Sufijo INF → Leer sesiones (informe) → Leer Liquidaciones (informe) → Preparar informe actual`. Se eliminó la regla `RNG_INF` de Switch2.
> 2. **Filtro por día (corrige el desborde de 4096)**: el informe ahora toma la fecha del día de la columna **`FE`** del maestro (primera fila con dato), la convierte de `dd/mm/aaaa` a `aaaa-mm-dd`, y filtra las sesiones por `s_fecha == fechaDía`. Así solo procesa las sesiones del día (pendientes/en curso/completadas de hoy), no el histórico acumulado. Esto hace imposible superar el límite de WhatsApp. Validado con datos reales (FE=06/06/2026 excluye correctamente sesiones de fechas anteriores).

> **Corrección reporte de comprobantes (v14):** `Transformar para reporte` devolvía el historial completo ignorando el filtro de fecha. Causa: tomaba los comprobantes con `$('Leer comprobantes (reporte)').all()` (la lectura CRUDA, sin filtrar) en lugar de los datos ya filtrados. Como entre el filtro y la transformación hay otra lectura (`Leer referencias bancos`) que reemplaza el item, no se podía usar `$input`. Solución: `Transformar para reporte` ahora toma los comprobantes desde `$('Filtrar fecha COMP').all()` (filtrados), con fallback a la lectura cruda. (`Transformar para reporte1` de sesiones ya funcionaba porque usa `$input.all()` directo del filtro.) Validado: `Filtrar fecha COMP` es ancestro de `Transformar para reporte`, cadena llega a Excel.

> **Corrección nombres de reporte (v15):** el nombre del Excel, el asunto y el mensaje del correo detectaban el tipo solo con `buttonId === 'INFORME_SESIONES'` → con los botones nuevos siempre decían "Liquidaciones". Se unificó una lógica (IIFE) en los 3 puntos que:
> - **Tipo**: Sesiones si `INFORME_SESIONES` / `REP_SES_*` / `RNG_SES_*` / contexto `PENDIENTE_RANGO|SES|`; si no, Liquidaciones.
> - **Periodo**: "del día dd/MM/yyyy" (DIA), "del <d1> al <d2>" (RANGO, fechas del texto), "Historial completo" (HISTORIAL).
> Resultado: nombre de archivo `Reporte_de_<Tipo>_<periodo>.xlsx`, asunto `Reporte de <Tipo> - <periodo>`, y mensaje coherente. Validado en 8 escenarios (día/rango/historial × sesiones/liquidaciones).

> **Corrección estado residual de reportes (v15) — bug de contexto sucio:** al pedir sesiones por rango y luego comprobantes, el segundo reporte heredaba tipo/periodo del anterior (decía "sesiones", traía histórico). **Causa:** el `contexto` (`PENDIENTE_RANGO|tipo|suc`) se guardaba en `autorizados` para recuperar la selección al recibir las fechas escritas, pero **nunca se limpiaba**; como `Resolver Sucursal` lo lee al inicio de cada mensaje, los reportes siguientes lo heredaban. **Triple fix:**
> 1. **Nombres (Excel/asunto/mensaje)**: el contexto solo se considera si el mensaje actual es respuesta de fechas real (sin `buttonId` + con fechas en el texto). Si viene por botón, el `buttonId` manda y el contexto viejo se ignora.
> 2. **Filtros de fecha**: misma regla — modo RANGO por contexto solo si es respuesta de fechas real.
> 3. **Limpieza**: nuevo nodo `Limpiar contexto reporte` (rama paralela de `Router reporte por fechas`) que vacía `autorizados.contexto` al consumir la respuesta de fechas.
> Validado con la secuencia exacta del bug: con contexto sucio, los reportes por botón ya no se contaminan.

> **Corrección (v16) — separar reportes del campo `contexto` del agente:** el campo `contexto` de `autorizados` es de uso EXCLUSIVO de la memoria conversacional del agente Groq (lo escribe `Update row in sheet` desde `Basic LLM Chain1`). El flujo de reportes lo estaba reutilizando indebidamente para el rango pendiente, lo que podía corromper/borrar la memoria de Groq en medio de una conversación. **Solución:** se creó una columna dedicada **`rep_pendiente`** en `autorizados` para el estado de reportes por rango. Todos los nodos de reporte (`Guardar rep_pendiente`, `Limpiar rep_pendiente`, filtros, nombres, resumen WA, router de fechas) ahora usan `rep_pendiente`; el campo `contexto` quedó intacto y exclusivo del agente. Verificado: el único uso de `contexto` es el del agente Groq.
>
> **🛠️ Manual:** crear la columna **`rep_pendiente`** en la pestaña `autorizados` (puede ir vacía; el flujo la gestiona).
