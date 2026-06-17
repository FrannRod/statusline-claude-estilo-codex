# Statusline para Claude Code estilo Codex

Guía autosuficiente para replicar en otra computadora una statusline de Claude Code
que muestra modelo, esfuerzo, **límites de uso 5h / semanal**, contexto restante,
directorio y rama de git:

```
Ready · Opus 4.8 high · 5h 49% left · weekly 78% left · Context 96% left · ~ · main
```

Los porcentajes cambian de color: verde (≥50%), amarillo (≥20%), rojo (<20%).

---

## 1. Requisitos

Necesitás estas herramientas en el `PATH` (en Ubuntu/Debian: `sudo apt install jq curl bc`):

- `jq` — parseo de JSON
- `curl` — consulta al endpoint de uso
- `bc` — aritmética de porcentajes (hay fallback si no está)
- Claude Code ya instalado y **logueado** (debe existir `~/.claude/.credentials.json`)

---

## 2. De dónde salen los datos

| Dato | Origen |
|------|--------|
| Modelo y directorio | JSON que Claude Code pasa por stdin al statusLine |
| Esfuerzo (`high`, etc.) | `~/.claude/settings.json` → `.effortLevel` |
| **5h / weekly % left** | `GET https://api.anthropic.com/api/oauth/usage` con el token OAuth local |
| Context % left | último `usage` del transcript de la sesión (input + cache), sobre la ventana del modelo |
| Rama git | `git rev-parse` sobre el directorio actual |

### Sobre el endpoint de uso

Los límites de 5h y semanal **no** vienen en el JSON del statusLine. Se obtienen del
mismo endpoint que usa el comando `/usage`:

```
GET https://api.anthropic.com/api/oauth/usage
Authorization: Bearer <accessToken>
anthropic-beta: oauth-2025-04-20
anthropic-version: 2023-06-01
```

El `accessToken` se lee de `~/.claude/.credentials.json` → `.claudeAiOauth.accessToken`.
La respuesta es del tipo:

```json
{
  "five_hour":  { "utilization": 49.0, "resets_at": "..." },
  "seven_day":  { "utilization": 21.0, "resets_at": "..." }
}
```

`utilization` es el **% usado**; el "% left" mostrado es `100 - utilization`.

> Es tu propio token consultando tu propio uso contra la API de Anthropic. No sale nada
> a terceros. El resultado se cachea 60 s y se refresca en segundo plano para que el
> render nunca se bloquee ni sature la API.

> **Ventana de contexto:** el script asume 1M de tokens si el modelo contiene `1m`
> (p. ej. `opus[1m]`), si no 200k. Ajustá la variable `window` si usás otra config.

---

## 3. Crear el script

Guardá esto como `~/.claude/statusline-command.sh`:

