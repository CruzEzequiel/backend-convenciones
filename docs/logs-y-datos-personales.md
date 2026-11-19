# Logs y datos personales

En producción no debe quedar **ningún dato de usuarios** en los registros. En Cloud Run, lo que se imprime en la consola va a
Cloud Logging, así que la consola es un canal público para el equipo.

## Dos canales

| Canal | Qué lleva | Dónde existe |
|---|---|---|
| **Consola** (`get_logger`) | Solo textos **fijos**, sin ningún valor variable | Desarrollo y producción |
| **Registro de detalle** (`record_event`) | Todo el detalle: textos, nombres, teléfonos, identificadores, cuerpos de error | **Solo desarrollo** |

### Consola: textos fijos

Cada llamada a `logger.info`, `warning` o `error` lleva **un único texto literal**, sin f-strings, sin `%s` y sin variables:

```python
logger.info("Mensaje de texto recibido")          # bien
logger.info(f"Mensaje de {name}: {text}")         # mal: lleva datos
```

Si un dato importa para diagnosticar, **va a `record_event`**, no al mensaje de consola. Los textos dicen qué pasó, no con quién.

### Registro de detalle

`utils/event_recorder.py` expone `record_event(evento, datos)`, que agrega una línea JSON a `logs/event_record.jsonl`:

```python
record_event("meta_call_rejected", {"status": 400, "body": response_body})
```

- Solo escribe si el registro a archivo está activo (que exige el interruptor por código **y** `IS_DEVELOPMENT`). En producción
  no hace nada y no deja rastro.
- **Nunca interrumpe el flujo:** si no puede escribir, lo avisa con un texto fijo y sigue.

### Revisión automática

La regla de los textos fijos se puede comprobar de forma estática, recorriendo el árbol sintáctico del código:

```python
for node in ast.walk(tree):
    if isinstance(node, ast.Call) and isinstance(node.func, ast.Attribute) \
            and node.func.attr in ("debug", "info", "warning", "error", "exception", "critical") \
            and isinstance(node.func.value, ast.Name) and node.func.value.id == "logger":
        fijo = len(node.args) == 1 and not node.keywords and isinstance(node.args[0], ast.Constant)
```

La misma revisión prohíbe `print` en el código de la app. Los **comandos del operador** (el ejecutor de migraciones, la
inicialización) sí imprimen con `print`, porque solo muestran versiones y descripciones, nunca datos de usuarios.

## Qué nunca se escribe

- **Secretos y tokens:** la autenticación se enmascara en cualquier registro (`Authorization: Bearer ***`).
- **La contraseña de la base:** se quita de cualquier mensaje de error antes de registrarlo, y los errores de conexión se
  relanzan con un texto fijo (`from None`).
- **Los payloads completos de un tercero:** no se guardan en la base. Se guarda lo mínimo del modelo (el mensaje, no el sobre).
- **Los errores de un tercero con datos personales:** a la base va un resumen (`código: título`); el cuerpo completo, solo al
  registro de detalle.
- **Las acciones registradas** llevan en su `payload` únicamente lo necesario para ejecutarse.

## Registros de tráfico

Un middleware puede registrar la petición y la respuesta completas, con un identificador común. Solo actúa si `DEBUG` está
activo, es decir, **nunca en producción**.

## Cuando un dato viaja en la URL

Algunos terceros mandan un valor en la URL (por ejemplo, el token de verificación de un webhook, en `?hub.verify_token=`). Cualquier
registro de la URL lo contiene. Hay tres capas:

1. Arrancar `uvicorn` con `--no-access-log`, para que la aplicación no imprima la URL de cada petición.
2. Excluir de Cloud Logging el registro de solicitudes de Cloud Run que la contiene, con un filtro como
   `httpRequest.requestUrl:"hub.verify_token"` en el receptor `_Default` (lo excluido no se almacena).
3. Si el valor ya se expuso, cambiarlo **en los dos lados a la vez** (el servicio y el tercero) y verificar de nuevo.

## Pendiente conocido

Con la consola en textos fijos, en producción no se puede diagnosticar **por qué** falló algo (por ejemplo, el código de error de
un tercero). Una tabla interna de errores, con códigos y contexto técnico sin datos personales, cubriría ese hueco.
