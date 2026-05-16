---
source: Claude Code session
date: pre-bootstrap (archival)
project: go_sport
status: archival discussion — contains mature decisions applied to code

verification_note: This is an archival discussion that closes the open investigation
"modal-in-stateful-shell-route" deferred during bootstrap. The decisions in this
discussion were applied to go_sport code on dev branch. Verify decision quality
(maturity, reasoning) but do not require strict timing alignment with current code.

key_topics:
- Bug investigation: modal bottom sheet behavior after returning from background
- Root cause: GlobalKey timing in branch-navigator-key resolver pattern
- Decision: replace branch-resolver pattern with useRootNavigator true
- Three equivalent solutions compared, chose simplest
---

# Discussion: Modal in StatefulShellRoute — bug fix and approach simplification

---

я заметил баг в приложении
когда приложение запущенно работает нормально, ботомщит плеера открывается поверх шела
когда переключаюсь на лругое приложение потом возвращаюсь обратно боттомшин перестает закрывать шел и ботомнавигешн а открвается над ним

Grep "bottomSheet|BottomSheet|showModalBottomSheet|player" (glob: lib/**/*.dart)
37 lines of output
Globpattern: "lib/**/*shell*.dart"
Found 2 files
Globpattern: "lib/**/*player*.dart"
Found 14 files
Read c:\flutter\go_sport\lib\core\navigation\main_shell.dart
Read c:\flutter\go_sport\lib\features\player\presentation\player\full_player_screen.dart
Read c:\flutter\go_sport\lib\core\navigation\routes.dart
Grep "branchNavigatorKey|StatefulShellBranch|navigatorKey" (glob: lib/**/*.dart)
11 lines of output
Read c:\flutter\go_sport\lib\core\navigation\app_router.dart (lines 31-110)

Нашёл причину бага. Сначала покажу её ниже прежде чем что-то менять.

Read c:\flutter\go_sport\lib\features\player\presentation\player\radio_full_player_screen.dart
Read c:\flutter\go_sport\lib\features\player\presentation\player\mini_player_widget.dart

Нашёл причину. Код менять не буду пока не согласуем — предлагаю вариант исправления.

## Причина бага

В main_shell.dart:29-33:

```dart
final activeBranchContext = activeBranchNavigatorKey?.currentContext ?? context;
```

Этот контекст передаётся в FullPlayerScreen.show(activeBranchContext) / RadioFullPlayerScreen.show(activeBranchContext), которые вызывают showModalBottomSheet(context: …, useRootNavigator: false) (full_player_screen.dart:19-22, radio_full_player_screen.dart:21-24).

Что происходит:

**Первый запуск** — при первом build() MainShell GlobalKey ветки ещё не привязан к элементу, поэтому activeBranchNavigatorKey.currentContext == null, и срабатывает fallback на context самого MainShell. showModalBottomSheet с useRootNavigator: false берёт ближайший Navigator сверху — это корневой Navigator. Шит открывается во весь экран, поверх BottomNav и MiniPlayer. Это то, что ты видишь как «нормально».

**После возврата из фона** — MainShell пересобирается, branch-навигатор уже смонтирован, currentContext возвращает реальный контекст ветки. Теперь showModalBottomSheet(context: branchContext, useRootNavigator: false) ограничен именно branch Navigator'ом — сидит только в области body, а BottomNav и MiniPlayer остаются сверху видимыми. Это и есть баг, который ты описал.

То есть поведение зависит от случайности — успел ли GlobalKey получить контекст к моменту тапа.

## Предлагаемое исправление

Если задуманное поведение — шит всегда на весь экран (как «нормально»):

1. В full_player_screen.dart:22 и radio_full_player_screen.dart:24 поменять useRootNavigator: false → true.
2. В main_shell.dart убрать branchNavigatorKeyResolver и вызов FullPlayerScreen.show(context) / RadioFullPlayerScreen.show(context) делать с context самого MainShell — тогда у корневого Navigator гарантированно стабильный контекст при любом сценарии. Параметр branchNavigatorKeyResolver из app_router.dart:169 тоже удаляем.

