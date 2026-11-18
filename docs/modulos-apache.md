# Práctica 2.1 — Módulos de Apache

**Alba Casado García** · 2º DAW · Despliegue de aplicaciones web

En esta práctica he activado los módulos de Apache que vamos a necesitar en el proyecto TaskFlow. Lo he hecho en mi máquina virtual de Ubuntu Server 22.04, conectada por SSH desde PowerShell (`ssh -p 2222 alba@localhost`).

## Qué es un módulo de Apache

Un módulo es como un "plugin" de Apache: un trozo de código que le añade una función que por sí solo no tiene. Apache de base sabe servir archivos, pero para hacer HTTPS, redirigir URLs o pasarle peticiones a otra aplicación necesita su módulo correspondiente.

Al listar los módulos se ven dos tipos:

- **static**: vienen metidos dentro de Apache y siempre están cargados (por ejemplo `core_module`).
- **shared**: se pueden activar con `a2enmod` y desactivar con `a2dismod`.

Hay que activarlos porque Apache, por defecto, solo carga lo básico. Así gasta menos recursos y tiene menos cosas que puedan dar problemas de seguridad. Si en la configuración usas una directiva de un módulo que no está activo (por ejemplo `ProxyPass` sin `proxy`), Apache da error y no arranca.

## Pasos que he seguido

### 1. Ver los módulos activos

~~~bash
apache2ctl -M
~~~

Me salió la lista de módulos cargados y ninguno de los 5 que pide la práctica estaba activo.

### 2. Activar los módulos

~~~bash
sudo a2enmod proxy proxy_http ssl rewrite headers
~~~

Se activaron los 5. Además se activó solo `socache_shmcb`, porque `ssl` lo necesita para guardar en caché las sesiones SSL.

### 3. Reiniciar Apache y comprobar

~~~bash
sudo systemctl restart apache2
systemctl status apache2
~~~

Apache reinició sin errores y sale `active (running)`.

Para comprobar que los módulos están cargados filtré la lista con `grep`:

~~~bash
apache2ctl -M | grep -E "proxy|ssl|rewrite|headers"
~~~

~~~
headers_module (shared)
proxy_module (shared)
proxy_http_module (shared)
rewrite_module (shared)
ssl_module (shared)
~~~

Están los 5.

## Para qué sirve cada módulo

### proxy

Hace que Apache funcione como **proxy inverso**: recibe la petición del navegador y se la pasa a otra aplicación (en nuestro caso, el backend de TaskFlow). Él solo no sabe hablar ningún protocolo, por eso va junto con `proxy_http`.

~~~apache
ProxyPreserveHost On
ProxyPass        /api/ http://127.0.0.1:8000/
ProxyPassReverse /api/ http://127.0.0.1:8000/
~~~

### proxy_http

Es el complemento de `proxy` para que pueda reenviar peticiones por **HTTP/HTTPS**. Es el que permite conectar Apache con el backend que corre con Gunicorn en el puerto 8000.

~~~apache
<VirtualHost *:80>
    ServerName taskflow.local
    ProxyPass        / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/
</VirtualHost>
~~~

### ssl

Sirve para que Apache pueda usar **HTTPS** (puerto 443). Cifra la comunicación entre el navegador y el servidor usando un certificado.

~~~apache
<VirtualHost *:443>
    ServerName taskflow.local
    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/taskflow.crt
    SSLCertificateKeyFile /etc/ssl/private/taskflow.key
</VirtualHost>
~~~

### rewrite

Permite **reescribir y redirigir URLs** con reglas. El uso más típico es mandar todo lo que entra por HTTP a HTTPS.

~~~apache
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
~~~

### headers

Permite **añadir, cambiar o quitar cabeceras HTTP** de las respuestas. Se usa para cabeceras de seguridad o para configurar CORS en la API.

~~~apache
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "DENY"
Header set Access-Control-Allow-Origin "*"
~~~

## Otros módulos interesantes

Hay otros módulos que no he activado en esta práctica pero que también se usan mucho:

- **deflate**: comprime las respuestas para que la web cargue más rápido (este ya venía activo).
- **expires**: le dice al navegador cuánto tiempo guardar en caché los archivos estáticos.
- **proxy_balancer**: reparte las peticiones entre varios backends (balanceo de carga).
- **proxy_wstunnel**: para hacer proxy de WebSockets.
- **http2**: activa HTTP/2, que es más rápido que HTTP/1.1.
- **security2 (ModSecurity)**: un cortafuegos de aplicaciones web que bloquea ataques típicos.

## Nota

Al ejecutar `apache2ctl -M` sale el aviso `AH00558: Could not reliably determine the server's fully qualified domain name`. Es solo un aviso porque no hay `ServerName` global configurado; no es un error y Apache funciona bien.
