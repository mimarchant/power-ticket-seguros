# Solicitudes a QA DevOps Custodio (DAC)

Workflow para pedir a DevOps, desde un ticket NSPUM / NSPRM existente:

- **Crear o modificar variables de entorno (secretos)** de un repositorio de Bitbucket.
- **Crear un repositorio** nuevo en Bitbucket.

Documentación de referencia: [Solicitar Variables para Repositorio](https://cnsv.atlassian.net/wiki/spaces/BS/pages/1751089300). Ejemplo real: NSPUM-619.

`cloudId` para todas las llamadas: `cnsv.atlassian.net`.

---

## Cómo funciona

La solicitud **no es un comentario suelto**. Se hace moviendo el ticket por el workflow:

| Paso | Transición | Estado destino | Campo del modal | Valor |
|---|---|---|---|---|
| A (solo si el ticket está en "POR HACER (DESARROLLADOR)") | `51` Iniciar Desarrollador | EN PROGRESO (DESARROLLADOR) | ninguno | |
| B | `61` Terminar | TERMINADO (DESARROLLADOR) | Resultado Tarea Solicitada al Desarrollador `customfield_10581` (multiselect) | `[{"id": "10415"}]` Analisis y estimacion de Desarrollo Terminado |
| C | `1701` A QA Custodio (DAC) | POR HACER (QA DEVOPS) | Solicitud QA Devops Custodio (DAC) `customfield_10801` (multiselect) + **Comentario** | Ver tabla de tipos de solicitud |

Tipos de solicitud (`customfield_10801`):

| Solicitud | ID |
|---|---|
| Crear o Mmodificar Secretos en Repositorio de Bitbucket | `19010` |
| Crear repositorio | `11574` |

Otras opciones del mismo campo (no cubiertas aún): Acceso a Repositorio `11575`, Crear Pipeline en Repositorio de Bitbucket `19009`, Crear Plan de Build de Bamboo `11576`, Crear Plan de Deploy de Bamboo `11577`.

El recuadro "Comentario" del modal es el comentario estándar de la transición. Por API se envía en `update.comment` dentro de `transitionJiraIssue`.

Cuando DevOps termina, el ticket vuelve a "POR HACER (DESARROLLADOR)" y el dev retoma con "Iniciar Desarrollador".

---

## Reglas obligatorias

1. **Siempre preguntar el ticket** NSPUM / NSPRM donde se hace la solicitud. Ofrecer los tickets del usuario (búsqueda de POWER.md) como opciones, pero no elegir uno por él. El sitio (público o privado) se deduce de la key y se confirma con el usuario.
2. **Siempre preguntar el repositorio** (o repositorios). **No asumir el repo abierto en el workspace del agente**: la solicitud suele ser para otro repo. Se puede mencionar el repo abierto como opción, nunca usarlo sin confirmación.
3. **Siempre preguntar el o los ambientes.** Disponibles según el sitio:
   - Sitio Privado (NSPRM): `INT`, `CERT`, `PROD`
   - Sitio Público (NSPUM): `CERT`, `PROD`
   Si el usuario pide un ambiente que no existe para ese sitio (por ejemplo INT en NSPUM), advertir y volver a preguntar.
4. **Siempre pedir al usuario las variables** (nombre y valor) por cada repo y ambiente. No inventar, completar ni reutilizar valores de otros repos o ambientes: cambian mucho entre repos. Si el usuario dice que son las mismas para varios repos o ambientes, pedirle confirmación explícita.
5. **`MODYO_TOKEN` siempre va vacío** (`MODYO_TOKEN=`). Nunca escribir un token ni ningún secreto en el texto: el comentario lo ve cualquiera con acceso al ticket. Si el usuario pega un valor para el token (u otra variable que parezca secreta: `TOKEN`, `SECRET`, `PASSWORD`, `KEY`), no usarlo, dejarla vacía y avisarle.
6. **El token siempre se saca del mismo repo**, para todos los repos y ambientes: `https://bitbucket.org/cnsvbucket/widget_fe-denuncio-vida-salud`. No preguntarlo. Cada bloque lleva siempre `MODYO_TOKEN=` al final de las variables y, después, la línea `Sacar token de este repo https://bitbucket.org/cnsvbucket/widget_fe-denuncio-vida-salud`. Si el usuario no incluyó `MODYO_TOKEN` en su lista, agregarlo igual.
7. **Vista previa completa** (ticket, estado actual, transiciones a ejecutar y texto exacto del comentario) antes de empezar.
8. **Confirmar cada transición por separado.** Antes de ejecutar A, B y C, pedir un "sí" explícito para esa transición. No encadenarlas con una sola confirmación.
9. **Las transiciones se ejecutan una por una, nunca en paralelo ni en el mismo bloque de llamadas.** Antes de cada una, esperar la respuesta de la anterior y confirmar con `getJiraIssue` (field `status`) que el ticket quedó en el estado esperado. Si no está en ese estado, detenerse e informar.
10. Si un paso falla, **no repetir los pasos ya ejecutados**. Informar el estado actual del ticket y reintentar solo el paso que falló.

---

## Paso 1: Datos de la solicitud (un solo mensaje)

Para variables de entorno:

```
Para la solicitud de variables de entorno necesito:
1. Ticket NSPUM/NSPRM donde se hace la solicitud (tus tickets en curso: NSPUM-659, ...):
2. Repositorio(s) de Bitbucket (nombre o link):
3. Ambiente(s): CERT / PROD   (en sitio privado también INT)
4. Variables por repo y ambiente, en formato NOMBRE=valor (MODYO_TOKEN lo agrego yo, vacío):
```

Para creación de repositorio: ver la sección "Creación de repositorio" más abajo.

Normalizar repositorios:

- `widget_landing-seguros-colectivos` → `https://bitbucket.org/cnsvbucket/widget_landing-seguros-colectivos/src/master/`
- Si entrega un link, usarlo tal cual.

## Paso 2: Validar el ticket

```
getJiraIssue
  cloudId: "cnsv.atlassian.net"
  issueIdOrKey: "<KEY>"
  fields: ["summary", "status", "project", "customfield_10351"]
```

| Estado actual | Pasos a ejecutar |
|---|---|
| POR HACER (DESARROLLADOR) | A → B → C |
| EN PROGRESO (DESARROLLADOR) | B → C |
| TERMINADO (DESARROLLADOR) | Solo C |
| POR HACER (QA DEVOPS) u otro | Detenerse: ya hay una solicitud en curso o el ticket está en otra etapa. Informar al usuario |

Si `customfield_10351` (Desarrollador Asignado) no es el usuario actual, advertir: las transiciones solo están disponibles para el dev asignado.

Confirmar con `getTransitionsForJiraIssue` (sin `expand`) que las transiciones necesarias estén disponibles. Si los IDs no coinciden, buscarlas por nombre ("Iniciar Desarrollador", "Terminar", "A QA Custodio (DAC)").

## Paso 3: Armar el texto

### Template variables de entorno (un bloque por repo y ambiente)

```
Estimados, para el repo <URL_REPO>
para el ambiente de  <AMBIENTE> favor agregar las siguiente variables de entorno
Para la variable MODYO_TOKEN favor obtener el token y lo coloquen en el secreto

<VARIABLE_1>=<valor>
<VARIABLE_2>=<valor>
...
MODYO_TOKEN=

Sacar token de este repo https://bitbucket.org/cnsvbucket/widget_fe-denuncio-vida-salud
```

- Las líneas "Para la variable MODYO_TOKEN...", `MODYO_TOKEN=` y "Sacar token de este repo..." van **siempre** en cada bloque, con ese mismo repo de origen.
- Copiar los nombres de variables **exactamente** como los da el usuario (por ejemplo `VUE_APP_NAME_ENVIROMENT` se escribe así, con el error ortográfico, porque así lo lee el código).
- `MODYO_TOKEN=` siempre al final de la lista de variables.

### Orden con varios ambientes y repos

1. Ordenar por ambiente: primero los bajos y al final PROD → `INT`, `CERT`, `PROD`.
2. Dentro de cada ambiente, los repos en el orden que los dio el usuario.
3. Separar cada bloque con una línea en blanco extra (como en NSPUM-619).

### Ejemplo (NSPUM-619, CERT)

```
Estimados, para el repo https://bitbucket.org/cnsvbucket/widget_landing-seguros-colectivos/src/master/
para el ambiente de  CERT favor agregar las siguiente variables de entorno
Para la variable MODYO_TOKEN favor obtener el token y lo coloquen en el secreto

MODYO_WIDGET_NAME=widget_landing-seguros-colectivos
MODYO_SITE_HOST=seguros-empresas
MODYO_ACCOUNT_URL=https://consorcio-spui.modyo.be/
VITE_MODYO_URL=https://consorcio-spui.modyo.be
MODYO_BUILD_COMMAND=build
MODYO_BUILD_DIRECTORY=dist
MODYO_VERSION=10
VUE_APP_NAME_ENVIROMENT=cert
VUE_APP_SPI_ENVIROMENT=spui
VITE_VUE_APP_MODYO_URL=https://consorcio-spui.modyo.be
MODYO_TOKEN=

Sacar token de este repo https://bitbucket.org/cnsvbucket/widget_fe-denuncio-vida-salud
```

Este ejemplo es solo de referencia de formato. **No usar sus valores** para otros repos sin que el usuario los entregue.

## Paso 4: Vista previa

```
Ticket: NSPUM-659 - [Impl] Mejoras Concurso Fotografía 110 años (estado: EN PROGRESO (DESARROLLADOR))
Solicitud DAC: Crear o Mmodificar Secretos en Repositorio de Bitbucket
Transiciones a ejecutar (te pediré confirmación en cada una):
  1. Terminar → Resultado: Analisis y estimacion de Desarrollo Terminado
  2. A QA Custodio (DAC) → con el comentario de abajo
Comentario:
<texto completo>
```

## Paso 5: Ejecutar transiciones (confirmar cada una)

Antes de cada llamada, preguntar: "¿Ejecuto la transición '<nombre>' en <KEY>?" y esperar un "sí".

### A. Iniciar Desarrollador (si aplica)

```json
transitionJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "issueIdOrKey": "<KEY>",
  "transition": { "id": "51" }
}
```

### B. Terminar

```json
transitionJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "issueIdOrKey": "<KEY>",
  "transition": { "id": "61" },
  "fields": {
    "customfield_10581": [{ "id": "10415" }]
  }
}
```

### C. A QA Custodio (DAC)

```json
transitionJiraIssue
{
  "cloudId": "cnsv.atlassian.net",
  "issueIdOrKey": "<KEY>",
  "transition": { "id": "1701" },
  "fields": {
    "customfield_10801": [{ "id": "19010" }]
  },
  "update": {
    "comment": [
      {
        "add": {
          "body": {
            "type": "doc",
            "version": 1,
            "content": [
              { "type": "paragraph", "content": [
                { "type": "text", "text": "Estimados, para el repo ..." },
                { "type": "hardBreak" },
                { "type": "text", "text": "para el ambiente de  CERT favor agregar las siguiente variables de entorno" }
              ]},
              { "type": "paragraph", "content": [
                { "type": "text", "text": "MODYO_WIDGET_NAME=..." },
                { "type": "hardBreak" },
                { "type": "text", "text": "MODYO_TOKEN=" }
              ]}
            ]
          }
        }
      }
    ]
  }
}
```

Construcción del ADF: cada grupo de líneas separado por una línea en blanco es un `paragraph`; dentro del párrafo, cada línea es un `text` seguido de `hardBreak`. Los bloques de distintos repos/ambientes se separan con un `paragraph` vacío adicional.

Para "Crear repositorio" usar `"customfield_10801": [{ "id": "11574" }]`.

**Fallback del comentario:** si la transición C falla por el formato del comentario, reintentar enviando `body` como string plano. Si vuelve a fallar, ejecutar la transición C **sin** `update` y luego publicar el texto con `addCommentToJiraIssue` (`contentFormat: "markdown"`). Informar al usuario cuál camino se usó y actualizar este documento con el formato que funcionó (pendiente de confirmar en la primera ejecución real).

## Paso 6: Confirmar al usuario

Verificar con `getJiraIssue` (fields: `status`, `comment`) y responder:

- Estado final (debe ser "POR HACER (QA DEVOPS)").
- Link: `https://cnsv.atlassian.net/browse/<KEY>`.
- Que el comentario quedó publicado.

---

## Creación de repositorio

Mismo flujo (Pasos 2, 4, 5 y 6), con `customfield_10801` = `11574` "Crear repositorio" y el texto del template de abajo.

### Datos (un solo mensaje)

Preguntar siempre el ticket y el o los nombres de repositorio. El resto se propone con los valores usados en NSPUM-619 y **el usuario debe confirmarlos**:

| Campo | Propuesta (NSPUM-619) |
|---|---|
| Nombre Proyecto | Canal-Digital-Widgets |
| Nombre Repositorio | **preguntar siempre** |
| Tecnología | Vue 3 |
| Cuenta AWS | No Aplica |
| Centro Costo | 5051 |
| Uso de UI-KIT/artifactory digital | Si |
| Despliegue en Bitbucket | Si |
| Línea de Negocio | vida |
| idAplicativo | igual al nombre del repositorio |
| fabrica | cnl |
| Responsable | Pablo Bustos (Pablo.bustos@consorcio.cl), Alessandra Muñoz (Alessandra.MunozO@consorcio.cl), Cristian Moreno (CristianMoreno.ExtAgilesoft@consorcio.cl), Carlos Moraga (Carlos.Moraga@consorcio.cl), Miguel Marchant (MiguelMarchant.ExtRYC@consorcio.cl), Alejandro Cortes (AlejandroCortes.ExtRYC@consorcio.cl) |
| proyecto | Sitio Publico (NSPUM) / Sitio Privado (NSPRM), según el ticket |

### Template (un bloque por repo, separados por una línea en blanco)

```
Estimados favor de crear repositorio
Nombre Proyecto: <Nombre Proyecto>
Nombre Repositorio: <nombre-repo>
Tecnología: <Tecnología>
Cuenta AWS: <Cuenta AWS>
Centro Costo: <Centro Costo>
Uso de UI-KIT/artifactory digital : <Si/No>
Despliegue en Bitbucket : <Si/No>
Línea de Negocio: <Línea de Negocio>
idAplicativo: <nombre-repo>
fabrica: <fabrica>
Responsable: <lista de responsables>
proyecto: <Sitio Publico / Sitio Privado>
```

Validar el nombre del repo: sin espacios, en minúsculas, con el prefijo habitual (`widget_`, `bff-`, etc.). Si no lo cumple, advertir antes de continuar.

---

## Troubleshooting

| Problema | Causa probable | Solución |
|---|---|---|
| Transición `61` o `1701` no aparece | El usuario no es el Desarrollador Asignado, o el ticket está en otro estado | Revisar estado y `customfield_10351`; informar al usuario |
| Error en `customfield_10581` o `customfield_10801` | Formato incorrecto | Enviar como arreglo de objetos con `id`: `[{"id": "..."}]` |
| Error en `update.comment` | El comentario espera otro formato | Ver "Fallback del comentario" |
| B ejecutado y C falló | Error en el paso C | El ticket queda en TERMINADO (DESARROLLADOR). Reintentar solo C, **no** volver a ejecutar B |
