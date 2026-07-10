# TechMarket Orders — Pipeline Blue-Green con Remediación Automática

**Evaluación Final Transversal (EFT) — AUY1104 Ciclo de Vida del Software II**
Repositorios: [`AUY1104-Benjamin-Santis-Main`](https://github.com/Santixie/AUY1104-Benjamin-Santis-Main) (recetas reutilizables) y [`AUY1104-Benjamin-Santis-Runner`](https://github.com/Santixie/AUY1104-Benjamin-Santis-Runner) (aplicación `demo-api`, representando el microservicio "Orders")

---

## 1. Contexto y mapeo de tecnologías

El caso de negocio "Operación Resiliencia en TechMarket" plantea la infraestructura sobre **Amazon EKS** y **Amazon ECR**. Este proyecto resuelve el mismo problema usando la infraestructura ya construida durante el semestre, que implementa los mismos estándares:

| En el enunciado | Usado en este proyecto | Por qué es equivalente |
|---|---|---|
| Amazon EKS | K3s sobre EC2 (AWS Learner Lab) | K3s implementa la API estándar de Kubernetes; los manifiestos (`Deployment`, `Service`) son idénticos a los que correrían en EKS |
| Amazon ECR | Docker Hub | Ambos son registries de imágenes OCI; el pipeline solo cambia el paso de login/push |

No se aprovisionó EKS ni ECR reales porque no aportan nada distinto a nivel de los conceptos evaluados (estrategias de despliegue, remediación automática, CI/CD).

---

## 2. Arquitectura general

```
┌─────────────────────────┐        ┌──────────────────────────────┐
│  Runner (repo alumno)    │        │  Main (repo central)          │
│  - src/ (API Express)    │        │  - build-api.yaml              │
│  - k8s/ (manifiestos)    │──uses──▶  - deploy-bluegreen.yaml       │
│  - client-bluegreen.yaml │        │    (health check + rollback)   │
└─────────────────────────┘        └──────────────────────────────┘
              │ push tag v*.*.*
              ▼
     ┌─────────────────┐
     │  GitHub Actions   │
     │  build → deploy   │
     └─────────────────┘
              │
              ▼
     ┌────────────────────────────────────┐
     │  Docker Hub (bnjmn265/demo-api:tag)  │
     └────────────────────────────────────┘
              │
              ▼
     ┌───────────────────────────────────────────────────┐
     │  Cluster K3s (EC2)                                  │
     │                                                      │
     │  Service demo-api (30090) ──▶ selector version=X    │  ← producción
     │  Service demo-api-blue-probe (30091) ──▶ Deployment  │  ← health-check
     │       demo-api-blue                                  │     únicamente,
     │  Service demo-api-green-probe (30092) ──▶ Deployment │     nunca recibe
     │       demo-api-green                                 │     tráfico real
     └───────────────────────────────────────────────────┘
```

**Idea central del diseño**: existen siempre **dos versiones desplegadas simultáneamente** (`blue` y `green`), pero el `Service` de producción (`demo-api`) solo apunta a una. Los otros dos `Service` fijos (`*-probe`) permiten que el pipeline valide la salud de la versión nueva **sin que ningún usuario real la vea**, antes de decidir si mueve el tráfico.

---

## 3. Ítem 1 — Plantillas reutilizables (repo Main)

Se separó la receta original monolítica en dos workflows independientes invocables vía `workflow_call`:

- **`build-api.yaml`**: instala dependencias, corre tests (`npm test`), construye la imagen Docker y la publica en Docker Hub con el tag exacto del commit/versión.
- **`deploy-bluegreen.yaml`**: recibe `image-name`, `image-tag` y `k3s-server-public-ip` como *inputs*, y `DOCKER_USERNAME`/`EA2_SSH_PRIVATE_KEY` como *secrets* — nada queda hardcodeado en el código, todo se inyecta desde el repo que invoca (`client-bluegreen.yaml`).

El repo `Runner` solo necesita declarar **qué** desplegar (`client-bluegreen.yaml`), delegando el **cómo** al repo central. Esto permite que cualquier otro proyecto del curso reutilice ambas recetas cambiando únicamente los `with:`.

---

## 4. Ítem 2 — Estrategia de despliegue: Blue-Green

### Por qué Blue-Green y no Canary o Rolling Update

| Estrategia | Downtime | Rollback | Riesgo en producción | Complejidad |
|---|---|---|---|---|
| **All-in-once** | Alto (corte total durante el despliegue) | Manual, lento | Muy alto — todo el tráfico ve la versión nueva de inmediato | Baja |
| **Rolling Update** | Bajo, pero con mezcla de versiones activas simultáneamente | Automático (K8s), pero mezcla ambas versiones mientras revierte | Medio — usuarios pueden recibir respuestas de ambas versiones a la vez | Baja |
| **Canary** | Muy bajo | Automático, gradual | Bajo — pero requiere más infraestructura de enrutamiento por peso de tráfico | Alta |
| **Blue-Green (elegida)** | Cero (el switch es instantáneo) | Automático e inmediato (revertir = apuntar el Service al color anterior) | Muy bajo — la versión nueva nunca recibe tráfico real hasta pasar el gate | Media |

Se eligió **Blue-Green** porque para un servicio crítico como "Orders" (pagos/pedidos), la prioridad es **cero usuarios expuestos a una versión no validada**. Canary hubiera exigido pesos de tráfico graduales (más complejo de implementar de forma confiable con `Service` nativo de Kubernetes sin un service mesh), mientras que Blue-Green resuelve el mismo objetivo con un mecanismo más simple y determinista: mientras el gate no pase, el tráfico real sigue 100% en la versión anterior.

### Cómo funciona el switch de tráfico

1. El pipeline detecta cuál color está activo leyendo el `selector` del `Service demo-api`.
2. Despliega la nueva imagen en el color **inactivo** (nunca en el que está sirviendo tráfico).
3. Corre la **Validación de Salud** contra el `Service` fijo de ese color (`*-probe`), nunca contra producción.
4. Solo si el health check pasa, hace `kubectl patch service demo-api` para mover el `selector.version` al nuevo color.

---

## 5. Ítem 3 — Remediación automática

### Dos capas de detección

Durante las pruebas se comprobó que la detección de fallos ocurre en **dos niveles independientes**, lo cual es un extra de robustez respecto al diseño original:

1. **Nativa de Kubernetes**: el `readinessProbe`/`livenessProbe` de cada `Deployment` apunta a `/health`. Si la nueva versión no responde bien, el pod nunca llega a `Ready`, y `kubectl rollout status --timeout=60s` falla.
2. **Explícita del pipeline**: un paso de `curl` reintenta 10 veces contra el puerto "probe" del color inactivo, verificando el código HTTP.

El pipeline contempla **ambos** puntos de fallo posibles (`id: deploy` y `id: healthcheck`, ambos con `continue-on-error: true`), y el paso de rollback se dispara si cualquiera de los dos falla:

```yaml
- name: 8 · ROLLBACK automático (el rollout o el health check fallaron)
  if: steps.deploy.outcome == 'failure' || steps.healthcheck.outcome == 'failure'
  run: |
    ssh ... "sudo k3s kubectl rollout undo deployment/demo-api-${{ steps.colors.outputs.target }}"
    exit 1
```

### Flujo de remediación

```
Detección (probe K8s o curl gate) → Acción (kubectl rollout undo sobre el color inactivo)
→ Notificación (log del job + estado final del pipeline en rojo, visible en GitHub Actions)
```

El job termina intencionalmente en rojo cuando se activa el rollback: esto es una decisión de diseño, no un error — refleja fielmente que el sistema detectó un problema real y actuó, información que debe quedar visible para quien revise el historial de despliegues.

---

## 6. Decisión de diseño clave: el problema del tag `:latest`

Durante las pruebas se detectó un defecto de diseño importante: los manifiestos originales fijaban `image: bnjmn265/demo-api:latest`. Como el pipeline hacía `docker push --all-tags` (publicando tanto el tag versionado como `:latest` apuntando a la misma imagen), **cada despliegue nuevo sobrescribía el contenido de `:latest`** — incluyendo cuando la imagen nueva estaba rota.

Esto rompía el rollback: `kubectl rollout undo` revierte a una revisión anterior del `Deployment`, pero si esa revisión también referenciaba `:latest`, Kubernetes terminaba re-descargando la imagen rota, no una versión anterior real.

**Solución aplicada**: los manifiestos usan un placeholder (`image: __IMAGE__`) que el pipeline sustituye dinámicamente por la imagen con el **tag exacto e inmutable** de esa versión (`sed -i "s|__IMAGE__|usuario/imagen:tag|g"`) antes de aplicar. Con esto, cada revisión de Kubernetes queda atada a una imagen específica e inmutable, y `rollout undo` funciona de forma confiable.

Este hallazgo se confirmó comparando los *digest* de las imágenes en Docker Hub: `v3.0.2-fallo` y `latest` compartían el mismo digest (evidencia del problema), mientras que tags anteriores tenían digests distintos.

---

## 7. Evidencia de pruebas realizadas

| Prueba | Resultado |
|---|---|
| Bootstrap inicial (2 Deployments + 3 Services) | ✅ Ambos colores `Running`, ambos `/health` respondiendo 200 |
| Despliegue exitoso con switch de tráfico (`v3.0.1-bluegreen`) | ✅ Detectó color activo, desplegó en el opuesto, pasó el gate, movió tráfico |
| Colisión detectada con pipeline legacy (mismo tag, mismo Service) | ✅ Diagnosticada y corregida (trigger del `client.yaml` legacy deshabilitado) |
| Fallo controlado real (`/health` forzado a HTTP 500, `v3.0.4-fallo`) | ✅ Rollback automático disparado, producción nunca afectada |
| Despliegue posterior con imagen restaurada (`v3.1.0-estable`) | ✅ Pipeline corrió limpio, cluster quedó en estado sano |

---

## 8. Impacto al negocio

La arquitectura implementada reduce directamente el riesgo operativo de TechMarket:

- **Cero downtime en despliegues**: el switch de tráfico es instantáneo y solo ocurre si la nueva versión ya demostró estar sana.
- **MTTR (tiempo medio de recuperación) cercano a cero** ante una versión defectuosa: el rollback es automático, no depende de que un operador humano detecte el problema y actúe a tiempo.
- **Reducción de errores humanos**: las variables de entorno (imagen, tag, IP del cluster) se inyectan automáticamente, eliminando pasos manuales propensos a error.
- **Trazabilidad**: cada despliegue, éxito o fallo, queda documentado en el historial de GitHub Actions con logs detallados de qué color se usó y por qué se tomó cada decisión.

---

## 9. Cómo reproducir

Requiere que el cluster tenga el bootstrap inicial aplicado (`k8s/demo-api-blue.yaml`, `k8s/demo-api-green.yaml`, `k8s/demo-api-service.yaml`) y la variable de repo `K3S_SERVER_PUBLIC_IP` actualizada con la IP pública vigente del EC2 (cambia en cada sesión de AWS Learner Lab).

```bash
git tag vX.Y.Z
git push origin vX.Y.Z
```

Esto dispara `client-bluegreen.yaml`, que encadena `build-api.yaml` → `deploy-bluegreen.yaml`.

---

## 10. Alcance y limitaciones

**Cubre**: estrategia Blue-Green completa, gate de salud en dos capas, rollback automático demostrado con fallo real, plantillas reutilizables parametrizadas.

**No cubre / posibles mejoras futuras**:
- No hay notificaciones externas (Slack/email) al activarse un rollback, solo el log del pipeline.
- El health check usa reintentos fijos (10 intentos, 3s de espera); en producción real convendría un backoff exponencial.
- No se implementó Canary como alternativa, dado que Blue-Green cubre mejor el requisito de cero riesgo para un servicio crítico.
- El diagrama de arquitectura es textual (ASCII); en un entorno real se recomendaría una herramienta de diagramación.

---

## 11. Declaración de uso de IA

Se utilizó Claude (Anthropic) como asistente durante el desarrollo de este encargo, para: apoyo en el diseño de la arquitectura Blue-Green, redacción y depuración de los workflows de GitHub Actions, diagnóstico de errores de sintaxis YAML, y estructuración de este README. Todas las decisiones de diseño, pruebas en el cluster real y validación de resultados fueron ejecutadas y verificadas por el estudiante.

## 12. Referencias (formato APA)

Kubernetes. (s. f.). *Deployments*. Kubernetes Documentation. https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

GitHub. (s. f.). *Reusing workflows*. GitHub Docs. https://docs.github.com/actions/using-workflows/reusing-workflows

Rancher Labs. (s. f.). *K3s: Lightweight Kubernetes*. https://k3s.io/

---
---

# Anexo: API de ejemplo — AUY1104 (Express + Docker)

API académica mínima en **Node.js** y **Express**, pensada para practicar contenedores y pruebas con `curl`. Responde siempre en **JSON**. Esta sección documenta la aplicación en sí (independiente de la infraestructura Blue-Green descrita arriba).

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
| `GET` | `/api/suma` | Suma `a` y `b` por query |
| `POST` | `/api/suma` | Suma `a` y `b` por cuerpo JSON |

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
│       ├── client.yaml           # Pipeline legacy (Rolling Update) — solo workflow_dispatch, ya no activo por tag
│       ├── client-bluegreen.yaml # Pipeline vigente: build + deploy Blue-Green
│       └── tests.yaml            # Pipeline de validación: solo tests en ramas de desarrollo
├── k8s/
│   ├── demo-api-blue.yaml    # Deployment color blue (imagen inyectada dinámicamente)
│   ├── demo-api-green.yaml   # Deployment color green (imagen inyectada dinámicamente)
│   ├── demo-api-service.yaml # 3 Services: demo-api (producción), blue-probe, green-probe
│   ├── deployment.yaml       # Manifiesto legacy (Rolling Update), no usado por el pipeline vigente
│   ├── service.yaml          # Service legacy, no usado por el pipeline vigente
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