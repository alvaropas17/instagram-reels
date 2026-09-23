# Plantilla — Workflow n8n: Instagram Reel → Notas

Automatiza: enlace de un Reel de Instagram → descarga del vídeo → análisis con Gemini → creación de una nota en Notion. Reporta el progreso por callbacks a una web externa (opcional).

Importable en cualquier instancia de n8n propia. No contiene credenciales ni claves.

---

## Qué incluye

- `instagram-reel-notas-videos.json` — workflow de n8n, listo para importar.

---

## Requisitos

- **n8n** (recomendado en Docker).
- **`yt-dlp`** y **`ffmpeg`** en el entorno donde corre n8n. Si usas Docker, deben estar **dentro de la imagen**.
- **Execute Command habilitado**: en n8n 2.x viene deshabilitado por defecto. Reactívalo (p. ej. con `NODES_EXCLUDE=[]`).
- Cuenta de Google con la **Generative Language API (Gemini)** habilitada.
- Una **integración de Notion** con acceso a la base de datos destino.

---

## 1. Importar

n8n → **Workflows → Import from File** → selecciona `instagram-reel-notas-videos.json`.

---

## 2. Credenciales

Crea las credenciales en n8n y asígnalas en los nodos correspondientes:

| Credencial | Tipo n8n | Nodos donde asignarla |
| --- | --- | --- |
| Gemini API | `googlePalmApi` | Iniciar subida (Gemini) · Subir bytes (Gemini) · Consultar estado (Gemini) · Generar contenido (Gemini) |
| Notion API | `notionApi` | Guardar en Notion · Añadir contenido a la ficha |

---

## 3. Configuración de nodos

Sustituye los marcadores por tus valores:

- **Webhook** → `path`: elige el tuyo (p. ej. `guardar-reel`). La URL queda `https://<tu-instancia>/webhook/<path>`.
- **Descargar vídeo (yt-dlp)**, **Leer vídeo desde disco** y **Releer vídeo para subida** → ruta del fichero. Ejemplos:
  - Linux/Docker: `/home/node/.n8n-files/{{ $json.jobId }}.mp4` (o `/data/...`)
  - Windows: `C:\Users\<usuario>\.n8n-files\{{ $json.jobId }}.mp4`
- **Guardar en Notion** → `databaseId`: `<NOTION_DATABASE_ID>`.
- **Notificar estado: completado** → sustituye la URL de la base de datos de Notion por la tuya.
- **Generar contenido (Gemini)** → modelo: por defecto `gemini-3.5-flash`. Cámbialo si quieres otro.

---

## 4. Variables de entorno (opcionales)

Los nodos de notificación usan:

- `CALLBACK_BASE_URL` → base de tu web de estado. Si no se define, se usa el `callbackBaseUrl` que llega en el webhook.
- `CALLBACK_SECRET` → valor que se envía en la cabecera `x-callback-secret` (solo si tu web lo valida).

Si no usas web de estado, ignóralas: los callbacks fallan de forma tolerante y el flujo continúa igual.

---

## 5. Trigger de entrada

`POST` al webhook con este cuerpo:

```json
{
  "enlace_reel": "https://www.instagram.com/reel/XXXXXXXXX/",
  "jobId": "<uuid>",
  "callbackBaseUrl": "https://<tu-web>"
}
```

---

## 6. Notas

- El flujo **no borra** el `.mp4` descargado. Añade un paso de borrado si no quieres acumularlos.
- Instagram puede **bloquear la descarga** desde IPs de datacenter; desde IP residencial suele funcionar mejor.
- **Cuota gratuita de Gemini**: ~20 peticiones/día por modelo (con `429` al agotarla).
- El workflow original iba acompañado de un **workflow de errores** (Error Trigger → callback). No está incluido aquí; si lo necesitas, hay que crearlo aparte y enlazarlo en **Settings → Error Workflow**.
