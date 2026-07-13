---
name: dag-task-runner
description: Descompone la tarea del usuario en un DAG de subtareas y las ejecuta con subagentes locales del Cursor SDK en orden topológico, mostrando estado en streaming en un canvas. Cada tarea tiene complejidad (HIGH/MED/LOW) que mapea a un modelo. Úsalo cuando pidan repartir trabajo, descomponer en DAG, correr subagentes en paralelo o partir una tarea grande en grafo de dependencias.
---

# DAG Task Runner

Descompone una tarea descrita por el usuario en un DAG JSON y ejecuta cada nodo como subagente local del Cursor SDK (con las salidas de los padres cosidas al prompt del hijo). El estado en vivo del DAG — incluido el streaming de cada subagente — se escribe en un `.canvas.tsx` que el runner reescribe en cada transición; el IDE recompila en caliente y se ve el paso `PENDING -> RUNNING -> FINISHED/ERROR`.

Esta skill puede vivir como skill de proyecto (`.cursor/skills/dag-task-runner`) o personal (`~/.cursor/skills/dag-task-runner`). El entry point instalado es `scripts/run_dag.ts` dentro del directorio de la skill. Define `DAG_RUNNER_DIR` para sobrescribir el directorio `scripts` auto-detectado.

## Cuándo usar

Actívala cuando el usuario diga algo como:

- "decompose this task", "break this into a DAG", "fan out subagents", "descompón esto en un DAG"
- "run this as a graph of subtasks" / "ejecuta esto como grafo de subtareas"
- una petición multi-paso donde unos pasos dependen de otros y otros pueden ir en paralelo

Sáltala si es una edición de un solo tiro, una pregunta rápida, o ya es lo bastante lineal para un solo turno de agente.

## Flujo

### Paso 1 — Generar el DAG JSON

Tú (el agente padre) redactas el DAG según la tarea. Esquema:

```json
{
  "title": "<short human-readable title for the run>",
  "models": {
    "HIGH": "gpt-5.3-codex",
    "MED": "composer-2",
    "LOW": "auto-low"
  },
  "tasks": [
    {
      "id": "<unique kebab-case id>",
      "depends_on": ["<id>", "..."],
      "complexity": "HIGH | MED | LOW",
      "subtask_prompt": "<self-contained prompt for the subagent>"
    }
  ]
}
```

Reglas:

- Cada entrada de `depends_on` debe referenciar el `id` de otra tarea.
- Sin ciclos. El runner rechaza DAGs cíclicos al parsear.
- `complexity` elige el modelo del subagente (tabla abajo). `HIGH` para razonamiento novedoso/complejo, `MED` para implementación típica, `LOW` para tareas mecánicas/lookup.
- El `models` de nivel superior opcional puede sobrescribir el mapa complejidad → modelo de este DAG.
- `subtask_prompt` debe leerse como petición autónoma — el runner antepone un resumen corto de salidas upstream; no hace falta repetirlas.
- **No** pongas dos tareas que escriban el mismo archivo en el mismo rank (hermanos del mismo rank corren a la vez y harían race).

#### Maximizar paralelismo — es el sentido del runner

El runner ejecuta tareas del mismo rank **en paralelo** con `Promise.all`. Un DAG lineal `A → B → C → D` desperdicia esa capacidad. Antes de cerrar el DAG, descompón el problema para sacar trabajo independiente:

1. **Por defecto, sin dependencias.** Añade `depends_on` **solo** cuando el hijo no puede empezar sin la salida del padre. "Lógicamente sigue" no es dependencia.
2. **Parte investigación/lectura en un primer rank ancho.** Greps, docs, scans de deps, schemas, inventario de tests — casi siempre van en rank 1 sin aristas entre sí.
3. **Abre abanico tras la implementación.** Tests, docs, changelog, types, lint suelen depender de la misma tarea de implementación y de nada más — un solo rank, no una cadena.
4. **Usa diamantes, no líneas.** Si dos tareas alimentan a una tercera, modélalo: rank 1 con los dos padres, rank 2 el merge.
5. **Seguridad de escritura en el mismo rank.** La restricción dura: no pongas en el mismo rank dos tareas que escriban el mismo archivo. Serialízalas con `depends_on` o fusiónalas en una.

Barra de calidad: al bosquejar ranks (rank 1 → rank 2 → …), al menos un rank debería tener más de una tarea en problemas no triviales. Si tu DAG es una sola cadena de ranks de 1 tarea, casi seguro perdiste paralelismo — vuelve a mirar.

El ejemplo del runner (`examples/example_dag.json`) muestra el patrón: rank 1 abre dos investigaciones de solo lectura, rank 2 las fusiona en un diseño, rank 3 implementa, rank 4 abre otra vez a tests + docs.

