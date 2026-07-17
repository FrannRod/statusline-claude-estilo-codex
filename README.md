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
- Si la API limita la consulta de uso, respeta el tiempo de espera que devuelve el server y no reintenta antes; asi evita que el limite escale.
- No muestra datos de uso de mas de 15 minutos; prefiere no mostrar nada antes que un numero viejo.
- Permite apagar por completo la consulta de uso con un interruptor en el script.
- Registra su actividad en un log local, seguro para varias sesiones a la vez: se manda un solo request aunque tengas muchas terminales abiertas.

## Privacidad

La consulta de uso se hace contra la API de Anthropic usando tu propio token local de Claude Code. El resultado se guarda en cache en tu maquina y no se envia a terceros.

## Requisitos

Necesitas Claude Code instalado y logueado. Tambien hacen falta herramientas de shell comunes: `jq`, `curl`, `bc` y `git`.

## Como usarlo

Si queres que una IA lo instale o adapte por vos, pasale este archivo:

[statusline-claude-estilo-codex.md](./statusline-claude-estilo-codex.md)

Ese documento esta escrito como una receta operativa para agentes: explica de donde sale cada dato, crea el script, registra la statusline y deja pruebas para validarla sin abrir Claude Code.

Si preferis hacerlo manualmente, podes seguir el mismo Markdown paso a paso.

## Actualizar una instalacion existente

No hay parches sueltos que aplicar a mano. La guia [statusline-claude-estilo-codex.md](./statusline-claude-estilo-codex.md) siempre tiene la ultima version del script, y al final trae un changelog que explica en una o dos oraciones el porque de cada cambio.

Para poner al dia una instalacion vieja, pasale a una IA tu `~/.claude/statusline-command.sh` actual junto con esa guia y pedile que lo actualice: va a comparar tu script con el ultimo, aplicar las diferencias y preservar tus personalizaciones, usando el changelog para entender la intencion de cada cambio.

## Contenido

- `README.md`: explicacion humana del proyecto.
- `statusline-claude-estilo-codex.md`: guia tecnica detallada, pensada para que una IA pueda ejecutar o adaptar la instalacion.

## Ver también

- [notificaciones-claude-codex](https://github.com/FrannRod/notificaciones-claude-codex): avisos de escritorio cuando Claude Code o Codex terminan un turno, piden permiso o esperan input.
