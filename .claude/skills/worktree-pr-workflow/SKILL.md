---
name: worktree-pr-workflow
description: Convención de trabajo con git de este proyecto — un worktree por tarea, una rama y un pull request por tarea, Conventional Commits y merge con squash. Usala siempre que empieces una tarea nueva, antes de commitear o pushear, al abrir o actualizar un PR, al sincronizar con main o resolver conflictos, al dar una tarea por terminada, o cuando haya que preparar el repo para trabajar con worktrees, aunque el usuario no mencione git, ramas ni PRs explícitamente.
---

# Flujo de trabajo: worktree → rama → PR → squash

## Por qué existe esta convención

En este proyecto pueden trabajar varios agentes en paralelo (por ejemplo desde Orca), cada uno en su propio git worktree. Hay un solo humano revisando, así que el pull request es el punto de control: es el momento en que el usuario ve el diff completo antes de que llegue a `main`.

La regla de fondo es simple: **una tarea = un worktree = una rama = un PR**. `main` queda siempre estable y deployable; todo lo que está a medias vive en ramas.

Además, los agentes suelen ejecutarse con permisos amplios (en Orca, Claude Code arranca con `--dangerously-skip-permissions`). El worktree aísla los archivos, pero no protege `main` ni el remoto: por eso las reglas de abajo son los límites que de verdad importan.

## Reglas fijas

1. **Nunca commitees ni pushees directo a `main`.** Todo cambio de un agente entra por rama + PR, aunque sea chico. Así el usuario siempre revisa antes de que algo llegue a `main`.
2. **Abrí PRs, pero no los mergees sin aprobación explícita del usuario en la conversación.** Decidir qué entra a `main` es del usuario. "Se ve bien" sobre el código no es "mergealo".
3. **Nunca hagas force-push a `main`.** En tu propia rama, usá solo `git push --force-with-lease` (falla si el remoto cambió sin que lo sepas, en vez de pisarlo).
4. **No commitees secretos ni artefactos generados**: `.env`, credenciales, tokens, `node_modules/`, `vendor/`, builds. Si algo así aparece en `git status`, frená y revisá el `.gitignore`.
5. **Quedate dentro del alcance de la tarea.** Tocar archivos no relacionados genera conflictos con los otros worktrees que corren en paralelo y ensucia la revisión.
6. **Pedí confirmación antes de cualquier comando destructivo**: `git reset --hard`, `git clean -fdx`, `git branch -D`, `git worktree remove`, borrar archivos fuera de tu worktree. Pueden borrar trabajo sin commitear, propio o de otro agente. Si los worktrees viven dentro del repo (`.worktrees/`), un `git clean -fdx` en el checkout principal los elimina a todos.

## 1. Al empezar una tarea

Verificá dónde estás antes de escribir una línea de código:

```bash
git branch --show-current   # no debería ser main
git worktree list           # en qué worktree estás
git status                  # árbol limpio
```

- **Si estás en `main`**, no empieces. Pedile al usuario que cree un worktree para la tarea desde Orca, o crealo vos:
  ```bash
  # con el CLI de Orca, si está disponible (command -v orca)
  orca worktree create --name <tipo>/<slug> --agent claude
  # o con git puro
  git fetch origin
  git worktree add ../<repo>-<slug> -b <tipo>/<slug> origin/main
  ```
- **Revisá el entorno.** Un worktree nuevo es un checkout limpio: los archivos gitignoreados (`.env`, dependencias, caches) no vienen solos. Comprobá que existan. Si faltan, mirá si el repo tiene `.worktreeinclude` y `orca.yaml` configurados (ver "Setup del repo"); si no los tiene, avisale al usuario y proponé agregarlos. No copies secretos a mano a lugares versionados.
- **Instalá dependencias** con el gestor del proyecto si hace falta (`npm install`, `pnpm install`, `composer install`, `pip install -r requirements.txt`).
- **Si existe `DESIGN.md` o un plan para la tarea, leelo antes de codificar.**

