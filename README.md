# 32 — Куратор курса: AI-ревью PR в GitHub Actions

Задание 32 курса по AI (неделя 8, день 2). Продолжение куратора курса
(задания 31 и 33): пайплайн, в котором ассистент анализирует новый PR в
репозитории курса `AlexS8332/AITrenning` — получает diff и изменённые
файлы, ищет по документации проекта и по коду (RAG) и публикует ревью:
потенциальные баги, нарушения правил проекта и расхождения с документацией,
рекомендации — каждая находка привязана к строке diff.

Этот репозиторий — снимок модуля `tools/curator` корня курса на момент
сдачи (тег корня `v32`); снимок делает `curator snapshot 32`.

## Что сделано по условию

| Условие | Что есть |
|---|---|
| Пайплайн запускается на PR | `.github/workflows/review.yml` в корне курса: на `pull_request` (opened, synchronize, reopened) собирает куратор, строит индекс документации на BM25 (без эмбеддера, секунды), запускает `curator review -pr N -post` с `DEEPSEEK_API_KEY` из секретов и `GITHUB_TOKEN`; ревью уходит в PR комментарием с inline-замечаниями, тот же текст — в summary и артефакт; повторный коммит отменяет идущее ревью, повторное ревью того же sha не публикуется (маркер) |
| Ассистент получает diff и изменённые файлы | `internal/review`: разбор unified diff (hunk'и, новая нумерация, переименования, бинарные), источники — `git diff base...head`, GitHub API (`Accept: application/vnd.github.diff`) или файл патча; затронутые файлы агент читает целиком инструментом `read_file` |
| RAG: документация + код | Индекс дня 31 плюс новый вид документов `code`: doc-комментарии пакетов и сигнатуры экспортов Go (`go/parser`, `go/doc`) по `tools/edgerec` и `tools/curator`, функции `scripts/aitr.ps1` с комментариями. `docs_search` вызывается кодом до первого запроса — запрос из имён файлов, пакетов и функций diff и слов правил, встречающихся в нём (`&&`, `expectJS`, `push`, `ключ`) |
| Текст ревью: баги, архитектурные проблемы, рекомендации | Завершающий инструмент `review_done{findings[{file, line, kind: bug\|rule\|docs\|recommendation, severity, text, evidence}], summary, verdict}`, который проверяет код: строка — добавленная или контекстная в diff (иначе GitHub её не примет), `rule`/`docs` — только с `chunk_id` из выдачи и дословной цитатой правила, `bug` — со ссылкой на прочитанную строку; `verdict: ok` ⇔ находок нет и в сводке «замечаний нет»; дубли схлопываются |

## Что нужно

То же, что в задании 31 (Go 1.25+, `git`, ключ `DEEPSEEK_API_KEY`, каталог
курса). Для `-pr`/`-post` — `GITHUB_TOKEN` (или `GH_TOKEN`) и `-repo
owner/name` (в Actions — `GITHUB_REPOSITORY`). В GitHub Actions токену
`gh`, который пушит workflow, нужен scope `workflow`.

## Быстрый старт

```powershell
go build -o curator.exe ./cmd/curator
.\curator.exe index -root D:\Projects\AITrenning

.\curator.exe review -patch eval\review\planted\P4.patch     # подставной PR: force-push и печать ключа
.\curator.exe review -range main...HEAD                        # текущая ветка против main
.\curator.exe review -pr 1 -repo AlexS8332/AITrenning -post   # настоящий PR, публикация ревью
.\curator.exe serve -addr 127.0.0.1:8781                       # окно → вкладка «Ревью»
.\curator.exe bench -trial 15 -out examples/review             # испытание И-15
```

## Подставные PR (`eval/review`)

| PR | Файл | Что посажено | Вид |
|---|---|---|---|
| P1 | `tools/edgerec/report.go` | результат записи файла отброшен в `_ =` | bug |
| P2 | `scripts/aitr.ps1` | `&&` и тернарник — CLAUDE.md: PowerShell 5.1 без них | rule |
| P3 | `demos/31-course-help/scenario.json` | `expectJS` с присваиванием, `wait` 240 с без `cut` — README edgerec | rule |
| P4 | `scripts/publish.ps1` (новый) | `git push --force` и печать `$env:DEEPSEEK_API_KEY` — CLAUDE.md | rule |
| P5 | `docs/notes/README.md` | чистая правка опечатки | — |

Патчи — настоящие unified diff против дерева корня; тест `TestPlantedApply`
проверяет `git apply --check` в каталоге курса, чтобы они не протухли.
«Найдено» = тот же файл, разница строк ≤ 3, ожидаемый вид.

## Испытание И-15

Дорожки `main` / `norag` (механизм `review.rag` выключен — без
`docs_search`; правила без ссылки на документ тогда принимаются) / `same`.
Пороги: посаженные найдены ≥ 3/4 на `main`, P1 — на обеих; на P2–P4
«с RAG − без RAG» больше шума; на P5 `major` ≤ 1; находок вне hunk'ов 0;
вердикт согласован с посадкой 5/5; «не проверено» 0; запросов на ход ≤ 6.
Отчёт живого прогона — `examples/review/`.

## Как устроено

```
internal/review/
  types.go, diff.go     контракт; unified diff → файлы, hunk'и, строки; Commentable
  input.go, github.go   PR из git-диапазона, патча или GitHub API; клиент REST (httptest-подменяемый), маркер дедупа
  agent.go, markdown.go агент ревью: Preload docs_search, review_done с проверками, текст ревью и inline-комментарии
  planted.go            подставные PR и правило «найдено»
internal/docs/code.go   код как документы индекса
internal/bench/t15.go   И-15
internal/web/review.go  вкладка «Ревью»: список подставных PR и текущая ветка, diff с подсветкой, находки, вердикт, панель Action
cmd/curator/review.go   CLI review; bench -trial 15
.github/workflows/review.yml (корень курса)
```

## Тесты

```powershell
go vet ./... ; go test ./...
```

Без сети: GitHub — `httptest`, git — временные репозитории, модель —
`llmtest.Fake`; `TestPlantedApply` и `TestCodeDocsCourseRoot` ищут каталог
курса и пропускаются, если его нет.
