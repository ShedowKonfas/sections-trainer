# agent.md — тренажёр построения сечений (куб и тетраэдр)

## 0. Правила работы в этой директории
- **Не удалять файлы без явного разрешения Жени.** Не трогать системные файлы.
- Общение: прямо и кратко, без длинных вступлений; технические термины можно не объяснять.
- Требования неоднозначны → задать **один** уточняющий вопрос, не список.
- Ясная задача без рискованных операций → довести до конца без подтверждения каждого шага.
- Показывать diff/summary перед коммитом. При ошибке — минимальное исправление, не переписывать лишнее.
- Глобальные правила Жени лежат в `C:\Users\ShedowKonfas\.claude\CLAUDE.md` (auto-router "always on", уведомление `notify.ps1` в финале).

## 1. Назначение
Один самодостаточный HTML-файл — интерактивный тренажёр по построению сечений куба и тетраэдра секущей плоскостью.
Два режима: **«Изучить»** (сечение строится автоматически + пошаговое объяснение методом следов) и **«Задачи»**
(случайная задача с 3 точками на рёбрах, пользователь отмечает вершины сечения, проверка и счётчик).
Язык интерфейса — русский.

## 2. Файлы и git
```
D:\HTML\Section\
├── .git\
├── agent.md                 ← этот файл
└── sections-trainer.html    ← весь проект, ~48 КБ
```
- Репозиторий инициализирован **в этой папке** (не в `D:\HTML`).
- `origin`: `git@github.com:ShedowKonfas/sections-trainer.git`
- Ветка: `main`, один коммит `3630726 Add sections trainer (cube and tetrahedron)`.
- Команды: `git -C 'D:\HTML\Section' status --short`, `git -C 'D:\HTML\Section' add ...`, `git -C 'D:\HTML\Section' push origin main`.
- История расположения: `C:\Users\ShedowKonfas\sections-trainer.html` → `D:\Site` → `D:\HTML` → **current** `D:\HTML\Section\sections-trainer.html`.
  Старых копий нет (был перенос, не копирование). Если правка не находится — проверь путь.

## 3. Стек и запуск
- THREE.js **r0.160.0** через `<script type="importmap">` с unpkg:
  - `three` → `https://unpkg.com/three@0.160.0/build/three.module.js`
  - `three/addons/` → `https://unpkg.com/three@0.160.0/examples/jsm/`
- `OrbitControls` из `three/addons/controls/OrbitControls.js`.
- Всё в одном `<script type="module">`. Сборки, npm, тестов нет.
- Открывается двойным кликом в браузере; **нужен интернет** для CDN (офлайн-версии нет).
- Текущая дата/модель в сессиях не важны; важно, что код — ES module, поэтому `file://` работает в Chrome/Edge (importmap поддерживается).

## 4. Структура HTML/CSS
- `#app` = flex: `aside#panel` (350px, скролл) + `main#viewport` (3D-сцена).
- Панель сверху вниз:
  - `#bodySeg`: `data-body="cube" | "tetra"`.
  - `#modeSeg`: `data-mode="learn" | "task"`.
  - `#toolSeg`: `data-tool="point" | "line" | "erase"`.
  - `#chkExtend` — «Продлевать линию (след)».
  - `#chkFace` — «Линия по грани (в плоскости грани)».
  - `#btnUndoLine`, `#btnClearSketch`, `#btnClearAll`.
  - `#dynamic` — **пересоздаётся целиком** в `updatePanel()`; все обработчики кнопок навешиваются там же.
  - `#explainBox` — плеер объяснения (`.hidden` => `display:none`).
  - `.legend` — цвета и управление мышью.
- `state.explaining === true` → `#dynamic` скрывается, `#explainBox` показывается.

## 5. Состояние `state`
```js
{
  body: 'cube'|'tetra',
  mode: 'learn'|'task',
  learnPoints: [],   // {pos, edge|null, index|null}
  userPoints: [],    // {pos, edge|null, status:''|'ok'|'bad'}
  task: null,        // {points, plane, section, _solved?}
  solved: 0,
  explaining: false, step: 0,
  tool: 'point'|'line'|'erase',
  extend: false,     // chkExtend
  byFace: false,     // chkFace
  faceLock: null,    // {face, faceIndex, plane} при отрисовке линии по грани
  sketch: [],        // {a, b, extended, byFace?, face?}
  lineStart: null    // первая точка линии/снап
}
```
`let explanation = null` — результат `buildExplanation()`, Live-объект для плеера.

## 6. Геометрия и математика
### Тела (`SOLIDS`)
- **cube**: вершины ±1; `A(-1,-1,-1) B(1,-1,-1) C(1,-1,1) D(-1,-1,1)` — нижняя грань, `A1..D1` — верхняя (y=+1). 12 рёбер, 6 граней, порядок граней совпадает с порядком `faceMeshes`.
- **tetra**: 4 чередующиеся вершины куба `A(1,1,1) B(1,-1,-1) C(-1,1,-1) D(-1,-1,1)`. 6 рёбер, 4 грани.
- Индексы рёбер используются для подсветки: `colorEdge(i, color)`.

