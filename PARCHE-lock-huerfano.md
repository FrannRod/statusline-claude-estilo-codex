# Parche: lock huérfano que congela el cache de uso

## Cuándo aplicarlo

Aplicá este parche si **instalaste la statusline antes del 2026-06-17**, es decir, si tu
`~/.claude/statusline-command.sh` **no** limpia el lock huérfano antes de tomarlo.

Forma rápida de saber si te afecta:

```bash
grep -q "lock huérfano" ~/.claude/statusline-command.sh \
  && echo "YA PARCHEADO" \
  || echo "FALTA EL PARCHE"
```

Si dice `FALTA EL PARCHE`, seguí leyendo. Si dice `YA PARCHEADO`, no tenés que hacer nada.

### Síntoma típico

La statusline muestra `5h XX% left` / `weekly XX% left` **clavados en un valor viejo** que
**no coincide** con lo que reporta el comando nativo `/usage`. El cache quedó congelado en
una foto vieja y nunca se actualiza, aunque pasen horas.

## Por qué pasa

El script refresca el uso en segundo plano y usa un directorio como candado atómico para
que no corran dos fetches a la vez:

```bash
mkdir "$LOCK" 2>/dev/null || return 0
```

El problema: el lock se libera (`rmdir`) **solo si el fetch termina bien**. Si el proceso
en background muere antes de liberarlo —corte de red, timeout del `curl`, la terminal
cerrada/matada, el equipo suspendido— el directorio `~/.claude/.usage-cache.lock` queda
para siempre. A partir de ahí, **cada** refresh siguiente choca con el `mkdir` que falla y
hace `return 0` sin hacer nada: el cache nunca más se actualiza y la statusline muestra
eternamente la última foto buena.

El parche agrega una limpieza de lock huérfano: si el lock existe y tiene más de 30 s de
antigüedad (el `curl` tiene timeout de 10 s, así que 30 s solo puede significar que el
refresh anterior murió), se borra antes de intentar tomarlo. Así el sistema se
autorrepara en el siguiente render y nunca queda congelado de forma permanente.

## Cómo aplicarlo

### 1. Destrabá el lock y refrescá el cache ahora (recuperación inmediata)

```bash
rmdir ~/.claude/.usage-cache.lock 2>/dev/null
rm -f ~/.claude/.usage-cache.json   # fuerza un fetch limpio en el próximo render
```

### 2. Parchá el script para que no vuelva a pasar

Abrí `~/.claude/statusline-command.sh` y, dentro de la función `refresh_usage()`,
**reemplazá** esta línea:

```bash
refresh_usage() {
  mkdir "$LOCK" 2>/dev/null || return 0   # lock atómico contra fetches simultáneos
```

por este bloque:

```bash
refresh_usage() {
  # limpiar lock huérfano: si un refresh anterior murió sin liberarlo (corte de red,
  # terminal matada, timeout), el directorio queda para siempre y congela el cache.
  if [[ -d "$LOCK" ]]; then
    lock_age=$(( $(date +%s) - $(stat -c %Y "$LOCK" 2>/dev/null || echo 0) ))
    (( lock_age > 30 )) && rmdir "$LOCK" 2>/dev/null
  fi
  mkdir "$LOCK" 2>/dev/null || return 0   # lock atómico contra fetches simultáneos
```

#### Alternativa: parche automático (idempotente)

Si preferís no editar a mano, este comando inserta el bloque justo antes del `mkdir` del
lock, y solo si el parche todavía no está aplicado:

```bash
grep -q "lock huérfano" ~/.claude/statusline-command.sh || \
awk '/mkdir "\$LOCK" 2>\/dev\/null/ && !ins {
  print "  # limpiar lock huérfano: si un refresh anterior murió sin liberarlo (corte de red,"
  print "  # terminal matada, timeout), el directorio queda para siempre y congela el cache."
  print "  if [[ -d \"$LOCK\" ]]; then"
  print "    lock_age=$(( $(date +%s) - $(stat -c %Y \"$LOCK\" 2>/dev/null || echo 0) ))"
  print "    (( lock_age > 30 )) && rmdir \"$LOCK\" 2>/dev/null"
  print "  fi"
  ins=1
} { print }' ~/.claude/statusline-command.sh > ~/.claude/statusline-command.sh.tmp \
  && mv ~/.claude/statusline-command.sh.tmp ~/.claude/statusline-command.sh
```

> Si el comando no matchea (porque editaste el script o cambió el espaciado), aplicá el
> parche a mano con el bloque de arriba: es lo más seguro.

### 3. Verificá

```bash
# 1) El parche está presente
grep -q "lock huérfano" ~/.claude/statusline-command.sh && echo "parche OK"

# 2) El script no tiene errores de sintaxis
bash -n ~/.claude/statusline-command.sh && echo "sintaxis OK"

# 3) Render de prueba (debe imprimir la statusline con 5h / weekly actualizados)
echo '{"workspace":{"current_dir":"'"$HOME"'"},"model":{"id":"claude-opus-4-8[1m]","display_name":"Opus 4.8"}}' \
  | bash ~/.claude/statusline-command.sh; echo

# 4) Los valores ahora coinciden con /usage
jq -c '{five_hour:.five_hour.utilization, seven_day:.seven_day.utilization}' ~/.claude/.usage-cache.json
```

Recordá que `utilization` es el **% usado**; la statusline muestra `100 - utilization`
como `% left`. Comparalo contra lo que ves en `/usage` dentro de Claude Code.
