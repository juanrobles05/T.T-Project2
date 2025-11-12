# README - OBJETIVO 1: Aplicación Monolítica con Dominio y SSL

**Estudiante:** Juan Diego Robles de la Ossa  
**Colaboradores:** Sara Pineda, Santiago Betancur  
**Profesor:** Álvaro Enrique Ospina Sanjuán  
**Curso:** ST0263 - Tópicos Especiales en Telemática  
**Período:** 2025-2  

---

## 1. Descripción de la actividad

Esta actividad consiste en desplegar la aplicación **BookStore** (sistema de ecommerce de venta de libros de segunda mano) de forma monolítica en **dos máquinas virtuales** separadas en AWS, configurando un **dominio propio** con **certificado SSL** y un **proxy inverso con NGINX**.

El objetivo principal fue demostrar el conocimiento en la separación de componentes (aplicación y base de datos), configuración de servidores web, manejo de dominios DNS y certificados SSL para asegurar las comunicaciones.

---

## 1.1. Aspectos desarrollados de la actividad

- Creación y configuración de dos máquinas virtuales EC2 en AWS.
- Despliegue de MySQL en contenedor Docker en VM2 (Base de Datos).
- Despliegue de aplicación Flask (BookStore) en contenedor Docker en VM1.
- Configuración de NGINX como proxy inverso en VM1.
- Configuración de dominio propio (`proyecto2.tudominio.com`).
- Obtención e instalación de certificado SSL con Let's Encrypt (Certbot).
- Redirección automática de HTTP a HTTPS.
- Configuración de Security Groups para comunicación segura entre VMs.
- Validación del correcto funcionamiento de la aplicación con HTTPS.

---

## 1.2. Aspectos no desarrollados

---

## 2. Información general de diseño y arquitectura

### Arquitectura implementada
```
                    Internet
                       ↓
        [DNS] proyecto2.tudominio.com
                       ↓
         [Let's Encrypt - Certificado SSL]
                       ↓
            [VM1: NGINX + Flask App]
              (Proxy Inverso + SSL)
                       ↓
              [VM2: MySQL Database]
            (Contenedor Docker)
```

### Componentes principales

**VM1 - Servidor de Aplicación:**
- Sistema Operativo: Ubuntu 22.04 LTS
- NGINX: Proxy inverso y terminación SSL
- Flask App: Aplicación BookStore en contenedor Docker (puerto 5000)
- Certbot: Gestión automática de certificados SSL

**VM2 - Servidor de Base de Datos:**
- Sistema Operativo: Ubuntu 22.04 LTS
- MySQL 8.0: Base de datos en contenedor Docker (puerto 3306)
- Almacenamiento persistente mediante Docker volumes

### Flujo de comunicación

1. Usuario accede a `https://proyecto2.work.gd`
2. DNS resuelve a la IP pública de VM1
3. NGINX recibe la petición HTTPS (puerto 443)
4. NGINX valida certificado SSL
5. NGINX redirige la petición a Flask (localhost:5000)
6. Flask procesa la petición y consulta MySQL en VM2 (IP privada:3306)
7. MySQL retorna los datos a Flask
8. Flask genera la respuesta HTML
9. NGINX envía la respuesta cifrada al usuario

---

## 3. Descripción del ambiente de desarrollo y técnico

### Tecnologías utilizadas

- **AWS EC2:** Instancias t2.micro / t2.small
- **Docker:** Versión 24.x para contenedores
- **Docker Compose:** Versión 1.29.x para orquestación de contenedores
- **NGINX:** Versión 1.18+ como proxy inverso
- **Certbot:** Cliente ACME para Let's Encrypt
- **Python Flask:** Framework web de la aplicación
- **MySQL:** Versión 8.0 como base de datos
- **Ubuntu Server:** 22.04 LTS

### Repositorio base

- Aplicación BookStore: https://github.com/st0263eafit/st0263-252/blob/main/proyecto2/BookStore.zip

---

## 4. Descripción del ambiente de ejecución

### Configuración de VM1 (Aplicación)

**Características:**
- Tipo de instancia: t2.micro
- Sistema Operativo: Ubuntu 22.04 LTS
- Región: us-east-1
- Security Group: bookstore-app-sg

