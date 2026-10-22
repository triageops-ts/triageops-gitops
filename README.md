# triageops-gitops

Repositorio GitOps de MediTriage (TriageOps). Contiene **solo los manifiestos de Kubernetes**
del backend, organizados con Kustomize en una base común y un overlay por ambiente.
El código de la aplicación vive en el repo `Equipo---MediTriage`.

## Estructura

```
triageops-gitops/
├── base/                      # Manifiestos comunes a todos los ambientes
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/                   # Ambiente de desarrollo
    │   ├── kustomization.yaml # namespace, imagen (images:) y parches
    │   └── patch-*.yaml
    └── prod/                  # Ambiente de producción
        ├── kustomization.yaml
        └── patch-*.yaml
```

Los manifiestos de `base/` son copia de la carpeta `k8s/` del repo de la app (entrega de la
semana pasada), con una sola diferencia: se les quitó `namespace: triageops`, porque ahora
el namespace lo fija cada overlay.

## Qué NO está aquí

El **namespace** y el **secret** (`triageops-secrets`) no están en `base/` ni en los overlays:
los crea Terraform (`terraform/main.tf` del repo de la app). Kustomize no los define, para
evitar que dos herramientas administren el mismo recurso (ver ADR 0005).

## Cómo conviven dev y prod en el mismo cluster

Cada ambiente usa **su propio namespace**: `triageops-dev` y `triageops-prod`. Así los
recursos mantienen los mismos nombres en ambos ambientes sin chocar, y los selectores del
Service y del Deployment no se mezclan.

Diferencias entre ambientes:

| | dev | prod |
|---|---|---|
| Namespace | `triageops-dev` | `triageops-prod` |
| Réplicas del Deployment | 1 | 2 |
| HPA (min / max) | 1 / 3 | 2 / 10 |
| CPU requests / limits | 100m / 250m | 250m / 500m |
| Memoria requests / limits | 128Mi / 256Mi | 256Mi / 512Mi |
| Host del Ingress | `dev.meditriage.local` | `meditriage.local` |
| `LOG_LEVEL` | `debug` | `info` |
| `ENVIRONMENT` | `development` | `production` |

## Imagen

La imagen se publica en GitHub Container Registry (GHCR), etiquetada con el SHA del commit,
y se fija en cada overlay con el campo `images:` de Kustomize (`newName` y `newTag`).
En `base/` el Deployment usa el nombre corto `meditriage-backend`, que Kustomize reemplaza.

```yaml
images:
  - name: meditriage-backend
    newName: ghcr.io/triageops-ts/meditriage-backend
    newTag: "e4516f192169443645113bc3897a75828c3ce1f5"   # SHA del commit
```

El job `promote` del CI (Tarjeta 5) actualiza el `newTag` del overlay correspondiente.
Para cambiarlo a mano:

```bash
cd overlays/dev
kustomize edit set image meditriage-backend=ghcr.io/triageops-ts/meditriage-backend:<sha>
```

### Acceso al paquete privado (`ghcr-secret`)

El paquete en GHCR es privado, por lo que el Deployment declara `imagePullSecrets: ghcr-secret`.
Ese secret **no se guarda en este repo** (contiene credenciales) y debe existir en el namespace
de cada ambiente antes de aplicar:

```bash
kubectl create secret docker-registry ghcr-secret -n triageops-dev \
  --docker-server=ghcr.io --docker-username=<usuario-github> --docker-password=<token-read:packages>

kubectl create secret docker-registry ghcr-secret -n triageops-prod \
  --docker-server=ghcr.io --docker-username=<usuario-github> --docker-password=<token-read:packages>
```

El token es un Personal Access Token de GitHub con permiso `read:packages`.

## Validar sin aplicar

```bash
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod
kubectl apply -k overlays/dev  --dry-run=client
kubectl apply -k overlays/prod --dry-run=client
```

## Aplicar (cluster local con Docker Desktop)

Primero Terraform debe haber creado los namespaces y el secret `triageops-secrets` de cada ambiente, y hay que crear el `ghcr-secret` (ver arriba). Con la
configuración actual (una variable `equipo_namespace`), se hace una vez por ambiente usando
workspaces, para que cada uno tenga su propio estado:

```bash
cd terraform
terraform workspace new dev
terraform apply -var equipo_namespace=triageops-dev

terraform workspace new prod
terraform apply -var equipo_namespace=triageops-prod
```

Después:

```bash
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod
```

Para probar los Ingress en local, agregar al archivo `hosts` de la máquina:
`127.0.0.1 dev.meditriage.local meditriage.local`.

## Notas para Argo CD (Tarjeta 4)

Cada `Application` debe apuntar a la ruta del overlay (`overlays/dev` o `overlays/prod`)
y usar como destino el namespace del ambiente. El repo no contiene secretos, por lo que
puede ser público.
