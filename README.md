# Agente de CV - Reto IA Banorte

Agente conversacional que representa la trayectoria profesional de Mario Alberto Torres Cruz. Responde sobre experiencia, proyectos, habilidades, formación e idiomas usando únicamente un CV público estructurado.

## Demo desplegada

- Health: `https://reto-banorte-mario-torres.onrender.com/health`
- Agent Card: `https://reto-banorte-mario-torres.onrender.com/.well-known/agent-card.json`
- Open Responses: `https://reto-banorte-mario-torres.onrender.com/v1/responses`

La API requiere `Authorization: Bearer <API_KEY>` cuando `API_KEY` está configurada.

## Arquitectura

```text
Cliente
  |
  | POST /v1/responses
  v
FastAPI
  |-- valida input, roles, tamaño y rate limit
  |-- carga app/cv.public.json
  |-- construye prompt del sistema
  v
Gemini
  |
  v
Respuesta con formato Open Responses
```

El CV se incluye completo en el contexto del modelo. No se utiliza RAG en esta versión: el documento es pequeño, estable y cabe en la ventana de contexto. Esto reduce complejidad, evita errores de recuperación y facilita la evaluación. RAG sería una evolución razonable para múltiples documentos, portafolios o información cambiante.

## Fidelidad, privacidad y alcance

- `app/cv.public.json` es la única fuente de verdad.
- El modelo debe reconocer cuando un dato no aparece y no puede inventar experiencia, cifras o herramientas.
- Teléfono y correo no se incluyen en el CV público.
- Los mensajes `system`, `developer` e `instructions` enviados por el cliente se descartan; las reglas válidas viven en el servidor.
- Las preguntas fuera del perfil profesional y los intentos de prompt injection se rechazan.
- Los logs registran metadatos operativos, no el contenido de las conversaciones.

Estas medidas son guardrails de aplicación, no una garantía absoluta del modelo. Por eso se acompañan de pruebas de contrato y privacidad.

## API

### Health check

```http
GET /health
```

### Respuestas

```http
POST /v1/responses
Content-Type: application/json
Authorization: Bearer <API_KEY>
```

Ejemplo:

```json
{
  "model": "cv-agent",
  "input": "¿Qué experiencia tiene Mario en RAG?"
}
```

El endpoint acepta texto o una lista de mensajes para conversaciones multi-turno. También soporta `stream: true` mediante SSE.

### Agent Card

```http
GET /.well-known/agent-card.json
```

Publica las capacidades del agente y sus interfaces A2A/Open Responses para facilitar el descubrimiento en la plataforma del reto.

## Ejecutar localmente

Requiere Python 3.12+.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt

$env:GEMINI_API_KEY = "tu_clave_de_Gemini"
$env:GEMINI_MODELS = "gemini-3.5-flash-lite"
$env:API_KEY = "clave_para_proteger_el_endpoint"
$env:PUBLIC_BASE_URL = "http://localhost:8000"

uvicorn app.main:app --port 8000
```

Comprobación:

```powershell
Invoke-RestMethod http://localhost:8000/health
Invoke-RestMethod http://localhost:8000/.well-known/agent-card.json
```

Consulta autenticada:

```powershell
$headers = @{ Authorization = "Bearer $env:API_KEY" }
$body = @{
  model = "cv-agent"
  input = "¿Qué proyectos ha desarrollado?"
} | ConvertTo-Json -Compress

Invoke-RestMethod `
  -Uri "http://localhost:8000/v1/responses" `
  -Method Post `
  -Headers $headers `
  -ContentType "application/json; charset=utf-8" `
  -Body $body
```

Nunca subas claves al repositorio.

## Pruebas

Las pruebas usan `MOCK_LLM=1`, por lo que no consumen cuota:

```powershell
$env:MOCK_LLM = "1"
python -m pytest -q
```

Cubren:

- contrato de respuesta;
- health check;
- tarjeta de agente;
- conversaciones multi-turno;
- rechazo de mensajes de sistema del cliente;
- autenticación;
- rate limiting;
- streaming;
- ausencia de correo y teléfono en el CV público.

## Despliegue en Render

1. Crear un Web Service conectado a este repositorio.
2. Seleccionar runtime Docker.
3. Configurar estas variables en **Environment**:

```text
GEMINI_API_KEY
GEMINI_MODELS=gemini-3.5-flash-lite
API_KEY
PUBLIC_BASE_URL=https://<servicio>.onrender.com
```

4. Guardar con **Save, rebuild, and deploy**.
5. Verificar `/health` y `/.well-known/agent-card.json`.
6. Registrar `/v1/responses` y la `API_KEY` en la plataforma del reto.

## Decisiones y trade-offs

| Decisión | Motivo | Trade-off |
|---|---|---|
| CV completo en prompt | Documento pequeño y estable | Menor escalabilidad que RAG |
| JSON separado del código | Fuente versionada y auditable | Requiere actualizar el archivo al cambiar el CV |
| Gemini configurable | Permite cambiar modelo sin modificar código | Dependencia de disponibilidad y límites del proveedor |
| API key y rate limit | Protegen un endpoint público | Rate limit en memoria no es distribuido |
| Guardrails en servidor | El cliente no puede cambiar las reglas principales | No reemplaza evaluación ni filtros adicionales |

## Limitaciones y siguientes pasos

- El rate limit es local a una instancia y se reinicia al reiniciar el servicio.
- La evaluación actual es de contrato y seguridad básica; falta un golden set publicado con métricas de calidad factual.
- La tarjeta declara interfaces de descubrimiento, mientras que la conversación principal usa el contrato Open Responses del reto.
- Próximas mejoras: golden set de 20-30 casos, evaluación de groundedness, métricas p50/p95, logs estructurados y RAG si el corpus crece.

