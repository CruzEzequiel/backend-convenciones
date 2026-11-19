# Modelos y tipado

Regla de fondo: **se debe saber exactamente qué se espera y qué se toma de cada respuesta.** Ningún dato cruza una frontera
como un `dict` suelto.

## Dos familias de modelos

| | Modelos del contrato | Modelos del dominio |
|---|---|---|
| **Qué reflejan** | El JSON de un tercero, tal cual (por ejemplo, el payload de un webhook) | Lo que el servicio necesita, con nombres propios |
| **Nombres** | Los del tercero (`from`, `id`, `type`...) | Los nuestros (`wamid`, `message_type`, `from_number`...) |
| **Dónde viven** | `endpoints/<recurso>/models/` | `domain/models/` |
| **Quién los usa** | El controller y el mapper | Los services y los repositorios |

El código fuera de `endpoints/` no debe importar modelos del contrato, solo del dominio. El **mapper** (`to_domain`) es el único
punto que conoce los dos.

### Modelos del contrato

- Reflejan el JSON **sin renombrar** campos ni usar alias. La única excepción son las palabras reservadas de Python
  (`from_` con `alias="from"`).
- Una base común con `extra="allow"`: si el tercero agrega un campo, no rompe nada.
- Casi todo es opcional: los terceros omiten campos según el caso. Lo que siempre viene se declara obligatorio.
- **Un comentario por campo** que dice qué es y de dónde viene.
- Una unión por tipo se resuelve con `Discriminator` y `Tag`, con una caída a un modelo `Unknown...` para los tipos que todavía
  no tienen modelo propio: se registran y se ignoran, en vez de rechazar la petición.

```python
Message = Annotated[
    Union[
        Annotated[TextMessage, Tag("text")],
        Annotated[ImageMessage, Tag("image")],
        Annotated[UnknownMessage, Tag("other")],
    ],
    Discriminator(_message_type),
]
```

### Modelos del dominio

- Con nombres propios, independientes del tercero. Si el tercero cambia de nombres, solo cambia el mapper.
- **Nada de valores por defecto que oculten un significado.** Si un mensaje puede ser entrante o saliente, `direction` es
  obligatorio y cada mapper lo indica; un mensaje sin dirección no debe poder crearse por descuido.
- Las fechas son `datetime` con zona horaria (UTC) y se guardan como `timestamptz`.
- Un modelo leído de la base puede traer solo parte de los campos: documéntalo en su docstring (por ejemplo, que un
  `conversation_key` calculado no sirve si el modelo no trae los datos de los que se calcula).

## Respuestas de la API

- **Cada ruta declara su `response_model`** y los errores que documenta (`responses={503: {"model": ...}}`). Ninguna devuelve un
  `dict`. `/docs` muestra así exactamente lo que responde.
- Los modelos de respuesta viven en `endpoints/<recurso>/models/<recurso>_response_model.py`, un campo por línea con su
  comentario.
- Los valores cerrados se tipan con `Literal`:

```python
class HealthResponse(BaseModel):
    status: Literal["ok", "error"]  # "ok" si la base de datos responde
    database: Literal["up", "down"]  # estado de la conexión con Postgres
```

- Un error documentado puede llevar el **mismo cuerpo** que el éxito (un `503` de salud que dice qué falla): así el cliente lo
  trata como parte del contrato y no como un fallo de red.
- Las listas paginadas tienen una forma fija: `{ "items": [...], "next_cursor": "..." }`.
- Las respuestas de error llevan **textos fijos**, sin detalles internos.

## Lo que sale de la base

El `Record` de `asyncpg` **no sale del repositorio**: cada función devuelve un modelo tipado, un `bool` o un identificador. Si
una columna `jsonb` llega como texto, el repositorio la convierte antes de armar el modelo.

## En el frontend

El espejo de esta regla en el cliente: el tipo de cada respuesta vive **junto a la función que llama a esa ruta** y es
exactamente lo que responde el backend; cada hook o componente declara lo que toma de él (por ejemplo, con `Pick<…>`). Ver
`frontend-convenciones`.