## 2. Nombre de la rama

Formato: `<tipo>/<descripcion-corta>` en kebab-case, sin tildes ni ñ, idealmente menos de 40 caracteres.

| Tipo | Para qué |
|---|---|
| `feat` | Funcionalidad nueva |
| `fix` | Corrección de bug |
| `refactor` | Cambio interno sin cambiar comportamiento |
| `docs` | Solo documentación |
| `test` | Tests nuevos o arreglados |
| `chore` | Mantenimiento, dependencias, config |
| `perf` | Mejora de rendimiento |

Ejemplos: `feat/export-csv`, `fix/123-login-race` (con número de issue si existe), `chore/actualizar-deps`.

En Orca, la rama se deriva del nombre del workspace: nombrá el workspace con este formato, o fijá el nombre en Advanced → Branch name.

## 3. Commits

Formato [Conventional Commits](https://www.conventionalcommits.org/):

```
<tipo>(<scope opcional>): <resumen en imperativo, máx. 72 caracteres>

<cuerpo opcional: por qué se hizo el cambio, no qué líneas cambiaron>
```

Ejemplos:
- `feat(facturas): agregar exportación a CSV`
- `fix(auth): evitar condición de carrera en el refresh del token`
- `refactor(api): extraer validación de requests a un middleware`

Idioma: español, salvo que el historial del repo ya esté en inglés; en ese caso seguí lo que haya.

Buenas prácticas:
- Commits chicos y en puntos estables. Los commits intermedios están bien: al hacer squash se aplastan en uno solo.
- Antes de cada commit, mirá `git status` y `git diff --staged`. Preferí `git add <rutas>` o `git add -p` antes que un `git add .` a ciegas.

## 4. Antes de abrir el PR

Recorré esta checklist completa:

1. **Sincronizá con `main`:**
   ```bash
   git fetch origin
   git rebase origin/main
   ```
   Rebase en lugar de merge: deja la rama lineal y es seguro porque la rama es tuya.
2. **Corré tests, lint y build.** Buscá los comandos en `package.json` (scripts), `composer.json` / `php artisan test`, `pytest`, `Makefile`. Si el proyecto no tiene tests, decilo en el PR en vez de inventar que pasaron.
3. **Revisá tu propio diff:**
   ```bash
   git diff origin/main...HEAD --stat
   git diff origin/main...HEAD
   ```
   Buscá código de debug (`console.log`, `dd()`, `var_dump`, `print`), TODOs nuevos, archivos que no corresponden a la tarea y cualquier secreto.
4. **Mirá el tamaño.** Apuntá a menos de ~400 líneas cambiadas. Un PR grande no se revisa bien; si la tarea creció mucho, proponele al usuario dividirla.

## 5. Abrir el PR

```bash
git push -u origin <rama>
gh pr create --title "<título en formato Conventional Commits>" --body-file <archivo>
# agregá --draft si el trabajo todavía no está listo para revisar
```

El título del PR va en formato Conventional Commits porque, con squash, se convierte en el mensaje del commit en `main`.

Cuerpo del PR (usá esta estructura siempre):

```markdown
## Qué
Resumen del cambio en 1-3 líneas.

## Por qué
Problema o necesidad que resuelve. Link a issue si existe (Closes #123).

## Cómo probarlo
Pasos concretos para verificarlo manualmente, y qué tests corriste.

## Notas
Decisiones tomadas, riesgos, limitaciones y cualquier hallazgo fuera de alcance.
```

Si el CLI de Orca está disponible, marcá el workspace como en revisión:
```bash
orca worktree set --worktree active --workspace-status in-review
```

Al terminar, informale al usuario: la URL del PR, un resumen de una línea y qué partes conviene que mire con más atención.

## 6. Mientras el PR está abierto

- **Cambios pedidos en la revisión:** commits nuevos en la misma rama y `git push`. No reescribas el historial para "limpiarlo": el squash lo hace al final.
- **`main` avanzó:** `git fetch origin && git rebase origin/main`, corré los tests de nuevo y `git push --force-with-lease`.
- **La CI falla:** investigá el log, arreglalo en la misma rama y explicá en el PR qué falló si no es obvio.

## 7. Merge y cierre (solo con aprobación explícita)

Cuando el usuario diga que se mergee:

```bash
gh pr merge --squash --delete-branch
```

Después:
- El worktree se elimina desde Orca (o con `git worktree remove <ruta>`), con confirmación del usuario.
- Si hay otros worktrees activos, avisale al usuario que `main` cambió y que conviene rebasearlos antes de seguir.

## 8. Resolver conflictos

```bash
git fetch origin
git rebase origin/main
# resolvé archivo por archivo, entendiendo ambos lados del conflicto
git add <archivo>
git rebase --continue
# si te perdés: git rebase --abort  (vuelve al estado previo)
```

Si un conflicto toca lógica del otro lado que no entendés, frená y preguntale al usuario en lugar de elegir una versión al azar. Al terminar, corré los tests y `git push --force-with-lease`.

## 9. Hallazgos fuera de alcance

Si encontrás un bug o una mejora no relacionada con la tarea, no lo arregles en esta rama. Anotalo en la sección "Notas" del PR, mencionáselo al usuario y sugerí un worktree aparte para eso.

## 10. Excepciones

El usuario (no los agentes) puede commitear directo a `main` cambios triviales: typos, README, comentarios. Si el usuario te pide explícitamente commitear algo directo a `main`, hacelo, pero recordale en una línea que la convención es pasar por PR.

## Setup del repo (una sola vez)

Aplicá esta sección cuando el usuario pida preparar un repo, o cuando detectes que faltan estas piezas. Mostrale los cambios antes de aplicarlos; los de GitHub, solo con su confirmación.

1. **`orca.yaml`** en la raíz: directorios gitignoreados grandes y reconstruibles que se comparten entre worktrees.
   ```yaml
   worktree:
     sharedDirectories:
       - node_modules
       - vendor
   ```
   Solo funcionan entradas que existan en el checkout principal y estén en `.gitignore`.
2. **`.worktreeinclude`** en la raíz: archivos gitignoreados que se copian a cada worktree nuevo.
   ```
   .env
   ```
3. **`.gitignore`** con `.env`, `node_modules/`, `vendor/` y, si los worktrees se crean dentro del repo, `.worktrees/`.
4. **`.github/pull_request_template.md`** con la estructura de la sección 5.
5. **Ajustes del repo en GitHub:**
   ```bash
   gh repo edit --enable-squash-merge --enable-merge-commit=false \
     --enable-rebase-merge=false --delete-branch-on-merge
   ```
   Protección de `main` (Settings → Rules): exigir PR y que pasen los status checks. **No exijas aprobaciones**: GitHub no permite que el autor apruebe su propio PR y, trabajando solo, quedarías bloqueado.
6. **Comando de setup en Orca** (configuración del repositorio): por ejemplo `npm install` o `composer install`, para que cada worktree nuevo quede listo.
7. **`AGENTS.md`** (lo leen Codex y otros agentes que no cargan skills de Claude): agregá una línea que resuma la convención — "una tarea = un worktree = una rama = un PR; nunca commitear a main; no mergear sin aprobación del usuario; Conventional Commits; squash merge".

## Resumen rápido

| Momento | Qué hacer |
|---|---|
| Empezar | Verificar que no estás en `main`, entorno completo, leer `DESIGN.md` si existe |
| Rama | `<tipo>/<slug>` |
| Commit | `tipo(scope): resumen`, revisando `git diff --staged` |
| Antes del PR | rebase sobre `origin/main`, tests, auto-revisión del diff, < ~400 líneas |
| PR | título convencional, cuerpo Qué / Por qué / Cómo probarlo / Notas |
| Merge | solo con aprobación explícita, squash, borrar rama y worktree |
| Conflictos | rebase, entender ambos lados, preguntar si hay dudas, `--force-with-lease` |
