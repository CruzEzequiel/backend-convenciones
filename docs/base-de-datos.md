# Base de datos

Acceso directo a PostgreSQL con `asyncpg`: sin ORM y con el SQL en archivos. El esquema vive solo en las migraciones (ver
[migraciones y datos de prueba](migraciones-y-datos-de-prueba.md)); no hay clases de tablas en el código.

## Conexión

`db/database_connection.py` abre **un pool por proceso** al arrancar la app (`lifespan` de `main.py`) y lo cierra al apagarla.
Nunca se crea uno por petición.

- **Si la base no responde, la app no arranca.** El error lleva un texto fijo (`No se pudo conectar a la base de datos`), sin la
  URL ni la contraseña; el detalle va al registro de eventos, con la contraseña quitada.
- Límites en `settings.py`: tamaño del pool, espera de una conexión nueva, espera de una conexión libre, tope de cada consulta
  y cierre de las conexiones inactivas.
- **Cloud Run abre un pool por instancia.** Conexiones totales = instancias × tamaño del pool: debe quedar por debajo de
  `max_connections` de la base (`SHOW max_connections;`), limitando `max-instances`.
- `acquire_connection()` entrega una conexión **sin** transacción automática; `get_transaction()` entrega una dentro de una
  transacción, con commit al salir bien y rollback si hay una excepción.
- **Transacciones cortas:** nunca se dejan abiertas durante esperas ni llamadas a un tercero.
- **Servicios de solo lectura:** cada conexión se abre con `server_settings={"default_transaction_read_only": "on"}`. Aunque
  una consulta futura intentara escribir, Postgres la rechaza.

## Repositorios: el SQL en archivos

Todo el SQL vive en `repositories/<recurso>/sql/`, **un archivo por consulta**, con el nombre de la función que la usa.
Ningún `.py` lleva SQL escrito dentro (ni siquiera un `SELECT 1`).

```
repositories/
├── sql_loader.py
└── conversation/
    ├── conversation_repository.py
    └── sql/register_inbound_message.sql
```

```python
_REGISTER_INBOUND_MESSAGE = load_sql(__file__, "register_inbound_message")

async def register_inbound_message(connection: asyncpg.Connection, message: Message) -> str:
    await connection.execute(_REGISTER_INBOUND_MESSAGE, key, message.phone_number_id, ...)
```

- El repositorio lee sus consultas **al importarse** y las guarda en constantes. Si falta un archivo, la app **no arranca**; no
  falla a mitad de una petición.
- Los `.sql` viajan dentro de `app/`, que es lo que copia la imagen de Docker.

### Cómo se escribe una consulta

- **Tablas con su esquema, siempre** (`FROM whatsapp.messages`). Sin `search_path` oculto.
- **Nada de `SELECT *`:** se piden exactamente las columnas que se necesitan, por nombre, una por línea.
- Palabras clave en mayúsculas, y `FROM`, `WHERE`, `ORDER BY`, `RETURNING` cada uno en su línea.
- **Siempre con parámetros** (`$1`, `$2`...), nunca valores pegados al texto. Son la forma de PostgreSQL y de `asyncpg`; los
  valores viajan aparte del SQL, así que ningún valor se interpreta como SQL. Se llenan **por posición**: si cambias el orden
  en el `.sql`, cambia la llamada en el `.py`. No sirven para nombres de tablas o columnas.
- **Encabezado de comentarios** con qué hace la consulta, **qué devuelve** (las columnas, y cuándo no devuelve nada) y qué es
  cada `$n`.

```sql
-- Toma una acción pendiente (pending -> in_progress) y cuenta el intento. Atómico: solo un proceso la obtiene.
-- Devuelve: id, solo si la tomó.
-- $1 id de la acción
UPDATE whatsapp.message_actions
SET status = 'in_progress', started_at = now(), attempts = attempts + 1
WHERE id = $1 AND status = 'pending'
RETURNING id
```

### Cómo se escribe una función

- **Recibe la conexión como primer argumento.** Quien la llama decide la transacción: el repositorio no abre las suyas, para
  poder juntar varias operaciones en una sola.
- **No imprime ni registra nada.** Los errores de la base suben a quien llama.
- Nombres que digan el efecto y la dirección: `register_inbound_message`, `advance_outbound_message_status`, `mark_action_done`.
- Devuelve un modelo tipado, un `bool` o un identificador cuando solo importa si algo pasó.

## Patrones de consistencia

Todos se resuelven en el SQL, para que sean atómicos:

| Necesidad | Patrón |
|---|---|
| **Idempotencia** (un reintento no duplica) | `INSERT ... ON CONFLICT (...) DO NOTHING RETURNING id`: devuelve una fila solo si se insertó |
| **Una fecha que solo avanza** | `GREATEST(columna, nueva)`: un evento que llega tarde no la hace retroceder |
| **Un estado que solo avanza** | `array_position($lista::text[], estado_nuevo) > COALESCE(array_position($lista::text[], estado_actual), 0)`; un estado repetido, tardío o desconocido no cambia nada |
| **Tomar una tarea una sola vez** | `UPDATE ... SET status = 'in_progress' WHERE id = $1 AND status = 'pending' RETURNING id`: si dos procesos lo intentan, solo uno obtiene la fila |
| **No pisar datos con vacíos** | `COALESCE(nuevo, existente)` en el `DO UPDATE`: un dato ausente no borra el que ya está |
| **JSON** | Columna `jsonb`; se envía con `json.dumps(...)` y `$n::jsonb`, y se lee convirtiendo el texto |

## Convenciones del esquema

- **Un esquema de Postgres propio** por dominio (por ejemplo `whatsapp`), no `public`: otras partes del sistema que compartan la
  base tienen el suyo sin mezclarse. Todo el SQL lo escribe explícito.
- `snake_case`, tablas en plural, `text` (no `varchar`), `timestamptz` siempre y `jsonb` para datos libres.
- Restricciones `CHECK` solo para valores que **controlamos nosotros**. Para los que manda un tercero (un estado de entrega, por
  ejemplo) no: si agrega un valor nuevo, la base lo rechazaría y fallaría todo el procesamiento.
- Nombres de restricciones: `ck_<tabla>_<regla>`, `uq_<tabla>_<columnas>` e índices `ix_<tabla>_<columnas>`.
- Cada columna lleva un comentario con su significado y, si importa, de dónde viene.
