# Crear módulos para Legion

Esta guía contiene la referencia técnica que no hace falta cargar para descubrir o usar un módulo.
Para una introducción rápida, volvé al [README](../../README.es.md).

## Empezar desde el template

Copiá [`_meta/template/module.md`](../template/module.md) a una carpeta de primer nivel:

```text
<module-name>/
├── module.md
├── .claude/
│   ├── agents/<agent-name>.md
│   ├── skills/<skill-name>/SKILL.md      # opcional
│   └── rules/                            # opcional
│       └── module-rules.md
├── scripts/                              # opcional
├── references/                           # opcional
├── assets/                               # opcional
└── <otros archivos necesarios>
```

Cada carpeta es un módulo instalable e independiente. `.claude/rules/` es la convención de
Claude Code; `provides_rules` apunta a un archivo concreto dentro de ella, aunque puede declarar
otro path interno. También se permiten manifests de
dependencias, schemas, fixtures y cualquier recurso que use el agente: no existe una lista
cerrada de subcarpetas. Todo path declarado por `agent_entrypoint`, `provides_skills` o
`provides_rules` debe permanecer dentro del módulo. Legion no carga el `AGENTS.md` interno del
repositorio al ejecutar el agente.

Un repositorio puede contener un único módulo (`module.md` en la raíz) o una colección de módulos
hermanos (`<subfolder>/module.md`, un nivel de profundidad). `/new-module` descubre todos los de
una colección y omite carpetas cuyo nombre empieza con `_`.

## Elegir el tipo

- `gate`: verifica una historia en una etapa declarada. Puede rechazarla cuando `blocking: true`,
  pero nunca reemplaza al `worktree-reviewer`.
- `generator`: lee un repo base o worktree y genera artefactos bajo demanda. No participa del
  ciclo de historias ni emite veredictos.
- `implementer`: escribe código en el worktree de una historia. Sólo corre cuando una subtask lo
  nombra como `[implementer:<module-name>]`.

## Contrato de `module.md`

Todos los tipos requieren:

- `type`: `gate`, `generator` o `implementer`.
- `tools`: lista explícita de capacidades.
- `agent_entrypoint`: path relativo al agente.

### Campos de `gate`

- `valid_stages` y `default_stage`.
- `default_activation`: `opt-in` o `always`.
- `writes_to`: subpath del worktree, o `""` si es read-only.
- `blocking`.
- Opcionales: `max_rejection_rounds`, `max_concurrent`, `requires_local_config`,
  `provides_skills` y `provides_rules`.

### Campos de `generator`

- `output`: `modules/output/<module-name>/<base-repo-name>/`.
- `scope`: `base-repo` o `worktree`.

Los campos de gate, `provides_skills` y `provides_rules` no aplican a generators.

### Campos de `implementer`

No agrega campos obligatorios. Puede usar `provides_skills` y `provides_rules`. No admite
`writes_to`, campos de stage, `blocking`, `max_concurrent` ni activación `always`: el permiso de
escritura cubre únicamente el worktree de la historia que lo pidió.

El template documenta cada campo y sus incompatibilidades.

## Escribir el agente

El frontmatter del agente debe declarar las mismas tools que `module.md`. Legion cruza ambas
listas durante la instalación.

Mantené el entrypoint enfocado en:

1. Inputs que recibe.
2. Flujo de decisión.
3. Límites de lectura y escritura.
4. Criterio de finalización.

Mové checklists, formatos y variantes a skills. Indicá en el agente exactamente cuándo leer cada
una. Una skill condicional no debe cargarse en ejecuciones que no la necesitan.

Las skills internas del agente son distintas de `provides_skills`:

- Skill interna: guía al agente del propio módulo.
- `provides_skills`: guía al implementador de una historia que activó el módulo.

## `provides_skills` y `provides_rules`

Disponibles para `gate` e `implementer`.

```yaml
provides_skills:
  - .claude/skills/oop-practices/SKILL.md
provides_rules: .claude/rules/module-rules.md
```

Todos los paths deben existir dentro del módulo.

`provides_rules` apunta a un Markdown itemizado por `rule_id` estable:

```markdown
### no-mutable-public-fields

Los campos públicos no deben ser mutables.

### max-function-length

Dividí una función cuando mezcla más de una responsabilidad.
```

Legion negocia estas reglas la primera vez que un proyecto activa el módulo. Los veredictos se
guardan por proyecto; actualizar o renegociar una regla no reabre historias finalizadas.

## Autoconfig

Si el template conserva placeholders `<...>`, `/new-module` ofrece completarlos:

1. Resuelve primero el tipo.
2. Puede inferir `tools` del agente.
3. Pregunta en lotes pequeños los valores restantes.
4. Escribe el manifiesto resuelto en la copia instalada.
5. Muestra el resultado en el preview antes de registrarlo.

Un valor real escrito por el autor nunca se reemplaza como si fuera un placeholder.

## Validación local

Instalá un módulo o una colección desde una instancia de Legion:

```text
/new-module <repo-or-path>
```

En una colección con dos o más módulos, cada carpeta atraviesa su propio preview y se registra
como `<repo>_<subfolder>`. El proceso es secuencial y termina con un resumen de módulos
registrados, descartados y omitidos. Si querés instalar sólo uno desde una copia local, apuntá
directamente a la carpeta que contiene su `module.md`.

Revisá el preview y probá el flujo correspondiente:

- `gate`: activalo en `## Modules` y verificá stage, `writes_to` y comportamiento blocking.
- `generator`: ejecutá `/run-module <name>` contra el repo y un worktree; ambos deben quedar sin
  cambios inesperados.
- `implementer`: usalo en una subtask; debe pasar por diseño y revisión como cualquier agente.
- Con `provides_rules`: confirmá que la negociación ocurre una sola vez por proyecto.

## Modelo de confianza

`/new-module` valida forma y hace un escaneo de riesgo best-effort; no demuestra intención.

- `Bash` permite comandos con el alcance del proceso host.
- `writes_to` y `output` expresan el contrato de escritura, no crean por sí solos un sandbox.
- Legion nunca amplía tools en el lanzamiento.
- Un cambio upstream de contrato obliga a un nuevo preview.
- El módulo nunca debe hacer commit, push ni tocar git state.

## Ciclo de vida

- `/module uninstall <name>`: depreca o elimina cuando ya no hay referencias activas.
- `/module activate <name>`: reactiva un módulo deprecado.
- `/module renegotiate <name> [rule_id]`: reabre reglas para el proyecto actual.
- `/run-module <name>`: ejecuta un generator bajo demanda.

Antes de publicar, comprobá que los permisos declarados sean el alcance real completo del módulo,
no sólo el mínimo necesario para pasar el preview.
