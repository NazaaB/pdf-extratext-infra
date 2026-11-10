# pdf-extratext-infra

Infraestructura del ecosistema **pdf-extratext**: orquesta todos los microservicios con un solo comando usando Traefik como API Gateway y balanceador de carga.

## Arquitectura del Sistema

```
        CLIENTE (curl / Frontend)
               │
               ▼
     ┌─────────────────────┐
     │  TRAEFIK GATEWAY    │  Puerto 80 (proxy) │ Puerto 8080 (dashboard)
     │  Reverse Proxy +    │
     │  Load Balancer      │
     └────────┬────────────┘
              │                     
    ┌─────────┴──────────┐          
    │ /extract           │ /summarize
    ▼                    ▼           
┌──────────────┐   ┌─────────────────┐
│ms-pdf-extract│   │  ms-ia-summary  │
│  × 5 réplicas│──▶│  × 1 instancia  │
│  1CPU / 512M │   │                 │
└──────────────┘   └────────┬────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │    OLLAMA    │
                    │  LLaMA 3.2  │
                    └──────────────┘
```

## Repositorios del Proyecto

| Repositorio | Descripción |
|---|---|
| [ms-pdf-extract](https://github.com/tomasreali/ms-pdf-extract) | Microservicio de extracción de texto de PDF |
| [ms-ia-summary](https://github.com/tomasreali/ms-ia-summary) | Microservicio de resumen con IA (Ollama) |
| [pdf-extratext-infra](https://github.com/tomasreali/pdf-extratext-infra) | Este repo: infraestructura y orquestación |
| [pdf-extratext](https://github.com/tomasreali/pdf-extratext) | Monolito original (versión de referencia) |

## Requisitos Previos

- Docker Desktop instalado y corriendo
- Git
- Los repos `ms-pdf-extract` y `ms-ia-summary` clonados en carpetas hermanas a esta:

```
Proyectos/
├── pdf-extratext-infra/   ← este repo
├── ms-pdf-extract/
└── ms-ia-summary/
```

## Cómo levantar todo el sistema

### 1. Clonar los repos en la estructura correcta
```bash
git clone https://github.com/tomasreali/pdf-extratext-infra.git
git clone https://github.com/tomasreali/ms-pdf-extract.git
git clone https://github.com/tomasreali/ms-ia-summary.git
```

### 2. Configurar variables de entorno
```bash
cd pdf-extratext-infra
cp .env.example .env
```

### 3. Levantar todo el ecosistema
```bash
docker compose up --build
```

### 4. Descargar el modelo de IA (solo la primera vez)
```bash
docker exec ollama ollama pull llama3.2
```

### 5. Verificar que todo funciona
```bash
# Health check del gateway
curl http://localhost/extract/health
curl http://localhost/summarize/health

# Traefik Dashboard
# Abrir en el navegador: http://localhost:8080
```

## Endpoints disponibles

| Método | URL | Descripción |
|---|---|---|
| GET | `http://localhost/extract/health` | Health check de ms-pdf-extract |
| POST | `http://localhost/extract` | Extrae texto de un PDF |
| POST | `http://localhost/extract-and-summarize` | Extrae texto + genera resumen con IA |
| GET | `http://localhost/summarize/health` | Health check de ms-ia-summary |
| POST | `http://localhost/summarize` | Genera resumen de texto con IA |

## Patrones de Resiliencia Implementados

### Circuit Breaker (en ms-pdf-extract)
Cuando `ms-ia-summary` no responde, el Circuit Breaker protege al sistema:
- **CLOSED**: funcionamiento normal, peticiones pasan
- **OPEN**: tras 5 fallas consecutivas, rechaza peticiones inmediatamente (evita cascada de errores)
- **HALF_OPEN**: después de 30 segundos, permite 1 petición de prueba para verificar recuperación

### Retry con Backoff Exponencial (en ms-pdf-extract)
Antes de registrar una falla en el Circuit Breaker, reintenta 3 veces:
- Intento 1: inmediato
- Intento 2: espera 1 segundo
- Intento 3: espera 2 segundos

### Rate Limit (en ms-ia-summary)
Máximo 3 resúmenes simultáneos. Si se supera el límite, responde 429 (Too Many Requests) inmediatamente en lugar de saturar a Ollama.

### Cache (en ms-ia-summary)
Los resúmenes ya generados se guardan en memoria. Si el mismo texto se solicita de nuevo, se devuelve el resultado cacheado sin llamar a Ollama.

### Degradación Elegante
Si `ms-ia-summary` cae o el Circuit Breaker está abierto, `ms-pdf-extract` sigue funcionando y devuelve la extracción de texto con `summary: null` y un mensaje explicativo en `summary_error`.

### SAGA con Orquestador (patrón aplicado en /extract-and-summarize)
El endpoint `/extract-and-summarize` implementa una SAGA donde `ms-pdf-extract` actúa como orquestador:
1. **Paso 1**: extrae el texto del PDF (siempre se ejecuta)
2. **Paso 2**: solicita el resumen a `ms-ia-summary`
3. **Compensación**: si el Paso 2 falla, no se hace rollback del Paso 1 (la extracción es stateless), sino que se devuelve el resultado parcial con el texto extraído y `summary_error` explicando la causa

Este diseño garantiza que el fallo de un servicio no invalida el trabajo ya realizado.

## Réplicas y Límites de Recursos

`ms-pdf-extract` corre con **5 réplicas**, cada una limitada a:
- 1.0 CPU
- 512 MB RAM

Traefik distribuye las peticiones entre las réplicas en modo **Round Robin** y excluye automáticamente del balanceo las réplicas que fallan el health check.
