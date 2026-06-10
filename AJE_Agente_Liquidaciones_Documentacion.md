# 📋 Documentación — AJE Agente Liquidaciones (Multisesiones)

> Workflow n8n para gestión automatizada de liquidaciones de transportistas vía WhatsApp, con análisis de comprobantes por IA (visión), soporte multisesión, soporte multisucursal y panel administrativo avanzado.

---

## 1. Resumen general

| Atributo | Valor |
|---|---|
| **Nombre** | AJE - Agente Liquidaciones (Multisesiones) |
| **Estado** | `active: true` |
| **Total de nodos** | ~205 (flujo base 168 + mejoras multisucursal + reportes avanzados) |
| **Canal de entrada** | WhatsApp Business (Meta Graph API v19.0) |
| **Motor IA** | Groq — `llama-3.1-8b-instant` (texto) + modelo de visión vía `$vars.IA_MODEL_VISION` |
| **Persistencia** | Google Sheets (3 spreadsheets, 13 pestañas activas) + Google Drive (imágenes) |
| **Reportería** | Excel (`spreadsheetFile`) + Gmail |
| **Error Workflow** | `AJE - Notificador de Errores` (workflow separado, debe estar activo) |

### Composición aproximada por tipo de nodo

> ⚠️ Los conteos exactos cambian con cada fase de mejoras. Los valores abajo son orientativos al estado v16.

| Tipo | Aprox. | Uso |
|---|---:|---|
| `googleSheets` | ~58 | Lectura/escritura de sesiones, comprobantes, autorizados, maestro, prompt, reporte |
| `if` | ~40 | Bifurcaciones de lógica |
| `whatsApp` | ~38 | Envío de mensajes al transportista/admin |
| `code` | ~22 | Mapeo, validación, normalización, parsing IA, ensamble de prompt, filtros de fecha |
| `httpRequest` | ~14 | Groq, descarga de imágenes, mensajes interactivos WA |
| `googleDrive` | 8 | Carpetas por día y guardado de comprobantes |
| `chainLlm` (LangChain) | 3 | Clasificador de intención + agente conversacional + agente admin |
| `switch` | ~5 | Enrutamiento por intención / tipo de mensaje / tipo de reporte |
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
| `SHEET_ID_CONTROL` | Spreadsheet de control (sesiones, comprobantes, autorizados, prompt_comprobante) |
| `SHEET_ID_MAESTRO` | Spreadsheet maestro de liquidaciones (`Detalle` y variantes por sucursal) |
| `SHEET_ID_REPORTE` | Spreadsheet de referencias para reporte (bancos) |
| `WA_PHONE_ID` | Phone Number ID de WhatsApp Business |
| `NOM_AGENTE` | Nombre del agente conversacional |
| `IA_MODEL_VISION` | Modelo de visión usado en Groq |
| `TOLERANCIA_BS` | Tolerancia en bolivianos para cierre de liquidación (default 1.00) |
| `DRIVE_FOLDER_COMPR` | Carpeta raíz de comprobantes en Drive |
| `DRIVE_FOLDER_RECHA` | Carpeta de rechazados/no legibles |
| `SUCURSALES` | Mapa de sucursales: `COM:\|LP:_LP\|CBB:_CBB` (formato `PREFIJO:SUFIJO_HOJA`) |
| `REPORTES_MULTISUCURSAL` | `true`/`false` — activa el menú de selección de sucursal en reportes admin |
| `NOMBRES_AJE` | Nombres válidos de la empresa para validación (separados por coma) |
| `CUENTAS_AJE` | Números de cuenta AJE para validación (separados por coma) |

---

## 3. Credenciales requeridas

| Servicio | Credencial |
|---|---|
| WhatsApp Trigger | `whatsAppTriggerApi` — "WhatsApp OAuth account" |
| WhatsApp Send | `whatsAppApi` — "WhatsApp account" |
| Google Sheets | `googleSheetsOAuth2Api` — "Google Sheets account" (id `FaAiIXMHXGGQkpkt`) — **todos los 58+ nodos de Sheets usan esta única credencial** |
| Google Drive | `googleDriveOAuth2Api` — "Google Drive OAuth2 API" |
| Gmail | `gmailOAuth2` — "Gmail OAuth2 API" |
| Groq | `groqApi` — "Groq account" |

---

## 4. Estructura de datos — Google Sheets

### 4.1 Spreadsheet de CONTROL (`$vars.SHEET_ID_CONTROL`)

#### Pestaña `sesiones` / `sesiones_LP` / `sesiones_CBB`
Núcleo del estado. Una pestaña por sucursal con encabezados idénticos:

| Columna | Descripción |
|---|---|
| `estado` | ACTIVO / PENDIENTE / CONFIRMADO / COMPLETADO |
| `phone` | Teléfono del transportista |
| `guia` | Código de guía normalizado `PREFIJO-0000000000XXXXXX` |
| `liquidacion_total` | Monto total a liquidar (del maestro) |
| `total_recaudado` | Acumulado de comprobantes válidos |
| `total_restante` | Saldo pendiente |
| `fecha_inicio` | Inicio de la sesión |
| `reintentos` | Contador de intentos fallidos (límite controlado) |
| `ultimo_movimiento` | Timestamp última actividad |
| `transportista` | Nombre del perfil WA |
| `foco_ant` | Sesión anterior en foco (multisesión) |
| `s_fecha` | Fecha de sesión (`yyyy-MM-dd`) |
| `sesion_foco` | Marca de sesión activa en foco |