### Ядро
- `planeFrom3(p1,p2,p3)` → `{n, d}` (нормализованная нормаль, `n·p = d`), `null` если коллинеарны.
- `computeSection(solid, plane)` — пересечение плоскости с рёбрами; вершины сортуются по углу вокруг центроида в базисе плоскости → выпуклый многоугольник. Возвращает `[{pos, edge, angle}]`.
- `planePlaneLine`, `lineLineIntersect`, `linePlaneIntersect`, `facePlane`, `faceHasEdge`, `faceContaining` — вспомогательные.
- `generateTask(solid)` — 800 попыток; 3 случайных ребра, t∈[0.2,0.8]; отбрасывает вырожденные (вершины совпадают с вершинами тела или друг с другом).
- `polyName(n)` — русское название многоугольника.
- `edgeName`, `faceName`, `ptEdgeLabel` (последний — для точек без ребра: показывает координаты).

## 7. Сцена
- Камера `PerspectiveCamera(45, ...)`, позиция `(4.4, 3.4, 6.2)`, цель `(0,0,0)`; OrbitControls с damping, zoom 2.5..30.
- Группы: `solidGroup`, `sectionGroup`, `pointsGroup`, `taskGroup`, `explainGroup`, `sketchGroup`, `previewGroup`.
- `buildSolid(key)` пересоздаёт грани и рёбра; у грани `mesh.userData = {face, faceIndex}`, у ребра `{edge, index}`.
- `clearGroup(g)` освобождает geometry/material/texture; всегда использовать его вместо ручного удаления.
- `makeLabel` = CanvasTexture + Sprite (depthTest:false); `makePoint` = Sphere + emissive.
- `resize()` через `ResizeObserver`; `animate()` через `requestAnimationFrame`.

## 8. Взаимодействие
`pointerdown/pointermove/pointerup` → клик, если <25px и <500ms. Вращение мышью — через OrbitControls, не мешает.

`handleClick(ev)`:
1. `state.explaining` → ignore.
2. `tool === 'line'` → `handleLineClick(ev)`.
3. `tool === 'erase'` → `handleEraseClick(ev)`.
4. иначе → `getSnapPoint(ev)` и постановка точки в `learnPoints` (до 3) или `userPoints` (до 8).

### Привязка (`getSnapPoint`)
Приоритет:
1. существующие точки (`collectCandidatePoints()`, радиус 0.3 в мире);
2. **пересечения линий** (`collectIntersections()`, экранный радиус 15px через `screenDist`) → `{type:'intersect'}`;
3. ближайшее ребро (`snapToNearestEdge`, t∈[0.1,0.9], порог 0.22);
4. точка на грани (`{type:'face'}`).

`collectIntersections()` — пересечения линий наброска между собой (`kind:'ll'`) и с рёбрами (`kind:'le'`), отсекает точки вне радиуса 3 от начала координат. Метки: жёлтая `0xffb300` (ll), красная `0xd81b60` (le).

### Инструмент «Линия»
- Два клика → `state.sketch.push({a,b,extended})`. Пока между кликами — пунктир `updatePreview()`.
- `chkExtend`: линия рисуется бесконечной (фиолетовая `0x8e24aa`) + красные точки пересечений с рёбрами.
- `chkFace` (`handleFaceLineClick`): 1-й клик по грани фиксирует `{face, faceIndex, plane}` и подсвечивает грань; 2-й клик проецируется на плоскость (`projectToPlane`), линия обрезается по контуру грани (`clipLineToFace`) → хорда, голубая `0x00acc1`.
- `renderSketch()` перерисовывает весь набросок, метки пересечений, подсветку `faceLock` и стартовую точку.

### Ластик
`handleEraseClick` ищет ближайшую точку (радиус 0.25) среди learn/user точек и концов линий наброска; удаляет её. В режиме «Задачи» есть ещё кнопка «Убрать точку» (удаляет последнюю).

### Очистка
- `clearSketch()` — только набросок и превью.
- `clearAll()` — learn + user точки, набросок, превью, faceLock, сброс цветов рёбер. **Условие задачи сохраняется**.

## 9. Режимы
- **learn**: 3 точки → `updateLearn()` считает плоскость и вызывает `renderSection(sec)`. Список точек обновляется.
- **task**: `newTask()` создаёт задачу, рисует M/N/K в `taskGroup`. `checkTask()` сравнивает пользовательские точки с `task.section` (допуск 0.18), ставит `status` ok/bad, считает счётчик и рисует сечение при полном совпадении.

## 10. Пошаговое объяснение (метод следов)
`buildExplanation()` возвращает `{given, plane, section, base, trace, tps, inter, sides, steps}`.
Выбор «основания» `base`: грань, не параллельная секущей плоскости, с максимумом вершин сечения на её рёбрах.
Шаги `steps`:
1. **Дано** — подписи точек (`ptEdgeLabel`, работает и без ребра).
2. **След секущей плоскости** — линия пересечения с основанием (`planePlaneLine`), точки X/Y от прямых MN/NK/MK.
3. **Вершины на основании** — пересечения следа с рёбрами основания (на ребре / на продолжении).
4. **Сторона k** — по одной на каждую сторону сечения, с названием грани.
5. **Сечение построено** — итоговый многоугольник с подписанными вершинами.

