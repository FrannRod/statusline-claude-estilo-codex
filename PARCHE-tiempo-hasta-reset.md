# Parche: tiempo hasta el reset en las etiquetas de uso

## Cuándo aplicarlo

Aplicá este parche si tu `~/.claude/statusline-command.sh` muestra los límites con
etiquetas fijas (`5h XX% left` / `weekly XX% left`) y querés que en su lugar muestren
**cuánto falta para el reset** de cada ventana.

Forma rápida de saber si te afecta:

```bash
grep -q "fmt_left" ~/.claude/statusline-command.sh \
  && echo "YA PARCHEADO" \
  || echo "FALTA EL PARCHE"
```

Si dice `FALTA EL PARCHE`, seguí leyendo. Si dice `YA PARCHEADO`, no tenés que hacer nada.

### Qué cambia

Antes:

```text
Ready · Opus 4.8 high · 5h 49% left · weekly 78% left · Context 96% left · ~ · main
```

Después:

```text
Ready · Opus 4.8 high · 2.6h 49% · 6.9d 78% · Context 96% left · ~ · main
```

El primer par ahora dice **cuánto falta para el reset** de cada ventana (la de 5h en
horas, la semanal en días, con 1 decimal) seguido del **% que te queda**. Así ves el
restart de un vistazo sin tener que hacer la cuenta.

## Por qué se puede hacer sin tocar el fetch

La respuesta del endpoint `GET /api/oauth/usage` ya trae, para cada ventana, un campo
`resets_at` (ISO-8601) además de `utilization`:

```json
{
  "five_hour": { "utilization": 52, "resets_at": "2026-07-07T22:09:59+00:00" },
  "seven_day": { "utilization": 4,  "resets_at": "2026-07-14T17:59:59+00:00" }
}
```

Como el script cachea la respuesta entera, esos timestamps ya están en
`~/.claude/.usage-cache.json`. El parche solo cambia la parte que **arma** la línea:
calcula `resets_at − ahora` y lo formatea en horas o días. No toca la consulta ni el cache.

## Cómo aplicarlo

### 1. Parchá el script

Abrí `~/.claude/statusline-command.sh` y hacé este único reemplazo.

**Buscá** el bloque de rate limits:

```bash
# ---------- rate limits desde el cache ----------
rl_str=""
if [[ -f "$CACHE" ]]; then
  fh=$(jq -r '.five_hour.utilization // empty' "$CACHE" 2>/dev/null)
  sd=$(jq -r '.seven_day.utilization // empty' "$CACHE" 2>/dev/null)
  if [[ -n "$fh" ]]; then
    fl=$(printf '%.0f' "$(echo "100 - $fh" | bc -l 2>/dev/null || echo "$((100 - ${fh%.*}))")")
    rl_str+="${SEP}$(pct_color "$fl")5h ${fl}% left${RST}"
  fi
  if [[ -n "$sd" ]]; then
    sl=$(printf '%.0f' "$(echo "100 - $sd" | bc -l 2>/dev/null || echo "$((100 - ${sd%.*}))")")
    rl_str+="${SEP}$(pct_color "$sl")weekly ${sl}% left${RST}"
  fi
fi
```

**Reemplazá por:**

```bash
# ---------- rate limits desde el cache ----------
# En vez de etiquetas fijas ("5h" / "weekly") mostramos cuánto FALTA para el reset
# (campo resets_at del endpoint): la ventana de 5h en horas y la semanal en días,
# ambas con 1 decimal, seguido del % restante. Ej: "3.0h 50%  6.5d 96%".
rl_str=""
if [[ -f "$CACHE" ]]; then
  fh=$(jq -r '.five_hour.utilization // empty'  "$CACHE" 2>/dev/null)
  fh_rst=$(jq -r '.five_hour.resets_at // empty' "$CACHE" 2>/dev/null)
  sd=$(jq -r '.seven_day.utilization // empty'  "$CACHE" 2>/dev/null)
  sd_rst=$(jq -r '.seven_day.resets_at // empty' "$CACHE" 2>/dev/null)

  # tiempo restante hasta un reset ISO-8601 → "<n><suf>" con 1 decimal (nunca negativo).
  # LC_ALL=C en el printf: bc emite el decimal con punto y un locale con coma lo rechaza.
  #   $1 = ISO timestamp   $2 = divisor (3600=h, 86400=d)   $3 = sufijo   $4 = fallback
  fmt_left() {
    local rst; rst=$(date -d "$1" +%s 2>/dev/null) || { printf '%s' "$4"; return; }
    local n; n=$(LC_ALL=C printf '%.1f' "$(echo "($rst - $now) / $2" | bc -l 2>/dev/null)")
    [[ "$n" == -* ]] && n="0.0"
    printf '%s%s' "$n" "$3"
  }

  if [[ -n "$fh" ]]; then
    fl=$(printf '%.0f' "$(echo "100 - $fh" | bc -l 2>/dev/null || echo "$((100 - ${fh%.*}))")")
    rl_str+="${SEP}$(pct_color "$fl")$(fmt_left "$fh_rst" 3600 h 5h) ${fl}%${RST}"
  fi
  if [[ -n "$sd" ]]; then
    sl=$(printf '%.0f' "$(echo "100 - $sd" | bc -l 2>/dev/null || echo "$((100 - ${sd%.*}))")")
    rl_str+="${SEP}$(pct_color "$sl")$(fmt_left "$sd_rst" 86400 d weekly) ${sl}%${RST}"
  fi
fi
```

Qué hace el reemplazo:

1. **Lee también `resets_at`** de cada ventana (además de `utilization`).
2. **`fmt_left`** convierte ese timestamp en cuánto falta, en la unidad que le pases
   (`3600` → horas, `86400` → días), con 1 decimal. Si el dato quedó viejo y el reset ya
   pasó, muestra `0.0` en vez de un negativo; si `resets_at` faltara, cae al label fijo
   (`5h` / `weekly`) que le pasás como cuarto argumento.
3. **`LC_ALL=C`** en ese `printf`: `bc` emite el decimal con punto y un locale con coma
   decimal (p. ej. `es_AR`) lo rechazaría con "número no válido".

> El bloque de la nota de rate-limit que va justo después (`if [[ -f "$RLMARK" ]]; then …`)
> no cambia: su fallback sigue diciendo `5h rate-limited (retry Xm)`, porque en ese caso
> no hay un tiempo de reset confiable para mostrar.

> Si preferís no editar a mano, lo más simple y seguro es reemplazar el archivo entero
> por el script de la guía [statusline-claude-estilo-codex.md](./statusline-claude-estilo-codex.md),
> que ya viene con todo aplicado.

### 2. Verificá

```bash
# 1) El parche está presente
grep -q "fmt_left" ~/.claude/statusline-command.sh && echo "parche OK"

# 2) El script no tiene errores de sintaxis
bash -n ~/.claude/statusline-command.sh && echo "sintaxis OK"

# 3) Render de prueba: los límites deben verse como "N.Nh XX%" y "N.Nd XX%"
echo '{"workspace":{"current_dir":"'"$HOME"'"},"model":{"id":"claude-opus-4-8[1m]","display_name":"Opus 4.8"}}' \
  | bash ~/.claude/statusline-command.sh; echo
```

Si el endpoint te está limitando en este momento vas a ver `5h rate-limited (retry Xm)`
en vez de los porcentajes: es lo esperado y no depende de este parche.
