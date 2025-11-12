# Proyecto 2 - Despliegue de Aplicaciones en la Nube

**Estudiantes:** Sara Pineda, Juan Diego Robles, Santiago Betancur  
**Profesor:** Álvaro Enrique Ospina Sanjuán  
**Curso:** ST0263 - Tópicos Especiales en Telemática  
**Período:** 2025-2  
**Universidad EAFIT**  

---

## Descripción General del Proyecto

Este repositorio contiene la documentación completa del **Proyecto 2**, enfocado en el despliegue de aplicaciones web en la nube utilizando diferentes arquitecturas y tecnologías. El proyecto se divide en cuatro objetivos principales que exploran distintas estrategias de despliegue, desde aplicaciones monolíticas hasta arquitecturas de microservicios, pasando por escalamiento automático y orquestación con Kubernetes.

### Aplicación Base: BookStore

La aplicación principal utilizada es **BookStore**, un sistema de ecommerce para la venta de libros de segunda mano desarrollado con:
- **Backend:** Python Flask
- **Base de Datos:** MySQL 8.0
- **Frontend:** HTML/CSS/JavaScript con Bootstrap

### Tecnologías y Servicios Utilizados

A lo largo del proyecto se trabajó con:
- **AWS Services:** EC2, EFS, ALB, Auto Scaling Groups, EKS
- **Contenedores:** Docker, Docker Compose
- **Orquestación:** Kubernetes (K3s, Amazon EKS)
- **Proxy Inverso:** NGINX
- **Certificados SSL:** Let's Encrypt (Certbot), AWS Certificate Manager
- **Arquitecturas:** Monolítica, Microservicios
- **Patrones:** Escalamiento horizontal, Alta disponibilidad, Load Balancing

---

## Estructura del Proyecto

El proyecto está organizado en cuatro objetivos principales, cada uno con su propia documentación detallada:

### 📘 [Objetivo 1: Aplicación Monolítica con Dominio y SSL](./README_Objetivo1.md)

**Descripción:** Despliegue de la aplicación BookStore en arquitectura de dos capas (aplicación y base de datos) en máquinas virtuales separadas, con configuración de dominio propio y certificado SSL.

**Aspectos principales:**
- Separación de componentes: VM1 (aplicación) y VM2 (base de datos)
- NGINX como proxy inverso
- Configuración de DNS con dominio `.work.gd`
- Certificado SSL con Let's Encrypt
- Redirección automática HTTP → HTTPS
- Comunicación segura entre VMs mediante Security Groups

**URL de acceso:** https://proyecto2.work.gd

---

### 📗 [Objetivo 2: Escalamiento Automático con Load Balancer](./README_Objetivo2.md)

**Descripción:** Implementación de escalamiento horizontal automático utilizando AWS Auto Scaling Groups y Application Load Balancer para garantizar alta disponibilidad y capacidad de respuesta ante cargas variables.

**Aspectos principales:**
- Application Load Balancer (ALB) con listeners HTTP/HTTPS
- Auto Scaling Group (2-4 instancias)
- Launch Template con AMI personalizada
- MySQL con replicación master-slave en EC2
- EFS para almacenamiento compartido de archivos
- Health checks multinivel
- Política de escalamiento basada en CPU (70% threshold)
- Gunicorn como servidor WSGI de producción

**URL de acceso:** http://bookstore-load-balancer-944350086.us-east-1.elb.amazonaws.com

---

### 📙 [Objetivo 3: Despliegue en Kubernetes (K3s)](./README_Objetivo3.md)

**Descripción:** Despliegue de la aplicación BookStore en un clúster de Kubernetes utilizando K3s (distribución ligera de Kubernetes) con Ingress Controller, certificados SSL automatizados y almacenamiento persistente.

**Aspectos principales:**
- Clúster K3s en AWS EC2 (t3.medium)
- MySQL con PersistentVolumeClaim (5GB)
- Despliegue de aplicación con 2 réplicas
- Traefik Ingress Controller (incluido en K3s)
- Certificado SSL automatizado con cert-manager y Let's Encrypt
- Secrets para gestión de credenciales
- Init Containers para gestión de dependencias
- Namespace dedicado: `bookstore`
- Alta disponibilidad con health checks

**Ventajas de K3s:**
- Menor consumo de recursos (~512MB RAM)
- Instalación simple y rápida
- 100% compatible con Kubernetes estándar
- Componentes esenciales incluidos por defecto
- Ahorro de ~70% vs Amazon EKS

**URL de acceso:** https://proyecto2objetivo3.work.gd

---

### 📕 [Objetivo 4: Microservicios en Amazon EKS](./README_Objetivo4_Opcion2.md)

**Descripción:** Despliegue de una aplicación basada en microservicios (Sock Shop) en Amazon Elastic Kubernetes Service (EKS), demostrando arquitecturas distribuidas y orquestación a nivel empresarial.