`renderExplanation(i)` накапливает геометрию по шагам; `addFaceHighlightTo(group, face, color, opacity)` (и обёртка `addFaceHighlight` для `explainGroup`) подсвечивает грань. Плеер: `updateExplainBox()` → Назад/Далее/Закрыть → `closeExplanation()`.
`openExplanation()` временно скрывает `taskGroup/pointsGroup/sectionGroup/sketchGroup`.

## 11. Цвета (не менять без причины)
| Что | Hex |
|---|---|
| M / N / K | `0xd81b60` / `0x7b1fa2` / `0x00897b` |
| Сечение (заливка / контур) | `0xffa726` / `0xff6d00` |
| След / продлённая линия | `0x1565c0` (объяснение) / `0x8e24aa` (набросок) |
| Набросок | `0x43a047` |
| Линия по грани | `0x00acc1` |
| Пересечение линий / с ребром | `0xffb300` / `0xd81b60` |
| Ребро по умолчанию / подсвеченное | `0x2b3a55` / `0x94a3b8` |
| Подсветка грани при объяснении / при линии по грани | `0xffe082` / `0x26c6da` |
| Цвета ответа (ok / bad / нейтральный) | `0x2e7d32` / `0xc62828` / `0x1565c0` |

## 12. Карта ключевых функций
```
математика   planeFrom3, planePlaneLine, lineLineIntersect, linePlaneIntersect,
             facePlane, faceHasEdge, faceContaining, computeSection, generateTask
рисование    buildSolid, addPolygon, renderSection, makePoint, makeLabel, labelAt,
             clearGroup, addFaceHighlightTo, addFaceHighlight, renderSketch, updatePreview
подбор       getSurfaceHit, snapToNearestEdge, getFaceHit, projectToPlane, screenDist,
             segmentIntersection, collectIntersections, clipLineToFace,
             collectCandidatePoints, getSnapPoint
клики        handleClick, handleLineClick, handleFaceLineClick, handleEraseClick
режимы       updateLearn, newTask, updateTaskVisual, checkTask, updatePanel,
             setBody, setMode, setTool, clearSketch, clearAll
объяснение   buildExplanation, renderExplanation, updateExplainBox, openExplanation, closeExplanation
init         обработчики #bodySeg/#modeSeg/#toolSeg, chkExtend/chkFace,
             btnUndoLine/btnClearSketch/btnClearAll → buildSolid → updateSegActive → updatePanel → resize → animate
```

## 13. Проверка синтаксиса (обязательна после правок JS)
```powershell
$c = Get-Content -Raw 'D:\HTML\Section\sections-trainer.html'
$m = [regex]::Match($c, '(?s)<script type="module">(.*?)</script>')
Set-Content -Path 'C:\Temp\opencode\check.mjs' -Value $m.Groups[1].Value -Encoding UTF8
node --check 'C:\Temp\opencode\check.mjs'
```
Ожидаемый вывод: `SYNTAX OK` (папка `C:\Temp\opencode` уже создана). Тестов и линтеров в проекте нет — выдумывать не нужно.

## 14. История изменений (контекст)
1. Создан тренажёр: learn/task, авто-сечение, плеер объяснений, генератор задач, счётчик.
2. Перенесён на D: в `D:\HTML`, затем пользователь сам переложил в `D:\HTML\Section`.
3. Добавлены: линия по двум точкам + «Продлевать линию (след)» + набросок.
4. Добавлены: ластик (удаление точки кликом) и кнопка «Убрать точку» в задачах.
5. Последнее: привязка точек к пересечениям линий, «Линия по грани», «Очистить всё», маркеры пересечений.

## 15. Известные ограничения и идеи
- `lineEdgeIntersections()` осталась в коде неиспользуемой (дублируется `collectIntersections`) — можно удалить при чистке.
- Точки без ребра корректно показываются, но объяснение для них даётся в координатах, а не как «на ребре XY».
- Снап к пересечениям работает в экранных пикселях: при сильном зуме радиус 15px может захватывать лишнее.
- Возможные направления развития: подписи у точек-пересечений, продление «линии по грани» до рёбер, автоматическое построение сечения из нарисованных линий, офлайн-версия с локальным three.js.
- `raycaster.params.Line.threshold = 0.12`.

## 16. Git workflow
Рабочее дерево было чистым на момент написания этого файла. Перед пушем: показать `git status --short` и summary изменений, затем:
```powershell
git -C 'D:\HTML\Section' add agent.md sections-trainer.html
git -C 'D:\HTML\Section' commit -m "Add agent guide for section trainer"
git -C 'D:\HTML\Section' push origin main
```
Push идёт по SSH в `git@github.com:ShedowKonfas/sections-trainer.git`. Если SSH-ключ не настроен — сообщить Жене, не переписывать remote.
