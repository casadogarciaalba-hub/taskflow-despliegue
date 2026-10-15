# Práctica 1.3 - Instalación básica de Gunicorn

Alba Casado García - 2º DAW

La práctica la he hecho en mi máquina virtual de Ubuntu Server, que es la versión 22.04. Como en la consola de VirtualBox no se puede pegar, me conecté por SSH desde PowerShell (puse una regla de reenvío de puertos del 2222 al 22) y todos los comandos los he ido metiendo desde ahí.

## 1. Instalar Python 3.13 y crear el entorno virtual

Mi Ubuntu traía Python 3.10 y la práctica pide 3.13 o más, así que añadí el repositorio de deadsnakes, que es donde están las versiones nuevas de Python:

```bash
sudo apt install -y software-properties-common && sudo add-apt-repository -y ppa:deadsnakes/ppa
```

Luego instalé Python 3.13 y también `python3.13-venv`, que si no lo pones no te deja crear el entorno virtual:

```bash
sudo apt update && sudo apt install -y python3.13 python3.13-venv
python3.13 --version
```

Me sale `Python 3.13.16`.

Después creé la carpeta del backend, dentro el entorno virtual y lo activé. Se sabe que está activado porque sale `(venv)` al principio de la línea. El entorno virtual sirve para que las librerías del proyecto se instalen solo ahí y no en todo el sistema.

```bash
mkdir -p ~/taskflow/backend && cd ~/taskflow/backend
python3.13 -m venv venv
source venv/bin/activate
```

## 2. Instalar dependencias y Gunicorn

Para hacer el `pip install -r requirements.txt` primero necesitaba el fichero, así que lo creé con lo que usa mi main.py, que es fastapi y uvicorn:

```bash
printf "fastapi\nuvicorn\n" > requirements.txt
pip install -r requirements.txt
pip install gunicorn
```

## 3. Añadir el main.py de la práctica 1.1

El main.py lo tenía en Windows en el proyecto de VS Code. Creé la carpeta `app` en el Ubuntu y luego lo pasé con scp desde la terminal de VS Code, junto con el `__init__.py` (sin él no funciona lo de `app.main:app`).

```bash
mkdir -p app
```

```powershell
scp -P 2222 C:\Users\TP3402\Desktop\taskflow_api\app\main.py C:\Users\TP3402\Desktop\taskflow_api\app\__init__.py alba@127.0.0.1:~/taskflow/backend/app/
```

Con `cat app/main.py` miré que había llegado bien y que tenía los tres endpoints, /health, /version y /ping.

## 4. Arrancar Gunicorn

```bash
gunicorn app.main:app --worker-class uvicorn.workers.UvicornWorker --bind 127.0.0.1:8000
```

Se ve que está escuchando en 127.0.0.1:8000, que usa el UvicornWorker y que pone Application startup complete. Como esta terminal se queda pillada con Gunicorn, abrí otra pestaña y me volví a conectar por SSH para hacer las pruebas.

## 5. Comprobar con curl

Puse el `-i` para que salgan también las cabeceras y se vea el 200 OK y que es JSON.

```bash
curl -i http://127.0.0.1:8000/health    # 200 OK  {"status":"ok"}
curl -i http://127.0.0.1:8000/version   # 200 OK  {"version":"1.0.0"}
curl -i http://127.0.0.1:8000/ping      # 200 OK  {"message":"pong"}
```

Los tres devuelven 200 y JSON. Además probé una ruta que no existe para ver que de verdad es mi aplicación la que contesta y no una página por defecto. Devuelve el 404 en JSON, que es el error típico de FastAPI, y en server pone uvicorn. Si fuera la página por defecto de un servidor saldría HTML.

```bash
curl -i http://127.0.0.1:8000/noexiste  # 404 Not Found  {"detail":"Not Found"}
```

Las capturas están en el PDF.

## 6. Qué hace el comando y por qué hace falta UvicornWorker

`gunicorn` arranca el servidor. `app.main:app` es dónde está la aplicación: `app.main` es el archivo app/main.py y el `app` de después de los dos puntos es la variable donde está creado el FastAPI. `--worker-class uvicorn.workers.UvicornWorker` hace que los workers sean de Uvicorn y `--bind 127.0.0.1:8000` es la IP y el puerto donde escucha.

Lo del UvicornWorker es obligatorio por lo de WSGI y ASGI. WSGI es la forma antigua de que un servidor se comunique con una app de Python, y es síncrono, o sea, va petición por petición (es lo que usa Flask por ejemplo). ASGI es lo nuevo, asíncrono, que permite async/await y atender varias peticiones a la vez. Gunicorn de normal solo sabe WSGI, y FastAPI es ASGI, así que no la puede ejecutar él solo. Con el UvicornWorker, Gunicorn lleva los procesos y dentro de cada uno va Uvicorn, que es el que entiende ASGI.

## 7. Por qué Gunicorn y no solo Uvicorn

Con Uvicorn solo también funcionaría, pero Uvicorn es más para desarrollo. Gunicorn sirve para producción porque gestiona los procesos: puedes poner varios workers para usar todos los núcleos, si uno se cae lo vuelve a levantar y se puede reiniciar sin cortar el servicio. Básicamente Gunicorn controla y Uvicorn ejecuta la app.

## 8. Servidor de aplicaciones y el bind

Apache es un servidor web, recibe las peticiones HTTP y sirve páginas y archivos. Gunicorn es un servidor de aplicaciones, lo que hace es ejecutar el código de Python. Tiene un proceso master que crea los workers y los vigila, y los workers son los que atienden las peticiones.

Que Gunicorn escuche solo en 127.0.0.1 es para que no se pueda entrar desde fuera de la máquina. Lo normal es poner Apache delante como proxy inverso, que recibe lo de fuera y se lo pasa a Gunicorn por local.

En el --bind se pone IP:puerto. Con 127.0.0.1 solo se entra desde la propia máquina. localhost es lo mismo pero con nombre, que apunta a 127.0.0.1. Y 0.0.0.0 es escuchar en todas las interfaces, así que se podría conectar cualquiera de la red.

## 9. Parar, arrancar y errores

Para arrancarlo hay que entrar en la carpeta, activar el venv y lanzar el comando:

```bash
cd ~/taskflow/backend
source venv/bin/activate
gunicorn app.main:app --worker-class uvicorn.workers.UvicornWorker --bind 127.0.0.1:8000
```

Para pararlo, Ctrl + C en esa terminal o `pkill gunicorn` desde otra.

Errores que te pueden salir: si pone que no encuentra gunicorn es que no has activado el venv. Si dice No module named 'app' es que no estás en la carpeta backend o falta el __init__.py. Si dice Connection in use es que el puerto 8000 está pillado por otro Gunicorn que se quedó abierto, y se quita con pkill gunicorn. Si el curl da Failed to connect es que Gunicorn no está arrancado. Y si lo arrancas sin el --worker-class de Uvicorn falla al hacer una petición, por lo de WSGI/ASGI.

A mí al principio no me dejaba instalar nada porque dpkg se había quedado a medias de una actualización, lo arreglé con `sudo dpkg --configure -a`.
