# Patrones de procesamiento

Patrones para servicios que reciben eventos de un tercero (webhooks) y reaccionan a ellos.

## Vocabulario

Se define en **un solo lugar** (el docstring del service que despacha) y se usa igual en todos los demás.

| Término | Significado |
|---|---|
| **Evento** | Una petición completa del tercero (el sobre de la entrega). Puede traer varios mensajes. No tiene identificador propio ni se guarda como tal |
| **Mensaje** | El contenido: algo que un usuario envió. Tiene su identificador único, que genera el tercero |
| **Status** | Un aviso sobre algo que nosotros enviamos (entregado, leído, fallido). No es un mensaje: solo cambia el estado de uno ya guardado |
| **Registrar** | Guardar en la base lo que llegó |
| **Procesar** | Actuar sobre lo recibido |

Evento y mensaje no son lo mismo: el tercero **reintenta eventos** (reenvía la petición completa) y por eso los duplicados se
evitan **por mensaje**, con su identificador.

## Registrar antes de responder, procesar después

Los dos momentos, en este orden:

1. **Registrar**, **antes** de responder `200`. Solo escrituras rápidas y en **una sola transacción**: o se guarda todo el
   evento o nada. Es lo que hace que un evento "cuente como recibido".
2. **Procesar**, **después** del `200`, en segundo plano (`BackgroundTasks`). Puede tardar (esperas, llamadas a un tercero) y el
   tercero no debe esperar. Si algo falla aquí, ya no cambia la respuesta que recibió el tercero: el evento sigue registrado.

**Si no se logra registrar, se responde `500` y el evento se considera no recibido:** el tercero reintenta (con frecuencia
decreciente, durante días), y como los duplicados se ignoran, reintentar es seguro. Un reintento no cuesta nada del lado del
tercero; solo es una petición más para el servicio.

Cada service de un tipo expone una función por momento:

| Función | Momento | Qué hace |
|---|---|---|
| `register(connection, ...)` | Antes del `200` | Guarda en la base (recibe la conexión de la transacción que abre el despachador) |
| `handle(...)` | Después del `200` | Deja constancia en el log y, cuando corresponda, actúa |

### Qué se guarda

- Un mensaje de un tipo con modelo propio: su conversación y el mensaje con su contenido.
- Un tipo **sin** modelo propio: el mensaje con su tipo y **sin contenido**, para que el historial no tenga huecos.
- Un status: avanza el estado del mensaje saliente al que se refiere; si ese mensaje no está en la base, se ignora.
- Un mensaje **sin datos para identificar la conversación**: no se guarda, se avisa con un texto fijo y se responde `200`.
  Reintentarlo no lo arreglaría, y con un `500` el tercero lo reintentaría durante días.

## Idempotencia

Se garantiza en la base, no en el código (ver [base de datos](base-de-datos.md)):

- Los mensajes se insertan con `ON CONFLICT DO NOTHING` sobre su identificador único; la función indica si insertó (`True`) o era
  duplicado (`False`).
- Una acción se registra solo para un mensaje **nuevo**, y la tabla no admite dos del mismo tipo para el mismo mensaje: un
  webhook repetido no genera una segunda acción.

## Acciones registradas (outbox)

Una **acción** es una tarea pendiente ante un mensaje (por ejemplo, responder). Se **registra** antes del `200`, en la misma
transacción que el mensaje, y se **ejecuta** después.

```
registrar (antes del 200)  →  pending
ejecutar  (después)        →  in_progress  →  done | failed
```

- **Tomar** una acción (`pending` → `in_progress`) es **atómico**: si dos procesos lo intentan, solo uno la obtiene.
- Cada tipo de acción vive en su carpeta de `actions/` con su archivo `<tipo>_action.py`: su constante `ACTION_TYPE`, su
  `build_payload` (lo mínimo que necesita para ejecutarse) y su `execute`. El contrato de la acción vive solo ahí.
- Un manejador común (`action_handler.py`) toma la acción, llama a la función de su tipo (`ACTION_HANDLERS`) y guarda el
  resultado **en una sola transacción**: el mensaje que salió, la fecha de la conversación y la acción como `done` con su
  identificador resultante.
- Si la función falla lanza `ActionFailed` con un **texto fijo** y sin datos de usuarios: la acción queda `failed` con ese
  resumen en `last_error` y no se guarda ningún mensaje saliente. El detalle va al registro de eventos.
- **No hay transacción abierta durante la espera ni las llamadas a un tercero:** cada paso usa una transacción corta.
- **No hay reintentos.** Una acción `failed`, o una que quedó en `pending` o `in_progress` porque el proceso murió, se queda así y
  se puede consultar en la base. Las pendientes viejas no se reejecutan solas.
- Una acción que falla no impide ejecutar las siguientes del mismo evento.
- Quitar un tipo de acción es borrar su carpeta y su línea en `ACTION_HANDLERS`.

Los modelos del resultado y de los errores van en su sitio: `actions/models/` para el modelo y `action_errors.py` para la
excepción.

## Segundo plano y Cloud Run

Por omisión, Cloud Run limita la CPU de una instancia **después de responder**. Lo que corre en segundo plano puede quedarse a
medias: para que termine hay que desplegar con la CPU siempre asignada (`--no-cpu-throttling`) o aceptar el riesgo. Una
instancia que se apaga a mitad de una acción deja la acción en `pending` o `in_progress`.

## Paginación por cursor

Las API de lectura paginan **por cursor** (*keyset*), no por número de página: el listado cambia mientras se consulta.

- Un orden estable con desempate (`last_message_at` y `conversation_key`).
- El cursor es **opaco**: lo codifica el servidor; uno inválido responde `400`, y un `limit` fuera de rango, `422`.
- Forma fija de la respuesta: `{ "items": [...], "next_cursor": "..." }`; `next_cursor` es `null` en la última página.
- Los datos derivados (por ejemplo, si una ventana de tiempo sigue abierta) los calcula el servidor.
