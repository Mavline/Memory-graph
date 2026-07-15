# Инструкция по созданию структуры и памяти проекта

Status: переносимая модель для новых проектов и машин.

Цель этой инструкции — создать проект, где код, стратегические документы,
проверки, память и история решений не расходятся между собой. Проект должен
оставлять чистые хвосты данных: никаких безликих handoff/evidence/status
markdown/json файлов, никаких параллельных «памятей», никаких устных
отклонений от записанного.

## Главный принцип

В проекте есть ровно три рода знания, и у каждого одно место хранения:

1. **Стратегические документы** в `docs/` — что строим и какие
   продуктовые, архитектурные и бизнес-решения приняты, включая порядок
   этапов и дорожную карту. Канонический документ один — `docs/spec.md`.
   Дополнительных стратегических документов может быть сколько нужно, но
   каждый подчиняется правилам раздела «Стратегические документы».
   Тематические рабочие документы агентов (выводы, поиски, исследования)
   живут в утверждённом один раз канале `docs/knowledge/` — см. тот же
   раздел.
2. `.agent/memory.db` — живая процессная память через `memgraph`: задачи,
   статусы, доказательства, блокеры, решения, gotchas, review verdicts,
   события workflow и embedding/FTS-поиск.
3. `AGENTS.md` — единственный рантайм-промпт: как работать (процессы памяти,
   проверки, ролей, git-дисциплины, иерархии источников). `CLAUDE.md`
   содержит ровно одну строку-указатель `@AGENTS.md`.

Всё остальное — это либо код, либо тесты, либо краткие runtime-инструкции для
агентов. Если информация процессная, она идёт в `.agent/memory.db`. Если это
продуктовое или техническое решение проекта, она идёт в стратегические
документы. Если это правило того, как агент обязан работать, оно идёт в
`AGENTS.md`. Нельзя дублировать одно и то же в нескольких местах с разными
формулировками.

## Почему память не используется сама собой

Правильных правил мало. Память можно описать идеально и всё равно получить
агента, который в неё не пишет и из неё не читает. Проактивное использование
держится на трёх взаимоусиливающих механизмах; убери любой — и память тихо
отмирает:

1. **Подача.** На старте каждой сессии агент должен выполнять
   repo-local wrapper (`memgraph/memgraph session-context` на macOS/Linux или
   `powershell.exe -NoProfile -ExecutionPolicy Bypass -File memgraph/memgraph.ps1 session-context` на Windows) и получать срез
   памяти в контекст. Для Claude это делается репозиторным SessionStart-хуком в
   `.claude/settings.json`; для Codex — правилом в проектном `AGENTS.md` и тем
   же wrapper-command на старте сессии или через локальный hook, если он
   доступен. Без этой подачи свежий агент (а в
   ролевом цикле каждая роль стартует с чистого листа) не видит, что память
   существует, и не вспоминает прежние указания — пользователю приходится
   повторять одно и то же. Startup-интеграция должна быть **репозиторной**, а не
   только в глобальном конфиге машины: глобальный hook или локальная привычка не
   переезжают на другую машину вместе с репозиторием.
2. **Вознаграждение.** `recall` должен находить по смыслу, а не только по
   точному слову. С семантическим поиском запрос «почему узел зависает» находит
   запись, где сказано иначе; с одним лишь FTS точное слово не угадывается,
   `recall` отдаёт пустоту, и агент делает вывод, что память бесполезна, — и
   перестаёт ею пользоваться. Пустой recall обучает игнорировать память.
3. **Принуждение.** Явное правило записи «по ходу работы» (см. шаблон
   `AGENTS.md`, блок MEMORY WRITE CADENCE) с перечнем моментов записи. Без него
   агент копит всё на финал и обычно не пишет вовсе.

Эти три механизма закладываются при bootstrap, а не добавляются после того, как
выяснилось, что агент памятью не пользуется. Признаки отсутствия одного из них:
агент пишет только «по крику», переспрашивает уже сказанное, ведёт карту системы
в голове, а не в памяти.

## Разделение поверхностей: `CLAUDE.md`, `AGENTS.md`, стратегические документы

Три поверхности не пересекаются по содержанию. Это правило защищает от
расхождения инструкций между агентами и от расползания проектной информации в
рантайм-промпт.

- **`CLAUDE.md`** — ровно одна строка: `@AGENTS.md`. Никаких дублирующих
  правил, никакой отдельной памяти, никакого Claude-специфичного форка
  инструкций. Тексты для агентов в `CLAUDE.md` не пишутся: он только
  указывает на `AGENTS.md`.
- **`AGENTS.md`** — только указания по процессам работы: память, проверка,
  роли, git, иерархия источников, запрет догадок, дисциплина продолжения.
  Промпт должен быть project-agnostic. Продуктовая, архитектурная,
  контентная, платформенная и бизнес-информация, а также дорожная карта в
  `AGENTS.md` **не попадают** — для них есть стратегические документы. Если
  правило про продукт оказалось в `AGENTS.md`, это ошибка размещения.
- **Стратегические документы** (`docs/*.md`) — вся проектная информация: что
  строим, какие решения приняты, этапы, ограничения, контракты. Сюда не
  пишется процессное состояние (статусы, доказательства, вердикты) — оно в
  memgraph.

## Стратегические документы (один и более)

Канонический стратегический документ — `docs/spec.md`. Он обязателен.

Дополнительные стратегические документы разрешены без ограничения по числу,
но каждый обязан соблюдать четыре условия:

1. **Явное утверждение.** Отдельный документ создаётся только когда
   пользователь его утвердил. Агент не плодит стратегические документы по
   своему усмотрению.
2. **Связанность.** Канонический `docs/spec.md` ссылается на каждый
   дополнительный документ и объясняет, зачем он существует.
3. **Одна зона ответственности.** У документа одна чёткая тема (например,
   карта ключевых слов, индекс фаз, контракт работы агентов, PRD конкретной
   фазы). Темы не пересекаются между документами.
