# Agente de CV - Reto IA Banorte

Agente conversacional público que representa la trayectoria profesional de Mario Alberto Torres Cruz. Responde preguntas sobre experiencia, proyectos, habilidades, formación e idiomas mediante un endpoint HTTP compatible con el contrato utilizado por la plataforma del reto.

Repositorio: https://github.com/toresum89/reto-banorte-mario-torres

## URLs de producción

- Health check: `https://reto-banorte-mario-torres.onrender.com/health`
- Agent Card: `https://reto-banorte-mario-torres.onrender.com/.well-known/agent-card.json`
- Endpoint: `https://reto-banorte-mario-torres.onrender.com/v1/responses`

## Arquitectura

```text
Cliente de la plataforma
        |
        | POST /v1/responses
        v
FastAPI
  |-- autenticación Bearer
  |-- validación de input y conversación
  |-- rate limit por IP
  |-- carga del CV público
  |-- prompt del sistema y guardrails
        |
        v
Google Gemini
        |
        v
Respuesta estructurada o stream SSE
```

La tarjeta publicada en `/.well-known/agent-card.json` describe el agente, sus capacidades, la interfaz A2A/HTTP+JSON y la interfaz Open Responses esperada por la plataforma.

## Estrategia de conocimiento

El CV vive en `app/cv.public.json`, separado del código y versionado en Git.

Esta versión utiliza el CV completo dentro del prompt del sistema y no RAG en tiempo de consulta. La decisión es intencional:

- el corpus es pequeño;
- el contenido es estable;
- el documento cabe ampliamente en el contexto del modelo;
- no se necesita una base vectorial;
- se evita perder información durante chunking o recuperación;
- la solución es más simple de operar y evaluar.

RAG sería apropiado si el agente incorporara varios documentos, un portafolio extenso, certificaciones, políticas o información que cambiara con frecuencia. En ese caso habría que añadir chunking, embeddings, base vectorial, filtros de metadatos, reranking y métricas de groundedness.

## Fidelidad, privacidad y alcance

La fidelidad se controla usando `cv.public.json` como única fuente de verdad. El prompt del sistema ordena al modelo:

- no inventar cargos, años, cifras, logros o herramientas;
- reconocer cuando un dato no aparece en el CV;
- no convertir responsabilidades en logros;
- no afirmar niveles de dominio no documentados;
- hablar en tercera persona y responder en el idioma del usuario.

La privacidad se aplica antes de llamar al modelo. El JSON público no contiene teléfono ni correo personal. Por tanto, esos datos nunca llegan al contexto del modelo. Además, el prompt rechaza solicitudes de información privada o de contacto no autorizado.

El agente limita su alcance al perfil profesional. Rechaza preguntas ajenas, solicitudes para revelar el prompt, intentos de cambiar las reglas y mensajes de sistema/developer enviados por el cliente. Esos mensajes se descartan porque las instrucciones válidas pertenecen al servidor.

Estos controles son guardrails de aplicación; no sustituyen una evaluación continua ni constituyen una garantía absoluta contra todos los comportamientos de un modelo.

## Contrato HTTP

### Health check

```http
GET /health
```

Respuesta:

```json
{
  "status": "ok",
  "models": ["gemini-3.5-flash-lite"],
  "auth": true
}
```

### Open Responses

```http
POST /v1/responses
Authorization: Bearer <API_KEY>
Content-Type: application/json
```

Entrada mínima:

```json
{
  "model": "cv-agent",
  "input": "¿Qué experiencia tiene Mario en RAG?"
}
```

`input` puede ser texto o una lista de mensajes `user` y `assistant` para mantener contexto multi-turno. El campo `stream: true` activa una respuesta SSE.

El servidor valida:

- que exista `input`;
- que el contenido sea texto o mensajes válidos;
- que el último mensaje sea del usuario;
- máximo de mensajes;
- máximo de caracteres;
- autenticación;
- límite de solicitudes por IP.

### Agent Card

```http
GET /.well-known/agent-card.json
```

La tarjeta permite que la plataforma descubra el agente y sus interfaces. En producción, `PUBLIC_BASE_URL` debe apuntar a la URL pública de Render.

