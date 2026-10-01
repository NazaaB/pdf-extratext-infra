# pdf-extratext-infra

Infraestructura del ecosistema **pdf-extratext**. Docker Compose maestro con **Traefik** como API Gateway y balanceador de carga.

## 🏗️ Arquitectura

```
Cliente → Traefik (puerto 80) → /extract             → ms-pdf-extract (5 réplicas)
                               → /extract-and-summarize → ms-pdf-extract (5 réplicas)
                               → /summarize           → ms-ia-summary
                                                              ↓
                                                            Ollama
```

## 📁 Estructura de carpetas

```
pdf-extratext-infra/
├── docker-compose.yml      ← Orquesta todo el ecosistema
├── traefik/
│   ├── traefik.yml         ← Configuración estática de Traefik
│   └── dynamic.yml         ← Configuración dinámica (middlewares)
├── .env / .env.example
├── .gitignore
└── README.md
```

## 📋 Requisitos previos

- **Docker** y **Docker Compose** instalados
- Los repos `ms-pdf-extract` y `ms-ia-summary` clonados al **mismo nivel** que este repo:

```
Desarrollo de Software/
├── ms-pdf-extract/         ← Microservicio de extracción
├── ms-ia-summary/          ← Microservicio de resumen con IA
└── pdf-extratext-infra/    ← Este repo (infraestructura)
```

## 🚀 Levantar todo el ecosistema

```bash
# 1. Copiar variables de entorno
cp .env.example .env

# 2. Levantar todo (un solo comando)
docker compose up --build

# 3. En otra terminal, descargar el modelo de Ollama (solo la primera vez)
docker exec ollama ollama pull llama3.2
```

## ✅ Verificar que funciona

```bash
# Health check de extracción (pasa por Traefik, distribuido entre 5 réplicas)
curl http://localhost/health

# Dashboard de Traefik (ver réplicas y balanceo)
# Abrir en el navegador: http://localhost:8080

# Probar extracción
curl -X POST http://localhost/extract -F "file=@test.pdf"

# Probar extracción + resumen (comunicación inter-servicios)
curl -X POST http://localhost/extract-and-summarize -F "file=@test.pdf"

# Probar resumen directo
curl -X POST http://localhost/summarize \
  -H "Content-Type: application/json" \
  -d '{"text": "Texto largo de ejemplo para que la IA lo resuma correctamente."}'
```

## 🔌 Puertos

| Servicio | Puerto | Descripción |
|----------|--------|-------------|
| Traefik (HTTP) | 80 | Punto de entrada principal |
| Traefik Dashboard | 8080 | Panel de administración |
| Ollama | 11434 | Motor de IA |

## ⚙️ Configuración de réplicas y recursos

El microservicio `ms-pdf-extract` está configurado con:
- **5 réplicas** para balanceo de carga
- **Límites por réplica:** 1.0 CPU, 512MB RAM
- **Health checks** cada 10 segundos para detectar réplicas caídas

## 📦 Variables de entorno

| Variable | Default | Descripción |
|----------|---------|-------------|
| `MAX_FILE_SIZE_MB` | 10 | Tamaño máximo de PDF permitido |
| `OLLAMA_URL` | `http://ollama:11434` | URL del servidor Ollama |
| `OLLAMA_MODEL` | `llama3.2` | Modelo de IA a utilizar |