4. **Без дублирования и без процессного состояния.** Документ не повторяет
   содержимое других стратегических документов и не хранит статусы, вердикты,
   блокеры или транскрипты — это уходит в memgraph.
5. **Только дописывание, без затирания истории.** Стратегический документ — это
   растущая чистая карта текущего состояния, а не «активный контекст», который
   переписывается при каждом изменении. Прошлые находки не удаляются ради
   «чистоты»: опровергнутый факт помечается на месте `[SUPERSEDED <дата UTC>:
   почему]` с указателем, а вытесненное рассуждение и сам пивот уходят в
   memgraph (`write-decision`). В конце документа ведётся датированный журнал
   ревизий: каждая существенная правка добавляет одну строку (дата UTC, что
   изменилось, доказательство). Чистая карта и сохранённый аудит-след
   сосуществуют: карта показывает «как сейчас», журнал и memgraph — «как к этому
   пришли». Вести документ как один перезаписываемый снимок, стирающий историю,
   запрещено.
6. **Живой документ, а не «написал и забыл».** Если документ — это план,
   декомпозиция или дерево узлов, по которому строится документация (например
   реверс-инжиниринг системы), он дополняется в тот же ход, когда разобран
   очередной узел, и не отстаёт, пока движутся `spec.md` или memgraph.

Дорожная карта и этапность — это тоже стратегическое знание: либо раздел
`docs/spec.md`, либо отдельный утверждённый документ (например `phase-index.md`
плюс `prd-phase-*.md`). Отдельный обязательный файл `plan.md` не требуется;
важно, что план — это стратегический документ, а не процессная запись.

Реальные образцы такой модели:

- один канонический `docs/spec.md` плюс утверждённый `docs/keyword-map.md`,
  на который `spec.md` ссылается и который отвечает только за ключевые слова и
  объём страниц первого релиза;
- один канонический `docs/spec.md` плюс `docs/phase-index.md`,
  `docs/AI_AGENT_OPERATING_CONTRACT.md`, `docs/document-cross-reference-map.md`
  и набор `docs/prd-phase-*.md`, где этапность вынесена в фазовые документы под
  навигатором.

Что входит в стратегические документы:

- продуктовый или системный объём;
- non-goals и запрещённый объём;
- уже принятые архитектурные решения;
- модель данных и контракты API, только когда проверены или явно помечены как
  предварительные;
- границы безопасности, приватности, авторизации, хранения, деплоя, рантайма;
- обещания, видимые пользователю, и запрещённые заявления;
- правила приёмки, которые должны пережить отдельные задачи;
- этапы, дорожная карта, зависимости, гейты приёмки фаз и условия остановки.

Чего в стратегических документах быть не должно:

- статус задач и «текущий прогресс»;
- review verdicts;
- транскрипты команд;
- handoffs;
- временные блокеры;
- черновые заметки агентов.

Если процессный факт стал устойчивым правилом, его повышают из memgraph в
стратегический документ или в `AGENTS.md` намеренно, отдельным коммитом.

### Тематические рабочие документы (`docs/knowledge/`)

Помимо стратегических документов проекту разрешён один канал тематических
рабочих документов — `docs/knowledge/**/*.md`: выводы, результаты поисков,
исследования, технические разборы и накопленные знания, которые агент ведёт
сам. Канал утверждается один раз — принятием этой инструкции; отдельное
утверждение каждого файла не требуется, и ссылка из `docs/spec.md` не
обязательна. Индексируются эти файлы тем же жизненным циклом документов, что и
остальные `docs/**` (см. «Индексация документов»): роль источника —
`thematic_doc`, полный вес авторитетности `1.0` в recall.

Правила каждого тематического файла:

- одна устойчивая тема на файл; темы не пересекаются между файлами;
- содержательные Markdown-заголовки — они же границы фрагментов индекса;
- фиксируются выводы, основания, источники и оставшиеся неизвестные;
- ссылки на файлы проекта — только относительные от корня репозитория;
- файл — курируемое текущее знание. В отличие от стратегических документов
  правило «только дописывание» здесь не действует: устаревший вывод не должен
  конкурировать в поиске с актуальным. Историю правок хранит Git, а значимые
  опровержения фиксируются типизированными записями (`write-claim`,
  `write-decision`).

Запрещено в тематических файлах: статусы задач, handoff и прогресс; сырые логи
и транскрипты; временные рабочие заметки; неподтверждённые выводы,
сформулированные как принятые решения.

Тематический документ не переопределяет стратегию, policy или decision, а его
фрагмент в результатах recall — поисковый указатель на источник, не
подтверждённый факт. Повышение знания всегда явное: тематический документ,
затем проверка, затем `write-claim` или `write-decision`, затем — при
необходимости — изменение `docs/spec.md` или планового документа.

## Что значит «помнить всё»

Помнить всё не значит складывать сырые логи в проект. Каноническую память
проекта, наоборот, нужно коммитить: `.agent/memory.db` является переносимым
состоянием проекта для работы с разных машин. Если память не попадает в git,
новая машина получает код без текущих решений, статусов, блокеров и истории
проверок.

Помнить всё значит, что каждое workflow-значимое действие получает структурную
запись в `.agent/memory.db`:

- открытие и закрытие задачи;
- frozen scope и owned/forbidden files;
- команды проверки, exit codes и краткие выводы;
- review verdicts `PASS`, `FAIL`, `UNKNOWN`;
- блокеры и точное условие разблокировки;
- decisions, gotchas, assumptions, risks;
- commit SHA, push status и следующий executable step.

Сырые артефакты не становятся новой памятью по умолчанию. Если большой лог,
скриншот или внешний отчёт действительно нужен для будущего аудита, он хранится
вне репозитория или в явно утверждённом artifact store, а в memgraph пишется
краткая запись с указателем, checksum или другим воспроизводимым
идентификатором.

