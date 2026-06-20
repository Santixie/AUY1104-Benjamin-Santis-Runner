# API de ejemplo — AUY1104 (Express + Docker)

API académica mínima en **Node.js** y **Express**, pensada para practicar contenedores y pruebas con `curl`. Responde siempre en **JSON**.

## Requisitos

- Node.js 20+ (ejecución local)
- Docker (ejecución en contenedor)

## Ejecución local

```bash
npm install
npm start
```

Por defecto escucha en el puerto **3000**: `http://localhost:3000`.

## Docker

Construir la imagen (desde esta carpeta):

```bash
docker build -t auy1104-api-ejemplo .
```

Ejecutar el contenedor:

```bash
docker run --rm -p 3000:3000 -ti auy1104-api-ejemplo
```

Si el puerto **3000** de tu equipo ya está ocupado, usa otro puerto en el host (el primero del mapeo) y deja **3000** como puerto del contenedor:

```bash
docker run --rm -p 8080:3000 -ti auy1104-api-ejemplo
```

En ese caso las URLs de los ejemplos serían `http://localhost:8080/...`.

## Endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/health` | Estado del servicio |
| `GET` | `/api/saludo` | Saludo en JSON; query opcional `nombre` |
| `POST` | `/api/echo` | Devuelve en JSON el cuerpo enviado |

Cualquier otra ruta responde **404** con JSON: `{ "error": "Ruta no encontrada" }`.

## Ejemplos con `curl`

Sustituye `localhost:3000` por `localhost:8080` (u otro) si mapeaste el contenedor distinto, por ejemplo `-p 8080:3000`.

### `GET /health`

```bash
curl -s http://localhost:3000/health
```

### `GET /api/saludo`

Sin parámetros (usa el nombre por defecto `estudiante`):

```bash
curl -s http://localhost:3000/api/saludo
```

Con query `nombre`:

```bash
curl -s "http://localhost:3000/api/saludo?nombre=Duoc"
```

### `POST /api/echo`

Envía JSON en el cuerpo; la API responde con estado **201** y el objeto recibido en `recibido`.

```bash
curl -s -X POST http://localhost:3000/api/echo \
  -H "Content-Type: application/json" \
  -d '{"curso":"AUY1104","modulo":"Docker"}'
```

### Ruta inexistente (404)

```bash
curl -s http://localhost:3000/api/no-existe
```

## Estructura del proyecto

```
AUY1104-Benjamin-Santis-Runner/
├── .github/
│   └── workflows/
│       ├── client.yaml       # Pipeline principal: build, push y deploy al activar un tag
│       └── tests.yaml        # Pipeline de validación: solo tests en ramas de desarrollo
├── k8s/
│   ├── deployment.yaml       # Manifiesto de despliegue de la API en K3s
│   ├── service.yaml          # Servicio NodePort expuesto en el puerto 30090
│   ├── nginx-deployment.yaml # Deployment nginx (auxiliar)
│   └── nginx-service.yaml    # Servicio nginx NodePort 30080 (auxiliar)
├── src/
│   ├── index.js              # Servidor Express con los endpoints de la API
│   └── lib/
│       └── ejemplo.js        # Lógica de negocio separada de las rutas
├── tests/
│   ├── app.test.js           # Tests de integración HTTP con supertest
│   └── ejemplo.test.js       # Tests unitarios de la lógica de negocio
├── coverage/                 # Reporte de cobertura generado por Jest (ignorado por git)
├── Dockerfile                # Imagen node:20-alpine, expone puerto 3000
├── .dockerignore
├── .gitignore
├── jest.config.js            # Configuración de Jest con cobertura lcov
├── package.json
└── README.md
```

## Variables de entorno

| Variable | Valor por defecto | Uso |
|----------|-------------------|-----|
| `PORT` | `3000` | Puerto donde escucha la app dentro del contenedor o en local |
