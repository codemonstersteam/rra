# Оценка простоты и понятности документации — научная база и автоматизация

> Спецификация модуля `audit-docs` / подсистема readability.
> Ссылается из [CONCEPT.md](./CONCEPT.md), секция «Критерий №2».

## Проблема

DORA (Google, 2022) свела оценку документации к субъективным шкалам Ликерта — «согласен / не согласен». Это даёт корреляцию с бизнес-результатами (25% рост производительности у команд с качественной документацией), но не даёт объективной метрики, которую можно прогнать в CI на каждый PR.

Науке есть что предложить. Ниже — три уровня оценки, каждый со своей научной базой, конкретным инструментом и местом в пайплайне.

## Три уровня оценки

### L1. Читаемость текста (формулы readability)

**Научная база.** Формулы с 80-летней историей валидации: Flesch Reading Ease (1948), Flesch-Kincaid Grade Level (1975), Gunning Fog, SMOG, Coleman-Liau, Dale-Chall. Армия США использует Flesch-Kincaid для оценки технических мануалов.

Для русского языка формулы адаптированы:

- **Оборнева И.В. (2006)**: `FRE(рус) = 206.836 − 1.52 × ASL − 65.14 × ASW`. Используется в макросе Microsoft Word.
- **Солнышкина М.И., Кисельников А.С. (КФУ, 2015–2018)**: `ФКМОД = 0.36 × ASL + 5.76 × ASW − 11.97` — модифицированная формула для научных и учебно-научных текстов.

**Ограничение.** Формулы ловят поверхностные характеристики (длина слов, длина предложений). Термин `WebAuthn` будет считаться «сложным», хотя для целевой аудитории он элементарен. Российская школа (Микк, Солнышкина) разграничивает четыре конструкта: читабельность, понятность, сложность, трудность — формулы покрывают только первый.

**Инструменты:**

- `textstat` (Python, PyPI) — Flesch, FK Grade, Gunning Fog, SMOG, Coleman-Liau, ARI, Dale-Chall
- `py-readability-metrics` (Python, PyPI) — аналогичный набор с расширенной интерпретацией
- Для русского: кастомный скрипт с формулой Оборневой (слогоделение через `pyphen` или ручной алгоритм)

**Что делает в CI.** Считает score для каждого `.md`-файла. Сравнивает с порогом (FK Grade ≤ 12 для технической документации). Выводит delta между текущей и предыдущей версией файла.

### L2. Стиль и когнитивная нагрузка (prose linting)

**Научная база.** Cognitive Load Theory (Sweller, 1988): три типа нагрузки — intrinsic (сложность задачи), extraneous (нагрузка от подачи), germane (продуктивная). Prose-линтинг снижает extraneous load.

Конкретные правила, которые имеют научное обоснование:

- **Длина предложений.** Исследования eye-tracking показывают, что предложения > 25 слов увеличивают количество регрессий (возвратов глаз) и время перечитывания.
- **Пассивный залог.** Увеличивает количество трансформаций при декодировании, повышает extraneous load.
- **Несогласованная терминология.** Дридзе Т.М. (1984): «эффект смысловых ножниц» — расхождение между намерением автора и интерпретацией читателя. Один термин = одно написание.
- **Номинализация и причастные обороты.** Текстометр (textometr.ru) выделяет причастия и пассивные формы как факторы, затрудняющие чтение русского текста.

**Инструменты:**

- **Vale** (Go, MIT, vale.sh) — prose-линтер, работает офлайн, поддерживает Markdown/AsciiDoc/RST, кастомные правила в YAML. Готовые пакеты стилей: Microsoft, Google, write-good. GitHub Action: `errata-ai/vale-action@v2`.
- **markdownlint-cli2** (Node.js) — структурный линтинг: заголовки, вложенность, ссылки, code-блоки. 40+ правил. GitHub Action: `DavidAnson/markdownlint-cli2-action`.

**Кастомные правила Vale для RRA:**

```yaml
# SentenceLength.yml
extends: occurrence
message: "Предложение слишком длинное (%s слов). Максимум 25."
scope: sentence
level: warning
max: 25
token: '\b\w+\b'
```

```yaml
# Terminology.yml — единый словарь проекта
extends: substitution
message: "Используй '%s' вместо '%s'."
level: error
ignorecase: true
swap:
  вебавтн: WebAuthn
  пасскей: Passkey
  эндпоинт: endpoint
```

**Что делает в CI.** Блокирует PR при нарушении терминологии (error). Предупреждает о длинных предложениях и пассивном залоге (warning). Комментирует PR через GitHub Action.

### L3. Структурная полнота и свежесть (information architecture)

**Научная база.** DORA 2022: findability и reliability как ключевые атрибуты. Дридзе Т.М. (1984): информативность как относительная характеристика — текст информативен только тогда, когда понятен конкретному читателю. JTBD-модель из статьи «Документация как продукт»: четыре потребителя нанимают документацию для разных задач.

