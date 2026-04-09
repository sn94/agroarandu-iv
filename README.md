# 🌐 Despliegue de Sitios Web con Traefik + Nginx (Docker Compose)

Este documento describe paso a paso cómo levantar en un servidor Ubuntu un **proxy inverso Traefik** y hospedar múltiples sitios web seguros (HTTPS) con **certificados automáticos de Let’s Encrypt**.  
Se ejemplifica con el despliegue del sitio **agroaranduiv.bareiroing.com**, alojado mediante un contenedor **Nginx**.

---

## 🧱 Estructura del proyecto

El servidor contiene dos stacks principales:

```

/home/<USER>/
├── traefik/              → Proxy inverso global (80/443)
│   ├── docker-compose.yml
│   └── letsencrypt/
│       └── acme.json     -> Donde se guardan los certificacdos ssl
└── agroarandu-iv/        → Sitio web Nginx con Traefik labels
├── docker-compose.yml
├── nginx/
│   └── nginx.conf
└── www/
├── index.html
└── 404.html

````

---

## ⚙️ 1. Despliegue del proxy Traefik

Traefik actúa como proxy inverso y gestor automático de certificados SSL.

### 📄 `traefik/docker-compose.yml`
```yaml
networks:
  web:
    external: true

services:
  traefik:
    image: traefik:latest
    container_name: traefik
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.le.acme.httpchallenge=true
      - --certificatesresolvers.le.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.le.acme.email=postmaster@aksolutionspy.com
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --log.level=DEBUG
      - --accesslog=true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    networks:
      - web
    dns:
      - 1.1.1.1
      - 8.8.8.8
    restart: unless-stopped
````

### 🧩 Crear la red compartida

```bash
docker network create web
```

### 🔐 Permisos del archivo ACME

```bash
mkdir -p traefik/letsencrypt
touch traefik/letsencrypt/acme.json
chmod 600 traefik/letsencrypt/acme.json
```

### 🚀 Levantar Traefik

```bash
cd /home/<USER>/traefik
docker compose up -d
```

Traefik escuchará en los puertos **80 y 443**, y todos los sitios se publicarán a través de esta red `web`.

---

## 🌿 2. Despliegue del sitio agroaranduiv.bareiroing.com

El sitio usa **Nginx** como servidor estático. Traefik lo detecta automáticamente mediante labels.

### 📄 `agroarandu-iv/docker-compose.yml`

```yaml
networks:
  web:
    external: true

services:
  agroaranduiv:
    image: nginx:1.27-alpine
    container_name: agroaranduiv
    restart: unless-stopped
    networks:
      - web
    volumes:
      - ./www:/usr/share/nginx/html:ro
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    labels:
      - traefik.enable=true
      - traefik.docker.network=web
      - traefik.http.routers.agroaranduiv.rule=Host(`agroaranduiv.bareiroing.com`)
      - traefik.http.routers.agroaranduiv.entrypoints=websecure
      - traefik.http.routers.agroaranduiv.tls.certresolver=le
      - traefik.http.routers.agroaranduiv-http.rule=Host(`agroaranduiv.bareiroing.com`)
      - traefik.http.routers.agroaranduiv-http.entrypoints=web
      - traefik.http.routers.agroaranduiv-http.middlewares=agroaranduiv-https
      - traefik.http.middlewares.agroaranduiv-https.redirectscheme.scheme=https
      - traefik.http.middlewares.agroaranduiv-https.redirectscheme.permanent=true
      - traefik.http.services.agroaranduiv.loadbalancer.server.port=80
    healthcheck:
      test: ["CMD-SHELL", "wget -q --spider http://127.0.0.1/ || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
```

---

## 🧩 3. Configuración de Nginx

Archivo: `agroarandu-iv/nginx/nginx.conf`

