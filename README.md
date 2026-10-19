# herdr-execution

Skill de Claude Code para paralelizar trabajo: a partir de un prompt, el orquestador arma una tabla corta de tareas y lanza workers de Claude Code en paralelo, cada uno en su propio pane de [Herdr](https://herdr.dev). No hay plan escrito, ni ola de research por defecto, ni parada de aprobación: arranca después de un único bloque de supuestos.

## Es autocontenido

No depende de ningún otro skill. Se puede instalar solo este. Todo lo necesario (comandos de Herdr, estados de los workers, manejo de fallas, reglas de evidencia) está dentro de `SKILL.md`.

## Requisitos

- Claude Code corriendo dentro de un pane de Herdr (`HERDR_ENV=1`).
- El binario `herdr` en el `PATH`.
- Un repo git (el modo build usa `git worktree`).
- `pnpm` para los gates por defecto (`lint`, `test`, `build`); si el proyecto usa otros comandos, se usan los del proyecto.

Opcional: el MCP de Obsidian, solo si la tarea depende de conocimiento del proyecto. Si no está conectado, se ignora sin avisar.

## Instalación

Copiar esta carpeta al directorio de skills:

```bash
cp -r herdr-execution ~/.claude/skills/
```

## Uso

El skill tiene `disable-model-invocation: true`: Claude no lo carga solo, hay que invocarlo explícitamente.

```
/herdr-execution <qué hay que construir o investigar>
```

## Cómo funciona

1. **Setup:** carpeta de reportes en `~/.cache/herdr-reports/<slug>-<fecha>`.
2. **Lectura previa:** el orquestador lee `CLAUDE.md`, dependencias y los archivos que toca la tarea.
3. **Bloque de supuestos:** tabla `ID · objetivo · modo · modelo · archivos` y arranca. Solo pregunta por decisiones de producto, acciones destructivas o sensibles, y push/PR/`main`.
4. **Contrato primero:** si hay build en paralelo, el orquestador escribe las interfaces compartidas antes de despachar.
5. **Workers:** máximo 4 panes vivos, cada uno con un brief (`brief-template.md`) y un reporte propio.
6. **Aceptación:** reporte `PASS` en archivo, diff dentro del scope, verificación re-ejecutada por el orquestador, review independiente si toca pagos/auth/seguridad/migraciones, y gate completo tras el merge.

## Modos

| Modo | Cuándo | Workers |
|---|---|---|
| Build (por defecto) | La tarea se divide en 2+ piezas con archivos disjuntos | Un worktree y una rama por worker |
| Research | El usuario pide investigación o hay una pregunta que no se responde leyendo el repo | Solo lectura, con allowlist de herramientas |

Cuándo **no** usarlo: una pregunta o un cambio chico (hazlo directo), o piezas que tocan los mismos archivos (un solo worker).

## Qué no hace

Ejecutar el skill autoriza commits y merges locales sobre `feature/<slug>`. No autoriza push, PR, merge a `main`, borrar worktrees o ramas, ni deploys: cada uno necesita un sí explícito.

## Archivos

| Archivo | Contenido |
|---|---|
| `SKILL.md` | Instrucciones del skill |
| `brief-template.md` | Plantillas de brief para research, build y review |
