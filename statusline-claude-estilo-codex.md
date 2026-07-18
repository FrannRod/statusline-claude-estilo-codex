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
> a terceros. El resultado se cachea ~2 min y se refresca en segundo plano para que el
> render nunca se bloquee ni sature la API.

> **El polling normal está bien; el maltrato no.** `/api/oauth/usage` tolera que lo consultes
> a un ritmo razonable (probado: cada 1-2 min anda sin problemas). Pero tiene un rate-limit por
> token que **escala el castigo si lo martillás** (medido con una sonda abusiva de decenas de
> requests en minutos: ~14 min de espera la primera vez, hasta 1 hora si seguís golpeando; sale
> del castigo solo tras un rato de silencio). Para no caer nunca en eso, ante un `rate_limit_error`
> el script **respeta el `retry-after` exacto que manda el server**: guarda ese deadline y no
> vuelve a consultar hasta que pase (nunca reintenta dentro del ban, que es lo que dispara la
> escalada). Sólo se cachean respuestas buenas (con `.five_hour`), así un error nunca pisa el
> último dato bueno. Mientras dure una espera, la statusline lo aclara: con dato fresco muestra
> `... rate-limited, retry Xm`; sin dato útil, `5h rate-limited (retry Xm)`.

> **Datos viejos no se muestran.** Si el último dato bueno tiene más de 15 min (`STALE`), la
> statusline **no** muestra los porcentajes (un número viejo no sirve para decidir nada); queda
> sólo la nota de rate-limit explicando por qué no hay número.

> **Ventana de contexto:** el script asume 1M de tokens si el modelo contiene `1m`
> (p. ej. `opus[1m]`), si no 200k. Ajustá la variable `window` si usás otra config.

---

## 3. Crear el script

Guardá esto como `~/.claude/statusline-command.sh`:

