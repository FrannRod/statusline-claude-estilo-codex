# Statusline para Claude Code estilo Codex

Guía autosuficiente para replicar en otra computadora una statusline de Claude Code
que muestra modelo, esfuerzo, **límites de uso 5h / semanal**, contexto restante,
directorio y rama de git:

```
Ready · Opus 4.8 high · 2.6h 49% · 6.9d 78% · Context 96% left · ~ · main
```

El primer par muestra **cuánto falta para el reset** de cada ventana (5h en horas,
semanal en días, 1 decimal) seguido del **% que te queda**. Así ves de un vistazo el
restart sin tener que hacer la cuenta.

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

> **Solo se cachean respuestas buenas.** El endpoint a veces responde con
> `rate_limit_error` (que también es JSON válido); el script lo detecta y **no** lo guarda,
> así nunca pisa el último dato bueno. Mientras el endpoint te limite, aplica backoff
> exponencial entre reintentos (1m → 2m → 4m → 8m → 15m tope, se resetea al primer éxito)
> y la statusline lo aclara: con dato bueno muestra `... rate-limited, retry Xm`; sin dato,
> `5h rate-limited (retry Xm)`.

> **Ventana de contexto:** el script asume 1M de tokens si el modelo contiene `1m`
> (p. ej. `opus[1m]`), si no 200k. Ajustá la variable `window` si usás otra config.

---

## 3. Crear el script

Guardá esto como `~/.claude/statusline-command.sh`:

```bash
#!/bin/bash
# Statusline estilo Codex para Claude Code.
#   Ready · <Modelo> <esfuerzo> · 5h XX% left · weekly XX% left · Context XX% left · <dir> · <rama>
#
# Los rate limits (5h / semanal) se obtienen del endpoint OAuth de Anthropic
# usando el accessToken de ~/.claude/.credentials.json. El resultado se cachea
# y se refresca en segundo plano para no bloquear el render ni saturar la API.

input=$(cat)

CRED="$HOME/.claude/.credentials.json"
CACHE="$HOME/.claude/.usage-cache.json"   # solo guarda respuestas BUENAS (con five_hour)
ATTEMPT="$HOME/.claude/.usage-attempt"    # marca de último intento (para throttlear reintentos)
BACKOFF="$HOME/.claude/.usage-backoff"    # intervalo de reintento actual (segundos)
RLMARK="$HOME/.claude/.usage-ratelimited" # existe mientras el endpoint nos rate-limitee
LOCK="$HOME/.claude/.usage-cache.lock"
TTL=60          # frescura deseada del cache bueno
RETRY_BASE=60   # intervalo base entre intentos sin dato bueno
RETRY_MAX=900   # tope del backoff (15 min)

# intervalo de reintento vigente: arranca en BASE y se duplica con cada rate-limit
RETRY=$(cat "$BACKOFF" 2>/dev/null)
[[ "$RETRY" =~ ^[0-9]+$ ]] || RETRY=$RETRY_BASE

# formatear segundos como "45s" / "5m"
fmt_dur() { local s=$1; if (( s >= 60 )); then printf '%dm' $(( s / 60 )); else printf '%ds' "$s"; fi; }

# ---------- helper: refrescar el cache de uso en segundo plano ----------
# Escribe en $CACHE únicamente si la respuesta trae .five_hour.utilization.
# Así un error (p. ej. rate_limit_error) nunca pisa el último dato bueno.
refresh_usage() {
  # limpiar lock huérfano: si un refresh anterior murió sin liberarlo (corte de red,
  # terminal matada, timeout), el directorio queda para siempre y congela el cache.
  if [[ -d "$LOCK" ]]; then
    lock_age=$(( $(date +%s) - $(stat -c %Y "$LOCK" 2>/dev/null || echo 0) ))
    (( lock_age > 30 )) && rmdir "$LOCK" 2>/dev/null
  fi
  mkdir "$LOCK" 2>/dev/null || return 0   # lock atómico contra fetches simultáneos
  (
    touch "$ATTEMPT"   # registrar el intento ANTES de pegarle al endpoint
    tok=$(jq -r '.claudeAiOauth.accessToken // empty' "$CRED" 2>/dev/null)
    if [[ -n "$tok" ]]; then
      curl -s -m 10 \
        -H "Authorization: Bearer $tok" \
        -H "anthropic-beta: oauth-2025-04-20" \
        -H "anthropic-version: 2023-06-01" \
        "https://api.anthropic.com/api/oauth/usage" -o "$CACHE.tmp"
      if jq -e '.five_hour.utilization != null' "$CACHE.tmp" >/dev/null 2>&1; then
        # éxito: guardar dato bueno y resetear el backoff
        mv "$CACHE.tmp" "$CACHE"
        echo "$RETRY_BASE" > "$BACKOFF"
        rm -f "$RLMARK"
      elif jq -e '.error.type == "rate_limit_error"' "$CACHE.tmp" >/dev/null 2>&1; then
        # rate-limit: duplicar el intervalo (con tope) y marcar el estado
        next=$(( RETRY * 2 )); (( next > RETRY_MAX )) && next=$RETRY_MAX
        echo "$next" > "$BACKOFF"
        touch "$RLMARK"
      fi
      rm -f "$CACHE.tmp"
    fi
    rmdir "$LOCK" 2>/dev/null
  ) >/dev/null 2>&1 &
  disown 2>/dev/null
}

# ---------- decidir si hay que refrescar ----------
# El cache bueno marca "qué tan viejo es el dato"; ATTEMPT marca "cuándo reintenté".
# Si no hay dato bueno, throttleamos por ATTEMPT para no quedar en loop golpeando
# un endpoint rate-limiteado.
now=$(date +%s)
cache_age=999999
[[ -f "$CACHE" ]] && cache_age=$(( now - $(stat -c %Y "$CACHE" 2>/dev/null || echo 0) ))
attempt_age=999999
[[ -f "$ATTEMPT" ]] && attempt_age=$(( now - $(stat -c %Y "$ATTEMPT" 2>/dev/null || echo 0) ))

if (( cache_age > TTL )) && (( attempt_age > RETRY )); then
  refresh_usage
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

# nota de rate-limit: el endpoint nos está limitando, los límites pueden estar
# desactualizados; aclaramos cada cuánto reintentamos.
if [[ -f "$RLMARK" ]]; then
  rt=$(fmt_dur "$RETRY")
  if [[ -z "$rl_str" ]]; then
    rl_str="${SEP}${YEL}5h rate-limited${RST}${DIM} (retry ${rt})${RST}"
  else
    rl_str+="${SEP}${DIM}rate-limited, retry ${rt}${RST}"
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
- **Etiquetas de los límites:** por defecto muestran el tiempo hasta el reset
  (`resets_at`) en horas/días. Si preferís las etiquetas fijas `5h`/`weekly` con hora de
  reset, ajustá `fmt_left` (usa `date -d` sobre el ISO-8601).
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
| `5h` / `weekly` clavados en un valor viejo (no coinciden con `/usage`) | Lock huérfano: un refresh anterior murió sin liberar `~/.claude/.usage-cache.lock`. El script lo autorrepara solo (limpia locks de más de 30 s); si querés destrabarlo al instante: `rmdir ~/.claude/.usage-cache.lock` |
| Aparece `rate-limited` y no los porcentajes | El endpoint `/api/oauth/usage` te está limitando. Es esperado: el script aplica backoff y se recupera solo al pasar la ventana. Si nunca tuviste datos, esperá unos minutos |
```
