# Configuración y entornos

## Un solo `settings.py`

Toda la configuración vive en `config/settings.py`, y se separa por su origen:

- **Comportamiento** (debug, logs, interruptores de funciones, tiempos, límites): **por código**, se cambia en el archivo.
- **Secretos y valores de despliegue** (tokens, `DATABASE_URL`, `PORT`): **variables de entorno**. En local se cargan desde
  `.env` con `python-dotenv` (sin pisar las que ya estén definidas); en Cloud Run se definen en el servicio.
- `.env.example` documenta todas las variables, **sin valores**. `.env` está en `.gitignore`.
- No se usa un gestor de secretos: los secretos son variables de entorno del servicio.

## La bandera de desarrollo

`IS_DEVELOPMENT` distingue el equipo del desarrollador de producción:

```python
IS_DEVELOPMENT = os.getenv("IS_DEVELOPMENT", "").strip().lower() == "true"
```

- **Solo el valor `true` activa el modo desarrollo.** Si la variable falta o tiene cualquier otro valor, se asume **producción**.
- **En producción no se define.** Un servicio nunca debe quedar en modo desarrollo por omisión.
- Las funciones que muestran datos de usuarios exigen **dos cosas**: el interruptor por código **y** la bandera del entorno.

```python
DEBUG_ENABLED = True              # interruptor por código
DEBUG = DEBUG_ENABLED and IS_DEVELOPMENT   # valor efectivo: lo único que usa el resto del código
```

## Fallar cerrado

`config/startup_checks.py` valida la configuración al arrancar, antes de aceptar tráfico:

- **En producción**, si falta una variable obligatoria, la app **no arranca**. El error nombra las variables que faltan,
  **nunca sus valores**.
- **En desarrollo** solo avisa, con textos fijos, para no estorbar las pruebas locales.
- Una dependencia obligatoria (la base) que no responde también impide el arranque, con un texto fijo.
- La validación corre al importar `main.py`, antes de crear la aplicación.

Qué es obligatorio depende de las funciones activas: un token para llamar a un tercero solo se exige si la función que lo usa
está encendida.

## Elegir la base por entorno

La base la decide `IS_DEVELOPMENT`:

| Entorno | Qué usa |
|---|---|
| Desarrollo | La base local de Docker (definida por código), con `DATABASE_URL_DEVELOPMENT` opcional para apuntar a otra. **Ignora** `DATABASE_URL` |
| Producción | `DATABASE_URL`, obligatoria |

Ignorar `DATABASE_URL` en desarrollo significa que el código que corre en el equipo **no puede** apuntar a la base remota por
accidente. Por eso los datos de prueba y los comandos destructivos nunca la alcanzan.

## Lo que solo existe en desarrollo

- La documentación interactiva (`/docs`, `/redoc`, `/openapi.json`): apagada en producción.
- El registro de tráfico y el registro de detalle (ver [logs y datos personales](logs-y-datos-personales.md)).
- Los datos de prueba y su carga.
- Los servicios sin login (ver [seguridad](seguridad.md)).

## Variables habituales

| Variable | Para qué |
|---|---|
| `IS_DEVELOPMENT` | `true` solo en el equipo del desarrollador |
| `DATABASE_URL` | Obligatoria en producción: la URL de Postgres (con el socket de Cloud SQL en Cloud Run) |
| `DATABASE_URL_DEVELOPMENT` | Opcional, solo desarrollo: otra base local |
| `PORT` | Lo define Cloud Run; en local, un valor por omisión |
| Secretos del tercero | Tokens y secretos de firma, según el servicio |