#### Pestaña `comprobantes` / `comprobantes_LP` / `comprobantes_CBB`
Datos extraídos del comprobante por IA + validación. Una pestaña por sucursal con encabezados idénticos:

`referencia`, `monto`, `fecha`, `hora`, `tipo_pago`, `banco_billetera`, `banco_origen`, `cuenta_destino`, `cuenta_origen`, `nombre_depositante`, `nombre_beneficiario`, `agencia`, `legible`, `fecha_registro`, `monto_confianza`, `tipo_pago_confianza`, `referencia_confianza`, `fecha_confianza`, `phone`, `guia`, `link_img`, `estado_user`, `redondeo`, `s_fecha`, `advertencias`, `validacion`.

#### Pestaña `autorizados`
Control de acceso y memoria del agente admin.

| Columna | Descripción |
|---|---|
| `phone` | Teléfono del usuario |
| `nombre` | Nombre del transportista/admin |
| rol | `ADMIN` o `OPERATIVO` |
| `sucursal` | Código de sucursal del transportista (`COM`, `LP`, `CBB`) — ancla la sucursal |
| `contexto` | Memoria conversacional del agente admin Groq (**uso exclusivo del agente**, no mezclar) |
| `ult_mss` | Último mensaje procesado |
| `rep_pendiente` | Estado pendiente de reporte por rango de fechas (exclusivo del flujo de reportes) |

> ⚠️ `contexto` y `rep_pendiente` son campos de uso exclusivo y separado. `contexto` es escrito únicamente por `Update row in sheet` desde `Basic LLM Chain1`. `rep_pendiente` es escrito/leído/limpiado únicamente por los nodos de reporte.

#### Pestaña `prompt_comprobante`
Fuente de las reglas de extracción del prompt de visión (Zona B dinámica). Columnas:

`id`, `campo`, `orden`, `regla`, `activo`, `creado_por`, `fecha_creacion`, `modificado_por`, `fecha_modificacion`, `nota`

- `campo`: identifica el campo de extracción (MONTO, REFERENCIA, FECHA, etc.)
- `orden`: enteros en incrementos de 10 para permitir inserción futura sin reordenar
- `activo`: `TRUE`/`FALSE` — controla si la regla se incluye en el prompt
- Solo las reglas con `activo=TRUE` y `regla` no vacía son ensambladas por el nodo `Ensamblar prompt`

### 4.2 Spreadsheet MAESTRO (`$vars.SHEET_ID_MAESTRO`)
- **Pestaña `Detalle`** / **`Detalle_LP`** / **`Detalle_CBB`** — fuente de verdad de las guías y su monto total a liquidar. Se consulta en `buscamos la guia`, `Leer Liquidaciones`, `buscamos la guia1`. La pestaña se selecciona dinámicamente según la sucursal del transportista.

### 4.3 Spreadsheet REPORTE (`$vars.SHEET_ID_REPORTE`)
- **Pestaña `referencias bancos`** (gid 1288535529) — catálogo de bancos/cuentas para enriquecer el reporte de comprobantes.

---

## 5. Arquitectura del flujo

### 5.1 Entrada y normalización

```
WhatsApp Trigger ("User envia mensaje")
   └─> Filtrar eventos validos        (descarta status updates / eventos vacíos)
        └─> Mapeamos datos del mensaje (extrae phone, tipo, texto, imageId, buttonId,
            messageId, comandos FIN/CERRAR/LISTO/TERMINAR, guiaCierre de botones)
             └─> Buscar autorizado     (valida phone contra hoja 'autorizados')
                  └─> Resolver Sucursal (lee columna 'sucursal', mapea con $vars.SUCURSALES,
                      expone sucursal_codigo y sucursal_sufijo — fallback a COM si desconocido)
                       └─> Es autorizado?
                            ├─ NO  -> NO AUTORIZADO (corta)
                            └─ SÍ  -> Es ADMIN?
```

El nodo **"Mapeamos datos del mensaje"** soporta doble formato de payload (Meta nativo y formato traducido al español), detecta comandos de cierre y extrae `guiaCierre` de botones `COMPLETAR_GUIA_*`. Expone `messageId` para trazabilidad.

### 5.2 Bifurcación por rol

