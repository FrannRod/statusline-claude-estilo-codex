# Parche: rate-limit del endpoint de uso (cache envenenado + backoff)

> **⚠️ Reemplazado (2026-07-17).** El backoff exponencial que describe este parche quedó
> superado: medimos que `/api/oauth/usage` **escala el castigo** si reintentás dentro de un
> ban (~14 min → 1 hora), así que adivinar el intervalo de reintento es contraproducente.
> El script actual **respeta el `retry-after` exacto del server** (guarda el deadline en
> `~/.claude/.usage-retry-at` y no consulta hasta que pase) y **no muestra datos de más de
> 15 min**. Si querés la versión al día, copiá el script completo de
> [statusline-claude-estilo-codex.md](./statusline-claude-estilo-codex.md), que ya lo trae.
> Este documento se conserva sólo como referencia histórica (el bug del cache envenenado que
> explica sigue siendo válido y está resuelto en el script nuevo).

## Cuándo aplicarlo

Aplicá este parche si **instalaste la statusline antes del 2026-06-18**, es decir, si tu
`~/.claude/statusline-command.sh` guarda en cache **cualquier** respuesta JSON (incluido un
error) y **no** tiene backoff cuando el endpoint te rate-limitea.

Forma rápida de saber si te afecta:

```bash
grep -q "five_hour.utilization != null" ~/.claude/statusline-command.sh \
  && echo "YA PARCHEADO" \
  || echo "FALTA EL PARCHE"
```

Si dice `FALTA EL PARCHE`, seguí leyendo. Si dice `YA PARCHEADO`, no tenés que hacer nada.

### Síntoma típico

La statusline muestra el contexto y todo lo demás, pero **desaparecen `5h XX% left` y
`weekly XX% left`** y no vuelven más, aunque pasen horas. Si mirás el cache, ves un error
en vez de los datos de uso:

```bash
cat ~/.claude/.usage-cache.json
# {"error":{"type":"rate_limit_error","message":"Rate limited. Please try again later."}}
```

## Por qué pasa

El endpoint `GET /api/oauth/usage` (el mismo que usa `/usage`) a veces responde con un
`rate_limit_error`. Eso **también es JSON válido**, y la versión vieja del script solo
chequeaba que la respuesta fuera JSON antes de guardarla en cache:

```bash
&& jq -e . "$CACHE.tmp" >/dev/null 2>&1 \
&& mv "$CACHE.tmp" "$CACHE"
```

Resultado: el error se guarda en el cache y lo "envenena". Como ese JSON no tiene
`.five_hour.utilization`, la sección de límites queda vacía y la statusline deja de
mostrarlos. Peor todavía: al no haber dato bueno, cada render sigue reintentando y
golpeando un endpoint que ya te está limitando, así que tarda en recuperarse.

El parche hace tres cosas:

1. **El cache solo guarda respuestas buenas** (las que traen `.five_hour.utilization`).
   Un error nunca pisa el último dato válido.
2. **Throttle de reintentos**: una marca de "último intento" (`~/.claude/.usage-attempt`)
   separa *cuándo reintenté* de *cuál es el último dato bueno*, para no quedar en loop.
3. **Backoff exponencial**: cuando el endpoint devuelve `rate_limit_error`, el intervalo
   entre intentos se duplica (1m → 2m → 4m → 8m → 15m tope) y se resetea al primer fetch
   exitoso. Mientras dura, la statusline lo aclara: con dato bueno muestra
   `... rate-limited, retry Xm`; sin dato, `5h rate-limited (retry Xm)`.

## Cómo aplicarlo

### 1. Limpiá el cache envenenado y el estado (recuperación inmediata)

```bash
rm -f ~/.claude/.usage-cache.json ~/.claude/.usage-backoff ~/.claude/.usage-ratelimited
rmdir ~/.claude/.usage-cache.lock 2>/dev/null
```

### 2. Parchá el script

Abrí `~/.claude/statusline-command.sh` y hacé estos cuatro reemplazos.

#### Reemplazo A — bloque de variables

**Buscá:**

```bash
CRED="$HOME/.claude/.credentials.json"
CACHE="$HOME/.claude/.usage-cache.json"
LOCK="$HOME/.claude/.usage-cache.lock"
TTL=60   # segundos de validez del cache
```

**Reemplazá por:**

```bash
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
```

#### Reemplazo B — cuerpo del fetch en background

**Buscá** (dentro de `refresh_usage()`, el subshell `( ... ) &`):

```bash
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
```

**Reemplazá por:**

```bash
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
```

#### Reemplazo C — decisión de refrescar

**Buscá:**

```bash
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
```

**Reemplazá por:**

```bash
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
```

#### Reemplazo D — nota de rate-limit en la statusline

**Buscá** el final de la sección de rate limits (el bloque que arma `weekly`):

```bash
  if [[ -n "$sd" ]]; then
    sl=$(printf '%.0f' "$(echo "100 - $sd" | bc -l 2>/dev/null || echo "$((100 - ${sd%.*}))")")
    rl_str+="${SEP}$(pct_color "$sl")weekly ${sl}% left${RST}"
  fi
fi
```

**Reemplazá por** (agrega el bloque de la nota justo después):

```bash
  if [[ -n "$sd" ]]; then
    sl=$(printf '%.0f' "$(echo "100 - $sd" | bc -l 2>/dev/null || echo "$((100 - ${sd%.*}))")")
    rl_str+="${SEP}$(pct_color "$sl")weekly ${sl}% left${RST}"
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
```

> Si preferís no editar a mano, lo más simple y seguro es reemplazar el archivo entero
> por el script de la guía [statusline-claude-estilo-codex.md](./statusline-claude-estilo-codex.md),
> que ya viene con todo aplicado.

### 3. Verificá

```bash
# 1) El parche está presente
grep -q "five_hour.utilization != null" ~/.claude/statusline-command.sh && echo "parche OK"

# 2) El script no tiene errores de sintaxis
bash -n ~/.claude/statusline-command.sh && echo "sintaxis OK"

# 3) Render de prueba (debe imprimir la statusline; tras 1-2 renders aparecen 5h / weekly)
echo '{"workspace":{"current_dir":"'"$HOME"'"},"model":{"id":"claude-opus-4-8[1m]","display_name":"Opus 4.8"}}' \
  | bash ~/.claude/statusline-command.sh; echo

# 4) El cache ahora tiene datos buenos (no un error)
jq -c '{five_hour:.five_hour.utilization, seven_day:.seven_day.utilization}' ~/.claude/.usage-cache.json
```

Si el endpoint te está limitando en este momento, vas a ver `5h rate-limited (retry Xm)`
en vez de los porcentajes: es lo esperado. Se recupera solo cuando pasa la ventana, y el
backoff evita que el script siga golpeando el endpoint mientras tanto.