```bash
#!/bin/bash
# Statusline estilo Codex para Claude Code.
#   Ready · <Modelo> <esfuerzo> · 5h XX% left · weekly XX% left · Context XX% left · <dir> · <rama>

input=$(cat)

CRED="$HOME/.claude/.credentials.json"
CACHE="$HOME/.claude/.usage-cache.json"
LOCK="$HOME/.claude/.usage-cache.lock"
TTL=60   # segundos de validez del cache

# ---------- helper: refrescar el cache de uso en segundo plano ----------
refresh_usage() {
  mkdir "$LOCK" 2>/dev/null || return 0   # lock atómico contra fetches simultáneos
  (
    tok=$(jq -r '.claudeAiOauth.accessToken // empty' "$CRED" 2>/dev/null)
    if [[ -n "$tok" ]]; then
      curl -s -m 10 \
        -H "Authorization: Bearer $tok" \
        -H "anthropic-beta: oauth-2025-04-20" \
        -H "anthropic-version: 2023-06-01" \
        "https://api.anthropic.com/api/oauth/usage" -o "$CACHE.tmp" \
        && jq -e . "$CACHE.tmp" >/dev/null 2>&1 \
        && mv "$CACHE.tmp" "$CACHE"
      rm -f "$CACHE.tmp"
    fi
    rmdir "$LOCK" 2>/dev/null
  ) >/dev/null 2>&1 &
  disown 2>/dev/null
}

# ---------- decidir si hay que refrescar ----------
now=$(date +%s)
if [[ -f "$CACHE" ]]; then
  mtime=$(stat -c %Y "$CACHE" 2>/dev/null || echo 0)
  (( now - mtime > TTL )) && refresh_usage
else
  refresh_usage
  tok=$(jq -r '.claudeAiOauth.accessToken // empty' "$CRED" 2>/dev/null)
  [[ -n "$tok" ]] && curl -s -m 3 \
    -H "Authorization: Bearer $tok" \
    -H "anthropic-beta: oauth-2025-04-20" \
    -H "anthropic-version: 2023-06-01" \
    "https://api.anthropic.com/api/oauth/usage" 2>/dev/null \
    | jq -e . >/dev/null 2>&1 && true
fi

# ---------- campos del JSON de entrada ----------
cwd=$(echo "$input"        | jq -r '.workspace.current_dir // .cwd // empty')
model=$(echo "$input"      | jq -r '.model.display_name // empty')
model_id=$(echo "$input"   | jq -r '.model.id // empty')
transcript=$(echo "$input" | jq -r '.transcript_path // empty')
effort=$(jq -r '.effortLevel // empty' "$HOME/.claude/settings.json" 2>/dev/null)

# ---------- colores (tono atenuado tipo Codex) ----------
DIM=$'\033[2m'
FG=$'\033[38;5;252m'
CYAN=$'\033[38;5;81m'
MAG=$'\033[38;5;176m'
GRN=$'\033[38;5;114m'
YEL=$'\033[38;5;179m'
RED=$'\033[38;5;174m'
RST=$'\033[0m'
SEP="${DIM} · ${RST}"

pct_color() {
  local p=$1
  if   (( p >= 50 )); then printf '%s' "$GRN"
  elif (( p >= 20 )); then printf '%s' "$YEL"
  else                     printf '%s' "$RED"
  fi
}

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

# ---------- contexto desde el transcript ----------
ctx_str=""
window=200000
if echo "$model_id" | grep -qi '1m' || jq -e '(.model // "") | test("1m")' "$HOME/.claude/settings.json" >/dev/null 2>&1; then
  window=1000000
fi
if [[ -n "$transcript" && -f "$transcript" ]]; then
  used=$(tac "$transcript" 2>/dev/null \
    | jq -c 'select(.message.usage != null) | .message.usage' 2>/dev/null \
    | head -1 \
    | jq -r '(.input_tokens // 0) + (.cache_creation_input_tokens // 0) + (.cache_read_input_tokens // 0)' 2>/dev/null)
  if [[ "$used" =~ ^[0-9]+$ ]]; then
    left=$(( (window - used) * 100 / window ))
    (( left < 0 )) && left=0
    ctx_str="${SEP}$(pct_color "$left")Context ${left}% left${RST}"
  fi
fi

# ---------- directorio con ~ ----------
dir="$cwd"
[[ "$dir" == "$HOME" ]] && dir="~"
[[ "$dir" == "$HOME/"* ]] && dir="~${dir#$HOME}"

# ---------- rama git ----------
branch=""
if b=$(git -C "$cwd" rev-parse --abbrev-ref HEAD 2>/dev/null); then
  [[ -n "$b" && "$b" != "HEAD" ]] && branch="${SEP}${MAG}${b}${RST}"
fi

# ---------- modelo + esfuerzo ----------
model_str="$model"
[[ -n "$effort" ]] && model_str="$model $effort"

# ---------- salida ----------
printf "%sReady%s%s%s%s%s%s%s%s" \
  "$GRN" "$RST" \
  "$SEP" "${FG}${model_str}${RST}" \
  "$rl_str" \
  "$ctx_str" \
  "${SEP}${CYAN}${dir}${RST}" \
  "$branch" ""
```

Dale permisos de ejecución:

```bash
chmod +x ~/.claude/statusline-command.sh
```

---

## 4. Registrar el statusline en settings

Agregá esta clave a `~/.claude/settings.json` (si el archivo ya existe, solo añadí el
bloque `statusLine` sin pisar el resto):

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline-command.sh"
  }
}
```

---

## 5. Probar sin abrir Claude Code

```bash
echo '{"workspace":{"current_dir":"'"$HOME"'"},"model":{"id":"claude-opus-4-8[1m]","display_name":"Opus 4.8"}}' \
  | bash ~/.claude/statusline-command.sh; echo
```

Deberías ver la línea con los porcentajes. La primera vez puede tardar ~1-3 s (crea el
cache); después es instantáneo. Verificá el cache con:

```bash
jq -c '{five_hour:.five_hour.utilization, seven_day:.seven_day.utilization}' ~/.claude/.usage-cache.json
```

Luego abrí (o reiniciá) Claude Code y la statusline aparece sola.

---

## 6. Personalización rápida

- **Quitar un campo:** borrá el fragmento que arma `rl_str`, `ctx_str` o `branch`, y
  sacá la variable correspondiente del `printf` final.
- **Mostrar cuándo se resetea el límite:** la respuesta trae `five_hour.resets_at` /
  `seven_day.resets_at` (ISO-8601); podés parsearlo con `date -d`.
- **Cambiar colores:** editá los códigos ANSI 256 (`\033[38;5;NNNm`) al principio.
- **TTL del cache:** variable `TTL` (segundos).
- **Otra ventana de contexto:** ajustá `window` o la detección de `1m`.

---

## 7. Solución de problemas

| Síntoma | Causa probable |
|---------|----------------|
| No aparecen `5h` / `weekly` | No existe `~/.claude/.credentials.json` (no estás logueado) o falta `curl`/`jq` |
| `HTTP 401` al consultar | Token expirado: abrí Claude Code para que lo renueve |
| No aparece `Context` | El transcript aún no tiene `usage` (sesión recién iniciada) |
| Statusline vacía | Falta `jq`; instalalo con `sudo apt install jq` |
| Tarda en cada render | El refresco corre en background; si persiste, revisá conectividad a `api.anthropic.com` |
```
