# Pase a Producción de un TIS (Ticket Seguro)

Workflow para llevar un ticket NSPUM / NSPRM desde desarrollo terminado hasta la bandeja del explotador, que instala en producción.

Documentación de referencia: [Proceso de pase a producción de un TIS](https://cnsv.atlassian.net/wiki/spaces/BS/pages/1758199923). **Ojo:** el paso "A QA FUNCIONAL PROD (CHECKLIST)" con checklist manual que describe el documento **ya no se usa**. Lo reemplazan QAF-Automation-Manual y la transición "A QA Técnico Automático".

Ejemplo real: NSPUM-659 (pauta PAUT-76655).

`cloudId` para todas las llamadas: `cnsv.atlassian.net`.

---

## Resumen del flujo

El flujo tiene dos fases separadas por la revisión de QA.

**Fase 1: preparar y enviar a QA**

| # | Paso | Quién | Herramienta / transición |
|---|---|---|---|
| 1 | Iniciar Desarrollador | Agente | `51` |
| 2 | Crear Pauta (si no existe) | Agente | `1491` |
| 3 | Subir PDF de QA y EML de aprobación | **Usuario (manual)** | el agente verifica |
| 4 | Ejecutar QAF-Automation-Manual | **Usuario (manual)** | el agente lee el resultado |
| 5 | Terminar | Agente | `61` |
| 6 | A QA Técnico Automático | Agente | `1841` |

**Espera:** QA revisa y devuelve el ticket al desarrollador.

**Fase 2: enviar al explotador**

| # | Paso | Quién | Transición |
|---|---|---|---|
| 7 | Iniciar Desarrollador | Agente | `51` |
| 8 | Terminar | Agente | `61` |
| 9 | Asignar al Explotador (M) | Agente | `1651` |

Después del paso 9, el trabajo del equipo termina: queda esperar al explotador.

---

## Reglas obligatorias

0. **Antes de cualquier transición, hacer el "Diagnóstico de requisitos"** y mostrar qué está listo y qué falta. Si el usuario solo pregunta qué le falta, responder con el diagnóstico y no ejecutar nada.
1. **Siempre preguntar el ticket.** Ofrecer los tickets del usuario como opciones, pero no elegir por él. El sitio (NSPUM o NSPRM) se deduce de la key y se confirma.
2. **Siempre pedir los PR** que se instalan (uno o más). No sacarlos del repo abierto en el workspace.
3. **Las transiciones van una por una**, nunca en paralelo ni en el mismo bloque de llamadas. Antes de cada una, esperar la respuesta de la anterior y verificar con `getJiraIssue` (field `status`) que el ticket quedó en el estado esperado. Si no, detenerse e informar.
4. **Confirmar cada transición por separado** con un "sí" explícito del usuario.
5. **No ejecutar "Terminar" (paso 5)** sin pauta vinculada, adjuntos verificados y último checklist QAF exitoso.
6. **No ejecutar la Fase 2** sin el comentario de aprobación de QA ("Paso Aprobado para Producción").
7. Si un paso falla, **no repetir los ya ejecutados**. Informar el estado actual y reintentar solo el que falló.
8. No crear una pauta nueva si el ticket ya tiene una vinculada, salvo que el usuario lo pida explícitamente.

---

## Paso 0: Datos de entrada y detección de fase

### Datos (un solo mensaje)

```
Para el pase a producción necesito:
1. Ticket NSPUM/NSPRM (tus tickets en curso: ...):
2. PR(s) a instalar (links de Bitbucket):
```

Los PR solo se necesitan si hay que crear la pauta (paso 2). Si la pauta ya existe, no pedirlos.

### Leer el ticket

```
getJiraIssue
  cloudId: "cnsv.atlassian.net"
  issueIdOrKey: "<KEY>"
  fields: ["summary", "status", "project", "components", "customfield_10351", "customfield_10581", "issuelinks", "attachment", "comment"]
  responseContentFormat: "markdown"
```

Validar que `customfield_10351` (Desarrollador Asignado) sea el usuario actual. Si no, advertir: las transiciones solo están disponibles para el dev asignado.

### Señales en el ticket

| Señal | Dónde se ve |
|---|---|
| Pauta creada | `issuelinks` con tipo **"Pauta Producción"** (id `10012`) hacia un ticket `PAUT-XXXXX` |
| Adjuntos | `attachment`: PDF `QA - Evidencia de pruebas - <KEY>.pdf` y al menos un `.eml` |
| Resultado QAF | Último comentario del autor `svr_jira_checklist_funcional_automation` |
| Aprobación de QA | Comentario con el texto "Paso Aprobado para Producción" |

### Detectar desde qué paso seguir

| Estado actual | Aprobación de QA | Siguiente paso |
|---|---|---|
| POR HACER (DESARROLLADOR) | No | 1 (Iniciar, fase 1) |
| EN PROGRESO (DESARROLLADOR) | No | 2, 3 o 4, según pauta, adjuntos y QAF |
| TERMINADO (DESARROLLADOR) | No | 6 (A QA Técnico Automático), previa verificación de pauta, adjuntos y QAF |
| POR HACER (AUTOMATICO) / POR HACER (QA TECNICO) / en QA | — | Esperar a QA. Informar y detenerse |
| POR HACER (DESARROLLADOR) | Sí | 7 (Iniciar, fase 2) |
| EN PROGRESO (DESARROLLADOR) | Sí | 8 (Terminar, fase 2) |
| TERMINADO (DESARROLLADOR) | Sí | 9 (Asignar al Explotador) |
| POR HACER (EXPLOTADOR) | — | Ya está con el explotador. Nada que hacer |

La aprobación de QA cuenta solo si el comentario es **posterior** a la última ejecución de "A QA Técnico Automático". Si hay dudas, mostrar los comentarios relevantes y preguntar al usuario.

Mostrar al usuario el diagnóstico antes de seguir (ver "Diagnóstico de requisitos" abajo).

---

## Diagnóstico de requisitos (qué falta para avanzar)

Se ejecuta **siempre** al empezar un pase a producción, y también cuando el usuario pregunta "¿qué me falta?" o "¿qué necesito?" para un ticket. Es **solo lectura**: no ejecuta transiciones ni modifica nada.

### Qué revisar

Con la lectura del Paso 0 (`getJiraIssue` con `status`, `customfield_10351`, `customfield_10581`, `issuelinks`, `attachment`, `comment`), evaluar cada requisito:

| # | Requisito | Cómo se verifica | Lo resuelve |
|---|---|---|---|
| R1 | Usuario es el Desarrollador Asignado | `customfield_10351` = usuario actual | TL / reasignación |
| R2 | Ticket en estado de desarrollador | `status` en POR HACER / EN PROGRESO / TERMINADO (DESARROLLADOR) | — |
| R3 | Pauta creada | `issuelinks` con tipo "Pauta Producción" hacia `PAUT-XXXXX`. Mostrar key y estado de la pauta | Agente (Paso 2) |
| R4 | PDF de evidencia de QA | `attachment` con nombre exacto `QA - Evidencia de pruebas - <KEY>.pdf` | Usuario (subir) |
| R5 | Correo de aprobación de negocio | `attachment` con al menos un `.eml` | Usuario (subir) |
| R6 | Checklist QAF ejecutado | Existe comentario de `svr_jira_checklist_funcional_automation` **posterior** a la creación de la pauta y a la subida de los adjuntos | Usuario (rayo) |
| R7 | Checklist QAF exitoso | El último comentario QAF dice "Análisis completado con éxito" y no tiene ❌. Si falló, listar las filas ❌ con su observación | Usuario (corregir y re-ejecutar) |
| R8 | Pruebas de seguridad | Si el QAF las muestra como pendientes o la subtarea "Security tests for <KEY>" no está aprobada, avisar. "➖ No aplica" no bloquea | Ciberseguridad (esperar) |
| R9 | Aprobación de QA | Comentario "Paso Aprobado para Producción" posterior al último envío a QA | QA (esperar) |

Si un requisito depende de otro (por ejemplo, R7 sin R6), marcarlo como "pendiente" y no como fallido.

Si el QAF es **anterior** a la pauta o a los adjuntos, marcar R6 como "hay que volver a ejecutarlo": el resultado no refleja el estado actual.

### Qué habilita cada paso

| Para poder... | Se necesita |
|---|---|
| Crear la pauta (paso 2) | R1, R2, ticket en EN PROGRESO, PR(s) del usuario |
| Ejecutar QAF-Automation-Manual (paso 4) | R3, R4, R5 |
| Terminar y enviar a QA Técnico (pasos 5 y 6) | R3, R4, R5, R6, R7 |
| Fase 2 y Asignar al Explotador (pasos 7 a 9) | R9 (y R3 para la key de la pauta) |

### Formato de respuesta

```
Diagnóstico NSPUM-XXX - <resumen>
Estado: EN PROGRESO (DESARROLLADOR) · Fase 1

✅ Eres el desarrollador asignado
✅ Pauta creada: PAUT-76655 (PAUTA Aprobada)
❌ Falta el PDF "QA - Evidencia de pruebas - NSPUM-XXX.pdf"
❌ Falta el correo de aprobación de negocio (.eml)
⏳ Checklist QAF: no se puede ejecutar todavía (faltan los adjuntos)
⏳ Aprobación de QA: aún no aplica

Para avanzar:
1. Sube el PDF de QA y el .eml de aprobación.
2. Ejecuta QAF-Automation-Manual desde el rayo.
Cuando lo hagas, avísame y reviso de nuevo.
```

Reglas del formato:
- ✅ cumplido, ❌ falta o falló, ⏳ pendiente porque depende de otro paso o de otra persona.
- "Para avanzar" lista solo las acciones **siguientes**, en orden, indicando quién las hace (tú / yo / QA).
- Si el siguiente paso lo puede hacer el agente (crear pauta, Terminar, enviar a QA, asignar al explotador), ofrecerlo: "¿Quieres que cree la pauta?". No ejecutarlo sin el "sí".
- Si todo está listo para el siguiente paso, decirlo en una línea: "Todo listo para enviar a QA Técnico. ¿Lo hago?".

Repetir el diagnóstico después de cada acción del usuario ("ya subí los archivos", "ya corrí el QAF") antes de proponer el siguiente paso.

---

## Paso 1 y 7: Iniciar Desarrollador

Transición `51` → EN PROGRESO (DESARROLLADOR).

| Campo | ID | Valor |
|---|---|---|
| Componentes del Desarrollo | `customfield_10384` (multiselect) | Widget → `[{"id": "15648"}]` MODYO |

- Si el ticket incluye **Microservicios** (`components`), **preguntar** al usuario qué componente va. Ofrecer AWS (`15646`) como opción, sin asumirlo. Si incluye widget y microservicio, el valor puede llevar ambos.
- Comentario vacío.

```json
transitionJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "issueIdOrKey": "<KEY>",
  "transition": { "id": "51" },
  "fields": { "customfield_10384": [{ "id": "15648" }] }
}
```

Confirmado en NSPUM-659.

---

## Paso 2: Crear Pauta

**Antes de crear:** revisar `issuelinks`. Si existe un vínculo "Pauta Producción", la pauta ya existe: mostrar su key y estado y **no crear otra** (salvo pedido explícito). Saltar al paso 3.

Transición `1491` "Crear Pauta" (en bucle: el ticket sigue en EN PROGRESO). Crea un ticket nuevo `PAUT-XXXXX` vinculado al TIS.

### Agrupación de PR

- Repos `widget_*` → pestaña **Modyo**.
- Repos `bff-*` o `ms-*` → pestaña **AWS**.
- Si un PR no calza con ninguno, preguntar al usuario.

### Campos

| Campo | ID | Valor |
|---|---|---|
| Ambiente Pauta | `customfield_10416` | `{"id": "11494"}` Producción |
| Instrucciones de ejecución (Modyo) | `customfield_10689` | template Modyo |
| Resultado Esperado (Modyo) | `customfield_10690` | texto fijo |
| Plan de vuelta atrás (Modyo) | `customfield_10692` | texto fijo |
| Path o PullRequest de Control de Versiones (Modyo) | `customfield_10694` | PR(s) Modyo |
| Instrucciones de ejecución (AWS) | `customfield_10730` | template AWS |
| Resultado Esperado (AWS) | `customfield_10731` | texto fijo |
| Plan de vuelta atrás (AWS) | `customfield_10733` | texto fijo |
| Path o PullRequest de Control de Versiones (AWS) | `customfield_10734` | PR(s) AWS |

Llenar solo las pestañas que correspondan. El resto de los campos (Java, Oracle, .NET, etc.) quedan vacíos.

### Textos (fijos, no cambiarlos)

Cada punto o viñeta es un párrafo separado. Los PR van como link.

**Instrucciones de ejecución (Modyo)**

```
1. Ejecutar merge a pull request <PR_1>.

2.Validar ejecución correcta con el desarrollador.

3.Ejecutar publicación de widgets en modyo (Esta parte es ejecutada por Equipo banco ex Canal Digital, y ha sido previamente coordinada por desarrollador con Cristian Rosales).
```

Con varios PR: un punto "Ejecutar merge a pull request <PR>." por cada uno, numerados, y luego los pasos de validación y publicación con la numeración que siga.

**Resultado Esperado (Modyo)**

- NSPUM: `Pipeline ejecutado correctamente y widget desplegado en sitio publico`
- NSPRM: `Pipeline ejecutado correctamente y widget desplegado en sitio privado`

**Plan de vuelta atrás (Modyo)**

```
-Si el pipeline falla por error de sistema, validar con devops reejecución de dicho pipeline.

-Si la falla es por código, revertir merge al pull request con el problema y devolver ticket a desarrollador para validar camino a seguir.

-Si la falla ocurre en el primer paso de la pauta, desistir con los pasos posteriores y revertir merge. En caso que el despliegue haya quedado con errores, reejecutar ultimo pipeline de master exitoso.
```

**Path o PullRequest (Modyo y AWS):** los links de los PR de esa pestaña, uno por párrafo.

**Instrucciones de ejecución (AWS)**

```
1. Ejecutar merge a pull request <PR_1>.

2. Validar ejecución correcta con el desarrollador.
```

**Resultado Esperado (AWS)**

```
Pipeline ejecutado correctamente y pod inicializado correctamente en rancher.
```

**Plan de vuelta atrás (AWS)**

```
-Si el pipeline falla por error de sistema, validar con devops reejecuccion de dicho pipeline.

-Si la falla es por código, revertir merge al pull request con el problema y devolver ticket a desarrollador para validar camino a seguir.

-Si la falla ocurre en el primer paso de la pauta, desistir con los pasos posteriores y revertir merge.  En caso de que el despliegue haya quedado con errores, se debe ejecutar el ultimo pipeline exitoso.
```

En AWS, recordar al usuario la nota del documento: revisar si el servicio es Serverless o POD y si usa Vault o Secret Manager. No cambia los textos.

### Formato

Los textareas se envían en ADF: cada párrafo es un nodo `paragraph`, y los links usan la marca `link`. Si falla, reintentar como string plano con líneas separadas por `\n\n`.

```json
transitionJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "issueIdOrKey": "<KEY>",
  "transition": { "id": "1491" },
  "fields": {
    "customfield_10416": { "id": "11494" },
    "customfield_10689": { "type": "doc", "version": 1, "content": [ "..." ] },
    "customfield_10690": { "type": "doc", "version": 1, "content": [ "..." ] },
    "customfield_10692": { "type": "doc", "version": 1, "content": [ "..." ] },
    "customfield_10694": { "type": "doc", "version": 1, "content": [ "..." ] }
  }
}
```

### Después de crear

Leer `issuelinks` del TIS y tomar la key del vínculo "Pauta Producción". Mostrarla al usuario. Si no aparece, detenerse: no reintentar la creación sin preguntar, para no duplicar pautas.

---

## Paso 3: Adjuntos (manual)

El MCP no puede subir archivos. Pedir:

```
Sube estos archivos al ticket <KEY> y avísame:
1. QA - Evidencia de pruebas - <KEY>.pdf  (documento de QA)
2. El correo de aprobación de negocio en formato .eml
```

Verificar con `getJiraIssue` (field `attachment`):

- Existe un archivo con nombre exacto `QA - Evidencia de pruebas - <KEY>.pdf`.
- Existe al menos un archivo `.eml` (el nombre es libre).

Si falta alguno, decir cuál y no seguir.

Formatos de correo de referencia: [Formatos de Correos](https://cnsv.atlassian.net/wiki/spaces/BS/pages/1758494770).

---

## Paso 4: QAF-Automation-Manual (manual)

Es una regla de automatización (ícono del rayo), no una transición: el agente no puede ejecutarla.

```
Ejecuta "QAF-Automation-Manual" desde el ícono del rayo en <KEY>.
Tarda unos minutos y deja el resultado como comentario. Avísame cuando aparezca.
```

Leer el último comentario del autor `svr_jira_checklist_funcional_automation`:

- Primera línea **"Análisis completado con éxito…"** y sin filas con ❌ → seguir.
- Primera línea **"Análisis fallido…"** o alguna fila con ❌ → mostrar al usuario las filas fallidas con su observación (por ejemplo "Validación Ficha correctiva ❌ Ficha 8717 es correctiva, se debe cambiar prioridad del ticket."). El usuario corrige y vuelve a ejecutar la regla. No seguir hasta que el último resultado sea exitoso. Si no sabe cómo corregirlo, indicarle que lo valide con el TL (Pablo Bustos).
- Filas "➖ No aplica" no bloquean.

Si no aparece el comentario, pedir al usuario que espere y avise de nuevo.

---

## Paso 5 y 8: Terminar

Transición `61` → TERMINADO (DESARROLLADOR).

```json
transitionJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "issueIdOrKey": "<KEY>",
  "transition": { "id": "61" },
  "fields": { "customfield_10581": [{ "id": "10417" }] }
}
```

`10417` = "Desarrollo Terminado para Instalar en Produccion".

---

## Paso 6: A QA Técnico Automático

Transición `1841` → POR HACER (AUTOMATICO). Sin modal ni campos.

```json
transitionJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "issueIdOrKey": "<KEY>",
  "transition": { "id": "1841" }
}
```

Informar al usuario que ahora hay que esperar a QA. Cuando QA aprueba, deja comentarios como:

```
Paso Aprobado para Producción - Asignada al Desarrollador
QA Funcional: Actividad finalizada
Comentario: Aprobado de acuerdo a CheckList y Pauta de Instalación.
Pauta: PAUT-XXXXX
```

y el ticket vuelve a POR HACER (DESARROLLADOR). Con eso se retoma desde el paso 7.

Si QA devuelve el ticket con observaciones (sin "Paso Aprobado para Producción"), mostrar los comentarios al usuario y no ejecutar la Fase 2.

---

## Paso 9: Asignar al Explotador (M)

Disponible en TERMINADO (DESARROLLADOR) junto con "A QA Técnico Automático" (`1841`) y "Reiniciar" (`471`). Usar **solo** `1651`.

| Campo | ID | Valor |
|---|---|---|
| Solicitud a Explotador | `customfield_10338` (multiselect) | `[{"id": "10517"}]` Instalar Componentes de Pauta de Produccion en Ambiente Respectivo |
| Pauta Produccion | `customfield_10388` (texto) | Key del vínculo "Pauta Producción", en mayúsculas (ej. `PAUT-76655`). **Obligatorio en el modal** aunque la API lo marque opcional |
| Explotador Asignado | `customfield_10397` | vacío |
| Fecha de vencimiento | `duedate` | vacío |
| Comentario | — | vacío |

"Ninguno" en el modal equivale a no enviar `customfield_10338`.

Si el ticket no tiene vínculo "Pauta Producción", detenerse y avisar.

```json
transitionJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "issueIdOrKey": "<KEY>",
  "transition": { "id": "1651" },
  "fields": {
    "customfield_10338": [{ "id": "10517" }],
    "customfield_10388": "PAUT-76655"
  }
}
```

Verificar que el ticket quede en **POR HACER (EXPLOTADOR)** e informar al usuario: "El ticket quedó con el explotador. Por parte del equipo no queda nada pendiente".

---

## Vista previa

Antes de cada transición, mostrar:

```
Ticket: <KEY> - <resumen> (estado actual: <estado>)
Paso: <n> - <nombre de la transición>
Campos: <campo legible> = <valor legible>
Estado esperado después: <estado>
¿Ejecuto la transición?
```

Para "Crear Pauta", incluir los textos completos de cada campo.

---

## Troubleshooting

| Problema | Causa probable | Solución |
|---|---|---|
| Transición no aparece | No es el dev asignado o el ticket está en otro estado | Revisar estado y `customfield_10351` con `getTransitionsForJiraIssue` |
| Crear Pauta no genera vínculo PAUT | Error en la creación | No reintentar sin preguntar. Revisar comentarios y vínculos del ticket |
| Error en textareas de la pauta | Formato ADF rechazado | Reintentar como string plano |
| QAF con ❌ | Dato del ticket incorrecto | Mostrar la fila; el usuario corrige y vuelve a ejecutar la regla |
| Asignar al Explotador falla por "Pauta Produccion" | Campo vacío o key en minúsculas | Enviar la key del vínculo en mayúsculas |
| Terminar ejecutado y la siguiente transición falló | Error en el paso siguiente | El ticket queda en TERMINADO; reintentar solo el paso que falló |
