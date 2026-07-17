# Parche: rate-limit real (respetar retry-after), datos frescos y logging

## Cuándo aplicarlo

Aplicá este parche si tu `~/.claude/statusline-command.sh` maneja el rate-limit con
**backoff exponencial adivinado** (el enfoque de [PARCHE-rate-limit-backoff.md](./PARCHE-rate-limit-backoff.md),
archivos `.usage-backoff` / `.usage-ratelimited`) en vez de respetar el `retry-after`
que manda el server.

Forma rápida de saber si te afecta:

```bash
grep -q "usage-retry-at" ~/.claude/statusline-command.sh \
  && echo "YA PARCHEADO" \
  || echo "FALTA EL PARCHE"
```

Si dice `FALTA EL PARCHE`, seguí leyendo. Si dice `YA PARCHEADO`, no tenés que hacer nada.

## Qué cambia

1. **Respeta el `retry-after` exacto del server.** Ante un `rate_limit_error`, el script
   lee el header `retry-after` de la respuesta, guarda ese deadline en
   `~/.claude/.usage-retry-at` y **no vuelve a consultar hasta que pase**. Reemplaza al
   backoff exponencial que adivinaba el intervalo (`.usage-backoff` / `.usage-ratelimited`
   quedan obsoletos).
2. **Datos viejos no se muestran.** Si el último dato bueno tiene más de 15 min (`STALE`),
   la statusline **no** muestra los porcentajes; queda sólo la nota de rate-limit. Un
   número viejo no sirve para decidir nada.
3. **Formato de reset robusto.** `fmt_left` auto-escala a días/horas/minutos y ya no
   muestra el falso `0.0h` cuando la ventana no tiene `resets_at` (bug de `date -d ""`,
   que en GNU devuelve "ahora" en vez de fallar).
4. **Kill-switch `AUTO_FETCH`.** Con `AUTO_FETCH=0` la statusline deja de consultar el
   endpoint por completo (útil para dejar drenar un castigo o desactivar el uso).
5. **Log multi-sesión.** Registra fetch, resultado, rate-limit y contención de lock entre
   sesiones en `~/.claude/.usage-statusline.log` (append atómico, etiqueta de sesión,
   latido throttleado).

## Por qué

El endpoint `GET /api/oauth/usage` **tolera el polling normal** (cada 1-2 min anda sin
drama), pero tiene un rate-limit por token que **escala el castigo si lo martillás**:
medido con una sonda abusiva, la espera arrancó en ~14 min y escaló hasta **1 hora**;
reintentar *dentro* del ban no lo acorta y puede alargarlo. El backoff adivinado reintentaba
a los 15 min aunque el server pidiera esperar más, así que caía dentro del ban y lo
realimentaba. Respetar el `retry-after` exacto corta ese lazo: consultamos, y si nos
limitan, esperamos lo que el server diga y ni un segundo menos. Sale solo tras un rato de
silencio.

## Cómo aplicarlo

Los cambios tocan casi todo el cuerpo del script (variables, `refresh_usage`, la decisión
de refrescar, el formato y la nota de rate-limit), así que lo más simple y seguro es
**reemplazar el archivo entero** por el script de la guía
[statusline-claude-estilo-codex.md](./statusline-claude-estilo-codex.md), que ya viene con
todo aplicado.

### 1. Limpiá el estado obsoleto del backoff

```bash
rm -f ~/.claude/.usage-backoff ~/.claude/.usage-ratelimited
```

### 2. Reemplazá el script

Copiá el bloque `bash` completo de la guía sobre `~/.claude/statusline-command.sh` y dale
permisos de ejecución:

```bash
chmod +x ~/.claude/statusline-command.sh
```

### 3. Verificá

```bash
# 1) El parche está presente
grep -q "usage-retry-at" ~/.claude/statusline-command.sh && echo "parche OK"

# 2) El script no tiene errores de sintaxis
bash -n ~/.claude/statusline-command.sh && echo "sintaxis OK"

# 3) Render de prueba
echo '{"session_id":"test","workspace":{"current_dir":"'"$HOME"'"},"model":{"id":"claude-opus-4-8[1m]","display_name":"Opus 4.8"}}' \
  | bash ~/.claude/statusline-command.sh; echo

# 4) El log empieza a registrar actividad
tail -n 3 ~/.claude/.usage-statusline.log
```

Si el endpoint te está limitando en este momento vas a ver `5h rate-limited (retry Xm)`
con el tiempo que pidió el server; es lo esperado y se recupera solo. Si querés cortar
todo tráfico mientras drena, poné `AUTO_FETCH=0` en el script.
