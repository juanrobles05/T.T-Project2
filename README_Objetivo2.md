# README - OBJETIVO 2: AUTOESCALAMIENTO Y LOAD BALANCER.

**Estudiante:** Santiago Betancur  
**Colaboradores:** Sara Pineda, Santiago Betancur  
**Profesor:** Álvaro Enrique Ospina Sanjuán  
**Curso:** ST0263 - Tópicos Especiales en Telemática  
**Período:** 2025-2  

1.1. Aspectos cumplidos
Requerimientos Funcionales:

Despliegue de aplicación monolítica escalable en AWS:

Instancia EC2 template (bookstore-template) configurada con la aplicación BookStore
Base de datos MySQL externa con replicación master-slave en EC2 independiente
Sistema de archivos compartido mediante EFS para uploads
Proxy inverso NGINX configurado


Patrón de escalamiento implementado:

Application Load Balancer (ALB) con listeners HTTP 
Target Group con health checks en /health
Launch Template basado en AMI de bookstore-template
Auto Scaling Group: mínimo 2 instancias, máximo 4
Política de escalamiento por CPU (70% threshold)
Certificado SSL configurado con AWS Certificate Manager


Alta disponibilidad:

Base de datos MySQL con replicación master-slave
Múltiples Availability Zones (us-east-1a, us-east-1b)
Health checks automáticos en Target Group
Reemplazo automático de instancias fallidas


D




1.2. Aspectos NO cumplidos

RDS no pudo utilizarse debido a restricciones de AWS Academy (permisos insuficientes). Se implementó alternativa con MySQL en EC2 con replicación manual.


2. Arquitectura de Alto Nivel
Componentes principales:
Capa de balanceo de carga:

Application Load Balancer en múltiples AZs
Listeners HTTP:80 (redirige a HTTPS) y HTTPS:443
Target Group con health checks cada 30s

Capa de aplicación:

Auto Scaling Group con 2-4 instancias EC2 t3.small
Gunicorn como servidor WSGI (2 workers, 2 threads)
NGINX como proxy inverso
Docker Compose para gestión de contenedores

Capa de datos:

MySQL 8.0 en EC2 (master en 172.31.16.23)
Réplica MySQL en EC2 secundaria
EFS para archivos compartidos (uploads)

Seguridad:

Security Groups segregados (ALB, App, DB, EFS)
Certificado SSL/TLS via ACM
Acceso a base de datos solo desde security group de aplicación

Patrón de arquitectura:
Escalamiento horizontal con balanceador de carga, siguiendo el patrón:
Internet → ALB (HTTPS) → Target Group → ASG (2-4 EC2) → MySQL + EFS
Mejores prácticas aplicadas:

Separación de capas (presentación, aplicación, datos)
Infraestructura como código (Launch Template, User Data scripts)
Health checks multinivel (Docker, ALB, ASG)
Auto-healing mediante ASG
Configuración mediante variables de entorno
Logs centralizados en /var/log/bookstore-startup.log


3. Ambiente de Desarrollo
Tecnologías utilizadas:
Backend:

Python 3.9
Flask 2.3.0
Flask-SQLAlchemy 3.0.5
Gunicorn 21.2.0 (servidor WSGI de producción)
PyMySQL 1.1.0
python-dotenv 1.0.0

Base de datos:

MySQL 8.0
Replicación master-slave

Infraestructura:

Docker 24.x
Docker Compose 3.8
NGINX 1.24.0
Ubuntu 24.04 LTS

AWS Services:

EC2 (t3.small instances)
EFS (Elastic File System)
ALB (Application Load Balancer)
Auto Scaling Groups
CloudWatch

Configuración del proyecto:
Ruta raíz del proyecto:
/opt/bookstore/proyecto2/BookStore-monolith/

Variables de entorno (.env):
FLASK_ENV=production
DB_HOST=172.31.16.23
DB_PORT=3306
DB_USER=bookstore_user
DB_PASSWORD=bookstore_pass
DB_NAME=bookstore
SECRET_KEY=<clave-secreta>
UPLOAD_FOLDER=/mnt/efs/bookstore/uploads
docker-compose.yml:
yamlversion: '3.8'

services:
  flaskapp:
    build: .
    restart: always
    env_file:
      - .env
    ports:
      - "5000:5000"
    volumes:
      - /mnt/efs/bookstore/uploads:/app/uploads
    networks:
      - bookstore_net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  bookstore_net:
    driver: bridge
Dockerfile:
dockerfileFROM python:3.9-slim

WORKDIR /app

RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN mkdir -p /app/uploads

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--threads", "2", "--timeout", "60", "--access-logfile", "-", "--error-logfile", "-", "app:app"]
Configuración NGINX (/etc/nginx/sites-available/bookstore):
nginxserver {
    listen 80 default_server;
    server_name _;

    location /health {
        proxy_pass http://127.0.0.1:5000/health;
        proxy_set_header Host $host;
        access_log off;
    }

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
___________________________________________________________________
Script de inicio automático (/usr/local/bin/bookstore-startup.sh):
bash#!/bin/bash
exec > >(tee /var/log/bookstore-startup.log)
exec 2>&1

echo "$(date): Iniciando BookStore..."

APP_DIR="/opt/bookstore/proyecto2/BookStore-monolith"

# Crear directorio uploads
mkdir -p /mnt/efs/bookstore/uploads
chmod -R 777 /mnt/efs/bookstore/uploads

# Esperar Docker
for i in {1..30}; do
    if docker info > /dev/null 2>&1; then
        echo "$(date): Docker listo"
        break
    fi
    sleep 2
done

# Iniciar aplicación
cd ${APP_DIR}
docker-compose up -d

# Esperar aplicación
for i in {1..60}; do
    if curl -f http://localhost:5000/health > /dev/null 2>&1; then
        echo "$(date): Aplicación lista"
        break
    fi
    sleep 2
done

# Reiniciar NGINX
systemctl restart nginx

echo "$(date): Startup completado"
Compilación y ejecución:
bash# Build de la imagen Docker
docker-compose build --no-cache

# Iniciar servicios
docker-compose up -d

# Verificar logs
docker-compose logs -f

# Verificar health
curl http://localhost/health
___________________________________________________________________

4. Ambiente de Producción
Configuración en AWS:
Dominio (http): http://bookstore-load-balancer-944350086.us-east-1.elb.amazonaws.com 
Infraestructura:

Application Load Balancer:

Scheme: Internet-facing
Security Group: bookstore-alb-sg
Listeners: HTTP:80 (redirect), HTTPS:443
Certificado SSL: Sin certificado para optimizar tiempo en configuración

Target Group:

Nombre: bookstore-tg
Protocol: HTTP, Port: 80
Health check: /health (interval 30s, timeout 5s)
Healthy threshold: 2, Unhealthy threshold: 3


Auto Scaling Group:

Nombre: bookstore-asg
Launch Template: bookstore-launch-template
Min: 2, Desired: 2, Max: 4 instancias
Instance type: t3.small
AMI: bookstore-autoscaling-v1
Availability Zones: us-east-1a, us-east-1b
Scaling policy: Target tracking CPU 70%
Health check grace period: 300s


Security Groups:

bookstore-alb-sg:

Inbound: HTTP (80) y HTTPS (443) desde 0.0.0.0/0
Outbound: All traffic

bookstore-app-sg:

Inbound: HTTP (80) desde bookstore-alb-sg


Base de datos:

MySQL Master: 172.31.16.23:3306
MySQL Replica: EC2 secundaria
Usuario: bookstore_user
Base de datos: bookstore

EFS:

Mount point: /mnt/efs/bookstore/uploads
Security Group: Permite NFS (2049) desde bookstore-app-sg

Parámetros de configuración:
Configurados mediante variables de entorno en Launch Template User Data y archivo .env.
Lanzamiento del servidor:
El servidor se inicia automáticamente mediante:

Systemd service: bookstore.service (habilitado en boot)
User Data script en Launch Template
Auto Scaling Group lanza instancias según demanda

Guía de uso para usuarios:

Acceder a http://bookstore-load-balancer-944350086.us-east-1.elb.amazonaws.com
Registrarse como nuevo usuario
Iniciar sesión con credenciales
Explorar catálogo de libros
Publicar libros propios para venta
Realizar compras (simuladas)

Monitoreo:

CloudWatch Metrics: CPU, Network, HealthyHostCount
Logs: /var/log/bookstore-startup.log en cada instancia
ALB Access Logs (opcional)
Target Health en Target Group


5. Información Relevante
Decisiones técnicas importantes:

Gunicorn vs Flask dev server: Se utilizó Gunicorn por ser production-ready, soportar múltiples workers y mejor rendimiento bajo carga.
EFS vs EBS: EFS permite compartir archivos entre múltiples instancias del ASG, requisito para archivos de uploads.
MySQL en EC2 vs RDS: Por limitaciones de AWS Academy, se implementó MySQL en EC2 con replicación manual. RDS hubiera sido preferible por gestión automatizada.
Health checks multinivel: Docker healthcheck, NGINX endpoint y ALB target health para máxima confiabilidad.

Limitaciones conocidas:

EFS mount puede fallar si Security Groups no están correctamente configurados
Credenciales en AWS Academy expiran después de 4 horas de sesión
Replicación de MySQL es manual, no automática como en RDS

Referencias:

Documentación oficial de Flask: https://flask.palletsprojects.com/
AWS Auto Scaling documentation: https://docs.aws.amazon.com/autoscaling/
Gunicorn deployment: https://docs.gunicorn.org/
Docker Compose reference: https://docs.docker.com/compose/
NGINX reverse proxy: https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/
Repositorio base del proyecto: https://github.com/st0263eafit/st0263-252/tree/main/proyecto2