```nginx
server {
  listen 80;
  listen [::]:80;
  server_name _;

  root /usr/share/nginx/html;
  index index.html;

  client_max_body_size 20m;
  sendfile on;

  gzip on;
  gzip_types text/plain text/css application/javascript application/json image/svg+xml;

  location ~* \.(?:ico|css|js|gif|jpe?g|png|svg|woff2?)$ {
    expires 30d;
    access_log off;
    try_files $uri =404;
  }

  location / {
    try_files $uri $uri/ /index.html;
  }

  error_page 404 /404.html;
  location = /404.html {
    internal;
  }
}
```

---

## 🧾 4. Archivos web

Carpeta `agroarandu-iv/www`:

**index.html**

```html
<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Agroarandú UI</title>
  <style>
    body { font-family: system-ui, sans-serif; display: grid; place-items: center; min-height: 100vh; background:#f7f7f7; margin:0;}
    .card { background:#fff; padding:2rem; border-radius:16px; box-shadow:0 8px 30px rgba(0,0,0,.08); text-align:center; }
    h1 { color:#228b22; margin-bottom:.5rem; }
    code { background:#f0f0f0; padding:.2rem .4rem; border-radius:6px; }
  </style>
</head>
<body>
  <div class="card">
    <h1>Agroarandú UI</h1>
    <p>Despliegue correcto en <code>agroaranduiv.bareiroing.com</code></p>
  </div>
</body>
</html>
```

**404.html**

```html
<h1>Página no encontrada</h1>
<p>El recurso solicitado no existe.</p>
```

---

## 🌎 5. DNS y dominio

Configurar en tu proveedor (GoDaddy, Cloudflare, etc.):

| Tipo | Nombre / Subdominio | Valor (IP pública del servidor) | TTL  | Proxy                                                  |
| ---- | ------------------- | ------------------------------- | ---- | ------------------------------------------------------ |
| A    | agroaranduiv        | TU IP PUBLCA                  | Auto | **DNS Only (gris)** durante la emisión del certificado |

> Luego de emitido el certificado SSL, si usas Cloudflare, podés volver a “Proxy (naranja)” si querés usar su CDN.

---

## 🚀 6. Despliegue del sitio

```bash
cd /home/<USER>/agroarandu-iv
docker compose up -d
```

En ~30 segundos, visitá:
👉 **[https://agroaranduiv.bareiroing.com](https://agroaranduiv.bareiroing.com)**

Traefik obtendrá automáticamente el certificado SSL válido de Let’s Encrypt.

---

## 🧩 7. Publicar actualizaciones

Cada vez que generes tu build de React/Vite/Next:

```bash
npm run build
rsync -av --delete dist/ /home/<USER>/agroarandu-iv/www/
```

Los archivos se sirven **automáticamente** sin reiniciar el contenedor.

---

## 🧰 8. Comandos útiles

| Acción           | Comando                                       |
| ---------------- | --------------------------------------------- |
| Ver estado       | `docker ps`                                   |
| Logs Traefik     | `docker logs -f traefik`                      |
| Logs sitio       | `docker logs -f agroaranduiv`                 |
| Reiniciar sitio  | `docker compose restart agroaranduiv`         |
| Actualizar Nginx | `docker compose pull && docker compose up -d` |

---

## 🧠 9. Solución de problemas

| Síntoma              | Causa                               | Solución                                       |
| -------------------- | ----------------------------------- | ---------------------------------------------- |
| 404 desde Traefik    | Contenedor unhealthy o sin label    | Revisar healthcheck y `traefik.docker.network` |
| Error de certificado | DNS en proxy (Cloudflare naranja)   | Poner “DNS Only” y reemitir                    |
| Página vacía         | Archivos no copiados a `www/`       | Verificar contenido del directorio             |
| Traefik sin routers  | Falta `traefik.enable=true` o `web` | Revisar labels y red externa                   |

---

---

✍️ **Autores:**
Jorge Bareiro y Karen Vázquez — *AK Solutions Paraguay*
📅 Última actualización: Octubre 2025

```