**Servicios ejecutándose:**
```bash
# NGINX
sudo systemctl status nginx

# Docker (contenedor Flask)
docker ps
```

**Security Group (Inbound Rules):**

| Tipo | Puerto | Origen | Descripción |
|------|--------|---------|-------------|
| SSH | 22 | Mi IP | Acceso SSH |
| HTTP | 80 | 0.0.0.0/0 | HTTP (redirige a HTTPS) |
| HTTPS | 443 | 0.0.0.0/0 | HTTPS con SSL |

---

### Configuración de VM2 (Base de Datos)

**Características:**
- Tipo de instancia: t2.small
- Sistema Operativo: Ubuntu 22.04 LTS
- Región: us-east-1
- Security Group: bookstore-db-sg

**Servicios ejecutándose:**
```bash
# Docker (contenedor MySQL)
docker ps
```

**Security Group (Inbound Rules):**

| Tipo | Puerto | Origen | Descripción |
|------|--------|---------|-------------|
| SSH | 22 | Mi IP | Acceso SSH |
| MySQL | 3306 | SG de VM1 | Acceso desde aplicación |

---

## 5. Guía de instalación y configuración

### 5.1. Configuración de VM2 (Base de Datos)
```bash
# 1. Conectarse a VM2
ssh -i tu-llave.pem ubuntu@<IP-VM2>

# 2. Instalar Docker y Docker Compose
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu

# 3. Reiniciar sesión
exit
ssh -i tu-llave.pem ubuntu@<IP-VM2>

# 4. Crear directorio y docker-compose.yml
mkdir ~/database
cd ~/database
nano docker-compose.yml
```

**Contenido de `docker-compose.yml`:**
```yaml
version: '3.8'

services:
  db:
    image: mysql:8
    restart: always
    environment:
      MYSQL_DATABASE: bookstore
      MYSQL_USER: bookstore_user
      MYSQL_PASSWORD: bookstore_pass
      MYSQL_ROOT_PASSWORD: root_pass
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```
```bash
# 5. Iniciar MySQL
docker-compose up -d

# 6. Verificar
docker ps
docker logs database_db_1
```

---

### 5.2. Configuración de VM1 (Aplicación)
```bash
# 1. Conectarse a VM1
ssh -i tu-llave.pem ubuntu@<IP-VM1>

# 2. Instalar dependencias
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose nginx certbot python3-certbot-nginx unzip wget
sudo systemctl enable docker nginx
sudo systemctl start docker nginx
sudo usermod -aG docker ubuntu

# 3. Reiniciar sesión
exit
ssh -i tu-llave.pem ubuntu@<IP-VM1>

# 4. Descargar BookStore
cd ~
wget https://github.com/st0263eafit/st0263-252/raw/main/proyecto2/BookStore.zip
unzip BookStore.zip
cd BookStore

# 5. Modificar config.py para conectar a VM2
nano config.py
# Cambiar: @db por @<IP-PRIVADA-VM2>
# Ejemplo: mysql://bookstore_user:bookstore_pass@172.31.x.x:3306/bookstore

# 6. Modificar docker-compose.yml (eliminar servicio db)
nano docker-compose.yml
```

**Contenido de `docker-compose.yml`:**
```yaml
version: '3.8'

services:
  flaskapp:
    build: .
    restart: always
    environment:
      - FLASK_ENV=development
    ports:
      - "5000:5000"
```
```bash
# 7. Construir y ejecutar
docker-compose build
docker-compose up -d

# 8. Verificar
docker ps
curl http://localhost:5000
```

---

### 5.3. Configuración de NGINX
```bash
# 1. Crear configuración de NGINX
sudo nano /etc/nginx/sites-available/bookstore
```

**Contenido:**
```nginx
server {
    listen 80;
    server_name proyecto2.work.gd;

    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
```bash
# 2. Activar configuración
sudo ln -s /etc/nginx/sites-available/bookstore /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

### 5.4. Configuración de Dominio

En el proveedor de dominio, crear registro tipo A:
```
Tipo: A
Nombre: proyecto2
Valor: <34.192.165.180>
TTL: 300
```

Verificar propagación:
```bash
nslookup proyecto2.work.gd
```

---