```bash
# statusline-claude-estilo-codex · v2026-07-17 · https://github.com/FrannRod/statusline-claude-estilo-codex
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
ATTEMPT="$HOME/.claude/.usage-attempt"    # marca de último intento (piso duro entre consultas)
RETRYAT="$HOME/.claude/.usage-retry-at"   # epoch hasta el cual NO hay que consultar (del retry-after del server)
LOCK="$HOME/.claude/.usage-cache.lock"
AUTO_FETCH=1     # KILL-SWITCH: 0 = nunca consultar el endpoint automáticamente; 1 = comportamiento normal
TTL=120          # frescura deseada del dato bueno (2 min): el polling normal NO molesta al endpoint
STALE=900        # edad máxima del cache que consideramos útil (15 min): más viejo no se muestra
MIN_INTERVAL=60  # piso duro entre consultas pase lo que pase (anti-hammering)

# epoch hasta el cual el server nos pidió esperar (0 si no hay). Respetarlo es lo que
# evita reintentar dentro de un ban: el endpoint ESCALA el castigo (vimos 14min → 1h)
# si insistís, así que jamás consultamos antes de este deadline.
retry_at=$(cat "$RETRYAT" 2>/dev/null)
[[ "$retry_at" =~ ^[0-9]+$ ]] || retry_at=0

# formatear segundos como "45s" / "5m" / "2h"
fmt_dur() { local s=$1; if (( s >= 3600 )); then printf '%dh' $(( s / 3600 )); elif (( s >= 60 )); then printf '%dm' $(( s / 60 )); else printf '%ds' "$s"; fi; }

# ---------- logging (multi-sesión) ----------
# Etiqueta de sesión, derivada del session_id (o del nombre del transcript): estable por
# sesión, a diferencia del PID, que cambia en cada render (el script corre de nuevo cada vez).
SES=$(printf '%s' "$input" | jq -r '.session_id // .transcript_path // empty' 2>/dev/null)
SES=$(basename "$SES" .jsonl 2>/dev/null); SES=${SES:0:8}; [[ -z "$SES" ]] && SES="unknown"

LOG="$HOME/.claude/.usage-statusline.log"
LOG_MAX=200000   # ~200 KB; al pasarse se recorta a las últimas líneas

# log(): cada evento es UN solo append corto. Con O_APPEND, un write < PIPE_BUF (4 KB) es
# atómico en POSIX, así que varias sesiones escribiendo a la vez NO se pisan las líneas.
# El [ses] permite atribuir cada línea a su sesión.
log() { printf '%s [%s] %s\n' "$(date '+%F %T')" "$SES" "$*" >> "$LOG" 2>/dev/null; }

# should_beat(): throttle COMPARTIDO para el "latido" de estado, así el idle no inunda el
# log. Vía mtime de un marcador; a lo sumo una línea de estado cada 60s entre TODAS las
# sesiones (la carrera por el touch es benigna).
should_beat() {
  local beat="$HOME/.claude/.usage-log-beat" n last=0; n=$(date +%s)
  [[ -f "$beat" ]] && last=$(stat -c %Y "$beat" 2>/dev/null || echo 0)
  (( n - last >= 60 )) && { touch "$beat" 2>/dev/null; return 0; }
  return 1
}

# ---------- helper: refrescar el cache de uso en segundo plano ----------
# Escribe en $CACHE únicamente si la respuesta trae .five_hour.utilization.
# Así un error (p. ej. rate_limit_error) nunca pisa el último dato bueno.
refresh_usage() {
  # limpiar lock huérfano: si un refresh anterior murió sin liberarlo (corte de red,
  # terminal matada, timeout), el directorio queda para siempre y congela el cache.
  if [[ -d "$LOCK" ]]; then
    lock_age=$(( $(date +%s) - $(stat -c %Y "$LOCK" 2>/dev/null || echo 0) ))
    (( lock_age > 30 )) && { rmdir "$LOCK" 2>/dev/null && log "lock huérfano removido (edad ${lock_age}s)"; }
  fi
  # lock atómico contra fetches simultáneos entre sesiones. Si otra sesión lo tiene, no
  # consultamos (queda registrado para ver la coordinación multi-sesión).
  mkdir "$LOCK" 2>/dev/null || { log "fetch omitido: lock tomado por otra sesión"; return 0; }
  log "fetch: consultando endpoint (cache_age=${cache_age}s attempt_age=${attempt_age}s)"
  (
    touch "$ATTEMPT"   # registrar el intento ANTES de pegarle al endpoint
    tok=$(jq -r '.claudeAiOauth.accessToken // empty' "$CRED" 2>/dev/null)
    if [[ -n "$tok" ]]; then
      curl -s -m 10 -D "$CACHE.hdr" \
        -H "Authorization: Bearer $tok" \
        -H "anthropic-beta: oauth-2025-04-20" \
        -H "anthropic-version: 2023-06-01" \
        "https://api.anthropic.com/api/oauth/usage" -o "$CACHE.tmp"
      if jq -e '.five_hour.utilization != null' "$CACHE.tmp" >/dev/null 2>&1; then
        # éxito: guardar dato bueno y limpiar la espera
        mv "$CACHE.tmp" "$CACHE"
        rm -f "$RETRYAT"
        log "fetch OK: five_hour=$(jq -r '.five_hour.utilization' "$CACHE" 2>/dev/null)% seven_day=$(jq -r '.seven_day.utilization' "$CACHE" 2>/dev/null)%"
      else
        # error: si el server manda retry-after, lo respetamos EXACTO (no reintentar antes,
        # que reintentar dentro del ban lo escala 14min → 1h). Sin retry-after lo tratamos
        # como transitorio (red/timeout/5xx) y reintentamos en el próximo ciclo.
        ra=$(grep -i '^retry-after:' "$CACHE.hdr" 2>/dev/null | tr -d '\r' | awk '{print $2}')
        if [[ "$ra" =~ ^[0-9]+$ ]]; then
          until=$(( $(date +%s) + ra ))
          echo "$until" > "$RETRYAT"
          etype=$(jq -r '.error.type // "desconocido"' "$CACHE.tmp" 2>/dev/null)
          log "fetch limitado ($etype): retry-after=${ra}s (espera hasta $(date -d "@$until" '+%T'))"
        else
          log "fetch error sin retry-after (¿red/timeout/5xx?); reintenta en el próximo ciclo"
        fi
      fi
      rm -f "$CACHE.tmp" "$CACHE.hdr"
    else
      log "fetch abortado: sin accessToken en credenciales"
    fi
    rmdir "$LOCK" 2>/dev/null
    # recorte best-effort del log (mv es atómico; carrera entre sesiones benigna)
    if [[ -f "$LOG" ]] && (( $(stat -c %s "$LOG" 2>/dev/null || echo 0) > LOG_MAX )); then
      tail -n 400 "$LOG" > "$LOG.trim$$" 2>/dev/null && mv "$LOG.trim$$" "$LOG" 2>/dev/null
    fi
  ) >/dev/null 2>&1 &
  disown 2>/dev/null
}

# ---------- decidir si hay que refrescar ----------
# Consultamos SÓLO si se cumplen las tres: (1) el server no nos pidió esperar
# (now >= retry_at), (2) el dato bueno venció (cache_age > TTL), (3) respetamos un piso
# duro entre intentos (anti-hammering). El gate por retry_at es lo que evita reintentar
# dentro de un ban y que el castigo escale.
now=$(date +%s)
cache_age=999999
[[ -f "$CACHE" ]] && cache_age=$(( now - $(stat -c %Y "$CACHE" 2>/dev/null || echo 0) ))
attempt_age=999999
[[ -f "$ATTEMPT" ]] && attempt_age=$(( now - $(stat -c %Y "$ATTEMPT" 2>/dev/null || echo 0) ))

if (( AUTO_FETCH )) && (( now >= retry_at )) && (( cache_age > TTL )) && (( attempt_age > MIN_INTERVAL )); then
  refresh_usage
elif should_beat; then
  # latido de estado: por qué NO consultamos (throttled 60s, compartido entre sesiones).
  if (( ! AUTO_FETCH )); then
    log "estado: AUTO_FETCH=0 (fetch apagado); cache_age=${cache_age}s"
  elif (( retry_at > now )); then
    log "estado: en espera por rate-limit, faltan $(( retry_at - now ))s; cache_age=${cache_age}s"
  elif (( cache_age <= TTL )); then
    log "estado: cache fresco (age=${cache_age}s < TTL=${TTL}s), sin fetch"
  else
    log "estado: throttle por intento reciente (attempt_age=${attempt_age}s < ${MIN_INTERVAL}s)"
  fi
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
# Solo mostramos los porcentajes si el cache es fresco (<= STALE). Un dato de más de
# 15 min no sirve para decidir nada, así que preferimos NO mostrarlo (queda la nota de
# rate-limit explicando por qué no hay número) antes que mentir con un valor viejo.
if [[ -f "$CACHE" ]] && (( cache_age <= STALE )); then
  fh=$(jq -r '.five_hour.utilization // empty'  "$CACHE" 2>/dev/null)
  fh_rst=$(jq -r '.five_hour.resets_at // empty' "$CACHE" 2>/dev/null)
  sd=$(jq -r '.seven_day.utilization // empty'  "$CACHE" 2>/dev/null)
  sd_rst=$(jq -r '.seven_day.resets_at // empty' "$CACHE" 2>/dev/null)

  # tiempo restante hasta un reset ISO-8601, auto-escalado a d/h/m (1 decimal en d/h).
  # Si no hay resets_at (null/vacío) o la fecha es inválida, cae al label fijo ($2).
  # OJO: GNU `date -d ""` devuelve "ahora" en vez de fallar, así que el vacío se chequea a mano
  # (si no, un resets_at null se mostraría como el falso "0.0h").
  # LC_ALL=C en el printf: bc emite el decimal con punto y un locale con coma lo rechaza.
  #   $1 = ISO timestamp   $2 = label fallback (cuando no hay reset)
  fmt_left() {
    local iso=$1 fb=$2
    [[ -z "$iso" || "$iso" == "null" ]] && { printf '%s' "$fb"; return; }
    local rst; rst=$(date -d "$iso" +%s 2>/dev/null) || { printf '%s' "$fb"; return; }
    local s=$(( rst - now )); (( s < 0 )) && s=0
    if   (( s >= 86400 )); then LC_ALL=C printf '%.1fd' "$(echo "$s/86400" | bc -l)"
    elif (( s >= 3600  )); then LC_ALL=C printf '%.1fh' "$(echo "$s/3600"  | bc -l)"
    else                        printf '%dm' $(( s / 60 ))
    fi
  }

  if [[ -n "$fh" ]]; then
    fl=$(printf '%.0f' "$(echo "100 - $fh" | bc -l 2>/dev/null || echo "$((100 - ${fh%.*}))")")
    rl_str+="${SEP}$(pct_color "$fl")$(fmt_left "$fh_rst" 5h) ${fl}%${RST}"
  fi
  if [[ -n "$sd" ]]; then
    sl=$(printf '%.0f' "$(echo "100 - $sd" | bc -l 2>/dev/null || echo "$((100 - ${sd%.*}))")")
    rl_str+="${SEP}$(pct_color "$sl")$(fmt_left "$sd_rst" weekly) ${sl}%${RST}"
  fi
fi

# nota de rate-limit: si el server nos pidió esperar, lo aclaramos y mostramos cuánto
# falta para volver a consultar (su retry-after). Al vencer el deadline, la nota se va.
# Con AUTO_FETCH=0 no consultamos, así que no tiene sentido mostrar "retry".
if (( AUTO_FETCH )) && (( retry_at > now )); then
  rt=$(fmt_dur $(( retry_at - now )))
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
- **Apagar el fetch de uso (kill-switch):** poné `AUTO_FETCH=0`. La statusline deja de consultar el endpoint por completo (no muestra 5h/weekly ni la nota de rate-limit; todo lo demás sigue). Útil si el endpoint te dejó en un tier de castigo y querés que drene sin tráfico. Volvé a `1` para reactivarlo.
- **Log de diagnóstico:** el script registra los eventos relevantes (fetch, resultado, rate-limit con su `retry-after`, contención de lock entre sesiones, latido de estado) en `~/.claude/.usage-statusline.log`. Es multi-sesión: cada línea es un append atómico y lleva `[sesión]` para atribuirla; el latido de estado está throttleado a 1 línea/60s entre todas las sesiones para no inundar, y el archivo se recorta solo al pasar `LOG_MAX`. Para seguirlo en vivo: `tail -f ~/.claude/.usage-statusline.log`.
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
| Aparece `rate-limited` y no los porcentajes | El endpoint `/api/oauth/usage` te está limitando. Es esperado: el script respeta el `retry-after` que manda el server y no vuelve a consultar hasta que pase (la nota muestra cuánto falta). Se recupera solo. **No fuerces reintentos** (borrar `~/.claude/.usage-retry-at` a mano y pegarle de nuevo): el endpoint escala el castigo si insistís (de ~14 min a 1 hora) |
| Los porcentajes desaparecen pero el dato no es viejo | Puede ser un ban largo heredado: mirá `~/.claude/.usage-retry-at` (epoch hasta el que espera). Si ya pasó y sigue sin datos, esperá un ciclo de refresh (TTL) |

---

## 8. Actualizar una instalación existente

Si ya tenés un `~/.claude/statusline-command.sh` de una versión anterior, **no hay parches
que aplicar a mano**. Esta guía siempre trae la última versión del script (sección 3) y el
Changelog de abajo explica el *porqué* de cada cambio. Para ponerte al día, pasale a una IA
tu script actual + esta guía y pedile que lo actualice.

La **primera línea del script** trae la versión (una fecha) y la URL de este repo, justo para
esto: con `head -1 ~/.claude/statusline-command.sh` la IA sabe qué versión tenés instalada y
de dónde bajar la última.

```bash
head -1 ~/.claude/statusline-command.sh
# statusline-claude-estilo-codex · v2026-07-17 · https://github.com/FrannRod/statusline-claude-estilo-codex
```

La IA debería:

1. Leer la versión de la primera línea de tu script y compararla con la última entrada del
   Changelog (ambas son fechas).
2. Comparar tu script con el bloque `bash` de la sección 3 (la última versión).
3. Aplicar las diferencias, **preservando tus personalizaciones** (colores, `window`, `TTL`,
   `AUTO_FETCH`, campos que hayas sacado, etc.).
4. Dejar en la primera línea la fecha del último cambio del Changelog (la versión nueva).

El *cómo* (el diff) la IA lo deduce comparando; lo único que no es derivable —el *por qué*—
está en el Changelog.

## 9. Changelog

Sólo el *por qué* de cada cambio, una o dos oraciones. El *cómo* vive en el script de la
sección 3, que siempre refleja la última versión.

- **2026-07-17 — Rate-limit real y datos frescos.** Ante un `rate_limit_error` se respeta el
  `retry-after` exacto del server en vez de adivinar un backoff: el endpoint escala el castigo
  si se lo martilla (medido: ~14 min → 1 h) y reintentar dentro del ban lo empeora. Además: no
  se muestran datos de más de 15 min, `fmt_left` auto-escala a d/h/m y evita el falso `0.0h`,
  kill-switch `AUTO_FETCH` para cortar el tráfico, y log multi-sesión en
  `~/.claude/.usage-statusline.log`.
- **2026-07-07 — Tiempo hasta el reset.** Las etiquetas de uso muestran cuánto falta para el
  reset de cada ventana (campo `resets_at`) en vez de labels fijas, para ver el restart de un
  vistazo sin abrir `/usage`.
- **2026-06-18 — Cache sólo con respuestas buenas.** El endpoint a veces responde
  `rate_limit_error` (JSON válido) y eso envenenaba el cache, dejando la statusline sin
  límites; ahora sólo se cachean respuestas con datos reales.
- **2026-06-17 — Auto-reparación de lock huérfano.** Si un refresh en background muere sin
  liberar el lock (corte de red, timeout, terminal matada), el cache quedaba congelado para
  siempre; ahora un lock de más de 30 s se limpia solo en el próximo render.