Если проект по явному решению владельца коммитит доказательства и досье в
`docs/` (auditable-режим), их содержимое включается в поиск командой
`ingest-docs` (см. раздел «Индексация документов»), а не
дублируется в типизированных записях: указатель-вердикт остаётся краткой
записью, полный текст ищется как фрагменты документа.

## Целевая структура

Минимальная структура нового проекта:

```text
.
├── AGENTS.md                       # единственный рантайм-промпт: как работать
├── CLAUDE.md                       # ровно одна строка: @AGENTS.md
├── docs/
│   ├── spec.md                     # обязательный канонический документ
│   ├── knowledge/                  # тематические рабочие документы агентов (канал утверждён один раз)
│   └── <дополнительные>.md         # сколько нужно; каждый утверждён и связан из spec.md
├── .agent/
│   ├── memory.db                   # процессная память (memgraph)
│   └── tasks/
│       └── <TASK_ID>/
│           └── spec.md             # замороженная спецификация задачи
├── .env.example
└── .gitignore
```

Разрешено в `.agent/`:

```text
.agent/memory.db
.agent/tasks/<TASK_ID>/spec.md
```

Запрещено в `.agent/`: evidence.md, evidence.json, verdict.json, problems.md,
lead-review.md, memory-recall.md, orchestrator-boot.md, status.md, handoff.md,
raw/, screenshots/, reports/. Если агенту нужно доказательство, статус, review
verdict, blocker, handoff или итог шага, он пишет это в `.agent/memory.db`
через `memgraph`, а не создаёт очередной файл.

## `AGENTS.md` — обязательный шаблон

`AGENTS.md` — единственный runtime prompt проекта. В него пишутся только
правила исполнения, памяти, проверки, ролей, git-дисциплины и иерархии
источников. Не превращать его в PRD, roadmap или продуктовый справочник:
проектная информация живёт в стратегических документах.

Шаблон ниже project-agnostic — копируется в новый проект целиком; в нём
заменяются только имена стратегических документов в `Source Hierarchy` и в
`Startup Discipline` под фактический набор `docs/*.md` проекта.

