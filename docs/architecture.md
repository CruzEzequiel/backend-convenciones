# Arquitectura de apps FastAPI

Convenciones y estructura base para servicios backend en Python. El objetivo es que cualquier servicio nuevo arranque con
una organización consistente, personal y fácil de escalar.

## Stack base

- **Python 3.12 o superior** y **FastAPI** con **uvicorn**
- **Pydantic v2** — modelos del contrato, del dominio y de las respuestas
- **asyncpg** — acceso directo a PostgreSQL, con el SQL en archivos (sin ORM)
- **httpx** — cliente HTTP para llamar a APIs externas
- **python-dotenv** — variables de entorno en local
- **Docker** — para el despliegue (Cloud Run) y la base de desarrollo (Docker Compose)

---

## Estructura de carpetas

```
app/
├── main.py                      # FastAPI: validación de configuración, lifespan (pool), middleware y router
├── router.py                    # Índice único de rutas
│
├── config/
│   ├── settings.py              # Configuración global: por código y por entorno
│   └── startup_checks.py        # Validaciones al arrancar (fallar cerrado)
│
├── db/
│   ├── database_connection.py   # Pool de asyncpg, uno por proceso
│   ├── migrate.py               # Comando del operador: aplica las migraciones
│   └── init_database.py         # Comando del operador: migraciones y datos de prueba
│
├── endpoints/                   # Una carpeta por endpoint
│   └── <recurso>/
│       ├── <recurso>_controller.py
│       ├── <recurso>_service.py
│       └── models/
│           ├── __init__.py      # Vacío
│           └── <recurso>_response_model.py
│
├── repositories/                # Todo el SQL, en archivos
│   ├── sql_loader.py
│   └── <recurso>/
│       ├── <recurso>_repository.py
│       └── sql/<consulta>.sql
│
├── domain/models/               # Modelos internos, con nombres propios
├── clients/                     # Llamadas salientes a APIs externas
├── actions/                     # Ejecutor de tareas registradas (si aplica, ver patrones)
├── middleware/                  # Middlewares de FastAPI
└── utils/                       # logger.py, event_recorder.py, time_utils.py...

migrations/                      # Archivos SQL versionados
seeds/                           # Datos de prueba (no entran en la imagen)
scripts/                         # start_dev.sh y demás
docs/                            # Documentación (docs/develop/ no se versiona)
docker-compose.yml  Dockerfile  requirements.txt  .env.example
```

**Reglas de la estructura**
- Cada endpoint tiene su carpeta. Lo compartido (`config/`, `db/`, `repositories/`, `clients/`, `utils/`...) queda **fuera** de
  `endpoints/`.
- Todas las carpetas llevan `__init__.py`, y en las subcarpetas `models/` va **vacío**: evita importaciones circulares.
- `app/` es lo único que se copia a la imagen, junto con `migrations/`. Por eso los archivos `.sql` de los repositorios
  viven dentro de `app/`.

---

## Capas

El camino de una petición es **controller → service → repository** (y, si hay un tercero, **client**):

| Capa | Qué hace | Qué NO hace |
|---|---|---|
| **Controller** | HTTP: leer el request, validar, elegir el status y devolver el modelo de respuesta | Lógica de negocio, SQL |
| **Service** | La lógica: orquesta repositorios y clientes y arma la respuesta | Conocer HTTP (status, headers) |
| **Mapper** | Traduce el modelo del contrato (un JSON externo) al modelo del dominio | Decidir qué hacer con él |
| **Repository** | Ejecuta una consulta SQL **que vive en un archivo** y devuelve un modelo tipado | Abrir transacciones propias, imprimir |
| **Client** | Llama a una API externa y valida lo que envía | Lanzar errores de red: los registra y devuelve `None` |

### Router

Un solo archivo, `router.py`, es el índice completo de rutas: cada `path + método` apunta al controller de su endpoint, con su
`response_model` y los errores que documenta. No se reparte en varios archivos.

```python
api_router = APIRouter(prefix="/api")

api_router.add_api_route(
    "/health", health, methods=["GET"],
    response_model=HealthResponse,
    responses={503: {"model": HealthResponse, "description": "La base de datos no responde"}},
    tags=["health"],
)
```

### Controller

Lleva al inicio un docstring de módulo que dice qué hace la ruta y, si tiene pasos, el flujo numerado. Solo maneja HTTP:

```python
async def health(response: Response) -> HealthResponse:
    result = await health_service.get_health()
    if result.status == "error":
        response.status_code = 503
    return result
```

### Service

La lógica. No sabe de HTTP: devuelve modelos y deja que el controller decida el status. Cuando un service tiene dos momentos
(por ejemplo, *registrar* antes de responder y *procesar* después), lo explica en el docstring del módulo (ver
[patrones de procesamiento](patrones-de-procesamiento.md)).

### Client

Una carpeta `clients/` con un archivo por API externa. Valida con expresiones regulares los identificadores que va a poner en
una URL **antes** de llamar, enmascara la autenticación en cualquier registro y, si algo falla, lo registra y devuelve `None`
en lugar de lanzar: quien lo llama decide qué hacer.

---

## Modelos

Hay dos familias, con reglas distintas, y todas las respuestas van tipadas. Ver [modelos y tipado](modelos-y-tipado.md).

---

## Convenciones de nombres

| Qué | Convención | Ejemplo |
|-----|-----------|---------|
| Identificadores (variables, funciones, clases, archivos) | Inglés | `register_inbound_message` |
| Comentarios, docstrings y documentación | Español de México | — |
| Archivos | `snake_case.py` con el rol al final | `conversations_controller.py` |
| Controllers | `<recurso>_controller.py` | `health_controller.py` |
| Services | `<recurso>_service.py` | `health_service.py` |
| Mappers | `<tipo>_mapper.py` con `to_domain` | `text_message_mapper.py` |
| Repositorios | `<recurso>_repository.py` | `message_repository.py` |
| Consultas SQL | El nombre de la función que las usa | `sql/insert_message_if_new.sql` |
| Modelos | `<nombre>_model.py` (contrato y dominio) y `<recurso>_response_model.py` (respuestas) | `conversation_model.py` |
| Constantes privadas de módulo | `_MAYÚSCULAS` | `_CHECK_DATABASE` |
| Funciones que dicen qué hacen y en qué dirección | Verbo + objeto + dirección | `advance_outbound_message_status` |
| Nombres de tablas | `snake_case`, plural, con su esquema | `whatsapp.messages` |

Los nombres de las funciones y los archivos deben decir **qué hacen** sin tener que abrirlos. Si una función solo aplica a un
sentido (entrante o saliente), el nombre lo dice.

---

## Agregar un endpoint nuevo — checklist

1. Crear la carpeta `endpoints/<recurso>/` con su `__init__.py` y la subcarpeta `models/` (con su `__init__.py` vacío).
2. Escribir el modelo de respuesta en `models/<recurso>_response_model.py`, un campo por línea con su comentario.
3. Escribir la consulta en `repositories/<recurso>/sql/<consulta>.sql` y la función que la usa en `<recurso>_repository.py`.
4. Escribir `<recurso>_service.py` con la lógica.
5. Escribir `<recurso>_controller.py`: solo HTTP, con su docstring.
6. Registrar la ruta en `router.py` con `response_model` y los errores que documenta.
7. Si hay un tercero, agregar o ampliar su cliente en `clients/`.
8. Probar contra la base local con los datos de prueba y actualizar la documentación.
