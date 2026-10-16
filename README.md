# Práctica UD01.1: Despliegue de Servidores Web con Docker (Nginx & Apache)

**Módulo:** Despliegue de Aplicaciones Web (DAW)  
**Alumno:** [Tu Nombre]  
**Fecha:** 16/10/2026  

---

## 1. Arquitectura

Despliegue de dos servicios web en el puerto 80 público utilizando Docker Compose y un proxy inverso:
* **Nginx (`srv-nginx-proxy`)**: Escucha en el puerto público `80:80`. Sirve directamente el portal estático (`portal.empresa.local`) y redirige las peticiones de gestión a Apache.
* **Apache (`srv-apache-app`)**: Aloja la app de gestión (`app-gestion.empresa.local`) de forma aislada dentro de la red privada `red-corp`.

---

## 2. Estructura del Proyecto

```text
despliegue-docker/
├── docker-compose.yml
├── nginx/
│   ├── conf.d/default.conf
│   ├── html/index.html
│   └── logs/
└── apache/
    ├── htdocs/index.html
    └── logs/
```
## 3. Configuraciones

### `docker-compose.yml`
```yaml
version: '3.8'

services:
  nginx-proxy:
    image: nginx:1.25-alpine
    container_name: srv-nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/html:/usr/share/nginx/html:ro
      - ./nginx/logs:/var/log/nginx
    networks:
      - red-corp
    depends_on:
      - apache-app

  apache-app:
    image: httpd:2.4-alpine
    container_name: srv-apache-app
    restart: unless-stopped
    volumes:
      - ./apache/htdocs:/usr/local/apache2/htdocs:ro
      - ./apache/logs:/usr/local/apache2/logs
    networks:
      - red-corp

networks:
  red-corp:
    driver: bridge
```

### nginx/conf.d/default.conf
```
server {
    listen 80;
    server_name portal.empresa.local;

    server_tokens off;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

server {
    listen 80;
    server_name app-gestion.empresa.local;

    server_tokens off;

    location / {
        proxy_pass http://srv-apache-app:80;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
## 4. Despliegue y Verificación

1. **Resolución DNS local** (`/etc/hosts` o `hosts` de Windows):
   `127.0.0.1 portal.empresa.local app-gestion.empresa.local`

2. **Arranque de contenedores**:
   ```bash
   docker compose up -d

# prueba estado docker compose
docker-compose ps

![prueba Compose](nginx\html\img\prueba3.png)

# Prueba portal Nginx
curl -I -H "Host: portal.empresa.local" http://localhost

![prueba Nginx](nginx\html\img\prueba1.png)

# Prueba app Apache vía Proxy
curl -I -H "Host: app-gestion.empresa.local" http://localhost

![prueba Apache](nginx\html\img\prueba2.png)