## Seguridad y secretos

El proyecto utiliza dos secretos distintos:

- `GEMINI_API_KEY`: permite al servidor llamar al proveedor del modelo.
- `API_KEY`: protege el endpoint público frente a solicitudes no autorizadas.

Ninguna clave debe aparecer en el código, `cv.public.json`, README, commits o GitHub. Las claves se configuran como variables de entorno en local o en Render.

Para generar `API_KEY), utiliza un generador criptográficamente seguro o la opción de generación de secretos de Render. Si una clave se expone, debe revocarse y reemplazarse inmediatamente.

## Ejecución local

Requiere Python 3.12 o superior.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt

$env:GEMINI_API_KEY = "tu_clave_de_Gemini"
$env:GEMINI_MODELS = "gemini-3.5-flash-lite"
$env:API_KEY = "genera_una_clave_local"
$env:PUBLIC_BASE_URL = "http://localhost:8000"

uvicorn app.main:app --port 8000
```

Verificación:

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

## Pruebas

Las pruebas utilizan `MOCK_LLM=1` para evitar consumo de cuota:

```powershell
$env:MOCK_LLM = "1"
python -m pytest -q
```

La suite cubre:

- health check;
- tarjeta de agente y sus interfaces;
- forma de la respuesta;
- mensajes multi-turno;
- descarte de instrucciones del cliente;
- errores de validación;
- autenticación;
- rate limiting;
- streaming SSE;
- ausencia de correo y teléfono en el JSON público.

Resultado verificado localmente:

```text
10 passed
```

La suite valida contrato y controles básicos. Todavía no reemplaza un golden set de calidad semántica con evaluación factual.

## Despliegue en Render

1. Crear un **Web Service** en Render.
2. Conectar el repositorio de GitHub.
3. Seleccionar runtime **Docker**.
4. Usar el `Dockerfile` del repositorio.
5. En **Environment**, agregar:

```text
GEMINI_API_KEY=<secreto de Gemini>
GEMINI_MODELS=gemini-3.5-flash-lite
API_KEY=<secreto del endpoint>
PUBLIC_BASE_URL=https://reto-banorte-mario-torres.onrender.com
```

6. Guardar con **Save, rebuild, and deploy**.
7. Esperar el deploy y verificar:

```text
/health
/.well-known/agent-card.json
```

8. Registrar en la plataforma Banorte:

```text
https://reto-banorte-mario-torres.onrender.com/v1/responses
```

Usar como API key la variable `API_KEY`, no `GEMINI_API_KEY`.

En el plan gratuito, Render puede suspender el servicio después de un periodo sin tráfico. Para una demostración, comprobar `/health` antes de iniciar.

## Decisiones y trade-offs

| Área | Decisión | Justificación | Trade-off |
|---|---|---|---|
| Contexto | CV completo en el prompt | Corpus pequeño y estable | Menor escalabilidad que RAG |
| Fuente de verdad | JSON público versionado | Auditable y separado del código | Requiere actualizar el archivo |
| Modelo | Gemini configurable | Permite cambiar proveedor/modelo sin reescribir la API | Dependencia de disponibilidad y cuotas |
| Seguridad | Bearer API key y rate limit | Protege un endpoint público | Rate limit en memoria y por instancia |
| Privacidad | Datos sensibles fuera del contexto | Prevención más fuerte que confiar solo en instrucciones | Requiere disciplina al actualizar el CV |
| Operación | Docker + Render | Despliegue reproducible y HTTPS gestionado | Dependencia del proveedor de hosting |

## Limitaciones y trabajo futuro

- El rate limit vive en memoria y no es distribuido entre instancias.
- La evaluación actual cubre contrato y seguridad básica, no una métrica completa de calidad factual.
- No se usa RAG porque el tamaño actual del CV no lo justifica.
- La disponibilidad depende del proveedor del modelo y del plan de Render.
- Próximas mejoras: golden set de 20-30 casos, métricas factual/groundedness, latencia p50/p95, observabilidad más detallada y RAG si el corpus crece.