Escribe el JSON a un temp **y genera de inmediato el canvas inicial** para que el usuario lo abra mientras arrancan los subagentes. Ejecuta todo esto en un solo bloque de shell:

```bash
# 0. Locate the runner and pick a canvas path
resolve_runner_dir() {
  if [ -n "${DAG_RUNNER_DIR:-}" ] && [ -f "$DAG_RUNNER_DIR/run_dag.ts" ]; then
    printf '%s\n' "$DAG_RUNNER_DIR"
    return 0
  fi

  git_root="$(git rev-parse --show-toplevel 2>/dev/null || true)"
  for dir in \
    "$PWD/.cursor/skills/dag-task-runner/scripts" \
    "${git_root:+$git_root/.cursor/skills/dag-task-runner/scripts}" \
    "$HOME/.cursor/skills/dag-task-runner/scripts"
  do
    if [ -n "$dir" ] && [ -f "$dir/run_dag.ts" ]; then
      printf '%s\n' "$dir"
      return 0
    fi
  done

  echo "Could not find dag-task-runner/scripts. Copy .cursor/skills/dag-task-runner into this project, install it under ~/.cursor/skills, or set DAG_RUNNER_DIR." >&2
  return 1
}

RUNNER_DIR="$(resolve_runner_dir)"
CANVAS_PATH="$HOME/.cursor/projects/<workspace-slug>/canvases/dag-<slug>.canvas.tsx"

# 1. Write the DAG JSON
cat > /tmp/dag-<slug>.json <<'JSON'
{ "title": "...", "tasks": [ ... ] }
JSON

# 2. Ensure deps are installed (skips if already present)
[ -x "$RUNNER_DIR/node_modules/.bin/tsx" ] || \
  (cd "$RUNNER_DIR" && (pnpm install --silent || npm install --silent))

# 3. Generate the initial all-PENDING canvas (no CURSOR_API_KEY needed)
"$RUNNER_DIR/node_modules/.bin/tsx" "$RUNNER_DIR/run_dag.ts" \
  --init-only \
  --dag /tmp/dag-<slug>.json \
  --canvas-path "$CANVAS_PATH"

# 4. Best-effort auto-open of the canvas file; ignore failure in headless/non-macOS environments
open "$CANVAS_PATH" >/dev/null 2>&1 || true
```

La ruta del canvas es:

```
~/.cursor/projects/<workspace-slug>/canvases/dag-<slug>.canvas.tsx
```

`<workspace-slug>` sale del path absoluto del cwd sustituyendo `/` y caracteres especiales por `-`. Toma `pwd`, quita el `/` inicial y reemplaza cada `/` restante por `-`. Ejemplo: cwd `/Users/me/Code/myapp` → slug `Users-me-Code-myapp`. Usa el mismo `<slug>` que en el JSON del DAG para correlacionarlos.

### Paso 2 — Enlazar el canvas en el chat

Con el archivo ya en disco, publica un hipervínculo Markdown con el texto exacto `Open Canvas` y una URL `file://`, más la ruta absoluta de respaldo:

