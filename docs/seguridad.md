# Seguridad

Criterios para un servicio que recibe webhooks de un tercero y lee o escribe datos de personas.

## Principio: fallar cerrado

Ante la duda, la app **no arranca** o **rechaza**, nunca acepta de más. Ver [configuración y entornos](configuracion-y-entornos.md).

- En producción, si falta una variable obligatoria o un secreto, el servicio no inicia y el error nombra la que falta (no su valor).
- Sin la bandera `IS_DEVELOPMENT=true` se asume producción.

## Webhooks de un tercero

- **Firma:** HMAC-SHA256 sobre el **cuerpo crudo** de la petición, comparado con `hmac.compare_digest` (tiempo constante).
  Una firma inexistente o incorrecta responde `401` **sin tocar la base**. En producción sin el secreto de firma el servicio no
  arranca; en desarrollo solo avisa.
- **Límite de tamaño:** un tope para el cuerpo (por ejemplo, 2 MB) con `413`. Se rechaza sin leer si el tamaño **declarado**
  ya lo excede y, como el declarado puede mentir, **también se cuenta lo leído** mientras se recibe.
- **Verificación inicial:** el token se compara de forma estricta y, si no coincide, responde `403`.
- **Respuesta rápida:** el `200` se responde tras guardar lo recibido; lo que puede tardar corre después (ver
  [patrones de procesamiento](patrones-de-procesamiento.md)).
- **Un token en la URL** queda en cualquier registro de la URL: ver [logs y datos personales](logs-y-datos-personales.md).
- **Pendiente:** protección contra reenvíos de una petición válida (*replay*), y fijar las versiones de las dependencias.

## Llamadas salientes

- **Se validan los identificadores** (números, ids) con una expresión regular **antes** de ponerlos en una URL o un cuerpo.
- La autenticación se enmascara en los registros.
- El cliente registra los fallos y devuelve `None`; no propaga excepciones con detalles.

## Respuestas y superficie expuesta

- **Textos fijos** en los errores: nada de detalles internos ni trazas.
- La documentación interactiva (`/docs`, `/redoc`, `/openapi.json`) **solo en desarrollo**.
- **CORS restrictivo:** solo los orígenes necesarios, solo los métodos necesarios y sin cabeceras de más.
- Un servicio de **solo lectura** acepta únicamente `GET`: el resto responde `405`.

## Servicios sin login

Un servicio que sirve datos personales **sin autenticación** solo puede correr en el equipo del desarrollador:

- **Se niega a arrancar** si `IS_DEVELOPMENT` no es `true` (con un mensaje que explica por qué).
- **Escucha solo en `127.0.0.1`.**
- Solo acepta peticiones de los orígenes locales que lo consumen (CORS).
- Cuando exista autenticación, esas restricciones se levantan junto con el login.

## Base de datos

- **Solo lectura a nivel de la base** para los servicios que no escriben (ver [base de datos](base-de-datos.md)).
- Un usuario propio de la aplicación, no el administrador. A futuro, separar el que migra (con permisos de esquema) del que
  usa el servicio (solo datos) y, para consultas, uno de solo lectura.
- **En desarrollo nunca se llega a la base remota:** se ignora `DATABASE_URL` y los datos de prueba se rechazan fuera de
  desarrollo.
- Conexiones con SSL obligatorio y sin redes autorizadas abiertas; en Cloud Run se usa el socket de Cloud SQL.
- La contraseña nunca aparece en un registro ni en un mensaje de error.

## Contenedor

- Se ejecuta como un usuario **sin privilegios**.
- La imagen **no contiene** `.env`, `seeds/`, `docker-compose.yml` ni documentación.
- Solo se instalan las dependencias necesarias.

## Datos personales

- Nombres y teléfonos solo se muestran a quien tiene acceso y **nunca** se escriben en la consola.
- No se guardan payloads completos; se guarda lo mínimo.
- Antes de producción hay que definir **cuánto tiempo se conservan** los datos de contacto.
