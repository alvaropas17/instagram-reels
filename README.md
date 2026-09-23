# Plantilla — Workflow n8n: Instagram Reel → Notas

Automatiza: enlace de un Reel de Instagram → descarga del vídeo → análisis con Gemini (vídeo + audio) → creación de una nota en Notion. Opcionalmente reporta el progreso a una web externa por callbacks.

Importable en cualquier instancia propia de n8n. **No contiene credenciales ni datos personales** (usa marcadores que sustituirás tú).

---

## Contenido

- `instagram-reel-notas-videos.json` — workflow de n8n, listo para importar.
- `README.md` — este documento.

---

## Cómo funciona (resumen del flujo)

```
Webhook
 → Normalizar URL → ¿Enlace de Instagram?
 → Descargar vídeo (yt-dlp) → Leer vídeo desde disco → Calcular tamaño y MIME
 → Iniciar subida (Gemini) → Releer vídeo para subida → Subir bytes (Gemini)
 → Esperar 2 s → Consultar estado (Gemini) → Contar intento → ¿Vídeo activo? (bucle)
 → Generar contenido (Gemini) → Preparar datos para Notion
 → Guardar en Notion → Añadir contenido a la ficha → Notificar estado: completado
```

Los nodos **"Notificar estado: …"** envían cada fase a una web de estado (opcional).

---

# Puesta en marcha (paso a paso, en orden)

## 0. Requisitos previos

- **n8n** instalado (recomendado en **Docker**; n8n ya avisa de que correrlo fuera de un contenedor está deprecado).
- **`yt-dlp`** y **`ffmpeg`** accesibles para el proceso de n8n.
- Una cuenta de Google con la **Generative Language API (Gemini)** habilitada.
- Una **integración de Notion** con acceso a una base de datos.

## 1. Preparar n8n (variables de entorno)

| Qué | Variable | Valor |
| --- | --- | --- |
| Habilitar **Execute Command** (n8n 2.x lo trae deshabilitado) | `NODES_EXCLUDE` | `[]` |
| Permitir leer/escribir en la carpeta de descarga | `N8N_ALLOWED_READ_FILE_ACCESS_PATHS` · `N8N_ALLOWED_WRITE_FILE_ACCESS_PATHS` · `N8N_RESTRICT_FILE_ACCESS_TO` | tu carpeta (p. ej. `~/.n8n-files`) |
| Permiso para que los Code nodes lean variables de entorno (si usas el bloque `$env`) | `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` |

Ejemplo (PowerShell, arranque directo):

```powershell
$env:NODES_EXCLUDE="[]"; n8n start
```

## 2. Instalar dependencias (ejemplo Windows)

```powershell
winget install --id yt-dlp.yt-dlp -e
winget install --id yt-dlp.FFmpeg -e   # o Gyan.FFmpeg
yt-dlp --version
ffmpeg -version
```

Actualiza `yt-dlp` con frecuencia (los extractores de Instagram cambian a menudo):

```powershell
winget upgrade --id yt-dlp.yt-dlp -e
```

> En **Docker**, `yt-dlp` y `ffmpeg` deben ir **dentro de la imagen** (no hay shell del host).

## 3. Importar el workflow

n8n → **Workflows → Import from File** → selecciona `instagram-reel-notas-videos.json`.

## 4. Crear y asignar credenciales

| Credencial | Tipo n8n | Dónde asignarla |
| --- | --- | --- |
| Gemini API | `googlePalmApi` ("Google Gemini(PaLM) Api") | Iniciar subida (Gemini) · Subir bytes (Gemini) · Consultar estado (Gemini) · Generar contenido (Gemini) |
| Notion API | `notionApi` | Guardar en Notion · Añadir contenido a la ficha |

**Gemini**
- El nombre "(PaLM)" es heredado: es la credencial correcta de Gemini.
- Campos: **Host** = `https://generativelanguage.googleapis.com` y **API Key**.
- **Allowed HTTP Request Domains = All** (imprescindible para usarla en nodos HTTP Request).
- Las claves nuevas de AI Studio tienen formato **`AQ...`** (ya no `AIza...`).
- Comprobar la clave:
  ```powershell
  curl.exe -s "https://generativelanguage.googleapis.com/v1beta/models?key=TU_CLAVE"
  ```
  Si lista modelos → OK. `401` → clave o método de auth incorrectos.