```
Es ADMIN?
 ├─ SÍ (Admin) -> Is message int? (reporte) -> Switch2 (7 salidas)
 │                   ├─ REPORTE        -> ¿Multisuc? COMP -> Menú sucursal / directo
 │                   │                    -> Sufijo COMP -> Leer comprobantes (reporte)
 │                   │                    -> Menú rango COMP -> Filtrar fecha COMP
 │                   │                    -> Leer referencias bancos -> Transformar para reporte
 │                   │                    -> ¿Hay datos hoy? -> Generar Excel -> Gmail -> WA
 │                   ├─ INF_ACTUAL     -> ¿Multisuc? INF -> Menú sucursal / directo
 │                   │                    -> Sufijo INF -> Leer sesiones (informe)
 │                   │                    -> Leer Liquidaciones (informe)
 │                   │                    -> Preparar informe actual -> WA informe actual
 │                   ├─ SES_PRIMERO    -> ¿Multisuc? SES -> Menú sucursal / directo
 │                   │                    -> Sufijo SES -> Leer sesiones (reporte)
 │                   │                    -> Menú rango SES -> Filtrar fecha SES
 │                   │                    -> Transformar para reporte1 -> Excel -> Gmail -> WA
 │                   ├─ GUIA           -> Normalizar Guia2 -> cierre por admin
 │                   ├─ REP_COMP       -> (respuesta menú sucursal COMP) -> Sufijo COMP -> ...
 │                   ├─ REP_SES        -> (respuesta menú sucursal SES)  -> Sufijo SES  -> ...
 │                   └─ REP_INF        -> (respuesta menú sucursal INF)  -> Sufijo INF  -> ...
 │
 │  Rama de texto admin: ¿Respuesta de fechas? (detecta formato fecha + rep_pendiente)
 │   ├─ SÍ -> Router reporte por fechas -> Switch tipo (fechas) -> lectura dedicada -> filtro
 │   │         └─ paralelo: Limpiar rep_pendiente (vacía autorizados.rep_pendiente)
 │   └─ NO -> Sanitizar respuesta admin -> Basic LLM Chain1 (agente + Structured Output Parser)
 │
 └─ NO -> Es OPERATIVO? -> Is message int? -> Switch (4 ramas de sesión/comprobante)
```

### 5.3 Lógica de sesión del transportista (`Switch`)
Cuatro salidas que enrutan según si tiene/recibe sesión o comprobante pendiente:
- **CON_SESION / REC_SESION** → "Buscamos la sesion PENDIENTE"
- **CON_COMPROBANTE / REC_COMPROBANTE** → "Buscamos comprob PENDIENTE"

Maneja confirmaciones interactivas (`acepta?`, `confirma?`), activación de estado (`ESTADO ACTIVO`), cancelación y limpieza de falsas sesiones (`ELIINAMOS FALSA SESION`).

### 5.4 Clasificación de intención (`Switch1`)
Alimentado por el chain **"Intencion del mensaje"** (Groq `llama-3.1-8b-instant`) que devuelve una sola palabra:
- **ESTADO** → consulta de saldo → "Buscamos su liquidacion"
- **CIERRE** → "Cierre de caja"
- **GUIA** → "Normalizar Guia" (inicia/activa sesión)
- **false** → mensaje genérico / agente conversacional

### 5.5 Procesamiento de comprobantes (imagen)

```
Es IMG?
 -> OBTENER ID DE LA IMG
 -> DESCARGAR IMG (Graph API)
 -> CONVERTIR A BASE64
 -> Leer reglas prompt        (lee pestaña 'prompt_comprobante' de $vars.SHEET_ID_CONTROL)
 -> Ensamblar prompt          (Code node: construye ZONA A [fija] + ZONA B [reglas activas
                               del Sheet, agrupadas y ordenadas por campo] + ZONA C [fija].
                               Recupera imageBase64/imageMimeType desde 'CONVERTIR A BASE64'
                               para no perder la imagen al leer el Sheet.
                               Fallback defensivo: si no hay reglas activas, devuelve error
                               sin llamar a Groq.)
 -> ANALIZAR CON GROQ (visión) (usa promptComprobante ensamblado + imagen en base64)
 -> FILTRAR DATOS             (parse defensivo: try/catch en JSON.parse y en acceso a campos;
                               devuelve { error:true, legible:false, motivo_error } si falla,
                               en lugar de lanzar excepción)
 -> Validar datos             (cuentas AJE desde $vars.CUENTAS_AJE, nombres desde $vars.NOMBRES_AJE,
                               fecha futura, banco)
 -> existe comprobante?       (anti-duplicado por referencia — ⚠️ solo dentro de la hoja de
                               la sucursal activa, ver §11)
 -> Buscamos/Creamos carpeta del día (Drive)
 -> Guardamos imagen -> permisos lectura
 -> registrar comprobante
 -> Actualizar sesión con montos
 -> Validar restante liq (tolerancia $vars.TOLERANCIA_BS)
 -> liq completa? -> COMPLETAR LIQ
```

Existe una **segunda cadena espejo** (`ANALIZAR CON GROQ1`, `FILTRAR DATOS1`, `DESCARGAR IMG2/IMG3`, `Es IMG?1`) para flujos paralelos (sesión pendiente / cierre). Hay control de reintentos (`reintentos ++`, `reintentos 0`, `Error limite`) para imágenes no legibles.

