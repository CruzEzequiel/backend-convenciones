# Despliegue

Servicios FastAPI en **Cloud Run**, con **Cloud SQL (PostgreSQL)**, construidos desde un repositorio de Git con un `Dockerfile`.

## Imagen de Docker

```dockerfile
FROM python:3.13-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PORT=8080

WORKDIR /srv

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app
COPY migrations ./migrations

RUN useradd --create-home appuser && mkdir -p logs && chown appuser logs
USER appuser

EXPOSE 8080
CMD exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT} --no-access-log
```

- Solo se copian `app/` y `migrations/`. **No** `seeds/`, `.env`, `docker-compose.yml` ni documentación (`.dockerignore`).
- Se ejecuta como un usuario sin privilegios.
- `--no-access-log`: la aplicación no imprime la URL de cada petición (ver [logs y datos personales](logs-y-datos-personales.md)).
- Los archivos `.sql` de los repositorios van dentro de `app/`, por eso viajan en la imagen.

## Cloud SQL

- **Misma región** que Cloud Run y **la misma versión de PostgreSQL** que el entorno de desarrollo (no se puede cambiar después).
- El tamaño más pequeño que alcance (los tipos compartidos requieren la edición Enterprise); se puede **detener** el cómputo
  cuando no se prueba (`--activation-policy=NEVER`).
- SSL obligatorio y **sin redes autorizadas**: el servicio entra por el socket de Cloud SQL, que ya viaja cifrado.
- Una base y un usuario propios de la aplicación. La contraseña, larga y aleatoria; con símbolos que no obliguen a codificar la
  URL (`-`, `_`, `.`).
- Copias de seguridad diarias si hay datos de personas.
- El esquema lo crea el Job de migraciones; si el usuario no puede crear esquemas, se le da `GRANT CREATE ON DATABASE`.

### `DATABASE_URL` con el socket

```
postgresql://USUARIO:CONTRASEÑA@/BASE?host=/cloudsql/PROYECTO:REGIÓN:INSTANCIA
```

Sin host ni puerto (entre `@` y `/` no va nada) y con el nombre de conexión de la instancia después de `/cloudsql/`. Sin comillas.
Si la contraseña lleva `@ / : # ? %`, se codifican.

## Servicio de Cloud Run

| Campo | Valor |
|---|---|
| Origen | El repositorio de Git, con el tipo de compilación **Dockerfile** |
| Autenticación | Sin autenticar si lo llama un tercero (la firma protege) |
| Facturación | Por instancia (CPU siempre asignada) si hay trabajo en segundo plano |
| Instancias | Mínimo `0`; máximo tal que instancias × tamaño del pool quede bajo `max_connections` |
| Puerto | `8080` |
| Variables | Los secretos y `DATABASE_URL`. **No** `IS_DEVELOPMENT` |
| Conexiones de Cloud SQL | La instancia |

La cuenta de servicio necesita el rol **Cliente de Cloud SQL**. Un servicio puede desplegarse **antes** de aplicar las migraciones:
arranca, y las peticiones responden `500` (y el tercero reintenta) hasta que el esquema exista.

## Migraciones: un Job de Cloud Run

Se aplican con un **Job** que usa **la misma imagen**, la misma instancia asociada y la misma `DATABASE_URL`, y ejecuta
`python -m app.db.migrate`:

| Campo | Valor |
|---|---|
| Imagen | La que construyó el servicio **para el commit que se quiere aplicar** |
| Comando y argumentos | `python` y `-m`, `app.db.migrate` |
| Reintentos | `0` |
| Región | La misma. No se puede cambiar después, ni el nombre |

- Quien lo lanza necesita permiso sobre el Job, **no** la contraseña de la base.
- **Orden de cada despliegue:** construir la imagen, ejecutar el Job y, solo si termina bien, actualizar el servicio.
- El Job **sigue apuntando a la imagen que se le puso**: cuando haya migraciones nuevas, se edita para elegir la del commit
  nuevo antes de ejecutarlo. Una imagen de un commit anterior aplicaría las migraciones viejas.
- Es repetible: ejecutarlo otra vez dice `No hay migraciones pendientes`.

## Conectar el tercero

La URL del webhook es la del servicio más la ruta, y la verificación usa un token que debe coincidir en los dos lados. Si el
tercero firma con un secreto, ese secreto **debe ser idéntico** en el servicio: con uno distinto la verificación pasa, pero cada
evento real se rechaza con `401`.

## Problemas comunes

| Síntoma | Causa probable |
|---|---|
| El servicio no inicia y el error dice `Faltan las variables de entorno: …` | Falta esa variable en el servicio |
| `No se pudo conectar a la base de datos` | Falta asociar la conexión de Cloud SQL, falta el rol *Cliente de Cloud SQL* o hay una errata en el nombre de conexión o la contraseña |
| Los webhooks responden `500` | Las migraciones no se aplicaron (o el Job usó una imagen de un commit viejo) |
| Una acción en segundo plano queda a medias | La CPU se limitó tras responder: facturación por instancia |
| Verificación correcta pero no llegan eventos | El secreto de firma distinto, o el campo del webhook no está suscrito |
