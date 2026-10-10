# TaskFlow API

Primer servidor con FastAPI de la práctica 1.1.

## Cómo arrancar

1. Crear el entorno virtual: python -m venv venv
2. Activarlo: venv\Scripts\activate
3. Instalar dependencias: pip install -r requirements.txt
4. Arrancar el servidor: uvicorn app.main:app --reload
5. Abrir la documentación: http://127.0.0.1:8000/docs

## Endpoints

- GET /health → {"status": "ok"}
- GET /version → {"version": "1.0.0"}
- GET /ping → {"message": "pong"}