### 5.5. Configuración de Certificado SSL
```bash
# Obtener certificado con Certbot
sudo certbot --nginx -d proyecto2.work.gd

# Seguir las instrucciones:
# 1. Ingresar email
# 2. Aceptar términos (A)
# 3. No compartir email (N)
# 4. Seleccionar opción 2 (Redirect HTTP to HTTPS)

# Verificar renovación automática
sudo certbot renew --dry-run
```

---

## 6. Información de acceso

### URL de acceso público

**Aplicación:** https://proyecto2.work.gd

### Credenciales de acceso

**Base de datos MySQL (VM2):**
- Host: IP privada de VM2
- Puerto: 3306
- Database: bookstore
- Usuario: bookstore_user
- Contraseña: bookstore_pass
- Root password: root_pass

---

## 7. Guía de uso

1. Acceder al dominio: https://proyecto2.work.gd
2. La aplicación BookStore se carga con certificado SSL válido
3. Funcionalidades disponibles:
   - Registro de usuarios
   - Inicio de sesión
   - Visualización de catálogo de libros
   - Compra de libros (simulado)
   - Pago (simulado)
   - Envío (simulado)

---

## 8. Resultados y validación

### Pruebas realizadas

✅ **Conectividad de red:**
- Comunicación entre VM1 y VM2 establecida
- MySQL accesible desde VM1 en puerto 3306

✅ **Funcionamiento de la aplicación:**
- Flask corriendo correctamente en puerto 5000
- Conexión exitosa a base de datos MySQL
- Registro y login de usuarios funcional
- Visualización de catálogo de libros

✅ **Proxy inverso:**
- NGINX redirigiendo correctamente a Flask
- Headers HTTP configurados apropiadamente

✅ **SSL/TLS:**
- Certificado SSL válido emitido por Let's Encrypt
- Redirección automática HTTP → HTTPS
- Conexión segura (candado verde en navegador)
- Calificación SSL Labs: A

✅ **DNS:**
- Dominio resuelve correctamente a IP de VM1
- Propagación DNS completa

---

## 9. Problemas encontrados y soluciones

### Problema 1: Contenedor MySQL se reinicia constantemente

**Solución:** Eliminar la línea `command: --default-authentication-plugin=mysql_native_password` del docker-compose.yml ya que MySQL 8.4+ no soporta este parámetro.

### Problema 2: Flask no puede conectarse a MySQL

**Solución:** Verificar Security Groups de AWS para permitir tráfico en puerto 3306 desde VM1 hacia VM2.

### Problema 3: Certbot falla al obtener certificado

**Solución:** Verificar que el dominio apunte correctamente a la IP pública de VM1 y que el puerto 80 esté abierto en el Security Group.

---

## 10. Información adicional y conclusiones

El despliegue de la aplicación BookStore en arquitectura de dos capas permitió:

- Comprender la separación de responsabilidades entre aplicación y datos
- Implementar prácticas de seguridad mediante SSL/TLS
- Configurar proxy inverso con NGINX
- Gestionar infraestructura en AWS con Security Groups
- Automatizar la renovación de certificados SSL

**Conclusión:**

La práctica permitió demostrar conocimientos en:
- Administración de servidores Linux
- Contenedorización con Docker
- Configuración de servidores web (NGINX)
- Gestión de certificados SSL
- Configuración de DNS
- Arquitectura de aplicaciones web de dos capas
- Seguridad en comunicaciones (HTTPS)

Este objetivo sienta las bases para los siguientes objetivos donde se implementará escalamiento, alta disponibilidad y orquestación con Kubernetes.

---

## 11. Referencias

- Docker Documentation: https://docs.docker.com/
- NGINX Documentation: https://nginx.org/en/docs/
- Let's Encrypt Documentation: https://letsencrypt.org/docs/
- Certbot Documentation: https://certbot.eff.org/docs/
- AWS EC2 Documentation: https://docs.aws.amazon.com/ec2/
- Flask Documentation: https://flask.palletsprojects.com/
- MySQL Documentation: https://dev.mysql.com/doc/
- Curso ST0263 - EAFIT

---

## 12. Autor

**Juan Diego Robles de la Ossa**
Estudiante de Ingeniería de Sistemas
Universidad EAFIT
Medellín, Colombia
2025-2
