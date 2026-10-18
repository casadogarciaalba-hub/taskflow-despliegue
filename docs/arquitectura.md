# Práctica 1.4: Instalación directa vs. contenedor

Alba Casado García · 2º DAW

En la práctica 1.3 instalé el backend de TaskFlow directamente en el Ubuntu (Python 3.13, entorno virtual y Gunicorn). En esta práctica he desplegado el mismo backend dentro de un contenedor Docker. Aquí comparo las dos formas.

## 1. Contenedor vs. máquina virtual

Mi Ubuntu Server es una máquina virtual de VirtualBox, y dentro he puesto un contenedor Docker, así que he usado las dos cosas:

| | Máquina virtual | Contenedor |
|---|---|---|
| Aislamiento | Total: tiene su propio sistema operativo y su propio kernel | Comparte el kernel del Ubuntu; solo aísla procesos, archivos y red |
| Peso | Varios GB (el disco de mi Ubuntu son 24 GB) | Pocos MB (mi imagen parte de `python:3.13-slim`) |
| Arranque | Minutos, tiene que arrancar el sistema entero | Segundos, solo arranca la aplicación |
| Uso de recursos | Reserva RAM y CPU para ella aunque no las use | Usa solo lo que necesita la aplicación |

Resumen: la máquina virtual "imita un ordenador entero" y el contenedor solo "encierra una aplicación".

## 2. Instalación directa vs. contenedor
~~~
INSTALACIÓN DIRECTA (práctica 1.3)        CONTENEDOR (práctica 1.4)

+-----------------------------+           +-----------------------------+
| Backend TaskFlow (Gunicorn) |           | +-------------------------+ |
| venv: fastapi, uvicorn...   |           | | Contenedor taskflow     | |
| Python 3.13 (deadsnakes)    |           | | Python 3.13 + librerías | |
|                             |           | | Backend (uvicorn)       | |
|                             |           | +-------------------------+ |
|                             |           |  Docker Engine              |
+-----------------------------+           +-----------------------------+
|      Ubuntu Server 22.04    |           |      Ubuntu Server 22.04    |
+-----------------------------+           +-----------------------------+
~~~

**Instalación directa**

- Ventajas: no hace falta aprender Docker, no gasta nada extra y es fácil ver los logs y archivos.
- Inconvenientes: tuve que instalar Python 3.13 desde otro repositorio y crear el venv a mano; si otra aplicación del servidor necesita otra versión de Python pueden chocar; y en otro servidor hay que repetir todos los pasos.

**Contenedor**

- Ventajas: todo lo que necesita el backend (Python, librerías y código) va dentro de la imagen, así que funciona igual en cualquier máquina con Docker; no ensucia el Ubuntu; se arranca, para, reinicia o borra con un comando; y si algo falla, borro el contenedor y creo otro en segundos.
- Inconvenientes: hay que instalar Docker y aprender a usarlo, las imágenes ocupan espacio, y para ver qué pasa dentro hay que usar comandos de Docker (`docker logs`, `docker exec`).

## 3. Caso TaskFlow: Docker para el backend y no para el servidor web

- **Backend en Docker:** es la parte que más cambia (código nuevo, librerías nuevas) y la que más depende de versiones concretas de Python. Con Docker cada versión del backend es una imagen nueva y se cambia sin tocar el sistema.
- **Servidor web (Apache) instalado directamente:** casi no cambia, ya lo trae Ubuntu en sus repositorios, se gestiona con `systemctl` y es el que da la cara a Internet (puertos 80/443, certificados). Meterlo en un contenedor no aporta mucho y añade una capa más.

## 4. Criterios de decisión: cuándo usar cada opción

| Usaría instalación directa si... | Usaría contenedor si... |
|---|---|
| Es un servicio estable del sistema (Apache, base de datos en un solo servidor) | La aplicación cambia a menudo y se despliegan versiones nuevas |
| Hay un solo servidor y una sola aplicación | Hay varias aplicaciones con versiones distintas en el mismo servidor |
| El servidor tiene pocos recursos | Quiero que funcione igual en mi PC, en clase y en el servidor |
| No se va a mover a otra máquina | Se va a desplegar en la nube o en varios servidores |

## 5. Qué aporta desplegar en la nube

- **Portabilidad:** la misma imagen que he construido en mi Ubuntu se puede subir a un registro (Docker Hub, por ejemplo) y ejecutarla en un servidor de la nube sin volver a instalar nada.
- **Escalabilidad:** si hay muchos usuarios, la nube puede arrancar más contenedores del backend; con la instalación directa habría que preparar otro servidor a mano.
- **Disponibilidad:** el proveedor se encarga del hardware, y si un contenedor falla se puede reiniciar solo.
- **Pago por uso:** solo pagas los recursos que gastas, y como los contenedores son ligeros, gastan menos que una máquina virtual por aplicación.

Por eso para la nube es mejor el contenedor: lo que pruebo en local es exactamente lo mismo que se despliega fuera.
