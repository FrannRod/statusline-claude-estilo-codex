# Statusline para Claude Code estilo Codex

## IMPORTANTE: primero ejecuta `/statusline` en Claude Code

Antes de instalar esta statusline, abri Claude Code y ejecuta el comando nativo:

```text
/statusline
```

Ese paso prepara la configuracion y las dependencias que Claude Code necesita para usar statuslines personalizadas. Despues de hacerlo, segui con esta guia.

Este repo contiene una guia para configurar una statusline de Claude Code con informacion parecida a la de Codex: modelo, esfuerzo, limites de uso, contexto disponible, directorio actual y rama de git.

El objetivo es tener una linea de estado util mientras trabajas, sin abrir comandos extra para saber cuanto margen queda.

## Que muestra

Un ejemplo de salida:

![Ejemplo de la statusline](./statusline.png?v=2)

```text
Ready · Opus 4.8 high · 2.6h 49% · 6.9d 78% · Context 96% left · ~ · main
```

Los dos primeros valores de uso muestran cuanto falta para el reset de cada ventana (la de 5 horas en horas, la semanal en dias, con 1 decimal) seguido del porcentaje que te queda. Asi ves el restart de un vistazo sin abrir `/usage`.

La statusline cambia colores segun el porcentaje restante: verde cuando hay margen, amarillo cuando conviene prestar atencion y rojo cuando queda poco.

## Que hace

- Lee el modelo y el directorio desde el JSON que Claude Code pasa al comando `statusLine`.
- Lee el nivel de esfuerzo desde `~/.claude/settings.json`.
- Consulta los limites de uso de 5 horas y semanal usando el token local de Claude Code.
- Calcula el contexto restante a partir del ultimo `usage` del transcript.
- Muestra la rama de git del proyecto actual.
- Cachea la consulta de uso para que la statusline no sea lenta ni consulte la API en cada render.

## Privacidad

La consulta de uso se hace contra la API de Anthropic usando tu propio token local de Claude Code. El resultado se guarda en cache en tu maquina y no se envia a terceros.

## Requisitos

Necesitas Claude Code instalado y logueado. Tambien hacen falta herramientas de shell comunes: `jq`, `curl`, `bc` y `git`.

## Como usarlo

Si queres que una IA lo instale o adapte por vos, pasale este archivo:

[statusline-claude-estilo-codex.md](./statusline-claude-estilo-codex.md)

Ese documento esta escrito como una receta operativa para agentes: explica de donde sale cada dato, crea el script, registra la statusline y deja pruebas para validarla sin abrir Claude Code.

Si preferis hacerlo manualmente, podes seguir el mismo Markdown paso a paso.

## Parches

Para instalaciones hechas antes de la fecha del parche. Las instalaciones nuevas ya vienen con todo aplicado.

| Fecha | Archivo |
|-------|---------|
| 2026-06-17 | [PARCHE-lock-huerfano.md](./PARCHE-lock-huerfano.md) |
| 2026-06-18 | [PARCHE-rate-limit-backoff.md](./PARCHE-rate-limit-backoff.md) |
| 2026-07-07 | [PARCHE-tiempo-hasta-reset.md](./PARCHE-tiempo-hasta-reset.md) |

## Contenido

- `README.md`: explicacion humana del proyecto.
- `statusline-claude-estilo-codex.md`: guia tecnica detallada, pensada para que una IA pueda ejecutar o adaptar la instalacion.

## Ver también

- [notificaciones-claude-codex](https://github.com/FrannRod/notificaciones-claude-codex): avisos de escritorio cuando Claude Code o Codex terminan un turno, piden permiso o esperan input.