**Инструмент:** shell/Go-скрипт, специфичный для RRA. Проверяет:

- Наличие обязательных файлов: README.md, AGENTS.md, docs/intent.md, docs/contracts-graph.md.
- README ссылается на ключевые файлы (grep по целевым именам).
- Drift: `git log` по `docs/` vs по коду — если документация старше кода на > N дней, флаг.
- Дублирование: совпадающие фрагменты > M токенов в разных файлах.

**Что делает в CI.** Warning при отсутствии файлов. Warning при drift. Error при критическом расхождении (intent.md отсутствует).

## Научные источники

### Классические формулы читаемости

1. Flesch R. (1948). A new readability yardstick. *Journal of Applied Psychology, 32*(3), 221–233.
2. Kincaid J.P. et al. (1975). Derivation of new readability formulas for Navy enlisted personnel. *Research Branch Report 8-75*, Naval Technical Training Command.
3. Gunning R. (1952). *The Technique of Clear Writing*. McGraw-Hill.

### Адаптация для русского языка

4. Микк Я.А. (1970). О факторах понятности учебного текста. Автореф. дисс. к.п.н. — Тарту.
5. Микк Я.А. (1981). *Оптимизация сложности учебного текста: В помощь авторам и редакторам*. — М.: Просвещение.
6. Мацковский М.С. (1976). Проблемы читабельности печатного материала. В: *Смысловое восприятие речевого сообщения* / Ред. Дридзе Т.М., Леонтьев А.А. — М.: Наука, С. 126–142.
7. Оборнева И.В. (2006). Автоматизированная оценка сложности учебных текстов на основе статистических параметров. Дисс. к.п.н. — М.
8. Солнышкина М.И., Кисельников А.С. (2015). Параметры сложности экзаменационных текстов. *Вестник ВолГУ, Серия 2, Языкознание, №1(25)*, 99–107.

### Когнитивная нагрузка и понимание текста

9. Sweller J. (1988). Cognitive load during problem solving: Effects on learning. *Cognitive Science, 12*(2), 257–285.
10. Дридзе Т.М. (1984). *Текстовая деятельность в структуре социальной коммуникации*. — М.
11. Graesser A.C. et al. (2004). Coh-Metrix: Analysis of text on cohesion and language. *Behavior Research Methods, 36*, 193–202.
12. Crossley S.A. et al. (2016). TAACO: Tool for Automatic Analysis of Cohesion. *Behavior Research Methods, 48*, 1227–1237.

### DORA и документация в DevOps

13. DORA / Google Cloud (2022). State of DevOps Report — Documentation Quality. https://dora.dev/capabilities/documentation-quality/
14. DORA 2022 Survey Questions. https://dora.dev/research/2022/questions/

### Разграничение конструктов

15. Солнышкина М.И. et al. (2015). On the problem of text characteristics: readability, comprehensibility, complexity, difficulty. *Филология. Теория & практика.* https://philology-journal.ru/en/article/phil20152320/fulltext

## Готовые проекты для переиспользования

| Инструмент | Язык | Что делает | Лицензия | Ссылка |
|---|---|---|---|---|
| textstat | Python | 10+ формул читаемости | MIT | https://pypi.org/project/textstat/ |
| py-readability-metrics | Python | Flesch, FK, Fog, Dale-Chall, ARI, SMOG | MIT | https://github.com/cdimascio/py-readability-metrics |
| Vale | Go | Prose-линтинг, кастомные правила стиля | MIT | https://vale.sh |
| vale-action | YAML | GitHub Action для Vale | MIT | https://github.com/errata-ai/vale-action |
| markdownlint-cli2 | Node.js | 40+ правил структуры markdown | MIT | https://github.com/DavidAnson/markdownlint-cli2 |
| markdownlint-cli2-action | YAML | GitHub Action для markdownlint | MIT | https://github.com/DavidAnson/markdownlint-cli2-action |
| Текстометр | Web | Сложность русского текста по CEFR | — | https://textometr.ru |
| Простой русский язык | Web/Bot | 5 формул для русского | — | https://plainrussian.ru |

## Связь с CONCEPT.md

Этот документ детализирует подсистему readability внутри модуля `audit-docs`. В CONCEPT.md секция «Критерий №2» ссылается сюда для научного обоснования и выбора инструментов. JTBD-симуляция через LLM (Часть 2 критерия №2) остаётся в CONCEPT.md — она не про читаемость текста, а про полноту информации для конкретного потребителя.

Граница ответственности: этот документ отвечает на вопрос «текст написан понятно?», JTBD-симуляция отвечает на вопрос «текст содержит нужное?». Оба необходимы, ни один не достаточен.
