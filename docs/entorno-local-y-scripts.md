# Entorno local y scripts

La regla: **un solo comando deja todo listo para probar.**

## Docker Compose para la base

`docker-compose.yml` levanta un Postgres de desarrollo, **de la misma versión que producción**:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: <contraseña local>   # solo desarrollo
      POSTGRES_DB: app
    ports:
      - "127.0.0.1:5432:5432"                 # solo el propio equipo
    volumes:
      - app_pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 3s
      timeout: 3s
      retries: 20
```

- El puerto se ata a `127.0.0.1`.
- Un **healthcheck**, para que los scripts esperen a que la base esté lista.
- **Volúmenes con nombre**, para conservar los datos.
- El **nombre de la carpeta** del repositorio es el nombre del proyecto de Compose y de él salen los nombres de los volúmenes: si
  la carpeta cambia de nombre, los volúmenes quedan vacíos. Al mover un repositorio, se conserva el nombre de su carpeta.
- Varios servicios en la misma máquina usan **puertos distintos** para su base.
- Las herramientas de apoyo (por ejemplo, un visor de datos) van en un **perfil** (`profiles: ["tools"]`) y no arrancan con
  `docker compose up -d db`.

## `scripts/start_dev.sh`

Un solo script prepara el entorno y arranca el servicio. Es **repetible**: ejecutarlo otra vez no rompe nada.

Pasos, en orden:

1. Comprobar que Docker está instalado y responde.
2. Comprobar el `.env`: si no existe, crearlo desde `.env.example` y detenerse para que se complete; avisar si falta la bandera
   de desarrollo.
3. Entorno virtual y dependencias, **solo si faltan o cambió `requirements.txt`** (con un archivo marca para saberlo).
4. Levantar la base **solo si no está saludable** y esperar a que lo esté.
5. Aplicar las migraciones pendientes y cargar los datos de prueba (`init_database --seed`): la base queda lista para pruebas.
6. Comprobar que el puerto está libre.
7. Opcionales con banderas: `--tunnel` (expone el servicio a internet mientras corra, para que el tercero llegue al equipo) y
   `--tools` (levanta y configura las herramientas de apoyo).
8. Arrancar la app **en primer plano**.

Detalles de implementación que importan:

- `set -euo pipefail` y `cd "$(dirname "$0")/.."`, para correr desde cualquier carpeta.
- La app y el túnel corren en **su propio grupo de procesos** (`setsid`): `Ctrl+C` solo llega al script, que les manda **una**
  señal de término y espera, para que la app cierre ordenadamente (incluido el pool de la base).
- `trap` para `EXIT`, `INT` y `TERM`.
- `--help` imprime las primeras líneas del propio script.
- Los errores empiezan con `ERROR:` y dicen qué hacer.

## Herramientas de apoyo

Una herramienta como Metabase se levanta con un perfil de Compose y se deja **configurada por script** (usuario administrador y
conexión a la base), con un script **idempotente**: si ya está configurada, no cambia nada. Su base interna va en un volumen
propio y su puerto es distinto de los demás.

## Probar contra un tercero desde el equipo

- El tercero necesita una URL pública: un **túnel** hacia el puerto local.
- Muchos terceros aceptan **una sola URL de webhook por aplicación**: al apuntarla a la nube, el túnel local deja de recibir.
- Las llamadas salientes se pueden simular en las pruebas sustituyendo las funciones del cliente.

## Verificación por fase

Cada fase de un plan termina con pruebas que se ejecutan contra algo real y reproducible:

- Una **base temporal** creada con las migraciones (o el SQL completo) y los datos de prueba, que se borra al terminar.
- Las llamadas a terceros **simuladas**, y bloqueada su salida de red en las pruebas de la imagen.
- La aplicación completa probada con el cliente de pruebas de FastAPI (`TestClient`), incluido el procesamiento en segundo plano.
- Casos de fallo **provocados**: una restricción que hace fallar la escritura (para probar el rollback), la base apagada, una
  firma inválida, un archivo `.sql` que falta.
- La revisión estática de la consola (textos fijos) y de que ningún `.py` lleve SQL.
- La imagen de Docker probada **como producción**: sin la bandera de desarrollo, con las variables obligatorias y con una base
  temporal.
- Si algo se verifica en el navegador, con un navegador automatizado.

Las pruebas se escriben para **fallar de verdad**: una comprobación que solo recorre una lista vacía no prueba nada, y un
registro que no captura nada no demuestra que no hay fugas.
