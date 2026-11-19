# Backend Convenciones

Convención personal para organizar aplicaciones backend en Python con una arquitectura clara, consistente y escalable.

Este repositorio documenta una forma de estructurar servicios FastAPI con PostgreSQL, separar responsabilidades y mantener
criterios repetibles entre aplicaciones. Nació de un webhook de WhatsApp (recibir, guardar y responder mensajes) y de su API
de lectura, y por eso varios ejemplos vienen de ahí; las reglas aplican a cualquier servicio.

## Contenido

- Una arquitectura base para servicios FastAPI: carpeta por endpoint y capas controller → service → repository.
- Modelos del contrato y del dominio, tipados de punta a punta.
- SQL en archivos, migraciones versionadas y datos de prueba.
- Configuración por entorno, fallar cerrado y banderas de desarrollo.
- Logs sin datos de usuarios y seguridad de los webhooks.
- Patrones de procesamiento: registrar antes de responder, ejecutar después e idempotencia.
- Scripts de entorno local y despliegue en Cloud Run con Cloud SQL.
- Cómo documentar y planear.

## Documentación

[Arquitectura de apps FastAPI](docs/architecture.md): estructura base, capas, router, nombres y cómo agregar un endpoint.

[Modelos y tipado](docs/modelos-y-tipado.md): modelos del contrato frente a modelos del dominio, mappers y respuestas tipadas.

[Base de datos](docs/base-de-datos.md): `asyncpg`, pool, transacciones, repositorios con el SQL en archivos y patrones de consistencia.

[Migraciones y datos de prueba](docs/migraciones-y-datos-de-prueba.md): migraciones SQL versionadas, ejecutor, datos de prueba e inicialización.

[Configuración y entornos](docs/configuracion-y-entornos.md): `settings.py`, la bandera de desarrollo, validaciones al arrancar y selección de la base.

[Logs y datos personales](docs/logs-y-datos-personales.md): consola con textos fijos, registro de detalle solo en desarrollo y qué nunca se escribe.

[Seguridad](docs/seguridad.md): firma de webhooks, límites, validación de identificadores y servicios sin login.

[Patrones de procesamiento](docs/patrones-de-procesamiento.md): registrar antes del 200, acciones registradas (outbox), idempotencia y paginación por cursor.

[Entorno local y scripts](docs/entorno-local-y-scripts.md): Docker Compose, un solo script que deja todo listo y verificación por fase.

[Despliegue](docs/despliegue.md): imagen de Docker, Cloud Run, Cloud SQL y migraciones con un Job.

[Documentación y planes](docs/documentacion-y-planes.md): idioma, qué documenta cada archivo, planes por fases y flujo de trabajo.

## Principios

- Separar responsabilidades antes de que el proyecto crezca.
- Cada cosa en un solo lugar: HTTP en los controllers, la lógica en los services y el SQL en archivos.
- Tipar todo: se debe saber exactamente qué se espera de cada respuesta.
- Fallar cerrado: si falta algo, es mejor no arrancar que arrancar inseguro.
- La consola dice textos fijos; el detalle va a un archivo, solo en desarrollo.
- Planear primero, construir por cortes pequeños y documentar solo lo que ya existe.
- Preferir convenciones simples sobre abstracciones innecesarias.

## Licencia

Este proyecto está licenciado bajo [Creative Commons Attribution 4.0 International](LICENSE.md).
