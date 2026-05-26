# Turismo Cañete — Presentación del Despliegue con Kubernetes

**Proyecto:** AS242S4_PII_T06  
**Fecha:** Mayo 2026  
**Integrantes:** Fabrizio Negrón y equipo

---

## 1. ¿Qué construimos?

Una plataforma web de turismo para la provincia de Cañete (Perú) llamada **Golden Valley**, compuesta por:

| Componente | Tecnología | Instancia EC2 |
|------------|-----------|---------------|
| Base de datos | MongoDB 7.0 | EC2 #1 — `100.55.177.41` |
| Backend API | Python Flask | EC2 #2 — `3.223.121.15` |
| Frontend | React + Vite + Nginx | EC2 #3 — `32.192.155.89` |

Cada componente corre en su **propia instancia EC2 de AWS**, orquestado con **Kubernetes (k3s)**.

---

## 2. ¿Por qué Kubernetes?

### Problema sin Kubernetes
Si desplegamos la app directamente en una EC2 con Docker o solo con Python:
- Si el proceso cae, **no se reinicia solo**
- No hay forma de actualizar sin tiempo de inactividad
- Escalar requiere intervención manual
- No hay separación clara entre servicios

### Solución con Kubernetes
| Beneficio | Cómo lo resuelve K8s |
|-----------|---------------------|
| **Alta disponibilidad** | Si el pod cae, el Deployment lo reinicia automáticamente |
| **Actualizaciones sin downtime** | RollingUpdate reemplaza pods gradualmente |
| **Separación de servicios** | Cada componente en su propio namespace y EC2 |
| **Configuración segura** | Secrets para variables sensibles como MONGO_URI |
| **Health checks** | Liveness y Readiness probes detectan fallos automáticamente |

---

## 3. ¿Por qué k3s y no k8s completo?

**k3s** es una distribución ligera de Kubernetes creada por Rancher, ideal para:
- Instancias EC2 pequeñas (t2.micro / t2.small)
- Un solo nodo por servicio
- Instalación en un solo comando
- Consume ~512MB de RAM vs ~2GB de Kubernetes completo

---

## 4. Arquitectura del sistema

```
┌─────────────────────────────────────────────────────────┐
│                     INTERNET                            │
└──────────────┬──────────────────────┬───────────────────┘
               │                      │
               ▼                      ▼
   ┌───────────────────┐   ┌───────────────────────┐
   │  EC2 #3 FRONTEND  │   │   EC2 #2 BACKEND      │
   │  32.192.155.89    │   │   3.223.121.15        │
   │  Puerto: 30080    │──▶│   Puerto: 30050       │
   │                   │   │                       │
   │  k3s cluster      │   │  k3s cluster          │
   │  Namespace:       │   │  Namespace:           │
   │  turismo-frontend │   │  turismo-backend      │
   │                   │   │                       │
   │  Pod: Nginx       │   │  Pod: Flask API       │
   │  (React build)    │   │  (Python 3.11)        │
   └───────────────────┘   └──────────┬────────────┘
                                      │
                                      ▼
                          ┌───────────────────────┐
                          │  EC2 #1 BASE DE DATOS │
                          │  100.55.177.41        │
                          │  Puerto: 30017        │
                          │                       │
                          │  k3s cluster          │
                          │  Namespace:           │
                          │  turismo-db           │
                          │                       │
                          │  Pod: MongoDB 7.0     │
                          │  PV: 5Gi en /data/    │
                          └───────────────────────┘
```

---

## 5. ¿Por qué 3 instancias separadas?

### Separación de responsabilidades
Cada capa tiene su propio ciclo de vida:
- Puedes actualizar el **frontend** sin tocar el backend ni la BD
- Puedes escalar el **backend** independientemente si hay más tráfico
- La **base de datos** tiene su propio almacenamiento persistente aislado

### Seguridad
- La BD solo acepta conexiones desde el backend (puerto 30017)
- El frontend solo habla con el backend (puerto 30050)
- Cada instancia tiene su propio Security Group en AWS

### Resiliencia
- Si el frontend cae, el backend y la BD siguen funcionando
- Si el backend se reinicia, la BD no se ve afectada

---

## 6. Componentes de Kubernetes usados

### Namespace
Agrupa todos los recursos de un servicio. Evita conflictos de nombres entre componentes.
```
turismo-db        → MongoDB
turismo-backend   → Flask API
turismo-frontend  → React/Nginx
```

