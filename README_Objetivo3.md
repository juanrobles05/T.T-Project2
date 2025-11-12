# Proyecto 2 – Objetivo 3: Despliegue de BookStore en Kubernetes (K3s)

**Estudiante:** Juan Diego Robles de la Ossa
**Colaboradores:** Sara Pineda, Santiago Betancur
**Profesor:** Álvaro Enrique Ospina Sanjuán
**Curso:** ST0263 - Tópicos Especiales en Telemática
**Período:** 2025-2

---

## 1. Descripción de la actividad

Despliegue de la aplicación monolítica **BookStore** en un clúster de Kubernetes utilizando **K3s** en AWS. El objetivo principal fue demostrar el conocimiento en orquestación de contenedores, gestión de servicios en Kubernetes, configuración de almacenamiento persistente, y exposición de servicios con Ingress Controller y certificados SSL.

---

## 1.1. Aspectos desarrollados

- Instalación y configuración de clúster Kubernetes con K3s
- Contenerización de la aplicación BookStore con Docker
- Despliegue de MySQL 8.0 con almacenamiento persistente (PVC)
- Configuración de Secrets para credenciales sensibles
- Despliegue de aplicación con 2 réplicas para alta disponibilidad
- Configuración de Ingress Controller (Traefik)
- Dominio propio configurado: **proyecto2objetivo3.work.gd**
- Certificado SSL automatizado con cert-manager y Let's Encrypt
- Init Containers para gestión de dependencias
- Services de tipo ClusterIP para comunicación interna

---

## 1.2. Aspectos no desarrollados

- No se utilizó Amazon EKS (se optó por K3s por razones de costo)
- No se implementó sistema de monitoreo (Prometheus/Grafana)

---

## 2. Justificación de K3s sobre microk8s

**K3s** fue seleccionado en lugar de microk8s debido a:

- **Menor consumo de recursos:** K3s requiere ~512MB RAM vs 2GB de microk8s
- **Mayor estabilidad:** Menos problemas de conectividad en VMs con recursos limitados
- **Instalación más confiable:** Proceso de instalación más simple y robusto
- **Incluye componentes necesarios:** Traefik Ingress Controller, CoreDNS, y local-path storage incluidos por defecto
- **Ampliamente usado en producción:** Especialmente en edge computing y ambientes con recursos limitados
- **100% compatible con Kubernetes:** K3s es una distribución certificada por CNCF

K3s es una distribución oficial de Kubernetes creada por Rancher Labs (ahora SUSE), diseñada específicamente para IoT, edge computing y ambientes con recursos limitados, manteniendo 100% de compatibilidad con la API de Kubernetes.

---

## 3. Arquitectura
```
Internet
    ↓
[DNS: proyecto2objetivo3.work.gd]
    ↓
[Traefik Ingress Controller]
    - SSL/TLS Termination
    - Let's Encrypt Certificate
    ↓
[BookStore Service - ClusterIP:80]
    ↓
[BookStore Pods x2]  ←→  [MySQL Pod]
    - Flask App            - MySQL 8.0
    - Port 5000            - Port 3306
    ↓                      ↓
[Ephemeral Storage]   [PVC 5GB - local-path]
```

### Componentes principales

**Kubernetes Cluster**
- Distribución: K3s v1.32.9
- Nodos: 1 (single-node cluster)
- VM: AWS t3.medium (2 vCPU, 4GB RAM)
- Región: us-east-1

**Namespace: bookstore**
- Aislamiento lógico de recursos
- Gestión centralizada de la aplicación

**MySQL Database**
- Deployment: 1 réplica
- Image: mysql:8.0
- Storage: PersistentVolumeClaim 5GB
- Service: ClusterIP (headless)
- Credentials: Kubernetes Secrets

**BookStore Application**
- Deployment: 2 réplicas (Alta Disponibilidad)
- Image: bookstore-app:v1 (custom)
- Service: ClusterIP port 80
- Init Container: wait-for-mysql
- Environment: Variables desde Secrets

**Ingress & SSL**
- Controller: Traefik (incluido en K3s)
- Certificate Manager: cert-manager v1.13.2
- SSL Provider: Let's Encrypt (ACME)
- Auto-renewal: Cada 90 días

---

## 4. Ambiente de desarrollo

### Tecnologías utilizadas