Это убирает зависимость от таймингов build и делает открытие плеера детерминированным.

Альтернатива — если наоборот хотели чтоб шит всегда сидел внутри branch-навигатора (т.е. BottomNav всегда виден): тогда нужно не трогать useRootNavigator: false, но резолвить контекст в момент тапа, а не в build — через branchNavigatorKeyResolver(currentIndex)!.currentContext! внутри callback'а.

Какой из вариантов реализуем?

---

а если только поменяем useRootNavigator: false → true. то как будет работать?

Да, работать будет — и это более минимальный фикс.

useRootNavigator: true заставляет showModalBottomSheet искать корневой Navigator через Navigator.of(context, rootNavigator: true), проходя вверх по дереву. Какой именно context передали (branch или shell) — уже не важно: оба вложены в root Navigator, поиск всё равно дойдёт до корня. Шит всегда будет открываться во весь экран, поверх BottomNav и MiniPlayer, независимо от того, получил ли GlobalKey ветки свой currentContext или сработал fallback.

Единственный «хвост» — код branchNavigatorKeyResolver в main_shell.dart и app_router.dart:48-59 станет ненужным (будет использоваться только для fallback в context, который и так эквивалентен). Но функционально всё будет корректно — можно почистить потом отдельно.

Если хочешь — делаем только две правки: false → true в full_player_screen.dart:22 и radio_full_player_screen.dart:24. Подтверждай — и внесу.

---

а если убрать этот код а usenavigator оставить false ?

Да, это тоже сработает — и даже чище.

Логика: MainShell сам сидит над branch-навигаторами (branch-навигаторы живут внутри body: navigationShell, а BottomNav и MiniPlayer — в bottomNavigationBar). Ближайший Navigator над самим MainShell — это уже корневой Navigator, созданный GoRouter.

Поэтому если передавать в FullPlayerScreen.show(context) именно context MainShell (а не branch-контекст), то showModalBottomSheet(context: shellContext, useRootNavigator: false) через Navigator.of(shellContext) найдёт корневой Navigator. Шит откроется во весь экран, поверх BottomNav и MiniPlayer — то, что нужно.

То есть функционально оба варианта эквивалентны:

A: оставить branchNavigatorKeyResolver, поменять false → true — минимум правок, но branchNavigatorKeyResolver становится мёртвым кодом.
B: убрать branchNavigatorKeyResolver, оставить false, передавать context MainShell — правок чуть больше, но код чище (выкидываем неиспользуемый резолвер из main_shell.dart и app_router.dart:41-59,169).

Делаем вариант B? Подтверждай — внесу правки.

---

твое исправление номер2
было именно про то чтобы передавать в FullPlayerScreen.show(context) именно context MainShell ?

Нет, я неточно сформулировал. Давай перечитаю что я писал.

Мой первый вариант был про "чтобы шит всегда на весь экран":
- useRootNavigator: false → true
- Убрать branchNavigatorKeyResolver, передавать context MainShell

То есть передача context MainShell была частью первого варианта, в паре с true.

Моя альтернатива была противоположной по цели — «если хотим чтоб шит всегда сидел внутри branch-навигатора, т.е. BottomNav всегда виден снизу»:
- Оставить useRootNavigator: false
- Оставить резолвер branch-контекста, но резолвить его в момент тапа, а не в build

То есть там наоборот нужен был branch-контекст, не shell-контекст.

Сейчас ты предлагаешь третью комбинацию: useRootNavigator: false + context MainShell. И по эффекту она эквивалентна моему первому варианту (шит во весь экран через root Navigator) — просто через другой механизм: не через флаг useRootNavigator, а за счёт того что контекст MainShell уже находится над всеми branch-навигаторами.

Итого три рабочие комбинации для «шит на весь экран»:

1. true + branch context
2. true + shell context
3. false + shell context ← твой вариант

И одна для «шит только в области body»:

4. false + branch context (резолв в момент тапа)

---

тоесть по сути мы передаем шторке контекс чтобы она открывалась в его навигаторе
а флаг useRootNavigator: true говорит шторке игнорировать и открыватся в корневом?