```markdown
# Agent Operating Instructions

FIRST RULE: operational state, work progress, evidence, blockers, review
verdicts, status, handoffs, gotchas, and run memory live only in
`.agent/memory.db` through `memgraph`. Do not write operational state into
specs, PRDs, control docs, role files, or `.agent/tasks/<TASK_ID>/spec.md`; the
task spec is only the initial frozen plan before coder handoff.

CLAUDE_POINTER_RULE: `CLAUDE.md` is not a duplicate runtime prompt. It must
contain exactly one line: `@AGENTS.md`. `AGENTS.md` is the canonical editable
agent instruction file. When these instructions change, update `AGENTS.md`
only.

COLLABORATION RULE: when more than one agent works the project, fix the
lead/reviewer and orchestrator/executor lanes explicitly and keep the tool
boundary asymmetric if the setup requires it. Coordination that the user has
not delegated to direct agent-to-agent calls passes through the user as concise
assignments, corrections, reviews, or next-step instructions. No agent kills or
manages another agent's processes unless the user explicitly asks.

FLOW CODE RULE: all new task titles, task specs, handoffs, closeouts, reviews,
blockers, reports, and next-step instructions use a single human-readable
`[wf-code=####]` as the only workflow identifier. The code is assigned by the
user's instruction, the handoff, or the lane's own sequence; it is never
derived from memgraph numeric workflow IDs, run IDs, object IDs, or other DB
numbers. The project may define its own prefix semantics (for example platform
work `1###`, content/spec work `2###`). If a DB tool emits numeric identifiers
while handling `[wf-code=####]`, keep them internal and continue using only
`[wf-code=####]`.

Status: project-agnostic runtime instructions. Project-specific product,
architecture, content, platform, business rules, and roadmap belong in the
repository source documents (`docs/spec.md` and approved additional documents),
not in these runtime prompt files. Until the user provides the main concept,
missing product/spec docs are `UNKNOWN`, not permission to invent them.

DOCUMENTATION DISCIPLINE: do not create sidecar planning, note, scratch,
status, evidence, blocker, or "analysis" files just to preserve working
context. Working context and operational memory go to `.agent/memory.db`
through `memgraph`. Durable product decisions are consolidated into the
canonical source document `docs/spec.md`, unless the user explicitly approves a
separate source document; an approved separate document must be linked from
`docs/spec.md` with a one-line reason. The one standing exception is the
thematic-knowledge channel `docs/knowledge/**/*.md`: curated topical findings,
research results, and investigation write-ups may be created there without
per-file approval, under the thematic rules in STRATEGIC-DOCUMENT DISCIPLINE.
Otherwise consolidate or delete the sidecar before commit.

MEMORY WRITE CADENCE: write to `.agent/memory.db` through `memgraph` DURING the
work, not as a closing summary. Each time, citing concrete evidence (file path,
command + exit code, log line, source reference), record at: run start/resume;
after any verification, observation, or experiment (`write-claim`); on a
non-trivial decision or pivot, or when a prior finding is overturned
(`write-decision` with what it supersedes); on a trap or gotcha
(`write-claim --type gotcha|risk`); on a source conflict; before any handoff,
self-review, or closeout; and on every `wf-status` transition. A turn that
produced a finding, a verdict, or a pivot but wrote nothing to memgraph is
incomplete; deferring all writes to the end is a violation. `recall` is hybrid
(semantic + keyword) — query by meaning, and never treat a keyword miss as proof
that nothing was recorded.

DOCUMENT INDEX LIFECYCLE: every create, update, move, or delete of a managed
document (`docs/**/*.md`, `AGENTS.md`, `.agent/tasks/**/spec.md`) finishes in
the same work unit as: commit the document, run `memgraph ingest-docs`, run
`memgraph verify-doc-index`, then commit the updated `.agent/memory.db` and
push both commits. A committed document whose content is absent from the index
is an unfinished tail. The engine enforces the commit boundary: production
ingest/verify require Git and a clean managed corpus, while `--dry-run`
previews drafts and reports `committable=false`; untracked gitignored files
are never indexed and are listed as `ignored_managed_paths`; paths that are or
cross a symlink are never indexed (Git commits the link, not the target bytes)
and are listed as `symlinked_managed_paths`. An empty corpus
is a healthy state — `verify-doc-index` reports ok with `corpus_empty=true` —
and sources retained only as fully de-indexed evidence tombstones are listed
separately in `retained_evidence_sources`.

STRATEGIC-DOCUMENT DISCIPLINE: durable system knowledge is split across the
approved strategic documents, never dumped into one rolling file. `docs/spec.md`
is the single clean current map; a topic that outgrows a section gets its own
approved document, linked from `docs/spec.md`; using the documents that already
exist is mandatory. Never delete or overwrite a past finding to tidy up: mark an
overturned fact in place `[SUPERSEDED <UTC date>: why]`, send the displaced
reasoning to memgraph, and append a one-line dated entry to the document's
revision log. A decomposition or plan document is a living deliverable, updated
the same turn a node is resolved — not a write-once file. Thematic knowledge
files under `docs/knowledge/` are the one channel outside this append-only
rule: keep each as curated current knowledge on a single stable topic (history
lives in Git; overturned findings become typed claims/decisions), never as
task status, handoffs, raw logs, or scratch notes, and never as a place that
overrides strategy, policies, or decisions.

## Startup Discipline

Before non-trivial work:

1. Run `git status --short --branch`.
2. If `.agent/memory.db` exists, run the repo-local wrapper:
   `memgraph/memgraph session-context` on macOS/Linux or
   `powershell.exe -NoProfile -ExecutionPolicy Bypass -File memgraph/memgraph.ps1 session-context` on Windows.
3. Read the source documents that exist, in source-hierarchy order.
4. Classify the lane, branch, worktree, open task/run records, dirty files,
   owned files, forbidden files, and the next safe action.

Navigation documents do not override the source hierarchy.

## Continuation Discipline

If the project is not complete and no active task remains, do not end with a
passive status-only response. State the next safe options for continuation,
ordered by practical priority, and identify which option can start immediately.

Every readiness, status, blocker, or "not complete" report must include the
next executable step or concrete handoff prompt in the same response. A
status-only answer is invalid unless the project is complete or a real blocker
prevents any next action.

After any analysis, review, or completed stage, do not wait for the user to ask
"what next?". End with one of: the next action already being taken; a concrete
ready-to-forward assignment for the correct executor; or the shortest specific
question truly required to unblock. If a minor correction or hygiene fix is safe
to do directly, do it instead of turning it into a handoff.

Do not leave tails: when a logical task changes files, memory records, or
generated assets, finish the loop in the same turn by verifying, recording the
result, removing superseded clutter, committing, pushing, and naming the next
concrete action. If a real blocker prevents that, report the blocker and the
exact remaining command or prerequisite.

## Source Hierarchy

Use this order:

1. User's latest explicit instruction.
2. `docs/spec.md`.
3. Approved additional strategic documents, in their declared order.
4. `AGENTS.md`.
5. Frozen `.agent/tasks/<TASK_ID>/spec.md`.
6. Memgraph recall as supporting context only.

If sources conflict, record the conflict in `.agent/memory.db` and stop unless
a higher-priority source resolves it.

## No Guessing

Do not invent values, routes, schemas, tests, roles, keys, product decisions,
option names, command flags, output paths, package behavior, DB columns, API
signatures, site purpose, or business claims.

Concrete technical claims are binding only when verified in the current session
by one of: a repository file read now; installed source or types read now; a
command run now with stdout/stderr and exit code; official documentation
fetched now. Memory, prior chat, old plans, examples, and model knowledge are
not verification. Missing evidence is `UNKNOWN` or a blocker. `UNKNOWN` blocks
approval.

High-risk work needs feasibility proof before freeze: build pipeline, runtime
adapter, deployment target, package resolution, external runtime, data access,
DB connectivity, storage, auth, cron/scheduler, and generated-output paths.
Paper plans are not proof. If two planned cycles are coupled, stop and supersede
the plan before opening more implementation work.

## Role Cycle

For every non-trivial logical task:

```txt
orchestrator -> task-spec-freezer -> coder -> integration test runner ->
fresh reviewer -> fixer if needed -> impacted tests -> fresh reviewer ->
logical reviewer-lead
```

The cycle ends only on blocker or completion of the declared logical block
after required verification, review, memgraph status, and commit/push when
tracked files changed. Do not stop after a role stage, command, micro-step, or
routine next action. Stop only for a real blocker: missing product decision,
missing secret, destructive/out-of-scope approval, unresolved source conflict,
missing runtime role, or unprovable high-risk boundary.

## Role Boundaries

- Orchestrator: reads sources, opens/closes memgraph task/run records, creates
  the initial frozen spec, classifies lanes and blockers.
- Task-spec-freezer: writes only the task spec before coder handoff.
- Coder/builder: edits only owned implementation files listed in the frozen
  spec.
- Test-runner/verifier: read-only for repo files; records evidence in memgraph.
- Reviewer: fresh context; read-only; assigns only `PASS`, `FAIL`, or
  `UNKNOWN`.
- Fixer: edits only directly affected owned files needed to resolve recorded
  findings, then stops for independent tests.
- Logical reviewer-lead: read-only final reasoning check and memgraph status.

No role may normalize a role-boundary violation into `PASS`. The builder must
not collect or package reviewer evidence. Fixer sanity checks do not replace
independent impacted tests.

## Memory And Evidence

Use repo-local memory through the memgraph wrapper only. Never edit
`.agent/memory.db` manually. Commit `.agent/memory.db` as the canonical portable
project memory; under `.agent/tasks/` commit only `<TASK_ID>/spec.md`.

Allowed `.agent/` layout:

```txt
.agent/
  memory.db
  tasks/<TASK_ID>/spec.md
```

Do not create markdown/json handoff, evidence, verdict, blocker, review,
lead-review, memory snapshot, or raw-output files under `.agent/`.

Evidence must cite concrete proof: file paths, commands, exit codes, output
summaries, HTTP traces, SQL smoke results, browser checks when relevant, and
the active `[wf-code=####]`. For mutating or generated commands, record
`git status --short` before and after.

## Memgraph Output Hygiene

Never stream raw memgraph stdout/stderr into the transcript. Raw output may
contain service DB identifiers, field names, handles, or diagnostics that are
not valid authored text. Use a sanitized wrapper when available, or redirect
raw output to a private sink and print only `[wf-code=####] <operation>:
ok|FAIL`.

## Git Discipline

Assume dirty work may belong to the user or another lane. Do not revert or
overwrite changes you did not make. Classify dirty files before editing. Use
destructive git commands only when the user explicitly asks for that exact
operation. Keep unrelated changes out of commits.

Commit and push are part of done: after every completed logical change with
repository changes, stage only the intended files, commit them, and push in the
same turn. Do not leave completed work local-only unless a real blocker exists
(missing remote access, failed required verification, unresolved ownership
conflict, or an explicit latest user instruction not to push).

## Runtime And Infrastructure

Do not start, stop, install, pull, bootstrap, deploy, tunnel, or bind
long-running services unless the user's latest instruction or tracked control
docs explicitly authorize that exact action. Read-only detection is allowed and
does not grant permission to start missing infrastructure. If a required
environment is absent, record `UNKNOWN/BLOCKER` with the exact prerequisite.

## Review Standard

Reviewer verdicts are only `PASS`, `FAIL`, or `UNKNOWN`. Missing evidence is
`UNKNOWN`. Code inspection alone cannot prove runtime, build, HTTP, DB, storage,
browser, or deployment behavior when the acceptance criterion requires those
boundaries. Before claiming completion, rerun the required verification and
report the commands, exit codes, changed files, active `[wf-code=####]`,
skipped/unknown checks, and residual risk.
```

Проектные дополнения к `AGENTS.md` допускаются только когда это настоящие
правила работы. Продуктовый объём, архитектура, цены, контентные правила,
выбор деплоя и бизнес-решения принадлежат стратегическим документам.

## `CLAUDE.md`

`CLAUDE.md` содержит ровно одну строку:

```markdown
@AGENTS.md
```

Никаких дублированных правил, никакого Claude-специфичного форка, никакой
отдельной политики памяти, никаких текстов для агентов. Цель — исключить
расхождение между поверхностями инструкций.

## Frozen task spec

Каждая нетривиальная логическая задача получает один замороженный файл:

```text
.agent/tasks/<TASK_ID>/spec.md
```

Он содержит: task id; прочитанные source-документы; точные owned files;
forbidden files; ожидаемое поведение; модель production-исполнения; явные
failure cases; требуемые тесты и проверки; evidence, который должен быть
записан в memgraph; условия блокера.

После старта реализации замороженная спецификация только для чтения. Если она
неверна — остановить задачу, записать блокер в memgraph и создать новую
задачу/спецификацию. Не править спецификацию задним числом, чтобы уже
написанный код выглядел корректным.

## Memgraph: установка и использование

Память идёт через единую обёртку-CLI с именем `memgraph` (или
проектно-локальный эквивалент). Обёртка владеет типизированными таблицами
памяти, FTS-индексом, векторным индексом sqlite-vec, метаданными эмбеддингов,
хешами текста эмбеддинга и транзакционной записью.

Инструмент памяти — это скилл `memgraph` (обёртка-CLI над каноническим движком).
Его НЕ копируют по файлам и НЕ ищут по машине — он либо уже установлен в
окружении агента, либо приложен этим комплектом рядом с этой инструкцией
(`./memgraph/`). Порядок:

1. Взять скилл: если в окружении уже есть memgraph-скилл — использовать его;
   иначе развернуть приложенный `./memgraph/` по его `install/README.md`.
   Вызывается обёртка (`memgraph` для macOS/Linux, `memgraph.ps1` для Windows),
   а не внутренний `scripts/*.py` напрямую.
2. Зависимости ставит сам скилл по своему `requirements.txt` (`openai`,
   `sqlite-vec` фиксированной версии, `tiktoken`); нативное расширение
   `sqlite-vec` вендорится рядом с обёрткой (`vendor/`) и пересобирается на
   каждой машине.
3. Переменная окружения `OPENAI_API_KEY` — единственная строго обязательная;
   нужна для `recall` и записей с эмбеддингом. Чисто-SQL команды
   (`session-context`, `next-wf` и подобные) работают без неё.
4. Подключить startup-интеграцию, которая на старте сессии в репозитории с
   `.agent/memory.db` выполняет repo-local wrapper и вкладывает вывод в
   контекст: `memgraph/memgraph session-context` на macOS/Linux или
   `powershell.exe -NoProfile -ExecutionPolicy Bypass -File memgraph/memgraph.ps1 session-context` на Windows. Claude подключается
   через `.claude/settings.json`, Codex — через проектный `AGENTS.md` или
   доступный локальный hook.

Модель эмбеддингов по умолчанию: `text-embedding-3-small`, 512 измерений.
Модель и размерность не меняются ad hoc.

Правила:

- `memgraph/memgraph session-context` на macOS/Linux или
  `powershell.exe -NoProfile -ExecutionPolicy Bypass -File memgraph/memgraph.ps1 session-context` на Windows в начале работы;
- точечный `memgraph/memgraph recall "<query>"` перед нетривиальной работой;
- write-хелперы для decisions, claims, gotchas, workflow status, reviews,
  blockers и принятого завершения задачи;
- никогда не писать строки в БД вручную;
- если эмбеддинги или запись падают — остановиться перед нетривиальной работой,
  если пользователь явно не принял write-ограниченную сессию.

Категории записи: `decision` (принятое архитектурное, продуктовое, провайдерское,
схемное или процессное решение); `claim` (статус, требование, ограничение,
риск, наблюдение, gotcha); `workflow_task` (жизненный цикл задачи); `run`
(сгруппированный жизненный цикл исполнения); `review_gate` (вердикт и findings
ревьюера); `orchestration_event` (assignment, checkpoint, blocker, completion,
handoff).

### Индексация документов (`ingest-docs`) и их жизненный цикл

Закоммиченные документы проекта — это источники, а не процессная память. Чтобы
`recall` находил их содержимое по смыслу, комплект содержит команду
`memgraph/memgraph ingest-docs`: она нарезает Markdown по заголовкам на
смысловые фрагменты (длинные секции дробятся по границам абзацев) и кладёт их в
сырой слой той же `.agent/memory.db` (`sources` + `chunks`) с проекцией в FTS
и векторный индекс.

Жизненный цикл: любое событие над управляемым документом — создан, изменён,
перемещён, удалён — завершается в том же рабочем блоке цепочкой «коммит
документа, `ingest-docs`, `verify-doc-index`, коммит обновлённой
`.agent/memory.db`, push обоих коммитов». Закоммиченный документ, содержимого
которого нет в индексе, — незакрытый хвост.

Правила:

- охват по умолчанию: `docs/**`, `AGENTS.md`, замороженные
  `.agent/tasks/**/spec.md`; `CLAUDE.md` и `memgraph/**` не индексируются;
- классификация источников: `AGENTS.md` получает роль `runtime_prompt`,
  `docs/evidence/` — `evidence`, `docs/knowledge/` — `thematic_doc`
  (тематические рабочие документы, полный вес авторитетности `1.0`), остальные
  `docs/` — `strategic_doc`, `.agent/tasks/**/spec.md` — `task_spec`.
  Классификация — производное состояние: при изменении правил она освежается
  даже у неизменённых файлов, без переэмбеддинга (смена категории переписывает
  только теги фрагментов);
- граница коммита: боевой запуск `ingest-docs` и `verify-doc-index` требует
  Git и чистого управляемого корпуса относительно `HEAD` — включая
  неотслеживаемые, удалённые и переименованные файлы; `--dry-run` работает на
  черновиках и без Git, но возвращает `committable=false`. Неотслеживаемые
  файлы под правилами `.gitignore` не индексируются и перечисляются в
  `ignored_managed_paths`; уже отслеживаемый файл остаётся в корпусе, даже
  если позднее попал под правило игнорирования;
- символические ссылки — вне воспроизводимого корпуса: Git коммитит саму
  ссылку, а не байты цели, поэтому управляемый путь, который является ссылкой
  или проходит через неё (даже чистой и отслеживаемой), никогда не
  индексируется. Идентичность источника — буквальный путь в репозитории,
  никогда не развёрнутая цель ссылки; такие пути перечисляются в
  `symlinked_managed_paths`;
- обновление инкрементально: файл с неизменным хешем пропускается, неизменные
  секции переиспользуются без переэмбеддинга, устаревшие фрагменты удаляются,
  а использованные как evidence — сохраняются, но полностью выводятся из
  индекса (надгробие для аудита: без строк в `index_docs`, FTS, векторном
  индексе и метаданных эмбеддинга);
- фрагменты не становятся типизированными записями автоматически: решения и
  клеймы по-прежнему пишутся вручную после проверки;
- одна память на репозиторий: **все** команды отказываются работать, когда
  `MEMGRAPH_DB` указывает за пределы `<repo>/.agent/memory.db`; обычный
  `--paths` только сужает управляемый набор и никогда его не расширяет —
  явный файл вне управляемой области (или вне репозитория) — это ошибка,
  каталог разворачивается только в свои управляемые `**/*.md`. Граница
  коммита и верификатор покрывают ровно управляемую область, поэтому обойти
  их через выбор путей нельзя;
- явное исключение по приказу владельца — разовый импорт внешнего Git-архива:
  `memgraph/memgraph ingest-docs --external-root <archive-git-root>` берёт ровно
  его tracked `*.md`, сохраняет абсолютный путь как provenance-ссылку и кладёт
  чанки в ту же repo-local `.agent/memory.db` с ролью
  `external_memory_archive`. Это не федерация и не вторая память: внешний текст
  остаётся низкоавторитетным источником и не создаёт автоматически decisions,
  claims, policies или workflow state;
- tracked Markdown внешнего архива должен быть чистым относительно `HEAD`;
  импорт сохраняет проверенный commit SHA в `meta`, чтобы можно было доказать,
  какие именно байты попали в проектную память;
- полнота проверяется командой
  `memgraph/memgraph verify-doc-index --external-root <archive-git-root>`:
  обязательны 1:1 охват без лишних архивных источников, совпадение текущего
  file hash, пересчитанных chunk/embedding-text hashes и archive revision, а
  также наличие каждого активного чанка в `index_docs`, FTS, `memory_vec` и
  `embedding_meta`; результаты `recall` для
  чанков возвращают явные `source_path`, `source_role` и `source_category`;
  чанки `external_memory_archive` получают прозрачный коэффициент
  авторитетности `0.75` после RRF (сырой `raw_score` сохраняется), чтобы близкое
  по смыслу актуальное доказательство проекта ранжировалось выше старого
  внешнего архива;
- без `--external-root` тот же verifier проверяет текущий default-набор
  документов репозитория. Пустой корпус — исправное состояние: `ok: true` с
  признаком `corpus_empty: true`. Оставшийся активный, частично
  проиндексированный или бесхозный управляемый источник — провал с
  перечислением в `unexpected_sources`. Корректные надгробия evidence
  (каждый фрагмент реально используется как доказательство, имеет статус
  `ignored` и полностью выведен из поисковых проекций) перечисляются отдельно
  в `retained_evidence_sources` и `retained_evidence_chunks` и проверку не
  валят; надгробие с оставшейся проекцией или без реальной ссылки — провал.
  Аудит охватывает каждый репозиторный источник в базе, а не только пути
  управляемой области: источник вне этой области никогда не попадал в корпус
  законно и проваливает проверку как `unexpected_sources` независимо от
  состояния его фрагментов — снисхождение для надгробий на него не
  распространяется. Грязный управляемый корпус выводится отдельным полем
  `dirty_managed_paths`, а не выдаётся за stale-источник;
- запускать по жизненному циклу выше — после коммита созданного, изменённого,
  перемещённого или удалённого документа; прогон без изменений — no-op.

## Переносимость между машинами

Память состоит из двух слоёв с разной переносимостью — это надо учитывать при
bootstrap на новой машине:

- **Хранилище** — файл `.agent/memory.db` (обычный SQLite). Полностью переносим:
  коммитится в git и переезжает как есть. Бинарный SQLite **не мёржится** —
  нельзя писать в память с двух машин одновременно; синхронизация через git по
  очереди.
- **Поиск** — слой эмбеддингов: загружаемое расширение `sqlite-vec` (нативная
  библиотека под конкретную ОС и архитектуру: `.dll` / `.so` / `.dylib`) и
  обращения к провайдеру эмбеддингов. Этот слой **не переносим**: бинарник
  расширения и venv пересобираются на каждой машине, ключ провайдера лежит в
  локальном `.env`. В `.gitignore` держать `memgraph/venv/`,
  `memgraph/vendor/*` и `.env` вне гита.

Практические следствия:

- **Векторы переносятся, эмбеддинг запроса — нет.** Дозаполненные векторы лежат
  в самой базе и переезжают с ней; но `recall` эмбеддит **сам запрос** на лету,
  поэтому на новой машине для живого семантического поиска всё равно нужен ключ
  провайдера. Без ключа `recall` и записи с эмбеддингом **падают** (работает
  только `session-context` и прочие чисто-SQL команды); автоматической деградации
  в FTS при отсутствии ключа у этого движка нет — FTS-only это отдельный
  Вариант B, который вообще не эмбеддит запрос.
- **Подача должна быть репозиторной.** Claude SessionStart-хук кладётся в
  `.claude/settings.json` внутри проекта; Codex получает то же требование через
  проектный `AGENTS.md` или доступный локальный hook. Нельзя полагаться только
  на глобальный конфиг машины — на новой машине подачи не будет. Команду писать
  под оболочку целевой ОС; `session-context` — чистый SQL, поэтому стартовая
  подача работает офлайн и до установки ключа.
- **Два варианта поднятия.** Вариант A — полный движок с эмбеддингами и
  семантическим recall (нужны рантайм провайдера, ключ и нативный `sqlite-vec`).
  Вариант B — деградированный FTS-only, без ключа и вектора (оффлайн или
  изолированная машина). По умолчанию целиться в A; B — осознанный fallback с
  понятным ограничением recall (см. «Почему память не используется сама собой»,
  звено 2).
- **UTF-8.** Если память не на английском, форсировать UTF-8 в обёртке и
  процессе (кодировка вывода консоли, `PYTHONUTF8` / `PYTHONIOENCODING`), иначе
  вывод `recall` / `session-context` приходит кракозябрами и обесценивает память.

## Code creation control

Работа над кодом начинается только после того, как: прочитана иерархия
источников; загружен контекст memgraph; объём задачи заморожен в
`.agent/tasks/<TASK_ID>/spec.md`; owned и forbidden files явные; известна
требуемая проверка; классифицирован уже существующий dirty-объём.

Исполнитель правит только owned files. Если нужен другой файл — остановиться и
обновить задачу через правильный процесс перед правкой.

Ревьюер не одобряет код по нарративу. Evidence обязан цитировать команды и exit
codes; релевантные пути; HTTP/browser/DB трейсы, когда критерий приёмки
затрагивает эти границы; пропущенные проверки и точные блокеры.

## Push discipline

Каждая принятая логическая задача закрывается до старта следующей:

1. прогнать требуемые проверки;
2. убедиться, что `git diff --check` проходит;
3. записать финальный evidence/status в memgraph;
4. застейджить только точные файлы;
5. закоммитить одну логическую единицу;
6. сразу запушить;
7. отчитаться: изменённые файлы, проверки, commit, push status, остаточный
   риск и следующий executable step.

Не оставлять принятую работу только локальной. Не сваливать несвязанные
принятые задачи в один коммит. Если tracked-изменений нет — сказать, что коммит
не нужен, и зафиксировать статус в memgraph.

## Что взять из зрелых проектов

Взять: строгую иерархию источников; одну рантайм-поверхность инструкций; живую
процессную память в `.agent/memory.db`; замороженные спецификации задач;
независимую проверку и ревью; `UNKNOWN` как блокер; точное владение файлами;
commit и push после каждой принятой логической задачи; чистые хвосты `.agent/`.

Не брать: дублированное содержимое `AGENTS.md` и `CLAUDE.md`; markdown/json
evidence-файлы под `.agent/tasks/**`; папки сырого вывода; исторический мусор
задач как нормальный workflow; продуктовые правила из другого репозитория;
старые процессные файлы, вытесненные memgraph.

## Bootstrap и четыре независимых процесса

Поднятие и ведение памяти состоит из четырёх независимых процессов. У каждого
свой триггер и своё определение «готово»; смешивать их нельзя.

### A. Инфраструктура памяти (пустая база)

Для нового проекта:

1. Создать `AGENTS.md` из шаблона выше; заменить имена стратегических
   документов в `Source Hierarchy` и `Startup Discipline` под проект.
2. Создать `CLAUDE.md` с единственной строкой `@AGENTS.md`.
3. Добавить `.env` и селективные правила `.agent/` в `.gitignore`: игнорировать
   сгенерированный мусор `.agent`, но оставить отслеживаемыми `.agent/memory.db`
   и `.agent/tasks/<TASK_ID>/spec.md`.
4. Развернуть скилл `memgraph` (уже установленный в окружении или приложенный
   `./memgraph/` — см. «Memgraph: установка и использование»); не копировать
   движок по файлам вручную.
5. Создать `.agent/memory.db` вызовом bootstrap скилла; он применяет
   каноническую схему (24 таблицы + FTS5 + sqlite-vec@512) и не трогает никакие
   файлы проекта:
   - macOS/Linux: `memgraph/venv/bin/python memgraph/scripts/bootstrap.py --target <repo>`
   - Windows: `memgraph\venv\Scripts\python.exe memgraph\scripts\bootstrap.py --target <repo>`
6. Добавить `OPENAI_API_KEY` в локальный `.env`; задокументировать в
   `.env.example`.
7. Выполнить `memgraph/memgraph session-context` на macOS/Linux или
   `powershell.exe -NoProfile -ExecutionPolicy Bypass -File memgraph/memgraph.ps1 session-context` на Windows.
8. Проверить подачу: открыть свежую Claude или Codex-сессию в репозитории и
   убедиться, что `session-context` доступен. Если нет — чинить команду
   startup-интеграции, путь или политику оболочки, пока не сработает.
9. Закоммитить bootstrap-файлы, включая пустую `.agent/memory.db`, исключая
   `.env` и запрещённые артефакты `.agent`.

Отсутствие документов — допустимое состояние этого процесса. Bootstrap не
создаёт, не ищет и не индексирует документы; `docs/spec.md` не является
предусловием пустой базы; `verify-doc-index` на пустом корпусе обязан
возвращать исправность с признаком `corpus_empty: true`. Никакие тестовые
записи (вроде «kit live») в базу не пишутся.

На шаге 9 этот процесс завершён: инфраструктура готова. Но сам bootstrap-коммит
уже добавил управляемый документ — `AGENTS.md`, — поэтому он одновременно
является первым событием процесса C: сразу после него выполняется `ingest-docs`,
`verify-doc-index` и коммит обновлённой `.agent/memory.db`. До этого прогона
состояние «инфраструктура готова, документы ещё не синхронизированы» — и
`verify-doc-index` честно покажет непустой корпус с пропущенным `AGENTS.md`.

### B. Типизированная память

Заполняется только реальными событиями по MEMORY WRITE CADENCE: policies,
decisions, claims, задачи и события — каждая запись вызвана настоящим фактом
работы и цитирует доказательство. Пустая типизированная память нового проекта —
норма, а не дефект. Проверка «вознаграждения» (см. «Почему память не
используется сама собой») выполняется, когда появилась первая настоящая
запись: `memgraph/memgraph recall "<запрос-парафраз>"` (Windows:
`powershell.exe -NoProfile -ExecutionPolicy Bypass -File memgraph/memgraph.ps1 recall "<запрос-парафраз>"`)
должен вернуть её по смыслу, с включённым векторным плечом, а не пустоту.

### C. Жизненный цикл документов

`docs/spec.md` обязателен перед продуктовой реализацией — но не перед
созданием пустой базы. Дополнительные стратегические документы создаются
только утверждёнными и связанными из `docs/spec.md`; тематические рабочие
документы (`docs/knowledge/`) агент создаёт без поштучного утверждения. Как
только управляемый документ создан, изменён, перемещён или удалён, в том же
рабочем блоке выполняется цепочка из раздела «Индексация документов»: коммит
документа, `ingest-docs`, `verify-doc-index`, коммит обновлённой
`.agent/memory.db`, push обоих коммитов.

### D. Подключение к существующему репозиторию (разовое обратное заполнение)

Только для проекта, где документы накопились до появления памяти:

1. Инвентаризация корпуса: перечислить управляемые документы и убедиться, что
   они закоммичены.
2. Полный `ingest-docs`.
3. `verify-doc-index`: охват N из N, без пропусков, лишних источников и
   несовпадающих хешей.
4. Проверка семантического поиска: `recall` парафразом находит фрагмент
   известного документа с корректными `source_path` и `source_role`.
5. Повторный `ingest-docs` — доказательство идемпотентности: no-op без новых
   эмбеддингов.

После обратного заполнения проект живёт по процессу C.

После bootstrap каждая задача идёт по одному циклу: прочитать источники,
вспомнить память, заморозить спецификацию, реализовать, проверить, отревьюить,
записать в memgraph, закоммитить, запушить.

## Модель сотрудничества (ведущий и исполнитель)

Когда над проектом работают несколько агентов, роли фиксируются явно. Типовая
расстановка: один агент — ведущий/ревьюер и владелец глобального контекста;
другой — оркестратор/исполнитель атомарных change-set'ов. Ведущий не
переписывает код сам, кроме мелких безопасных исправлений и гигиены; он пишет
точные задания, проверяет выполнение, ведёт приёмку.

Граница инструментов может быть асимметричной: какие агенты кого имеют право
вызывать — определяется рантаймом и явным разрешением пользователя.
Координация, которую пользователь не делегировал прямым вызовам между агентами,
идёт через пользователя: ведущий формулирует задания, корректировки, ревью и
следующий шаг, а пользователь передаёт их исполнителю.

Задания исполнителю объёмные и самодостаточные: не один прогон, а целый
логический блок, рассчитанный на длительное выполнение, с детально прописанным
объёмом, без допусков и домыслов исполнителя. Память проекта к этому моменту
уже существует и ведётся по этой инструкции. Ведущий не останавливается на
простых отчётах: задача закончена и проверена — он сразу выдаёт следующий шаг.
Остановка допустима только на реальном блокере; блокер — это требование искать
решение, а не повод завершить работу.
