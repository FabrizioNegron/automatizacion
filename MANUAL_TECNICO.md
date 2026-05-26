# Manual Técnico — Despliegue con Kubernetes
## Proyecto Turismo Cañete — AS242S4_PII_T06

**Fecha:** Mayo 2026  
**Stack:** Python Flask · React · MongoDB · Docker · Kubernetes (k3s) · AWS EC2

---

## Índice

1. Estructura del proyecto
2. Pre-requisitos
3. Fase 1 — Base de datos MongoDB en EC2
4. Fase 2 — Backend Flask en EC2
5. Fase 3 — Frontend React en EC2
6. Resumen de IPs y puertos
7. Comandos de mantenimiento

---

## 1. Estructura del proyecto

```
AS242S4_PII_T06-be/          ← Backend Python Flask
├── app/
│   ├── controllers/
│   ├── models/
│   ├── routes/
│   └── services/
├── k8s/
│   ├── mongodb/             ← Manifiestos EC2 #1
│   │   ├── 00-namespace.yaml
│   │   ├── 01-persistentvolume.yaml
│   │   ├── 02-configmap-init.yaml
│   │   ├── 03-deployment.yaml
│   │   └── 04-service.yaml
│   └── backend/             ← Manifiestos EC2 #2
│       ├── 00-namespace.yaml
│       ├── 01-secret.yaml
│       ├── 02-deployment.yaml
│       └── 03-service.yaml
├── .env
├── .dockerignore
├── Dockerfile
└── requirements.txt

AS242S4_PII_T06-fe/          ← Frontend React + Vite
├── src/
│   ├── pages/
│   ├── services/            ← Usan VITE_API_URL
│   └── components/
├── k8s/
│   └── frontend/            ← Manifiestos EC2 #3
│       ├── 00-namespace.yaml
│       ├── 01-deployment.yaml
│       └── 02-service.yaml
├── .env
├── .dockerignore
├── Dockerfile               ← Multi-stage: Node build + Nginx
└── nginx.conf
```

---

## 2. Pre-requisitos

- Cuenta en **AWS** con 3 instancias EC2 Ubuntu 22.04 LTS creadas
- Cuenta en **Docker Hub** (usuario: `fabrizionegron`)
- **Docker Desktop** instalado en tu PC
- Claves `.pem` para cada instancia
- Puertos abiertos en cada Security Group (ver sección 6)

---

## 3. Fase 1 — Base de datos MongoDB en EC2

**Instancia:** `100.55.177.41` | **Clave:** `bdk8s.pem`

### 3.1 Conectarse a la EC2

```bash
ssh -i "bdk8s.pem" ubuntu@100.55.177.41
```

### 3.2 Instalar k3s

```bash
sudo apt update && sudo apt upgrade -y
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes
```

### 3.3 Copiar manifiestos (desde tu PC, nueva terminal)

```bash
scp -i "bdk8s.pem" -r "C:\Users\MSI\Desktop\Juanita\AS242S4_PII_T06-be\k8s\mongodb" ubuntu@100.55.177.41:~/k8s-mongodb
```

### 3.4 Desplegar MongoDB

```bash
sudo kubectl apply -f ~/k8s-mongodb/00-namespace.yaml
sudo kubectl apply -f ~/k8s-mongodb/01-persistentvolume.yaml
sudo kubectl apply -f ~/k8s-mongodb/02-configmap-init.yaml
sudo kubectl apply -f ~/k8s-mongodb/03-deployment.yaml
sudo kubectl apply -f ~/k8s-mongodb/04-service.yaml
```

### 3.5 Verificar

```bash
# Esperar estado Running
sudo kubectl get pods -n turismo-db

# Ver logs — buscar: ✅ TurismoCaneteDB inicializada correctamente
sudo kubectl logs -n turismo-db deployment/mongodb

# Verificar datos dentro del pod
sudo kubectl exec -it -n turismo-db deployment/mongodb -- mongosh
> use TurismoCaneteDB
> db.distritos.countDocuments()   // debe retornar 6
> db.actividades.countDocuments() // debe retornar 8
> exit
```

### 3.6 Qué hace cada manifiesto