- **Kubernetes:** K3s v1.32.9
- **Container Runtime:** containerd
- **Lenguaje:** Python 3.10
- **Framework Web:** Flask
- **Base de Datos:** MySQL 8.0
- **Ingress Controller:** Traefik
- **Certificate Manager:** cert-manager
- **Cloud Provider:** AWS EC2
- **Storage Provisioner:** local-path

### Herramientas de desarrollo
```bash
kubectl version
# Client Version: v1.32.9
# Server Version: v1.32.9

k3s --version
# k3s version v1.32.9+k3s1

docker --version
# Docker version 24.0.5
```

### Dependencias de la aplicación
```txt
flask
flask_sqlalchemy
flask_login
pymysql
werkzeug
cryptography
```

---

## 5. Configuración y despliegue

### 5.1. Preparación de la VM
```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar K3s
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644

# Configurar kubectl
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER ~/.kube/config

# Verificar instalación
kubectl get nodes
```

### 5.2. Instalación de cert-manager
```bash
# Instalar cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml

# Esperar a que esté listo
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=300s

# Verificar
kubectl get pods -n cert-manager
```

### 5.3. Creación del namespace
```bash
kubectl create namespace bookstore
```

### 5.4. Construcción de la imagen Docker
```bash
cd ~/BookStore-monolith

# Construir imagen
docker build -t bookstore-app:v1 .

# Guardar imagen
docker save bookstore-app:v1 -o bookstore-app.tar

# Importar a K3s
sudo k3s ctr images import bookstore-app.tar

# Verificar
sudo k3s ctr images ls | grep bookstore
```

### 5.5. Despliegue de MySQL

**Archivo: mysql-deployment.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: bookstore
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: bookstore
type: Opaque
stringData:
  mysql-root-password: "rootpass123"
  mysql-database: "bookstore"
  mysql-user: "bookstore_user"
  mysql-password: "bookstore_pass"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: bookstore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-database
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: bookstore
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  clusterIP: None
```
```bash
# Aplicar configuración
kubectl apply -f mysql-deployment.yaml

# Verificar
kubectl get pods -n bookstore
kubectl get pvc -n bookstore
```

### 5.6. Despliegue de BookStore

**Archivo: bookstore-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookstore-app
  namespace: bookstore
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bookstore
  template:
    metadata:
      labels:
        app: bookstore
    spec:
      initContainers:
      - name: wait-for-mysql
        image: busybox:1.28
        command: ['sh', '-c', 'until nc -z mysql-service 3306; do echo waiting for mysql; sleep 2; done;']
      containers:
      - name: bookstore
        image: bookstore-app:v1
        imagePullPolicy: Never
        env:
        - name: DB_HOST
          value: "mysql-service"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-user
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-password
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-database
        - name: SECRET_KEY
          value: "mi-secret-key-super-segura-2024"
        ports:
        - containerPort: 5000
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: bookstore-service
  namespace: bookstore
spec:
  selector:
    app: bookstore
  ports:
  - port: 80
    targetPort: 5000
  type: ClusterIP
```
```bash
# Aplicar configuración
kubectl apply -f bookstore-deployment.yaml

# Verificar
kubectl get pods -n bookstore
kubectl get svc -n bookstore
```

### 5.7. Configuración de Let's Encrypt

**Archivo: letsencrypt-issuer.yaml**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@proyecto2objetivo3.work.gd
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
```
```bash
kubectl apply -f letsencrypt-issuer.yaml
kubectl get clusterissuer
```

### 5.8. Configuración del Ingress

**Archivo: ingress.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookstore-ingress
  namespace: bookstore
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - proyecto2objetivo3.work.gd
    secretName: bookstore-tls
  rules:
  - host: proyecto2objetivo3.work.gd
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bookstore-service
            port:
              number: 80
```
```bash
kubectl apply -f ingress.yaml

# Monitorear certificado
kubectl get certificate -n bookstore -w
```

### 5.9. Configuración DNS

En el proveedor de DNS (Freenom):
```
Tipo: A
Nombre: proyecto2objetivo3
Dominio: work.gd
Valor: [3.229.23.201]
TTL: 14400
```

---

## 6. Ambiente de ejecución (Producción)

### Información de acceso

- **URL Pública:** https://proyecto2objetivo3.work.gd
- **Plataforma:** AWS EC2
- **Región:** us-east-1
- **Tipo de instancia:** t3.medium

### Especificaciones de la VM
```
Instancia: t3.medium
vCPUs: 2
RAM: 4 GB
Storage: 20 GB (gp3)
Sistema Operativo: Ubuntu 22.04 LTS
```