> I created a live canvas: [Open Canvas](file:///Users/<user>/.cursor/projects/<workspace-slug>/canvases/dag-<slug>.canvas.tsx)
> Fallback path: `/Users/<user>/.cursor/projects/<workspace-slug>/canvases/dag-<slug>.canvas.tsx`

Usa siempre el texto de enlace `Open Canvas`. Usa la ruta absoluta en la URL `file://` y en el fallback, nunca `~/`. Hazlo **antes** del Paso 3 para que el usuario abra el canvas mientras arrancan los subagentes. El bloque del Paso 1 ya intenta `open`; si falla, sigue y confía en el enlace del chat.

### Paso 3 — Ejecutar el DAG

Asegura `CURSOR_API_KEY` (el runner falla rápido si falta) y lanza:

```bash
[ -n "$CURSOR_API_KEY" ] || { [ -f .env ] && set -a && source .env && set +a; }

"$RUNNER_DIR/node_modules/.bin/tsx" "$RUNNER_DIR/run_dag.ts" \
  --dag /tmp/dag-<slug>.json \
  --canvas-path "$CANVAS_PATH"
```

Mismo `--canvas-path` que el Paso 1. El runner:

1. Valida el DAG y reutiliza el canvas existente.
2. Por cada rank (topo-sort de Kahn), lanza tareas listas en paralelo como agentes locales del Cursor SDK y reescribe el canvas en cada transición, streameando texto del assistant a cada tarjeta.
3. Omite automáticamente tareas cuyo upstream falló (las marca `ERROR` con "Skipped: upstream task(s) … failed").
4. Captura texto final, status, tokens y duración de cada subagente.
5. Escribe un canvas final con estadísticas.
6. Con SIGINT/SIGTERM/SIGHUP, cancela subagentes en vuelo antes de finalizar el canvas.

#### Knobs de CLI

| Flag | Default | Uso |
|------|---------|-----|
| `--models-file <path>` | — | JSON con override parcial complejidad → modelo. |
| `--task-timeout-ms <ms>` | `1200000` (20 min) | Marca la tarea `ERROR` si corre demasiado. |
| `--stream-publish-ms <ms>` | `500` | Throttle de escrituras de streaming al canvas. |
| `--stream-idle-timeout-ms <ms>` | `300000` (5 min) | Marca `ERROR` si no llegan eventos de stream. |
| `--debounce <ms>` | `200` | Debounce de escritura del canvas. |

### Paso 4 — Resumir

Al salir el runner, resume qué completó/falló y vuelve a enlazar el canvas con el texto exacto `[Open Canvas](file:///Users/<user>/.cursor/projects/<workspace-slug>/canvases/dag-<slug>.canvas.tsx)`. Incluye la ruta absoluta de fallback solo si aporta.

## Complejidad → modelo

| Complejidad | Modelo             |
|-------------|--------------------|
| HIGH        | `gpt-5.3-codex`   |
| MED         | `composer-2`       |
| LOW         | `auto-low`         |

Sobrescribe cualquier subconjunto con `models` en el DAG, o pasa un perfil reutilizable con `--models-file <path>`. Precedencia: defaults < `models` del DAG < `--models-file`. El catálogo de modelos del Cursor SDK puede variar por cuenta; usa `Cursor.models.list()` de la docs del SDK para confirmar IDs.

## Auth

El runner lee `CURSOR_API_KEY` del entorno. Configúralo como suelas gestionar secretos:

```bash
export CURSOR_API_KEY=crsr_...
```

Si el workspace tiene un `.env` con la clave, fuente primero:

```bash
set -a && source .env && set +a
```

## Opciones CLI

| Flag                        | Default              | Notas                                                                              |
|-----------------------------|----------------------|------------------------------------------------------------------------------------|
| `--dag`                     | required             | Ruta al JSON del DAG.                                                              |
| `--canvas-path`             | composed from below  | Ruta absoluta completa al canvas. Preferida — flujo gestionado por el padre.        |
| `--canvas`                  | —                    | Stem del nombre del canvas (sin `.canvas.tsx`). Solo si no hay `--canvas-path`.     |
| `--canvases-dir`            | derived from cwd     | Override del directorio de canvases. Solo con `--canvas`.                          |
| `--cwd`                     | `process.cwd()`      | Working dir de cada subagente.                                                     |
| `--models-file`             | —                    | JSON con override parcial complejidad → modelo.                                    |
| `--debounce`                | `200` (ms)           | Debounce de escritura del canvas.                                                  |
| `--init-only`               | `false`              | Escribe canvas inicial todo-`PENDING` y sale. No requiere `CURSOR_API_KEY`.        |
| `--task-timeout-ms`         | `1200000` (20 min)   | Marca `ERROR` si supera esta duración.                                             |
| `--stream-publish-ms`       | `500` (ms)           | Throttle de streaming al canvas.                                                   |
| `--stream-idle-timeout-ms`  | `300000` (5 min)     | Marca `ERROR` si no llegan eventos de stream en esta ventana.                      |

## Limitaciones

- Solo runtime local — cada subagente opera sobre `--cwd` (por defecto donde invocaste el runner).
- Hermanos del mismo rank van en paralelo; no dejes que escriban los mismos archivos.
- Este runner no configura MCP servers inline ni sub-sub-agentes.
- Una tarea fallida omite automáticamente a todos los dependientes downstream (marcados `ERROR` con "Skipped: upstream task(s) … failed"). Evita llamadas API inútiles sin inputs.
- El texto streameado por tarea está limitado a `STREAM_CAP = 4000` chars. El contexto upstream al hijo a 2000 chars por padre.
- Timeouts marcan `ERROR` en vez de quedarse en `RUNNING` indefinidamente.
- SIGINT/SIGTERM/SIGHUP cancelan subagentes en vuelo y finalizan el canvas antes de salir.
- Rechazos no manejados inesperados del SDK se suprimen para no tumbar el runner; excepciones no capturadas se loguean y disparan shutdown limpio.

## Referencia

- Ejemplo de esquema DAG: `examples/example_dag.json` (hermano de esta skill tras instalar)
- Entry point del runner: `run_dag.ts` dentro de `$RUNNER_DIR`
- Docs Cursor SDK: https://cursor.com/docs/api/sdk/typescript
