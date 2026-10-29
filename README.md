# Agente de CV (Reto IA Banorte)

Agente conversacional que responde sobre mi trayectoria profesional. Expone `POST /v1/responses`
compatible con [Open Responses](https://www.openresponses.org/) y `GET /health`.

## Cómo funciona

1. El CV vive en `app/cv.public.json` (solo datos públicos) y se incluye completo en el prompt del sistema.
2. Cada solicitud se valida (tamaño, roles), se limita por IP y se envía a Gemini con reintentos y modelos de respaldo.
3. La respuesta se devuelve con el esquema Open Responses, con o sin streaming SSE.

## Ejecutar en local (PowerShell)

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:GEMINI_API_KEY = "tu_clave"      # nunca la subas al repositorio
uvicorn app.main:app --port 8000
```

Prueba: `Invoke-RestMethod http://localhost:8000/health`

## Pruebas

```powershell
python -m pytest -q        # contrato y privacidad, con modelo simulado (no gasta cuota)
```

## Despliegue (Render, Docker)

1. Subir este repositorio a GitHub.
2. En Render: **New > Web Service**, conectar el repositorio, entorno **Docker**.
3. Variables de entorno: `GEMINI_API_KEY`, `GEMINI_MODELS`, `API_KEY` (la que registrarás en la plataforma).
4. Comprobar `https://<tu-servicio>.onrender.com/health`.
5. Registrar la URL base y la API key en la sección Agentes de la plataforma.

El plan gratuito de Render duerme el servicio tras un rato sin tráfico: antes de una demo, llama a `/health`
para despertarlo.

## Decisiones técnicas

| Área | Decisión | Por qué |
|---|---|---|
| Contexto | CV completo en el prompt, sin RAG | El CV cabe con holgura; RAG añadiría pérdida de información posible y complejidad |
| Fuente de verdad | `cv.public.json` versionado | Separado del código; solo contiene datos públicos |
| Privacidad | Lo privado nunca entra al contexto ni al repositorio | Lo que el modelo no tiene, no lo puede revelar; el prompt es solo segunda capa |
| `instructions` del cliente | Se ignora, igual que los mensajes system/developer | Son texto no confiable; las reglas son solo del servidor |
| Modelo | Gemini `gemini-3.5-flash-lite` configurable por variable | Funcionó en pruebas; los modelos 2.5 dieron 404 y otros 503, por eso hay reintentos y lista de respaldo |
| Operación | Límite de uso por IP, tope de tokens, logs sin contenido | Endpoint público: controlar gasto y proteger conversaciones |

## Limitaciones

- El streaming sigue los eventos de la Responses API y falta validarlo contra el validador oficial.
- El límite de uso es en memoria: se reinicia al reiniciar el servicio.
- La capa gratuita de Gemini tiene límites de solicitudes y los modelos pueden dar 503 temporales.