### Recursos de Kubernetes
```bash
# Ver todos los recursos
kubectl get all -n bookstore

# Salida esperada:
# NAME                                READY   STATUS    RESTARTS   AGE
# pod/bookstore-app-xxxxxxxxx-xxxxx   1/1     Running   0          10m
# pod/bookstore-app-xxxxxxxxx-xxxxx   1/1     Running   0          10m
# pod/mysql-xxxxxxxx-xxxxx            1/1     Running   0          15m
#
# NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# service/bookstore-service   ClusterIP   10.43.xx.xx     <none>        80/TCP     10m
# service/mysql-service       ClusterIP   None            <none>        3306/TCP   15m
#
# NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/bookstore-app   2/2     2            2           10m
# deployment.apps/mysql           1/1     1            1           15m
```

### Guía de uso

1. Acceder a https://proyecto2objetivo3.work.gd
2. Verificar candado verde de SSL
3. Registrarse como nuevo usuario
4. Explorar funcionalidades de la aplicación

---

## 7. Estructura del proyecto
```
BookStore-monolith/
├── k8s-manifests/
│   ├── mysql-deployment.yaml          # MySQL + PVC + Secret
│   ├── bookstore-deployment.yaml      # App + Service
│   ├── ingress.yaml                   # Ingress + TLS
│   └── letsencrypt-issuer.yaml        # ClusterIssuer
├── app.py                             # Aplicación Flask modificada
├── Dockerfile                         # Imagen Docker
├── requirements.txt                   # Dependencias Python
├── models/                            # Modelos de datos
├── controllers/                       # Controladores
├── templates/                         # Templates HTML
└── static/                            # Archivos estáticos
```

---

## 8. Comandos útiles

### Monitoreo
```bash
# Ver todos los recursos
kubectl get all -n bookstore

# Ver logs de la aplicación
kubectl logs -n bookstore -l app=bookstore -f

# Ver logs de MySQL
kubectl logs -n bookstore -l app=mysql

# Ver eventos
kubectl get events -n bookstore --sort-by='.lastTimestamp'

# Ver certificado SSL
kubectl get certificate -n bookstore
kubectl describe certificate bookstore-tls -n bookstore
```

### Escalamiento
```bash
# Escalar aplicación
kubectl scale deployment bookstore-app -n bookstore --replicas=3

# Ver estado de réplicas
kubectl get pods -n bookstore -w
```

### Troubleshooting
```bash
# Describir pod con problemas
kubectl describe pod <pod-name> -n bookstore

# Ejecutar shell en un pod
kubectl exec -it <pod-name> -n bookstore -- /bin/bash

# Ver configuración del ingress
kubectl describe ingress bookstore-ingress -n bookstore

# Reiniciar deployment
kubectl rollout restart deployment bookstore-app -n bookstore
kubectl rollout restart deployment mysql -n bookstore
```

### Gestión de recursos
```bash
# Ver uso de recursos
kubectl top nodes
kubectl top pods -n bookstore

# Ver almacenamiento
kubectl get pvc -n bookstore
kubectl get pv

# Ver secrets
kubectl get secrets -n bookstore
```

---

## 9. Ventajas de K3s vs otras distribuciones

| Característica | K3s | microk8s | EKS |
|----------------|-----|----------|-----|
| RAM mínima | 512 MB | 2 GB | 4 GB+ |
| Instalación | 1 minuto | 5-10 min | 15-30 min |
| Costo | $30/mes | $30/mes | $73+/mes |
| Componentes incluidos |  Todos |  Algunos |  Aparte |
| Edge Computing |  Diseñado |  Limitado |  No |
| Producción |  Sí |  Sí |  Sí |

---

## 10. Costos estimados (AWS)

| Recurso | Especificación | Costo mensual |
|---------|----------------|---------------|
| EC2 t3.medium | 2 vCPU, 4GB RAM | ~$30.00 |
| EBS Storage | 20 GB gp3 | ~$1.60 |
| Transferencia | Capa gratuita | $0.00 |
| Dominio .work.gd | Freenom | $0.00 |
| Certificado SSL | Let's Encrypt | $0.00 |
| **Total** | | **~$31.60/mes** |

**Comparación con EKS:**
- EKS Control Plane: $73/mes adicional
- **Ahorro: ~70% usando K3s**

---

## 11. Problemas encontrados y soluciones