### Deployment
Define cómo correr el contenedor: imagen, variables de entorno, recursos, réplicas.
- Garantiza que siempre haya 1 pod corriendo
- Si el pod falla, lo recrea automáticamente

### Service (NodePort)
Expone el pod al exterior del cluster usando un puerto del nodo EC2.
```
MongoDB  → NodePort 30017
Backend  → NodePort 30050
Frontend → NodePort 30080
```

### PersistentVolume + PersistentVolumeClaim
Almacenamiento persistente para MongoDB en `/data/mongodb` del nodo EC2.
- Los datos **sobreviven** reinicios del pod
- `ReclaimPolicy: Retain` → los datos no se borran si se elimina el PVC

### ConfigMap
Almacena el script de inicialización de MongoDB (colecciones + datos semilla).
Se ejecuta automáticamente la primera vez que el pod arranca.

### Secret
Almacena la `MONGO_URI` de forma segura.
El pod la recibe como variable de entorno sin exponerla en el código.

### Health Checks (Probes)
- **Liveness:** si falla, Kubernetes reinicia el pod
- **Readiness:** si falla, Kubernetes deja de enviarle tráfico hasta que se recupere

---

## 7. ¿Por qué Docker Hub?

La imagen del backend y frontend se construye localmente y se sube a Docker Hub (`fabrizionegron/turismo-backend:latest` y `fabrizionegron/turismo-frontend:latest`).

Cuando Kubernetes despliega el pod en la EC2, descarga la imagen directamente desde Docker Hub. Esto permite:
- Desplegar en cualquier servidor sin copiar código
- Versionar las imágenes con tags
- Actualizar con un simple `kubectl rollout restart`

---

## 8. Demo en vivo — URLs del sistema

| Servicio | URL |
|---------|-----|
| **Frontend** | http://32.192.155.89:30080 |
| **Backend Swagger** | http://3.223.121.15:30050/apidocs/ |
| **Health check API** | http://3.223.121.15:30050/health |

---

## 9. Cómo volver a poner el sistema en uso

Si las instancias EC2 fueron apagadas o los pods cayeron:

### Verificar estado de los pods

```bash
# Base de datos
ssh -i "bdk8s.pem" ubuntu@100.55.177.41
sudo kubectl get pods -n turismo-db

# Backend
ssh -i "backK8s.pem" ubuntu@3.223.121.15
sudo kubectl get pods -n turismo-backend

# Frontend
ssh -i "frontk8s.pem" ubuntu@32.192.155.89
sudo kubectl get pods -n turismo-frontend
```

### Si los pods están en estado distinto a Running

```bash
# Reiniciar deployment (en cada EC2 correspondiente)
sudo kubectl rollout restart deployment/mongodb -n turismo-db
sudo kubectl rollout restart deployment/turismo-backend -n turismo-backend
sudo kubectl rollout restart deployment/turismo-frontend -n turismo-frontend
```

### Si k3s no está corriendo (EC2 fue reiniciada)

```bash
# k3s arranca automáticamente con el sistema, pero si no:
sudo systemctl start k3s
sudo kubectl get nodes
```

### Si hay que redesplegar todo desde cero

```bash
# En la EC2 correspondiente, aplicar los manifiestos de nuevo
sudo kubectl apply -f ~/k8s-mongodb/     # EC2 #1
sudo kubectl apply -f ~/k8s-backend/     # EC2 #2
sudo kubectl apply -f ~/k8s-frontend/    # EC2 #3
```

---

## 10. Actualizar el sistema (nueva versión)

Cuando se hace un cambio en el código:

### Backend
```bash
# En tu PC
docker build -t fabrizionegron/turismo-backend:latest .
docker push fabrizionegron/turismo-backend:latest

# En la EC2 del backend
sudo kubectl rollout restart deployment/turismo-backend -n turismo-backend
```

### Frontend
```bash
# En tu PC
docker build --build-arg VITE_API_URL=http://3.223.121.15:30050 -t fabrizionegron/turismo-frontend:latest .
docker push fabrizionegron/turismo-frontend:latest

# En la EC2 del frontend
sudo kubectl rollout restart deployment/turismo-frontend -n turismo-frontend
```

---

## 11. Comandos de diagnóstico rápido

```bash
# Ver todos los recursos de un namespace
sudo kubectl get all -n <namespace>

# Ver logs de un pod
sudo kubectl logs -n <namespace> deployment/<nombre>

# Describir un pod (ver errores detallados)
sudo kubectl describe pod -n <namespace> <nombre-del-pod>

# Ver eventos recientes
sudo kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```