**Aspectos principales:**
- Clúster Amazon EKS (versión 1.33)
- Aplicación Sock Shop con múltiples microservicios
- Arquitectura de microservicios completa:
  - Frontend (UI)
  - Catálogo de productos
  - Gestión de usuarios
  - Carrito de compras
  - Procesamiento de pedidos
  - Sistema de pagos
  - Gestión de envíos
- Bases de datos independientes por servicio (MongoDB, MySQL, Redis)
- RabbitMQ para mensajería
- LoadBalancer público (AWS ELB)
- Namespace dedicado: `sock-shop`
- Comunicación interna mediante servicios ClusterIP

**URL de acceso:** http://a8f516e41a68744099fd775f94b74956-261273995.us-east-1.elb.amazonaws.com

**Repositorio del desarrollo:** *https://github.com/Saraapl/microservicios*

---

## Evolución del Proyecto

El proyecto muestra una evolución natural en la complejidad y escalabilidad de las aplicaciones:

1. **Objetivo 1:** Fundamentos - Aplicación monolítica básica con SSL
2. **Objetivo 2:** Escalabilidad - Introducción de load balancing y auto scaling
3. **Objetivo 3:** Orquestación - Migración a Kubernetes con K3s
4. **Objetivo 4:** Microservicios - Arquitectura distribuida en EKS

---

## Competencias Desarrolladas

A lo largo del proyecto se desarrollaron competencias en:

### Infraestructura en la Nube
- Gestión de instancias EC2 en AWS
- Configuración de redes y Security Groups
- Uso de servicios gestionados (EFS, ALB, EKS)
- Optimización de costos en la nube

### Contenedores y Orquestación
- Creación de imágenes Docker
- Uso de Docker Compose
- Despliegue y gestión de clústeres Kubernetes
- Configuración de recursos de Kubernetes (Deployments, Services, Ingress, PVC)

### Arquitectura de Software
- Diseño de aplicaciones monolíticas
- Implementación de arquitecturas de microservicios
- Patrones de escalamiento horizontal
- Alta disponibilidad y tolerancia a fallos

### Seguridad y Redes
- Configuración de certificados SSL/TLS
- Gestión de secretos y credenciales
- Configuración de DNS
- Implementación de proxy inverso

### DevOps
- Automatización de despliegues
- Infraestructura como código
- Health checks y monitoreo básico
- Gestión de logs

---

## Comparativa de Soluciones

| Aspecto | Objetivo 1 | Objetivo 2 | Objetivo 3 | Objetivo 4 |
|---------|------------|------------|------------|------------|
| **Arquitectura** | Monolítica | Monolítica | Monolítica | Microservicios |
| **VMs/Nodos** | 2 VMs | 2-4 VMs | 1 VM K3s | EKS (múltiples) |
| **Escalabilidad** | Manual | Automática | Manual/Horizontal | Automática |
| **Load Balancer** | No | ALB | Ingress | ELB |
| **Orquestación** | Docker | Docker | Kubernetes | Kubernetes |
| **SSL** | Let's Encrypt | ACM | cert-manager | No configurado |
| **Costo mensual** | ~$20 | ~$60 | ~$32 | ~$100+ |
| **Complejidad** | Baja | Media | Media-Alta | Alta |
| **HA** | No | Sí | Limitada | Sí |

---

## Lecciones Aprendidas

### Desafíos Superados

1. **Limitaciones de AWS Academy:** Adaptación de soluciones (MySQL en EC2 vs RDS) debido a restricciones de permisos
2. **Gestión de recursos limitados:** Optimización con K3s en lugar de soluciones más pesadas
3. **Certificados SSL:** Configuración de diferentes métodos (Certbot, cert-manager, ACM)
4. **Networking en Kubernetes:** Comprensión de Services, Ingress y CNI

### Mejores Prácticas Aplicadas

- Separación de concerns (aplicación, datos, proxy)
- Uso de variables de entorno para configuración
- Implementación de health checks
- Documentación exhaustiva de cada paso
- Versionado de imágenes Docker
- Uso de Secrets para credenciales sensibles
- Configuración de almacenamiento persistente

---

## Recursos y Referencias

### Documentación Oficial
- [AWS Documentation](https://docs.aws.amazon.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Documentation](https://docs.docker.com/)
- [NGINX Documentation](https://nginx.org/en/docs/)
- [Flask Documentation](https://flask.palletsprojects.com/)

### Proyectos de Referencia
- [Sock Shop Microservices Demo](https://github.com/microservices-demo/microservices-demo)
- [K3s Documentation](https://docs.k3s.io/)
- [cert-manager Documentation](https://cert-manager.io/docs/)

### Repositorio Base
- [ST0263 - Proyecto 2](https://github.com/st0263eafit/st0263-252/tree/main/proyecto2)

---

## Autores

- **Sara Pineda** - [GitHub](https://github.com/Saraapl)
- **Juan Diego Robles de la Ossa** - [GitHub](https://github.com/juanrobles05)
- **Santiago Betancur**

**Universidad EAFIT**
**Medellín, Colombia**
**2025-2**