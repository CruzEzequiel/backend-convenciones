# Migraciones y datos de prueba

## Migraciones

El esquema se define en archivos SQL versionados dentro de `migrations/`, sin Alembic. Un ejecutor propio
(`python -m app.db.migrate`) los aplica en orden.

### Nombre del archivo

```
<mayor>.<menor>.<parche>_<descripción>.sql        ejemplo:  1.1.0_agregar_media_a_messages.sql
```

- **Versión:** tres números enteros, con orden **numérico** (`1.2.0` va antes que `1.10.0`). No se repite una versión ni se usa
  una menor que la última aplicada: el ejecutor lo rechaza.
- **Significado** (convención del proyecto; el ejecutor solo usa el orden): `mayor`, un cambio grande que rompe la
  compatibilidad con el código anterior; `menor`, un cambio compatible (una tabla o una columna nueva); `parche`, un ajuste
  pequeño (un índice, una restricción).
- **Descripción:** en español, minúsculas, con guiones bajos, sin acentos, empezando con un verbo (`crear_`, `agregar_`,
  `quitar_`, `cambiar_`).

### Reglas

1. **Solo hacia adelante.** Una migración aplicada **no se edita ni se borra**: el ejecutor guarda su checksum (SHA-256) y se
   detiene si el archivo cambia o falta. Un cambio sobre una tabla existente es una migración **nueva**.
2. **No hay migraciones de reversa.** Deshacer algo también es una migración nueva.
3. **Un cambio por archivo**, con un nombre que lo describa.
4. **Cada archivo corre dentro de una transacción** que abre el ejecutor, junto con su fila en el historial. Por eso no llevan
   `BEGIN` ni `COMMIT`, ni sentencias que no pueden correr dentro de una transacción (`CREATE INDEX CONCURRENTLY`, `VACUUM`).
   Si algo falla, todo el archivo se revierte y el ejecutor se detiene.
5. **Sin `IF NOT EXISTS`.** Cada migración se aplica una sola vez: que falle de forma visible es mejor que ocultar una diferencia
   entre bases.
6. **Sin datos de prueba:** van en `seeds/`.
7. **Compatibles hacia atrás.** Se aplican **antes** de arrancar el código nuevo, así que el código anterior debe seguir
   funcionando con el esquema nuevo: se agregan columnas nulas o con valor por defecto y, para quitar o renombrar algo, se hace
   en dos pasos (primero lo nuevo y el cambio de código; después, en otra migración, se quita lo viejo).

### El ejecutor

- Crea el esquema de la aplicación si no existe, antes de nada, y lleva el historial en `<esquema>.schema_migrations`
  (`version`, `description`, `checksum`, `applied_at`). Las migraciones asumen que el esquema ya existe. Si el usuario de la base
  no puede crearlo, el error dice el comando que lo arregla (`GRANT CREATE ON DATABASE <base> TO <usuario>;`).
- Un **bloqueo de Postgres** (`pg_advisory_lock`) hace que una segunda ejecución espere (hasta 60 s) y no corra a la vez.
- Rechaza: un nombre de archivo inválido, una versión repetida, un checksum cambiado, un archivo que falta y una migración
  pendiente con versión menor que una ya aplicada.
- **Solo necesita la base.** No exige los demás secretos de la app, para poder correr como Job con solo `DATABASE_URL`.
- **Nunca corre al arrancar la app.** Es un comando del operador y un paso del despliegue.
- Es un comando para quien opera: imprime con `print` qué aplicó (versiones y descripciones, que no son datos de usuarios).

## Datos de prueba

Archivos SQL en `seeds/`, aparte del historial de migraciones:

```
seeds/
├── 00_limpiar_datos_de_prueba.sql   # borra los datos de prueba anteriores
├── 01_conversaciones.sql
└── 02_mensajes.sql
```

- Nombre `<número>_<descripción>.sql`; se ejecutan en orden, cada uno en su transacción.
- **Datos ficticios y reconocibles:** llevan un prefijo (`SEED`) en sus claves. El primer archivo borra **solo** las filas con ese
  prefijo y lo que cuelga de ellas; nunca toca otros datos.
- **Fechas relativas a `now()`**, para que "hace un minuto" siempre lo sea. Por eso se recargan completos en cada arranque, en
  lugar de usar `ON CONFLICT DO NOTHING`, que dejaría las fechas viejas.
- **Cubren los casos del flujo:** una conversación con la ventana abierta, una con la ventana cerrada, una sin teléfono, una
  acción en cada estado.
- **Solo en desarrollo:** se cargan con `--seed` y se rechazan antes de tocar la base si `IS_DEVELOPMENT` no es `true`. Como en
  desarrollo la conexión es siempre a la base local, nunca llegan a la remota. `seeds/` no entra en la imagen.
- Los nombres de los archivos se validan **antes** de conectar: si hay un error, no se aplica ni siquiera una migración.

## Inicialización

`python -m app.db.init_database [--seed]` deja la base lista: verifica la conexión, aplica las migraciones pendientes (con el
código del ejecutor) y, con `--seed`, carga los datos de prueba. Es repetible, y el mismo bloqueo cubre ambos pasos. Supone que
la base y el usuario ya existen.

## Un servicio que no es dueño del esquema

Un segundo servicio que **solo lee** la misma base no migra. Para sus pruebas y su desarrollo tiene su **propia base local,
idéntica**, creada con un SQL completo:

- `database/schema.sql`: una **copia** de todas las migraciones, más el historial con las versiones aplicadas.
- `database/seed.sql`: una copia de los datos de prueba.
- Se aplica una sola vez sobre una base vacía (el script de arranque lo hace si el esquema falta).
- **Riesgo:** si el dueño cambia el esquema y la copia no se actualiza, las pruebas siguen pasando contra un esquema que ya no
  es el real. Quien cambia el esquema actualiza también la copia.
