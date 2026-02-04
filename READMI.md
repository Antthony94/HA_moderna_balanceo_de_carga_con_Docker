# 🧠 Guía para el Examen Docker + HAProxy

> Esta guía está pensada para **reconstruir toda la práctica desde cero en un examen sin conexión a Internet**.

---

## 📌 Idea clave del examen (lo primero que entendí)

Antes de escribir una sola línea, tuve claro **qué me estaban pidiendo realmente**:

* Montar **4 contenedores Docker**:

  * **3 servidores web (Nginx)**:

    * `app1` → sano
    * `app2` → roto (responde mal)
    * `app3` → sano
  * **1 balanceador (HAProxy)**

El balanceador debe comportarse así:

* 🌐 **Tráfico normal** → balanceo **L7 (HTTP)**
  Comprueba que la app responde `200 OK` en `/health`.

* 🔌 **Tráfico por `/14/`** → balanceo **L4 (TCP)**
  Solo comprueba si el puerto abre, no el contenido.

👉 Si tienes esto claro, el resto es solo escribir archivos.

---

## 1️⃣ Preparación del entorno (orden antes que código)

Lo primero que hice fue **crear la estructura de carpetas**. Esto evita perder tiempo y errores tontos en el examen.

```powershell
mkdir examen
cd examen
mkdir haproxy
mkdir backends

New-Item docker-compose.yml
New-Item haproxy/haproxy.cfg
New-Item backends/app1.conf
New-Item backends/app2.conf
New-Item backends/app3.conf
```

🧠 *En este punto todavía no hay Docker ni HAProxy funcionando. Solo estoy preparando el terreno.*

---

## 2️⃣ Backends Nginx (patrón que memoricé)

Aquí entendí algo clave:

> **No necesito memorizar Nginx entero, solo el bloque `server`.**

### 🟢 app1 y app3 (sanos)

Archivo: `backends/app1.conf`
(app3 es igual cambiando el nombre)

```nginx
server {
    listen 80;
    location /health { return 200 "app1 OK\n"; }
    location /login  { return 200 "app1 Login\n"; }
}
```

### 🔴 app2 (la rota a propósito)

Archivo: `backends/app2.conf`

```nginx
server {
    listen 80;
    location /health { return 500 "app2 FAIL\n"; }
    location /login  { return 500 "app2 FAIL\n"; }
}
```

🧠 *Esto es intencionado: HAProxy debe detectar que app2 está mal y sacarla del balanceo L7.*

---

## 3️⃣ HAProxy (la parte más importante del examen)

Este archivo fue el más crítico. Para no liarme, lo dividí **mentalmente en 4 bloques**.

Archivo: `haproxy/haproxy.cfg`

---

### 🔹 Bloque 1: Configuración básica

```haproxy
global
    log stdout format raw local0

defaults
    mode http
    timeout connect 5s
    timeout client 10s
    timeout server 10s
```

🧠 *Si se me olvida algo aquí no suele ser mortal. Lo importante viene después.*

---

### 🔹 Bloque 2: Frontend (el portero)

```haproxy
frontend mi_frontend
    bind *:80

    acl es_tcp path_beg /14/
    use_backend be_tcp if es_tcp

    default_backend be_http
```

🧠 *Aquí decido a qué backend va cada petición.*

* Si empieza por `/14/` → backend TCP
* Si no → backend HTTP

---

### 🔹 Bloque 3: Backends (la lógica real)

#### 🧠 Backend HTTP (L7)

```haproxy
backend be_http
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200
    server app1 app1:80 check
    server app2 app2:80 check
    server app3 app3:80 check
```

👉 Aquí **app2 cae automáticamente** porque devuelve 500.

---

#### 🔌 Backend TCP (L4)

```haproxy
backend be_tcp
    balance roundrobin
    option tcp-check
    http-request set-path %[path,regsub(^/14,)]
    server app1 app1:80 check
    server app2 app2:80 check
    server app3 app3:80 check
```

🧠 *Esta línea es la más difícil del examen:*
`set-path + regsub` elimina `/14` antes de llegar a Nginx.

---

### 🔹 Bloque 4: Estadísticas

```haproxy
listen stats
    bind *:8404
    stats enable
    stats uri /
```

⚠️ **IMPORTANTE**: Dar **ENTER al final del archivo** o HAProxy no arranca.

---

## 4️⃣ Docker Compose (el orquestador)

Aquí solo pensé una cosa:

> **1 balanceador + 3 apps**

Archivo: `docker-compose.yml`

```yaml
services:
  lb:
    image: haproxy:2.9-alpine
    ports:
      - "8080:80"
      - "8404:8404"
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - app1
      - app2
      - app3

  app1:
    image: nginx:alpine
    volumes:
      - ./backends/app1.conf:/etc/nginx/conf.d/default.conf:ro

  app2:
    image: nginx:alpine
    volumes:
      - ./backends/app2.conf:/etc/nginx/conf.d/default.conf:ro

  app3:
    image: nginx:alpine
    volumes:
      - ./backends/app3.conf:/etc/nginx/conf.d/default.conf:ro
```

🧠 *Si no tengo esa versión descargada, uso la que tenga (`docker images`).*

---

## 5️⃣ Ejecución y comprobaciones

```bash
docker compose up -d
```

Si falla:

```bash
docker compose logs lb
```

### ✅ Prueba L7

```bash
curl http://localhost:8080/login
```

Debe salir **OK**, nunca FAIL.

### ❌ Prueba L4

```bash
curl http://localhost:8080/14/login
```

A veces devuelve FAIL (entra app2).

### 🔥 Simular caída

```bash
docker stop app3
```

🧠 *El sistema sigue funcionando con app1.*

---

## 🧯 Consejos finales de supervivencia

* 📦 **Descargar imágenes antes del examen**
* 🧠 Memorizar: `set-path + regsub`
* 📐 YAML → 2 espacios, nunca tabs
* 🔁 Puerto ocupado → cambia 8080 por 9090

---

## ✅ Conclusión

Esta práctica **no va de memorizar comandos**, sino de:

* entender la arquitectura,
* saber qué hace cada bloque,
* y reconstruirla con lógica.

Si entiendes esto, el examen sale.