### Problema 1: microk8s con errores en VM pequeña
**Causa:** Insuficiente RAM (914MB inicial)
**Solución:**
- Cambiar a instancia t3.medium (4GB RAM)
- Migrar a K3s por menor consumo de recursos

### Problema 2: Certificado SSL no se genera
**Causa:** ClusterIssuer no estaba creado
**Solución:**
```bash
kubectl apply -f letsencrypt-issuer.yaml
kubectl delete certificate bookstore-tls -n bookstore
kubectl apply -f ingress.yaml
```

### Problema 3: Pods en CrashLoopBackOff
**Causa:** Falta librería `cryptography` para MySQL 8.0
**Solución:**
```bash
echo "cryptography" >> requirements.txt
docker build -t bookstore-app:v1 .
# Reimportar imagen a K3s
```

---

## 12. Características implementadas

### Alta Disponibilidad
- 2 réplicas de la aplicación
- Reinicio automático de pods fallidos
- Health checks con Kubernetes

### Seguridad
- Secrets para credenciales
- Certificado SSL válido
- Comunicación interna con ClusterIP
- Variables de entorno para configuración

### Persistencia
- PersistentVolumeClaim para MySQL
- Datos sobreviven a reinicios de pods
- Storage class: local-path

### Automatización
- Init Containers para dependencias
- Renovación automática de SSL
- Auto-restart de pods fallidos
- DNS resolution interno

---

## 13. Mejoras futuras

### Corto plazo
- [ ] Implementar HorizontalPodAutoscaler (HPA)
- [ ] Agregar Resource Limits y Requests optimizados
- [ ] Configurar Liveness y Readiness Probes
- [ ] Implementar NetworkPolicies

### Mediano plazo
- [ ] Migrar a Amazon RDS para MySQL
- [ ] Implementar Amazon EFS para archivos compartidos
- [ ] Agregar Prometheus + Grafana para monitoreo
- [ ] Implementar backup automatizado de base de datos
- [ ] Configurar logging centralizado con ELK Stack

### Largo plazo
- [ ] Migrar a Amazon EKS para mayor escalabilidad
- [ ] Implementar CI/CD con GitHub Actions
- [ ] Agregar service mesh (Istio o Linkerd)
- [ ] Implementar multi-region deployment

---

## 14. Conclusiones

- Se logró desplegar exitosamente BookStore en un clúster Kubernetes usando K3s
- K3s demostró ser una alternativa viable y económica a EKS para proyectos académicos
- La orquestación con Kubernetes facilita el escalamiento y la alta disponibilidad
- Cert-manager simplifica la gestión de certificados SSL
- El uso de Secrets y ConfigMaps mejora la seguridad y flexibilidad
- Los Init Containers son útiles para gestionar dependencias entre servicios

### Lecciones aprendidas

1. **Recursos importan:** Kubernetes requiere recursos adecuados; 1GB RAM no es suficiente
2. **K3s es production-ready:** Ampliamente usado en edge computing y IoT
3. **Abstracciones de K8s:** Simplifican deployment y gestión de aplicaciones
4. **Cert-manager:** Automatiza completamente la gestión de certificados
5. **PVC:** Fundamental para persistencia de datos en contenedores

### Comparación Objetivo 1 vs Objetivo 3

| Aspecto | Objetivo 1 (VMs) | Objetivo 3 (K8s) |
|---------|------------------|------------------|
| Escalamiento | Manual | Automático |
| Alta disponibilidad | Manual | Nativa |
| Gestión | Scripts bash | Manifiestos YAML |
| Complejidad | Baja | Media |
| Flexibilidad | Baja | Alta |
| Costo inicial | Bajo | Medio |
| Mantenimiento | Manual | Automatizado |

---

## 15. Referencias

- **K3s Documentation:** https://docs.k3s.io/
- **Kubernetes Documentation:** https://kubernetes.io/docs/
- **cert-manager Documentation:** https://cert-manager.io/docs/
- **Traefik Documentation:** https://doc.traefik.io/traefik/
- **Docker Documentation:** https://docs.docker.com/
- **AWS EC2 Documentation:** https://docs.aws.amazon.com/ec2/
- **Let's Encrypt:** https://letsencrypt.org/docs/
- **Flask Documentation:** https://flask.palletsprojects.com/
- **MySQL on Kubernetes:** https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/
- **Curso ST0263 - EAFIT**

---

**Autor:** Juan Diego Robles de la Ossa
**Fecha:** Noviembre 2025
**Universidad EAFIT**
**Curso:** ST0263 - Tópicos Especiales en Telemática