**Notion**
- Crea una integración interna en `https://www.notion.so/my-integrations`.
- **Comparte la base de datos con la integración** (··· → **Connections**). Sin esto, el token es válido pero la API devuelve `404`.

## 5. Configurar los nodos

Sustituye los marcadores por tus valores:

- **Webhook** → `path`: viene como `guardar-reel`. Cámbialo si quieres. La URL queda `https://<tu-instancia>/webhook/<path>`.
- **Descargar vídeo (yt-dlp)**, **Leer vídeo desde disco** y **Releer vídeo para subida** → ruta del fichero:
  - Linux/Docker: `/home/node/.n8n-files/{{ $json.jobId }}.mp4` (o `/data/...`)
  - Windows: `C:\Users\<usuario>\.n8n-files\{{ $json.jobId }}.mp4`
- **Guardar en Notion** → `databaseId`: `<NOTION_DATABASE_ID>`.
- **Notificar estado: completado** → sustituye la URL de la base de datos de Notion por la tuya.
- **Generar contenido (Gemini)** → modelo: por defecto `gemini-3.5-flash`.

## 6. Activar y disparar

`POST` al webhook con este cuerpo:

```json
{
  "enlace_reel": "https://www.instagram.com/reel/XXXXXXXXX/",
  "jobId": "<uuid>",
  "callbackBaseUrl": "https://<tu-web>"
}
```

En pruebas usa la **Test URL** del webhook (n8n escuchando con "Test workflow"); en producción, activa el workflow (toggle **Active**) y usa la URL `/webhook/...`.

## 7. (Opcional) Web de estado

Los nodos de notificación usan:

- `CALLBACK_BASE_URL` → base de tu web de estado. Si no se define, se usa el `callbackBaseUrl` que llega en el webhook.
- `CALLBACK_SECRET` → valor enviado en la cabecera `x-callback-secret` (solo si tu web lo valida).

Si no usas web de estado, ignóralas: los callbacks fallan de forma **tolerante** y el flujo continúa.

---

# Errores frecuentes y cómo resolverlos

## E1 · `Execute Command` no aparece, o el workflow falla al arrancar
**Causa:** en n8n 2.x está deshabilitado por defecto.
**Solución:** arranca con `NODES_EXCLUDE=[]`.

## E2 · `Access to the file is not allowed`
**Causa:** el acceso a ficheros está restringido a ciertas rutas.
**Solución:** incluye la carpeta de descarga en las rutas permitidas (`N8N_ALLOWED_READ_FILE_ACCESS_PATHS` / `N8N_ALLOWED_WRITE_FILE_ACCESS_PATHS` / `N8N_RESTRICT_FILE_ACCESS_TO`). Comprueba que la ruta del nodo es exactamente la permitida.

## E3 · `No file matching the selector "…mp4" found`
**Causa:** el selector de fichero se evaluó con un `jobId` vacío.
**Solución:** toma el id del nodo de origen explícitamente, p. ej. `{{ $('Normalizar URL').first().json.jobId }}`, tanto en la ruta de descarga como en el selector.

## E4 · `Module 'crypto' is disallowed`
**Causa:** los nodos **Code** de n8n no pueden usar `require('crypto')` (el task runner lo bloquea).
**Solución:** no uses `require(...)` en Code; usa los helpers de n8n y `$env` para lo que necesites.

## E5 · `This operation expects the node's input data to contain a binary file 'data'`
**Causa:** el nodo **HTTP Request** solo conserva el binario de entrada cuando la respuesta es un fichero (`responseFormat === 'file'`). Con respuesta JSON, descarta el binario.
**Solución:** inserta un nodo **Read File** ("Releer vídeo para subida") antes de "Subir bytes" y toma la URL de subida del nodo anterior:
```
{{ $('Iniciar subida (Gemini)').item.json.headers["x-goog-upload-url"] }}
```

