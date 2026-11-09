---
name: "cd-power-ticket-seguros"
displayName: "Ticket Seguros NSPUM NSPRM"
description: "Crea tickets 'Mejora o Nueva Funcionalidad' en NSPUM (sitio público) y NSPRM (sitio privado) a partir de una HDU de Jira, los asigna al dev y los vincula a la HDU. Solicita a QA DevOps Custodio (DAC) variables de entorno y creación de repositorios, y gestiona el pase a producción hasta el explotador. También busca tickets del usuario."
keywords: ["jira", "ticket", "seguros", "nspum", "nsprm", "hdu", "mejora", "nueva funcionalidad", "atlassian", "consorcio", "sitio publico", "sitio privado", "variables de entorno", "secretos", "dac", "devops", "repositorio", "bitbucket", "pase a produccion", "pauta", "explotador", "qaf"]
author: "Equipo Sitios - Consorcio"
---

# Ticket Seguros (NSPUM / NSPRM)

## Overview

Antes de desarrollar una HDU hay que crear el ticket en el Jira de seguros para poder crear la rama. Este power automatiza ese proceso siguiendo la documentación oficial del equipo:
[Creación Ticket Seguro Mejora o Nueva funcionalidad | Devs](https://cnsv.atlassian.net/wiki/spaces/BS/pages/1677787145).

Capacidades:

1. **Crear ticket** desde una HDU (proyecto SCP u otro) → crea, asigna (transición "Asignar Ticket") y vincula con la HDU.
2. **Buscar tickets** del usuario en NSPUM / NSPRM.
3. **Solicitar variables de entorno** (secretos) para uno o más repos de Bitbucket, vía transición a QA DevOps Custodio (DAC).
4. **Solicitar creación de repositorio** en Bitbucket, por la misma vía.
5. **Pase a producción**: iniciar, crear pauta, verificar adjuntos y checklist QAF, enviar a QA Técnico y, tras la aprobación, asignar al explotador.

Alcance actual de creación de tickets: solo el tipo **"Mejora o Nueva Funcionalidad"**.

## Available Steering Files

- **solicitudes-dac.md**: flujo completo para solicitar variables de entorno o creación de repositorio (transiciones Terminar → A QA Custodio (DAC), templates de texto y reglas). **Leerlo siempre** antes de hacer cualquiera de estas dos solicitudes.
- **pase-produccion.md**: flujo completo de pase a producción (Iniciar Desarrollador, Crear Pauta, adjuntos, QAF-Automation-Manual, Terminar, A QA Técnico Automático y Asignar al Explotador). **Leerlo siempre** antes de mover un ticket hacia producción.

## Onboarding

- Requiere el MCP remoto de Atlassian (incluido en `mcp.json`). La primera vez abre el navegador para autenticar con OAuth en `cnsv.atlassian.net`.
- Si ya tienes un servidor `atlassian` configurado en `~/.kiro/settings/mcp.json`, puedes deshabilitar el de este power para no tener herramientas duplicadas.
- `cloudId` para todas las llamadas: `cnsv.atlassian.net`.

## Reglas obligatorias para el agente

1. **Lo primero, antes de cualquier otro paso, es preguntar si el ticket es de Sitio Público (NSPUM) o Sitio Privado (NSPRM).** No inferirlo del contenido de la HDU, del board ni de tickets anteriores. Si el usuario ya lo mencionó en su mensaje, pedirle que lo confirme. No leer la HDU ni preguntar otros datos hasta tener esta respuesta.
2. **Nunca crear el ticket sin mostrar la vista previa completa y recibir confirmación explícita** del usuario. Crear tickets no se puede deshacer fácilmente.
3. **Siempre preguntar** Número de Ficha (madre) y Número de Ficha Hija. No sugerir ni inferir valores: cambian en el tiempo y las entrega el PO o TL.
4. **Siempre preguntar** quién es el Desarrollador Asignado. Se puede mencionar el asignado de la HDU como referencia, pero el usuario debe confirmarlo.
5. **Siempre preguntar** quién es el Mandante. No hay valor por defecto: cambia según la iniciativa. Se pueden mostrar los mandantes frecuentes como opciones, pero el usuario debe elegir uno o indicar otro.
6. Preguntar el resto de los datos variables **en un solo mensaje**, no campo por campo.
7. No modificar la HDU de origen (solo leerla y vincularla).
8. **Las escrituras en Jira se ejecutan una por una, nunca en paralelo ni en el mismo bloque de llamadas.** Crear, asignar y vincular son pasos dependientes: cada uno espera la respuesta del anterior. **Prohibido adivinar o calcular la key** del ticket nuevo (por ejemplo, "el último fue NSPUM-659, el nuevo será NSPUM-660"). La key se toma **solo** de la respuesta de `createJiraIssue`. Ver "Paso 5b: Validar la key".

---

## Constantes

### Proyectos

| Sitio | Key | Project ID |
|---|---|---|
| Sitio Público | `NSPUM` | 10284 |
| Sitio Privado | `NSPRM` | 10292 |

Tipo de incidencia: `Mejora o Nueva Funcionalidad` (id `10057`, igual en ambos proyectos).

### Campos fijos

| Campo | Field ID | Valor a enviar |
|---|---|---|
| Área de TI | `customfield_10347` | `{"id": "10656"}` (Sistemas de Canales Personas) |
| Gerencia B | `customfield_10403` | `{"id": "10672"}` (Gerencia de Desarrollo) |
| Proveedor | `customfield_10631` | `{"id": "16866"}` (Canal Digital Célula Ryodan) |
| QA Funcional Asignado | `customfield_10210` | `{"accountId": "712020:9029982f-b3c8-4695-abec-4616cdd65031"}` (Balford Montenegro) |

### Campos con valor por defecto (confirmar con el usuario)

| Campo | Field ID | Default | Alternativas |
|---|---|---|---|
| Gerencia Origen | `customfield_10634` | `{"id": "10589"}` Marketing y Clientes | Validar con PO/TL |
| Arquitecto Asignado (Célula) | `customfield_10266` | Landing nueva o actualización → Ibrahim Zerpa Ortiz `612fa37444c8ed00688967a3` | Proyecto desde 0 → el arquitecto del RAS (preguntar) |
| Proceso Afectado | `customfield_10665` | Landing → `[{"id": "15166"}]` SITIO_WEB_PUBLICO | Ver tabla de procesos |

Procesos Afectados frecuentes (el campo tiene ~2.800 opciones, **no cargar la lista completa**):

| Valor | ID |
|---|---|
| SITIO_WEB_PUBLICO | 15166 |
| AP-SUCURSAL-VIRTUAL | 12829 |
| POSTVENTA | 14646 |
| SEGURO_AUTO | 15074 |

Si el usuario indica otro proceso, enviarlo por nombre exacto: `[{"value": "NOMBRE_PROCESO"}]`.

### Campos que siempre se preguntan

| Campo | Field ID | Tipo | Formato |
|---|---|---|---|
| Componentes | `components` | multi | `[{"name": "Widget"}]`, `[{"name": "Microservicios"}]` o ambos. **Usar `name`, no `id`**: los IDs difieren entre NSPUM y NSPRM |
| Número de Ficha | `customfield_10248` | número | `8717` (número, no string). Sin el prefijo `2000` |
| Número de Ficha Hija | `customfield_11217` | texto | `"10157"` o `"GPTI-10096"` |
| Instalación en Producción Jira Funcional | `customfield_11285` | multi | `[{"value": "Dentro de Horario"}]`, `[{"value": "Fuera de Horario"}]` o ambos |
| Desarrollador Asignado | (transición) | usuario | accountId vía `lookupJiraAccountId` |
| Mandante | `customfield_10439` | usuario | `{"accountId": "..."}`. Sin default. Ver mandantes frecuentes abajo |

Mandantes frecuentes (solo como opciones para mostrar, **no asumir ninguno**):

| Persona | Contexto habitual | accountId |
|---|---|---|
| Laura Beatriz Besoain Klein | Marketing y Clientes | `712020:117031eb-07e9-46cf-a31f-c580a92057d7` |
| Josefina Ignacia Quiroga Garcia | Tareas SEO | `712020:a5ec33db-cb5f-41e4-a656-708014b0e4d6` |
| Catherine Alejandra Torres Antiqueo | Visto en NSPRM | `5fe0a9f334847e0069666a05` |
| Fernanda Antonia Valech Massu | Visto en NSPRM | `712020:5c4e22e9-3fee-4a6b-99e4-a5582ceba393` |

Si el usuario indica otra persona, buscarla con `lookupJiraAccountId` (nombre o correo). Si hay más de un resultado, preguntar cuál. Si no aparece, pedir que lo valide con el PO o TL.

### Asignación (transición)

| Dato | Valor |
|---|---|
| Transición | `1091` "Asignar Ticket" (Abierta → POR HACER (DESARROLLADOR)). Mismo ID en NSPUM y NSPRM |
| Jefe de Proyecto Asignado | `customfield_10348` → Pablo Javier Bustos Viveros `712020:e12a998e-96da-4bc4-91c5-98775749700d` |
| Desarrollador Asignado | `customfield_10351` → dev confirmado por el usuario |

### Vínculo con la HDU

Tipo de enlace `Relates` ("relates to").

---

## Workflow 1: Crear ticket desde una HDU

### Paso 0: Preguntar la ticketera (siempre primero)

Antes de cualquier otra acción, preguntar:

```
¿El ticket es de Sitio Público o Sitio Privado?
1. Sitio Público → NSPUM
2. Sitio Privado → NSPRM
```

- Si el usuario ya lo indicó, confirmar: "¿Confirmas que va en NSPUM (Sitio Público)?".
- Si la respuesta es ambigua, volver a preguntar. No asumir un valor por defecto.
- El proyecto elegido se usa en `projectKey` y se muestra en la vista previa.

### Paso 1: Obtener la HDU

Con la ticketera confirmada, pedir la HDU. El usuario entrega una key (`SCP-2236`) o un link. Extraer la key:

- `.../browse/SCP-2236` → `SCP-2236`
- `...?selectedIssue=SCP-2236` → `SCP-2236`

Si el usuario entregó la HDU junto con su primer mensaje, guardarla y no leerla hasta completar el Paso 0.

```
getJiraIssue
  cloudId: "cnsv.atlassian.net"
  issueIdOrKey: "SCP-2236"
  fields: ["summary", "description", "assignee", "parent", "status"]
  responseContentFormat: "markdown"
```

### Paso 2: Extraer datos de la HDU

- **Resumen** = `summary` de la HDU, tal cual (incluido prefijo como `[Impl]`).
- **Descripción** = primer párrafo de `description`: el texto antes del primer encabezado (`#`), tabla o lista. Quitar marcas markdown (`**`, links de imágenes). Si queda vacío, pedir al usuario el texto.
- **Solución** = mismo texto que la descripción.
- Asignado de la HDU (`assignee.displayName`) → solo como referencia al preguntar por el dev.

### Paso 3: Preguntar los datos variables (un solo mensaje)

Ejemplo:

```
Para crear el ticket en NSPUM necesito:
1. Número de Ficha (madre):
2. Número de Ficha Hija:
3. Componentes: Widget / Microservicios / ambos
4. Instalación en producción: Dentro de Horario / Fuera de Horario / ambos
5. Desarrollador asignado (la HDU está asignada a Miguel Marchant, ¿es él?):
6. ¿Es landing (nueva o actualización)? → Proceso SITIO_WEB_PUBLICO y arquitecto Ibrahim Zerpa. Si es un flujo, indica proceso y arquitecto.
7. Mandante (¿quién solicitó el cambio?): Laura Besoain / Josefina Quiroga (SEO) / Catherine Torres / Fernanda Valech / otro:
8. Gerencia Origen: Marketing y Clientes, ¿ok?
```

Validaciones antes de seguir:
- Ficha madre debe ser numérica. Si empieza con `2000` y es más larga, preguntar si se debe quitar el prefijo.
- Si la ficha madre parece una ficha hija o viceversa (por ejemplo, la madre es mayor que la hija y ambas son numéricas), advertir y pedir confirmación. Ya ocurrió que se cruzaron.
- Resolver accountId de personas nuevas con `lookupJiraAccountId` (`searchString`: nombre o correo). Si hay más de un resultado, preguntar cuál.

### Paso 4: Vista previa y confirmación

Mostrar una tabla con **todos** los campos y valores legibles (nombres, no IDs):

```
Proyecto: NSPUM (Sitio Público)
Tipo: Mejora o Nueva Funcionalidad
Resumen: [Impl] Mejoras Concurso Fotografía 110 años (votación y compartir)
Descripción / Solución: Mejoras al Concurso de Fotografía de los 110 años...
Componentes: Widget
Área de TI: Sistemas de Canales Personas
Gerencia B: Gerencia de Desarrollo
Proveedor: Canal Digital Célula Ryodan
Arquitecto: Ibrahim Zerpa
QA Funcional: Balford Montenegro
Número de Ficha: 8717 | Ficha Hija: 10146
Instalación: Dentro de Horario
Gerencia Origen: Marketing y Clientes
Mandante: <el que indiques>
Proceso Afectado: SITIO_WEB_PUBLICO
Luego: asignar a <dev> con JP Pablo Bustos y vincular con SCP-2236
¿Creo el ticket?
```

No continuar sin un "sí" explícito.

### Paso 5: Crear el ticket

```json
createJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "projectKey": "NSPUM",
  "issueTypeName": "Mejora o Nueva Funcionalidad",
  "summary": "<titulo HDU>",
  "description": "<primer párrafo>",
  "contentFormat": "markdown",
  "additional_fields": {
    "components": [{ "name": "Widget" }],
    "customfield_10347": { "id": "10656" },
    "customfield_10403": { "id": "10672" },
    "customfield_10631": { "id": "16866" },
    "customfield_10210": { "accountId": "712020:9029982f-b3c8-4695-abec-4616cdd65031" },
    "customfield_10266": { "accountId": "612fa37444c8ed00688967a3" },
    "customfield_10248": 8717,
    "customfield_11217": "10146",
    "customfield_11285": [{ "value": "Dentro de Horario" }],
    "customfield_10634": { "id": "10589" },
    "customfield_10439": { "accountId": "<accountId mandante confirmado>" },
    "customfield_10665": [{ "id": "15166" }],
    "customfield_10525": {
      "type": "doc",
      "version": 1,
      "content": [
        { "type": "paragraph", "content": [{ "type": "text", "text": "<primer párrafo>" }] }
      ]
    }
  }
}
```

Nota sobre **Solución** (`customfield_10525`): es un campo de texto enriquecido y se envía en ADF, como en el ejemplo. **Confirmado** en la creación de NSPUM-660: Jira aceptó el ADF. Solo si en algún caso falla por ese campo, reintentar como string plano.

Esta llamada va **sola**. No incluir en el mismo bloque la transición ni el vínculo.

### Paso 5b: Validar la key (obligatorio antes de los Pasos 6 y 7)

1. Esperar la respuesta de `createJiraIssue` y leer la `key` que devuelve. Si la respuesta no trae una key, **detenerse**: no asignar ni vincular. Buscar el ticket con `searchJiraIssuesUsingJql` (`project = <KEY_PROYECTO> AND summary ~ "<resumen>" AND created >= -10m`) y mostrar el resultado al usuario antes de seguir.
2. Confirmar que esa key es el ticket recién creado:

```
getJiraIssue
  cloudId: "cnsv.atlassian.net"
  issueIdOrKey: "<key devuelta por createJiraIssue>"
  fields: ["summary", "status", "project", "issuetype"]
```

Debe cumplirse **todo**:
- `summary` es igual al Resumen de la vista previa.
- `project.key` es el proyecto confirmado en el Paso 0.
- `status` es "Abierta".

Si algo no coincide, **detenerse**, no ejecutar los Pasos 6 y 7, e informar al usuario.

3. Recién entonces usar esa key en los Pasos 6 y 7, uno después del otro (primero asignar, esperar respuesta, luego vincular).

### Paso 6: Asignar (transición "Asignar Ticket")

```json
transitionJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "issueIdOrKey": "<key validada en el Paso 5b>",
  "transition": { "id": "1091" },
  "fields": {
    "customfield_10348": { "accountId": "712020:e12a998e-96da-4bc4-91c5-98775749700d" },
    "customfield_10351": { "accountId": "<accountId dev>" }
  }
}
```

Si la transición no está disponible, consultar `getTransitionsForJiraIssue` y buscar la de nombre "Asignar Ticket".

### Paso 7: Vincular con la HDU

```json
createIssueLink
{
  "cloudId": "cnsv.atlassian.net",
  "type": "Relates",
  "inwardIssue": "<key validada en el Paso 5b>",
  "outwardIssue": "<SCP-XXXX>"
}
```

### Paso 8: Confirmar al usuario

Verificar con `getJiraIssue` (fields: `status`, `assignee`, `customfield_10351`) y responder:

- Key y link: `https://cnsv.atlassian.net/browse/<KEY>`
- Estado (debe ser "POR HACER (DESARROLLADOR)") y dev asignado
- Vínculo con la HDU
- Si algún paso falló (transición o vínculo), indicarlo; el ticket ya existe y **no se debe crear de nuevo**.

---

## Workflow 2: Buscar tickets

Búsqueda por defecto ("mis tickets"):

```
searchJiraIssuesUsingJql
  cloudId: "cnsv.atlassian.net"
  jql: project in (NSPUM, NSPRM) AND issuetype not in subTaskIssueTypes() AND (assignee = currentUser() OR "Desarrollador Asignado" = currentUser()) ORDER BY created DESC
  fields: ["summary", "status", "project", "created", "assignee"]
  maxResults: 20
```

Variantes según lo que pida el usuario:

| Pedido | Agregar al JQL |
|---|---|
| Solo abiertos / pendientes | `AND statusCategory != Done` |
| Incluir subtareas (Security tests, Check list) | quitar `issuetype not in subTaskIssueTypes()` |
| Solo Mejoras | `AND issuetype = "Mejora o Nueva Funcionalidad"` |
| Solo sitio público / privado | cambiar a `project = NSPUM` / `project = NSPRM` |
| Por texto | `AND text ~ "palabra"` |
| Ticket asociado a una HDU | `issue in linkedIssues("SCP-2236")` |
| De otra persona | reemplazar `currentUser()` por el accountId (vía `lookupJiraAccountId`) |

Mostrar resultados como tabla: key (con link), resumen, estado, proyecto, fecha de creación.

Antes de crear un ticket nuevo, es buena práctica buscar `issue in linkedIssues("<HDU>") AND project in (NSPUM, NSPRM)` para evitar duplicados.

---

## Workflow 3 y 4: Solicitudes DAC (variables de entorno / crear repositorio)

Leer `steering/solicitudes-dac.md` (`readSteering` con `steeringFile: "solicitudes-dac.md"`) y seguirlo al pie de la letra. Resumen de sus reglas:

- Preguntar siempre el ticket, el o los repositorios (no asumir el repo abierto en el workspace), el o los ambientes (NSPRM: INT, CERT, PROD; NSPUM: CERT, PROD) y las variables.
- `MODYO_TOKEN` siempre vacío; nunca escribir secretos en el texto.
- Vista previa completa y confirmación de **cada** transición por separado.

---

## Workflow 5: Pase a producción

Leer `steering/pase-produccion.md` (`readSteering` con `steeringFile: "pase-produccion.md"`) y seguirlo al pie de la letra. Resumen de sus reglas:

- Preguntar siempre el ticket. Detectar la fase por el estado, el vínculo "Pauta Producción" y el comentario de aprobación de QA.
- Empezar siempre con el **diagnóstico de requisitos** (solo lectura): qué está listo (pauta, PDF, EML, QAF, aprobación de QA) y qué falta para el siguiente paso. Si el usuario pregunta "¿qué me falta?", responder solo con eso.
- Subir adjuntos y ejecutar QAF-Automation-Manual son pasos manuales del usuario; el agente solo los verifica.
- No terminar sin pauta, adjuntos y QAF exitoso. No asignar al explotador sin aprobación de QA.
- Confirmar **cada** transición por separado y verificar el estado antes de la siguiente.

---

## Troubleshooting

| Error | Causa probable | Solución |
|---|---|---|
| `Field 'customfield_10525' ... ADF` o "Operation value must be an Atlassian Document" | Solución enviado como string | Enviar en ADF |
| "Operation value must be a string" en Solución | Se envió ADF y la API espera texto | Enviar string plano |
| `Componentes ... is not valid` | Se envió id de otro proyecto | Enviar por `name` |
| `Número de Ficha ... number` | Se envió como string | Enviar como número |
| `Option value ... is not valid` en Proceso Afectado | Nombre no coincide exactamente | Pedir el nombre exacto al usuario (mayúsculas y guiones bajos) |
| Transición `1091` no aparece | El ticket no está en "Abierta" o falta permiso | Revisar `getTransitionsForJiraIssue`, informar al usuario |
| 401 / sin acceso | OAuth vencido | Reconectar el servidor MCP de Atlassian desde el panel de MCP |

## Mantenimiento

- Personas fijas (TL, QA, arquitecto, mandantes): si cambian, actualizar los accountId en la sección **Constantes**.
- Las fichas no se guardan aquí a propósito: siempre se preguntan.
- Para agregar otros tipos de ticket (Falla, Hotfix), consultar sus campos obligatorios con `getJiraIssueTypeMetaWithFields` (`requiredFieldsOnly: true`) y documentarlos en un archivo `steering/` aparte.
