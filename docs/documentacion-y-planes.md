# Documentación y planes

## Idioma

- **Identificadores en inglés** (variables, funciones, clases, archivos, tablas y columnas).
- **Comentarios, docstrings, documentación y mensajes en español de México**, con tuteo ("puedes", "tienes", "dime").
  Sin voseo.

## Qué documenta cada archivo

| Archivo | Contenido |
|---|---|
| `README.md` | Qué es, cómo se corre y la configuración. **Solo lo que ya está implementado** |
| `docs/ARCHITECTURE.md` | Estructura, flujos, decisiones técnicas, seguridad. Solo lo implementado |
| `migrations/README.md` | Las convenciones de las migraciones y la lista de archivos actuales |
| `app/repositories/README.md` | Las reglas de los repositorios y cómo agregar una consulta |
| `docs/develop/` | Los planes de trabajo. **No se versiona** (va en `.gitignore`) |

README y ARCHITECTURE se actualizan **en la misma entrega** que el código que describen, y solo con lo que existe. Lo que aún
no existe va en el plan, no en el README.

## En el código

- **Cada controller y cada service** abre con un docstring de módulo que dice **de qué trata** y qué hace, con el flujo numerado
  si lo tiene.
- Si un service tiene dos momentos, el docstring los nombra y dice qué función corresponde a cada uno.
- El **vocabulario del dominio** se define una sola vez, en el service que despacha, y los demás remiten a él.
- Cada campo de un modelo del contrato lleva un comentario de qué es y de dónde viene.
- Cada archivo `.sql` abre con un encabezado: qué hace, qué devuelve y qué es cada `$n`.

## Planes de trabajo

Los planes viven en `docs/develop/<tema>/`:

```
docs/develop/<tema>/
├── plan.md                              # el plan general
└── fases/
    ├── fase-1-<descripción>.md
    └── fase-2-<descripción>.md
```

### `plan.md`

1. **Objetivo y alcance**: qué incluye y qué **no** incluye.
2. **Restricciones** que condicionan el diseño.
3. **Decisiones**: una tabla con número, decisión, **estado** (`Confirmada` o `Pendiente`) y valor propuesto.
4. El flujo, el esquema y la estructura de código.
5. **Limitaciones** conocidas.
6. **Fases**: una tabla con el documento de cada una y cuándo "se considera lista".

### Cada fase

- **Objetivo** y las decisiones que usa.
- **Qué se hace**: una tabla de archivos con su contenido (nombres de archivo **explícitos**).
- **Reglas**.
- **Pruebas y criterio de "lista"**: una lista de comprobaciones concretas.
- **Fuera de esta fase**.

### Reglas de los planes

- **Un plan describe solo el estado actual**: sin "antes" y "ahora", sin historial. Si cambia el rumbo, se reescribe.
- Los planes **no se versionan**: son andamio de trabajo.
- Cada fase es pequeña y deja **algo que se puede comprobar por sí solo**.

## Flujo de trabajo

1. **Plan primero.** Para algo grande se escribe el plan y se espera la aprobación antes de construir. Se pregunta solo lo que
   es del desarrollador decidir, con una recomendación por delante.
2. **Cortes verticales.** Se construye por pedazos que se ven funcionando: primero una maqueta con elementos provisionales y,
   después, cada elemento que necesita datos reales se resuelve con su consulta en el backend.
3. **Cada fase se prueba antes de darla por buena** (ver [entorno local y scripts](entorno-local-y-scripts.md)) y se reporta
   con honestidad: si una prueba falla o se omitió, se dice.
4. **Se documenta lo implementado** en la misma entrega.
5. **Los commits los hace el desarrollador.** Lo destructivo o difícil de revertir se confirma antes.

## Estilo de lo escrito

- Frases directas y concretas; cada decisión con su razón.
- Tablas para comparar y listas para pasos; código solo cuando aclara.
- Mensajes de la app: textos fijos, que digan qué pasó y qué hacer, sin datos de usuarios.