> 📐 **Refactor futuro pendiente (Mejora #8):** las 4 cadenas de visión casi idénticas podrían extraerse a un sub-workflow `AJE - SUB - Procesar Comprobante`. Ver diseño detallado en §9.

### 5.6 Arquitectura del prompt de visión (Zona A / B / C)

El prompt enviado a Groq se construye en tres zonas:

| Zona | Contenido | Origen |
|---|---|---|
| **A** | Cabecera del sistema + escala de confianza | Fijo en nodo `Ensamblar prompt` |
| **B** | Reglas de extracción por campo (MONTO, REFERENCIA, FECHA, etc.) | Dinámico desde pestaña `prompt_comprobante` |
| **C** | Esquema JSON de salida + lógica del flag `legible` + umbral de confianza | Fijo en nodo `Ensamblar prompt` |

El umbral `legible` (actualmente `monto.confianza >= 0.7 AND referencia.confianza >= 0.7`) está hardcodeado en la Zona C del nodo `Ensamblar prompt`. Modificarlo requiere editar ese nodo directamente, no la hoja de reglas.

### 5.7 Multisesión (diferenciador del flujo)
Lógica para que un transportista tenga varias guías y se vaya activando la siguiente automáticamente:
- `Desactivar sesiones en foco anteriores`, `Activar sesion de la guia elegida`
- `Buscar siguiente sesion pendiente` (+ variante "(caja)") → `IF ¿Hay siguiente sesion?` → `Activar siguiente sesion` → `WA aviso sesion activada automaticamente`
- `volvemos a sesion anterior`, `tenia sesiones anteriores?` para rollback.

### 5.8 Reportería admin

**Informe actual (por WA):**
`Switch2[INF_ACTUAL]` → selección de sucursal (si `REPORTES_MULTISUCURSAL=true`) → `Leer sesiones (informe)` → `Leer Liquidaciones (informe)` → `Preparar informe actual` (filtra por `s_fecha == hoy`, evita desborde del límite de 4096 chars de WA) → `WA informe actual`.

**Reporte de comprobantes / sesiones (por email):**
`Switch2` → selección de sucursal → `Sufijo reporte` → `Leer [comprobantes|sesiones] (reporte)` → `Menú rango [COMP|SES]` → `Filtrar fecha [COMP|SES]` → generación Excel → `Enviar email con reporte` → `WA confirmación reporte`.

**Rango escrito:** el admin escribe `dd/mm/aaaa a dd/mm/aaaa` → `¿Respuesta de fechas?` detecta el formato + `rep_pendiente` → `Router reporte por fechas` → `Switch tipo (fechas)` → lectura dedicada → `Filtrar fecha` → generación.

---

## 6. Integraciones externas

| Endpoint | Uso |
|---|---|
| `graph.facebook.com/v19.0/{WA_PHONE_ID}/messages` | Envío de mensajes interactivos (botones), múltiples nodos HTTP |
| `api.groq.com/openai/v1/chat/completions` | Análisis de comprobantes por visión + cadenas de texto |
| WhatsApp node nativo | Mensajes de texto simples (múltiples nodos) |

---

## 7. Estado de hallazgos y mejoras

### ✅ Resueltos

| # | Descripción | Detalle |
|---|---|---|
| 1 | Reemplazar `gid` numérico por nombre de pestaña | ⏸️ Pospuesto para nodos de `autorizados` y `referencias bancos`; **aplicado de facto** para todos los demás en las Fases C y D (sesiones, comprobantes, maestro usan nombre dinámico) |
| 2 | ID spreadsheet hardcodeado | ✅ Unificado a `$vars.SHEET_ID_CONTROL` en el nodo "Desactivar sesiones en foco anteriores" |
| 3 | Dos credenciales Google Sheets | ✅ 58+ nodos consolidados en "Google Sheets account" (`FaAiIXMHXGGQkpkt`) |
| 4 | `JSON.parse` de Groq sin defensa | ✅ `try/catch` en `FILTRAR DATOS` y `FILTRAR DATOS1`; devuelven `{error:true, legible:false, motivo_error}` |
| 5 | Idempotencia del trigger (mínima) | ✅ `messageId` expuesto en `Mapeamos datos del mensaje`; deduplicación completa pospuesta (sin casos reales en producción) |
| 6 | Error Workflow global | ✅ Workflow separado `AJE - Notificador de Errores` implementado |
| 9 | CUENTAS_AJE/NOMBRES_AJE hardcodeados | ✅ Movidos a `$vars.CUENTAS_AJE` y `$vars.NOMBRES_AJE` con fallback |
| 10 | Sin documentación interna | ✅ 7 sticky notes agregadas (bloques: Entrada, Roles, Panel Admin, Clasificación, Comprobantes, Multisesión, Reportería) |

### ⬜ Pendientes

| # | Descripción | Prioridad |
|---|---|---|
| 7 | Concurrencia / condiciones de carrera en read→update de `sesiones` | 🟠 Alta |
| 8 | Extraer 4 cadenas de imagen duplicadas a sub-workflow | 🟡 Media (diseño en §9) |
| 11 | Etiquetar rama vacía de `Switch1` | 🟡 Baja |
| — | Erratas en nombres de nodos (`ELIINAMOS FALSA SESION`, `reintentos ` con espacio) | 🟡 Baja — riesgoso sin buscar todas las referencias `$('...')` primero |
| — | Anti-duplicado global de comprobantes entre sucursales | 🔴 Pendiente de consulta a Finanzas (ver §11) |

### ✅ Aspectos bien resueltos (base)
- Centralización de configuración en `$vars`.
- Normalización robusta de guías (`padStart(16,'0')`) y de texto (NFD, quita tildes/SA/SRL).
- Anti-duplicado de comprobantes por referencia (dentro de cada sucursal).
- Tolerancia configurable en Bs para cierre de liquidación.
- Soporte de doble formato de payload de WhatsApp.
- Separación clara de roles (ADMIN / OPERATIVO / no autorizado).
- Lógica de multisesión con activación automática de la siguiente guía.

---

## 8. Diagrama lógico resumido

```
WA Trigger
  → Filtrar eventos → Mapear mensaje → Buscar autorizado → Resolver Sucursal → ¿Autorizado?
       ├─ No → NO AUTORIZADO
       └─ Sí → ¿ADMIN?
             ├─ ADMIN → Switch2 (7 salidas) → {
             │            Reporte COMP (menú suc + rango) → Excel + Gmail
             │            Informe actual (menú suc) → WA
             │            Reporte SES (menú suc + rango) → Excel + Gmail
             │            Cierre guía admin
             │          }
             │          + Agente Admin (LLM + Structured Output Parser)
             │          + Rama fechas escritas → Router → filtro → Excel
             │
             └─ OPERATIVO →
                   ├─ Imagen → Descargar → Base64 → Leer reglas prompt → Ensamblar prompt
                   │           → Groq Visión → Filtrar (defensivo) → Validar
                   │           → Drive → Registrar comprobante → Actualizar saldos
                   │           → ¿Completa? → Cerrar liq → Siguiente sesión auto
                   └─ Texto  → Intención (Groq) → Switch1 →
                                 ESTADO → consulta saldo
                                 CIERRE → cierre de caja
                                 GUIA   → Normalizar Guia → nueva sesión
                                 false  → agente conversacional
```

---

## 9. Backlog de mejoras — estado y detalles

### #4 — `try/catch` en parseo de Groq ✅

Ambos nodos (`FILTRAR DATOS`, `FILTRAR DATOS1`) usan acceso seguro a `choices[0].message.content` y a `.valor`/`.confianza`, con `try/catch` en el `JSON.parse`. Ante cualquier fallo devuelven `{ error:true, legible:false, motivo_error }`.

> ⚠️ **Pendiente de cableado (acción en n8n):** los nodos ahora pueden emitir `error:true`, pero el flujo aún NO enruta esa salida. Agregar un nodo IF después de cada `FILTRAR DATOS` que evalúe `{{ $json.error === true }}` y lo conecte a un mensaje WA tipo *"No pude leer el comprobante, reenvíalo más claro"* (puede reutilizar la rama `reintentos ++` / `Error limite`).

### #5 — Idempotencia del trigger ✅ (mínima)

`messageId` expuesto en `Mapeamos datos del mensaje`. La deduplicación completa (hoja `procesados` + comparación) se pospuso: en semanas de producción no se observó ningún caso de duplicidad. El nodo `Filtrar eventos validos` descarta status updates y tipos no válidos, pero no compara IDs.

### #6 — Error Workflow ✅

Workflow separado `AJE - Notificador de Errores`: `Error Trigger` → `HTTP Request` → WhatsApp al `59163332108`. Campos del mensaje: nombre del workflow, nodo que falló, mensaje de error, hora e ID de ejecución. Usa `$vars.WA_PHONE_ID` y credencial `whatsAppApi`.

> ⚠️ **El JSON del Error Workflow se importa con `active: false`** — debe activarse manualmente.

#### 🛠️ Pasos manuales para activar el Error Workflow
1. **Importar** `AJE_-_Notificador_de_Errores.json` como workflow nuevo.
2. **Verificar** la credencial WhatsApp en el nodo "Notificar Admin por WhatsApp".
3. **Activar** ese workflow (toggle "Active").
4. En el **workflow principal** → menú **⋯ → Settings** → campo **"Error Workflow"** → seleccionar `AJE - Notificador de Errores`. Guardar.
5. Repetir paso 4 en cualquier otro workflow a monitorear.
6. **Probar**: forzar un error y confirmar que llega el WA.

> ⚠️ **Limitación WA:** mensajes de texto libre solo llegan dentro de la ventana de 24h. Para alertas 100% confiables, crear una plantilla (template) Meta aprobada, o mantener la ventana enviando un mensaje al bot cada día.

### #8 — Sub-workflow "Procesar comprobante" 📐 (diseño, refactor futuro)

**Objetivo:** eliminar la duplicación de las 4 cadenas de visión (`DESCARGAR IMG/IMG1/IMG2/IMG3`, `ANALIZAR CON GROQ/GROQ1`, `FILTRAR DATOS/DATOS1`).

**Diseño propuesto:**
1. Workflow nuevo `AJE - SUB - Procesar Comprobante` con `Execute Workflow Trigger`.
   - Inputs: `imageId` (o url), `phone`, `guia`, `transportista`, flag `modo` (`comprobante` | `guia`).
   - Cuerpo: `OBTENER ID` → `DESCARGAR IMG` → `CONVERTIR A BASE64` → `Leer reglas prompt` → `Ensamblar prompt` → `ANALIZAR CON GROQ` → `FILTRAR DATOS`.
   - Output: objeto normalizado (`monto`, `referencia`, `legible`, `error`, etc.).
2. Reemplazar las 4 cadenas por nodos `Execute Workflow`.

**Precauciones:** hacer en entorno de pruebas; pasar url/imageId (no el base64) para no romper el transporte entre workflows; conservar las dos variantes de parser (`FILTRAR DATOS` vs `FILTRAR DATOS1` tienen estructuras distintas).

**Beneficio:** ~12-15 nodos menos, un único punto de mantenimiento para el prompt de visión.

### #9 — CUENTAS_AJE / NOMBRES_AJE a `$vars` ✅

El nodo `Validar datos` lee de `$vars.NOMBRES_AJE` y `$vars.CUENTAS_AJE` (separados por coma). Fallback a valores hardcodeados si la variable no existe.

#### 🛠️ Paso manual opcional en n8n
Crear en *Settings → Variables*:
- `NOMBRES_AJE` = `AJE,INDUSTRIAS AJE,INDUSTRIAS AJE BOLIVIA,AJE BOLIVIA,AJE GROUP,AJEBOLIVIA,INDUSTRIAS AJEBOLIVIA`
- `CUENTAS_AJE` = `7015046986392,1041309287,1311415157`

### #10 — Sticky notes ✅ (parcial)

7 sticky notes agregadas: Entrada, Roles, Panel Admin, Clasificación de intención, Procesamiento de comprobantes, Multisesión, Reportería.

Renombrado de erratas (`ELIINAMOS FALSA SESION`, `reintentos ` con espacio sobrante) **pospuesto**: requiere buscar y reemplazar todas las referencias `$('...')` en el editor visual antes de renombrar.

---

## 10. Soporte multisucursal (COM / LP / CBB)

**Objetivo:** registrar guías, sesiones y comprobantes de 3 sucursales sin mezclar datos, en un solo workflow.

**Decisiones de diseño:**
- Mismos documentos Sheets, **pestañas separadas por sucursal**: `Detalle`/`Detalle_LP`/`Detalle_CBB`; `sesiones`/`sesiones_LP`/`sesiones_CBB`; `comprobantes`/`comprobantes_LP`/`comprobantes_CBB`. Principal sin sufijo.
- Un transportista pertenece a **una sola** sucursal → ancla en columna `sucursal` de `autorizados`.
- Prefijos y sufijos parametrizados en `$vars.SUCURSALES` = `COM:|LP:_LP|CBB:_CBB`.
- La fuente única del sufijo de hoja para operaciones es el **transportista** (`Resolver Sucursal.sucursal_sufijo`), no el prefijo de la guía.

**Fases de implementación:**

| Fase | Contenido | Estado |
|---|---|---|
| A | Variable `SUCURSALES` + columna `sucursal` en `autorizados` + nodo `Resolver Sucursal` | ✅ Aplicada |
| B | 3 nodos `Normalizar Guia` → multi-prefijo vía regex dinámica desde `SUCURSALES` | ✅ Aplicada (sin validación cruzada transportista/guía, por decisión) |
| C | 3 nodos de maestro → `sheetName` dinámico (`Detalle` + sufijo) | ✅ Aplicada |
| D | 42 nodos operativos (35 sesiones + 7 comprobantes) → `sheetName` dinámico | ✅ Aplicada |
| D+ | 6 nodos con prefijo COM hardcodeado en lógica (IS GUIA?, Transformar para reporte, etc.) | ✅ Aplicada |
| E | Crear pestañas nuevas y probar de punta a punta | ⬜ Manual |

> **Detalle Fase A:** nodo Code `Resolver Sucursal` insertado entre `Buscar autorizado` y `Es autorizado?`. Lee `autorizados.sucursal`, mapea con `$vars.SUCURSALES`, expone `sucursal_codigo` y `sucursal_sufijo`. Fallback a COM si el código es desconocido o vacío.

> **Detalle Fase B:** los 3 nodos `Normalizar Guia` detectan cualquier prefijo de `SUCURSALES` mediante regex dinámica ordenada por longitud (evita colisiones, ej. C vs CBB). Exponen `sucursal_guia` y `sufijo_guia`. No se valida que el prefijo coincida con la sucursal del transportista — el enrutamiento de hojas usa el sufijo del transportista, no el de la guía.

> **Detalle Fase D:** 42 nodos usan `={{ "sesiones" + ($('Resolver Sucursal').item.json.sucursal_sufijo || '') }}` (y equivalente para comprobantes). Total con hoja dinámica: **45** (35 sesiones + 7 comprobantes + 3 maestro). Quedan con `gid` hardcodeado solo: `autorizados` (gid 700117647) y `referencias bancos` (1288535529) — ambos no tienen variantes por sucursal.

> **Detalle Fase D+:** los 3 nodos `IS GUIA?/1/2` tenían regex `/COM-\d{6,}/` hardcodeada — se reemplazó por construcción dinámica desde `$vars.SUCURSALES`. Los nodos `Transformar para reporte`, `Transformar para reporte1` y `Preparar informe actual` tenían regex `/COM-0*(\d+)/` — se reemplazó por `/([A-Z]+)-0*(\d+)/` (grupo 1 = serie, grupo 2 = número). El prompt de `ANALIZAR CON GROQ1` actualizado para mencionar COM/LP/CBB.

> ⚠️ **Punto de mantenimiento:** el prompt de `ANALIZAR CON GROQ1` lista los prefijos COM/LP/CBB en texto plano. Si se agregan o cambian prefijos en `$vars.SUCURSALES`, actualizar ese prompt manualmente.

#### 🛠️ Pasos manuales para multisucursal

1. Crear variable `SUCURSALES = COM:|LP:_LP|CBB:_CBB` en *Settings → Variables*.
2. Agregar columna `sucursal` en la pestaña `autorizados` y rellenar por transportista (`COM`, `LP` o `CBB`).
3. Crear las pestañas nuevas (duplicar y borrar datos):
   - `sesiones_LP`, `sesiones_CBB` → mismos encabezados que `sesiones`
   - `comprobantes_LP`, `comprobantes_CBB` → mismos encabezados que `comprobantes`
   - `Detalle_LP`, `Detalle_CBB` → mismos encabezados que `Detalle` (incluida `SER_IMPREG`)
4. Probar: 1 transportista COM (no-regresión), 1 LP, 1 CBB — confirmar que cada uno escribe en sus propias hojas.

---

## 11. ⚠️ Riesgo conocido — Anti-duplicado entre sucursales

**Estado:** abierto / pendiente de decisión de Finanzas.

**Descripción:** al dividir comprobantes por sucursal, el anti-duplicado perdió alcance global. La verificación "¿ya existe esta referencia?" se hace solo dentro de la hoja de la sucursal activa. El mismo comprobante puede registrarse en hasta 3 hojas distintas. Confirmado con prueba real.

**Probabilidad en práctica:** baja por accidente (sucursales geográficamente distantes, transportistas distintos). Riesgo relevante: fraude deliberado.

**Razón del aplazamiento:** los 7 bancos en uso no confirman si el número de referencia es globalmente único. Un bloqueo global solo por referencia podría rechazar comprobantes legítimos de distintos bancos con la misma referencia.

**Tarea pendiente:** consultar a Finanzas si la clave de unicidad es `referencia` sola, o `referencia + banco` / `referencia + cuenta_destino`.

**Plan recomendado (Opción 3 — hoja índice global):**
- Crear hoja `comprobantes_index` con solo la clave de unicidad + sucursal + fecha.
- El anti-duplicado consulta solo esa hoja (1 lectura, escala a N sucursales).
- Los datos completos siguen en la hoja por sucursal.

**Mientras tanto:** riesgo aceptado conscientemente. El flujo queda operativo.

---

## 12. Prompt dinámico — Zona A / B / C

**Objetivo:** permitir ajustar las reglas de extracción de comprobantes sin redeployar el workflow.

**Arquitectura:**
- **Zona A** (fija): cabecera del rol del sistema, contexto boliviano, escala de confianza.
- **Zona B** (dinámica): reglas de extracción por campo, cargadas desde la pestaña `prompt_comprobante`.
- **Zona C** (fija): esquema JSON de salida esperado, campo `legible`, umbral de confianza.

**Flujo en el workflow:**
`CONVERTIR A BASE64` → `Leer reglas prompt` (Sheet) → `Ensamblar prompt` (Code) → `ANALIZAR CON GROQ`.

**Nodo `Ensamblar prompt`:**
1. Filtra reglas con `activo=TRUE` y `regla` no vacía.
2. Agrupa por campo y ordena cada grupo por `orden`.
3. Construye Zona B respetando `ORDEN_CAMPOS`; campos extra van al final.
4. Recupera `imageBase64`/`imageMimeType` desde `$('CONVERTIR A BASE64').item.json` (evita pérdida al leer el Sheet en medio del pipeline).
5. Fallback defensivo: si no hay reglas activas, devuelve `{ error:true, legible:false, motivo_error }` sin llamar a Groq.

**Reglas de confianza:**
El umbral `legible` (`monto.confianza >= 0.7 AND referencia.confianza >= 0.7`) es fijo en Zona C del nodo `Ensamblar prompt`. **Para ajustarlo hay que editar el nodo directamente**, no la hoja de reglas.

**Estrategia para comprobantes de baja calidad (ej. recibos físicos dot-matrix):** subir la tolerancia de confianza en las *reglas de Zona B* del campo específico (ej. Banco Económico), no bajar el umbral global de Zona C.

---

## 13. Reportes admin avanzados (multisucursal + rango temporal)

### Estado general: ✅ Aplicado (v16)

### Componentes implementados

| Componente | Estado |
|---|---|
| Sanitizador salida Groq (`Sanitizar respuesta admin`) | ✅ v10 |
| Variable `REPORTES_MULTISUCURSAL` + menú de sucursal condicional | ✅ v11 |
| Menú de rango temporal (Día actual / Rango de fechas / Historial) | ✅ v12 |
| Correcciones de filtro de rango, detección de tipo, mensaje WA | ✅ v13 |
| Informe actual separado del flujo de rango; filtro por día desde columna FE | ✅ v13 |
| Corrección `Transformar para reporte` tomando datos filtrados | ✅ v14 |
| Nombres dinámicos del Excel (Tipo + Período) en 8 escenarios | ✅ v15 |
| Fix contexto residual entre reportes consecutivos | ✅ v15 |
| Separación `rep_pendiente` vs `contexto` del agente | ✅ v16 |

### Diseño de menús encadenados

```
Tipo de reporte (REPORTE/SES_PRIMERO/INF_ACTUAL)
  └─ [si REPORTES_MULTISUCURSAL=true] Menú de sucursal (COM/LP/CBB)
       └─ [solo COMP y SES] Menú de rango (DÍA ACTUAL / RANGO DE FECHAS / HISTORIAL)
            └─ [si RANGO] Admin escribe "dd/mm/aaaa a dd/mm/aaaa"
                 └─ ¿Respuesta de fechas? → Router → filtro → Excel
```

El `buttonId` codifica toda la selección acumulada (ej. `RNG_COMP_LP_RANGO`). El rango escrito es la única excepción que usa persistencia en `autorizados.rep_pendiente`.

### Notas de implementación

> **Sanitizador Groq (v10):** nodo Code `Sanitizar respuesta admin` entre `Basic LLM Chain1` y `Botón reporte del día`. Limpia comillas dobles/simples rectas y tipográficas, normaliza saltos de línea, limita a ~1000 chars. El campo `contexto` sigue leyéndose del chain original (sin cambios).

> **Lecturas dedicadas de reporte (v11):** las lecturas `Leer comprobantes (reporte)`, `Leer sesiones (reporte)`, `Leer sesiones (informe)`, `Leer Liquidaciones (informe)` son nodos **independientes** de los usados en la rama operativa (transportistas). Esto evita que una lectura de Sheet borre campos de la pipeline de reportes.

> **Filtro de rango (v12/v13):** los nodos `Filtrar fecha COMP/SES/INF` extraen fechas con regex global `\d{1,2}/\d{1,2}/\d{4}` (acepta separadores: "a", "al", "-", "hasta", prefijos "del/desde"), fuerzan modo RANGO cuando llega texto con fecha, ordenan desde/hasta si vienen al revés. El formato de `s_fecha` es `yyyy-MM-dd`.

> **Informe actual (v13):** siempre muestra solo el día de hoy. La fecha se toma de la columna `FE` del maestro (primera fila con dato, convertida de `dd/mm/aaaa` a `aaaa-mm-dd`). No tiene menú de rango temporal. El límite de 4096 chars de WA no se supera.

> **Reporte de comprobantes (v14):** `Transformar para reporte` toma los comprobantes desde `$('Filtrar fecha COMP').all()` (ya filtrados), no desde la lectura cruda. Fallback a la lectura cruda si el nodo de filtro no es ancestro.

> **Nombres dinámicos del Excel (v15):** nombre de archivo `Reporte_de_<Tipo>_<periodo>.xlsx`, asunto y mensaje coherentes. Tipo = Sesiones si `INFORME_SESIONES`/`REP_SES_*`/`RNG_SES_*`/contexto `PENDIENTE_RANGO|SES|`; si no, Liquidaciones. Período = "del día dd/MM/yyyy" (DIA), "del \<d1\> al \<d2\>" (RANGO), "Historial completo" (HISTORIAL). Validado en 8 escenarios.

> **Fix contexto residual (v15):** el `contexto` de `autorizados` se leía al inicio de cada mensaje via `Resolver Sucursal`, lo que contaminaba reportes posteriores con el estado del anterior. Triple fix: (1) nombres/tipo solo consideran contexto si el mensaje es respuesta de fechas real (sin `buttonId` + con fecha en el texto); (2) filtros de fecha con la misma regla; (3) nodo `Limpiar contexto reporte` vacía `autorizados.contexto` al consumir la respuesta de fechas.

> **Separación `rep_pendiente` (v16):** `autorizados.contexto` es de uso EXCLUSIVO del agente Groq. Se creó la columna `rep_pendiente` para el estado de reportes por rango. Todos los nodos de reporte (`Guardar rep_pendiente`, `Limpiar rep_pendiente`, filtros, nombres, resumen WA, router de fechas) usan `rep_pendiente`. El campo `contexto` queda intacto.

> **🛠️ Manual (v16):** crear la columna `rep_pendiente` en la pestaña `autorizados` (puede ir vacía; el flujo la gestiona).

### Puntos de prueba prioritarios

1. **Formato `s_fecha`**: verificar que comprobantes Y sesiones tengan esa columna con formato `yyyy-MM-dd`.
2. **Detección de fechas escritas**: un texto normal del admin (no-fecha) debe seguir yendo al chat (`Basic LLM Chain1`) y solo las fechas con `rep_pendiente` pendiente se desvían.
3. **Encadenamiento multi-ejecución**: tipo → sucursal → rango son 3 mensajes/ejecuciones; `rep_pendiente` debe persistir entre ellas y limpiarse tras generar el reporte.
4. **Error de formato de fecha**: el filtro emite `_error_fechas:true`; actualmente no hay rama que avise al admin — considerar agregar mensaje "formato inválido".

---

## 14. Tareas pendientes en horizonte

| Tarea | Estado |
|---|---|
| Admin CRUD vía WA para gestionar reglas de `prompt_comprobante` (comandos `PROMPT NUEVA`, `PROMPT EDITAR`, `PROMPT BAJA`, etc.) | Diseñado, **explícitamente diferido** |
| Anti-duplicado global de comprobantes | Pendiente consulta a Finanzas (ver §11) |
| Ajuste del umbral `legible` si reglas de banco específico no resuelven los rechazos | Diferido a resultados de pruebas |
| Protección de concurrencia en read→update de `sesiones` (Mejora #7) | Pendiente |
| Cableado de la salida `error:true` de `FILTRAR DATOS` hacia rama de reintento WA | Pendiente (ver detalle en §9 / Mejora #4) |