| Archivo | Qué hace |
|---------|----------|
| `00-namespace.yaml` | Crea el namespace `turismo-db` |
| `01-persistentvolume.yaml` | Reserva 5Gi en `/data/mongodb` del nodo para persistir datos |
| `02-configmap-init.yaml` | Script JS que crea colecciones y carga datos semilla |
| `03-deployment.yaml` | Despliega el pod MongoDB 7.0 con health checks |
| `04-service.yaml` | Expone MongoDB en el puerto NodePort `30017` |

---

## 4. Fase 2 — Backend Flask en EC2

**Instancia:** `3.223.121.15` | **Clave:** `backK8s.pem`

### 4.1 Preparar el Dockerfile (ya hecho)

El backend tiene un `.dockerignore` que excluye:
- `.env` (Kubernetes inyecta MONGO_URI desde el Secret)
- `venv/`, `__pycache__/`, `.git/`, `k8s/`

### 4.2 Construir y subir la imagen

```bash
cd C:\Users\MSI\Desktop\Juanita\AS242S4_PII_T06-be
docker build -t fabrizionegron/turismo-backend:latest .
docker login
docker push fabrizionegron/turismo-backend:latest
```

### 4.3 Conectarse a la EC2

```bash
ssh -i "backK8s.pem" ubuntu@3.223.121.15
```

### 4.4 Instalar k3s

```bash
sudo apt update && sudo apt upgrade -y
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes
```

### 4.5 Copiar manifiestos (desde tu PC, nueva terminal)

```bash
scp -i "backK8s.pem" -r "C:\Users\MSI\Desktop\Juanita\AS242S4_PII_T06-be\k8s\backend" ubuntu@3.223.121.15:~/k8s-backend
```

### 4.6 Desplegar el backend

```bash
sudo kubectl apply -f ~/k8s-backend/00-namespace.yaml
sudo kubectl apply -f ~/k8s-backend/01-secret.yaml
sudo kubectl apply -f ~/k8s-backend/02-deployment.yaml
sudo kubectl apply -f ~/k8s-backend/03-service.yaml
```

### 4.7 Verificar

```bash
sudo kubectl get pods -n turismo-backend
sudo kubectl logs -n turismo-backend deployment/turismo-backend
```

Probar en el navegador:
```
http://3.223.121.15:30050/health
http://3.223.121.15:30050/apidocs/
```

### 4.8 Qué hace cada manifiesto

| Archivo | Qué hace |
|---------|----------|
| `00-namespace.yaml` | Crea el namespace `turismo-backend` |
| `01-secret.yaml` | Almacena `MONGO_URI` de forma segura |
| `02-deployment.yaml` | Despliega el pod Flask con la imagen de Docker Hub |
| `03-service.yaml` | Expone la API en el puerto NodePort `30050` |

---

## 5. Fase 3 — Frontend React en EC2

**Instancia:** `32.192.155.89` | **Clave:** `frontk8s.pem`

### 5.1 Preparar el frontend

Todos los servicios usan `import.meta.env.VITE_API_URL` en lugar de URLs hardcodeadas.
El `.env` del frontend contiene:
```
VITE_API_URL=http://3.223.121.15:30050
```

### 5.2 Construir y subir la imagen

El Dockerfile usa **multi-stage build**:
- **Stage 1 (builder):** Node 20 compila el proyecto con `npm run build`. La variable `VITE_API_URL` se pasa como `--build-arg` y queda embebida en el bundle.
- **Stage 2 (serve):** Nginx Alpine sirve los archivos estáticos del build.

```bash
cd C:\Users\MSI\Desktop\Juanita\AS242S4_PII_T06-fe
docker build --build-arg VITE_API_URL=http://3.223.121.15:30050 -t fabrizionegron/turismo-frontend:latest .
docker login
docker push fabrizionegron/turismo-frontend:latest
```

### 5.3 Probar localmente antes de subir

```bash
docker run -p 8080:80 fabrizionegron/turismo-frontend:latest
```
Abrir: `http://localhost:8080`

### 5.4 Conectarse a la EC2

```bash
ssh -i "frontk8s.pem" ubuntu@32.192.155.89
```

### 5.5 Instalar k3s

```bash
sudo apt update && sudo apt upgrade -y
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes
```