## E6 · HTTP `503 UNAVAILABLE` — "model is currently experiencing high demand"
**Causa:** el modelo está saturado o caído (no es tu configuración).
**Solución:** usa otro modelo; probado y estable: **`gemini-3.5-flash`** (alternativa rápida: `gemini-3.1-flash-lite`). Compruébalo antes con la lista de modelos.
> `retryOnFail` ayuda con picos, pero **no** arregla un modelo caído.

## E7 · `429 RESOURCE_EXHAUSTED` (quota exceeded)
**Causa:** cuota del **free tier** de Gemini agotada (~**20 peticiones/día** por modelo).
**Solución:** espera al reinicio diario, usa otro modelo, o pasa Gemini a plan de pago (Pay-as-you-go).

## E8 · `PROHIBITED_CONTENT` o `Unexpected end of JSON input` al preparar la nota
**Causa:** Gemini bloqueó el contenido (`promptFeedback.blockReason`, sin `candidates`), así que el parser recibe texto vacío y falla.
**Solución:** en "Preparar datos para Notion", comprueba primero `candidates` y `promptFeedback.blockReason` y lanza un error claro en lugar de intentar el `JSON.parse`.

## E9 · Los reintentos no esperan lo que configuras
**Causa:** n8n limita `waitBetweenTries` a **5000 ms**.
**Solución:** usa 5000 ms o menos.

## E10 · Nodo Notion: `Cannot read properties of undefined (reading 'text')`
**Causa:** en el nodo Notion v2.2, las propiedades `rich_text` necesitan **`richText: false`**; si no, intenta leer `value.text.text`.
**Solución:** añade `"richText": false` a cada propiedad `rich_text` ("Resumen", "Puntos clave").

## E11 · Notion `404 object_not_found` (y `search` devuelve 0)
**Causa:** una integración de Notion solo ve lo que se le comparte explícitamente.
**Solución:** comparte la base de datos con la integración (··· → **Connections**). Verifica `GET /v1/databases/<id>` → debe devolver **200**.

## E12 · Gemini `401 Authorization failed`
**Causa:** credencial incorrecta u obsoleta.
**Solución:** revisa formato de clave (`AQ...`), que **Allowed HTTP Request Domains = All**, y verifica con `curl .../models?key=TU_CLAVE`.

## E13 · yt-dlp: "redirected to the login page" / "rate-limit exceeded"
**Causa:** Instagram bloquea el acceso anónimo en algunos casos (más aún desde IPs de datacenter).
**Solución:** los Reels **públicos** suelen funcionar sin cookies. Si falla, usa `--cookies-from-browser firefox`. **Evita Chrome/Edge** (su cifrado DPAPI está roto en yt-dlp). Usa una cuenta secundaria y baja frecuencia; actualiza yt-dlp.

## E14 · El `.mp4` se acumula en disco
**Causa:** el flujo **no borra** el vídeo descargado.
**Solución:** añade un paso de borrado al terminar (también en la rama de error), o descarga a una carpeta temporal.

## E15 · Avisos de deprecación de n8n (task runners, "outside container")
**Causa:** el modo de runner interno y correr n8n sin contenedor están deprecados.
**Solución:** usa Docker y, si aplica, task runner en modo externo. No rompe, pero conviene migrar.

## E16 · Los Code nodes no ven `CALLBACK_*` en `$env`
**Causa:** el acceso a variables de entorno en nodos está bloqueado.
**Solución:** define las variables y pon `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. Si no las usas, los callbacks ya tienen fallback a `callbackBaseUrl` y a `http://localhost:3333`.

---

## Notas

- **El workflow de errores** (Error Trigger → callback) **no está incluido**. El original iba acompañado de uno; si lo quieres, créalo aparte y enlázalo en **Settings → Error Workflow**.
- **Retención de ejecuciones**: n8n poda ejecuciones antiguas según su configuración; para depurar, revisa la pestaña **Executions** del workflow.
- **Cuota/limitaciones de n8n**: la edición Community self-hosted no tiene límites de ejecuciones; la licencia *Sustainable Use* no permite revender n8n como servicio.