Да, ровно так — почти.

Маленький нюанс: контекст не «игнорируется» при useRootNavigator: true, он всё равно нужен как точка старта поиска. Flutter всё равно идёт вверх по дереву от этого контекста — разница только в том, где остановиться:

- useRootNavigator: false (по умолчанию) — Navigator.of(context) берёт ближайший Navigator, найденный при подъёме вверх от контекста.
- useRootNavigator: true — Navigator.of(context, rootNavigator: true) поднимается до самого корневого Navigator, пропуская все промежуточные.

Плюс контекст ещё используется чтобы унаследовать Theme, MediaQuery, Directionality и т.п. — независимо от флага.

Поэтому твой вариант (false + shell-контекст) работает за счёт того, что над MainShell нет промежуточных Navigator'ов — ближайший же сразу root. А вариант с true работает независимо от того, сколько навигаторов над контекстом — он всё равно долезет до корня.

---

давай сделаем как ты предлагал в начале
кажется это самы чистый вариант

Делаю правки.

Edit c:\flutter\go_sport\lib\features\player\presentation\player\full_player_screen.dart
Modified
Edit c:\flutter\go_sport\lib\features\player\presentation\player\radio_full_player_screen.dart
Modified

Теперь чищу branchNavigatorKeyResolver из MainShell и app_router.

Edit c:\flutter\go_sport\lib\core\navigation\main_shell.dart
Edit failed
Reason: может оставим ключ и просто не будем его передавать при открытии шторки? или он не нужен?

Проверю где ключи используются.

Grep "_homeBranchNavigatorKey|_musicBranchNavigatorKey|_radioBranchNavigatorKey|_branchNavigatorKeyResolver|BranchNavigatorKeyResolver|branchNavigatorKeyResolver" (glob: lib/**/*.dart)
15 lines of output

Разделю две вещи:

**1. Сами ключи** _homeBranchNavigatorKey, _musicBranchNavigatorKey, _radioBranchNavigatorKey (app_router.dart:41-46) — нужны, они передаются в StatefulShellBranch(navigatorKey: ...) (app_router.dart:175,206,304). Это часть архитектуры go_router: у каждой ветки свой Navigator, ключи позволяют к ним обращаться. Их оставляем.

**2. Резолвер** — функция _branchNavigatorKeyResolver, typedef BranchNavigatorKeyResolver, параметр branchNavigatorKeyResolver в MainShell и всё что с ним связано — используется только для получения контекста ветки при открытии шторки (activeBranchContext). Больше нигде в коде не задействовано. После фикса шторка открывается через context MainShell, и весь этот механизм становится мёртвым кодом.

Предлагаю удалить именно резолвер (пункт 2), а ключи оставить. Подтверждаешь — или хочешь оставить резолвер на будущее «вдруг пригодится»?

---

нет делаем как ты предложил

Edit c:\flutter\go_sport\lib\core\navigation\main_shell.dart
Removed 12 lines
Read c:\flutter\go_sport\lib\core\navigation\app_router.dart (lines 161-180)
Edit c:\flutter\go_sport\lib\core\navigation\app_router.dart
Removed 13 lines
Edit c:\flutter\go_sport\lib\core\navigation\app_router.dart
Removed 1 line

Готово. Итог правок:

- full_player_screen.dart:22 — useRootNavigator: true
- radio_full_player_screen.dart:24 — useRootNavigator: true
- main_shell.dart — удалён typedef BranchNavigatorKeyResolver, параметр branchNavigatorKeyResolver и вычисление activeBranchContext; в MiniPlayerWidget теперь передаётся context MainShell
- app_router.dart — удалена функция _branchNavigatorKeyResolver и её передача в MainShell

Ключи _homeBranchNavigatorKey / _musicBranchNavigatorKey / _radioBranchNavigatorKey оставлены — они нужны для StatefulShellBranch. Проверь поведение: открой плеер сразу после запуска и после возвращения из фона — шит теперь должен стабильно перекрывать шелл в обоих случаях.