### 5.6 Copiar manifiestos (desde tu PC, nueva terminal)

```bash
scp -i "frontk8s.pem" -r "C:\Users\MSI\Desktop\Juanita\AS242S4_PII_T06-fe\k8s\frontend" ubuntu@32.192.155.89:~/k8s-frontend
```

### 5.7 Desplegar el frontend

```bash
sudo kubectl apply -f ~/k8s-frontend/00-namespace.yaml
sudo kubectl apply -f ~/k8s-frontend/01-deployment.yaml
sudo kubectl apply -f ~/k8s-frontend/02-service.yaml
```

### 5.8 Verificar

```bash
sudo kubectl get pods -n turismo-frontend
sudo kubectl logs -n turismo-frontend deployment/turismo-frontend
```

Probar en el navegador:
```
http://32.192.155.89:30080
```

### 5.9 Qué hace cada manifiesto

| Archivo | Qué hace |
|---------|----------|
| `00-namespace.yaml` | Crea el namespace `turismo-frontend` |
| `01-deployment.yaml` | Despliega el pod Nginx con el build de React |
| `02-service.yaml` | Expone el frontend en el puerto NodePort `30080` |

---

## 6. Resumen de IPs y puertos

| Componente | IP EC2 | Puerto NodePort | URL de acceso |
|------------|--------|-----------------|---------------|
| MongoDB | 100.55.177.41 | 30017 | `mongodb://100.55.177.41:30017/TurismoCaneteDB` |
| Backend Flask | 3.223.121.15 | 30050 | `http://3.223.121.15:30050` |
| Frontend React | 32.192.155.89 | 30080 | `http://32.192.155.89:30080` |

### Security Groups requeridos

**EC2 MongoDB:**
| Puerto | Protocolo | Origen |
|--------|-----------|--------|
| 22 | TCP | Tu IP |
| 30017 | TCP | IP del backend (3.223.121.15) |

**EC2 Backend:**
| Puerto | Protocolo | Origen |
|--------|-----------|--------|
| 22 | TCP | Tu IP |
| 30050 | TCP | 0.0.0.0/0 |

**EC2 Frontend:**
| Puerto | Protocolo | Origen |
|--------|-----------|--------|
| 22 | TCP | Tu IP |
| 30080 | TCP | 0.0.0.0/0 |

---

## 7. Comandos de mantenimiento

### Ver estado de todos los pods

```bash
# MongoDB
sudo kubectl get all -n turismo-db

# Backend
sudo kubectl get all -n turismo-backend

# Frontend
sudo kubectl get all -n turismo-frontend
```

### Reiniciar un servicio

```bash
sudo kubectl rollout restart deployment/mongodb -n turismo-db
sudo kubectl rollout restart deployment/turismo-backend -n turismo-backend
sudo kubectl rollout restart deployment/turismo-frontend -n turismo-frontend
```

### Ver logs en tiempo real

```bash
sudo kubectl logs -f -n turismo-backend deployment/turismo-backend
```

### Actualizar a una nueva versión

```bash
# 1. Rebuild y push de la imagen
docker build -t fabrizionegron/turismo-backend:latest .
docker push fabrizionegron/turismo-backend:latest

# 2. Forzar descarga de la nueva imagen en la EC2
sudo kubectl rollout restart deployment/turismo-backend -n turismo-backend

# 3. Verificar que el rollout fue exitoso
sudo kubectl rollout status deployment/turismo-backend -n turismo-backend
```

### Eliminar y redesplegar todo (desde cero)

```bash
# ⚠️ Esto borra los datos de MongoDB
sudo kubectl delete namespace turismo-db
sudo kubectl delete pv mongodb-pv
sudo kubectl apply -f ~/k8s-mongodb/

# Backend y frontend (no tienen datos persistentes)
sudo kubectl delete namespace turismo-backend
sudo kubectl apply -f ~/k8s-backend/

sudo kubectl delete namespace turismo-frontend
sudo kubectl apply -f ~/k8s-frontend/
```

### Si k3s no responde después de reiniciar la EC2

```bash
sudo systemctl status k3s
sudo systemctl start k3s
sudo kubectl get nodes
```
