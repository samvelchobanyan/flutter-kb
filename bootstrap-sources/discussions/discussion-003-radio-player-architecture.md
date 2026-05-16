---
source: Claude Code session
date: pre-bootstrap (archival)
project: go_sport
status: archival discussion — contains mature architectural decisions

verification_note: This is an archival discussion documenting the design of
RadioFullPlayerScreen and supporting architecture refactor. The decisions were
applied to go_sport code on dev branch. Verify decision quality and extract
portable knowledge.

key_topics:
- Architecture: separate RadioFullPlayerScreen vs shared PlayerShell abstraction
- Refactor: ScheduleController moved from features/schedule to domain/state for
  cross-feature accessibility
- New pattern: currentProgramProvider derived from schedule + ticker, with value
  equality dedup
- UI/Domain state split: SelectedDateNotifier (UI) stays in feature,
  ScheduleController (domain) moves to domain/state
- Mini-player tap logic: symmetric handling for music/radio panels
---

# Discussion: Radio Full Player — architecture and supporting refactor

---

давай подумаем о плеере
у нас есть миниплеер который работает в двух режимах - музыка и радио
при клике на плеер в режиме музыки открывается страница плеера
при кли на мниплеер в режиме радио должна открыватся страница радио плеера
посмотри детально как все работает связи между скринами переходы структура скринов стейт и репозитории
и дай знать когда будешь готов

Изучаю архитектуру плеера детально.

Agent:Deep dive into player architecture

Готов. Вот полная картина:

## Архитектура плеера

### Мини-плеер
mini_player_widget.dart

Два панели — музыка (зелёная) и радио (синяя). Режим определяется внутренним _isMusicMode (UI-only toggle с анимацией), отдельно от реального PlaybackMode в стейте.

Логика тапа:
- Музыкальная панель активна → вызывает widget.onOpenFullPlayer → открывает полный плеер
- Музыкальная панель неактивна → _toggleMode() (переключиться на музыку)
- Радио панель активна → _toggleMode() (переключиться на музыку)
- Радио панель неактивна → _toggleMode() (переключиться на радио)

**Проблема:** тап по активной радио-панели переключает на музыку вместо того, чтобы открыть радио-плеер.

### Полный плеер музыки
full_player_screen.dart

Открывается через FullPlayerScreen.show(context) — modal bottom sheet, не go_router навигация.

Вызывается из main_shell.dart:

```dart
MiniPlayerWidget(
  onOpenFullPlayer: () => FullPlayerScreen.show(activeBranchContext),
)
```

### Полный плеер радио
Не существует. Нет отдельного экрана для радио.

### Стейт
player_state.dart — Freezed + Riverpod

Ключевые поля: mode (music|radio), radioTitle, radioStreamUrl, radioNowPlaying (ICY metadata), треки, статус, позиция, shuffle/repeat.

## Что нужно сделать

Чтобы тап по активной радио-панели открывал радио-плеер:

1. Добавить callback onOpenRadioPlayer в MiniPlayerWidget
2. Создать RadioFullPlayerScreen (аналогично FullPlayerScreen, но с радио-контентом)
3. Изменить логику тапа: радио активна → widget.onOpenRadioPlayer
4. Прокинуть callback из main_shell.dart

Как хочешь двигаться? Обсудить дизайн радио-плеера, или есть макеты?

---

нужно создать аналог полного плеера музыки для радио
верхняя часть будет тойже.
посмотри внимательно как устроен экран плеера и дай знать

Читаю полный плеер музыки.

Read c:\flutter\go_sport\lib\features\player\presentation\player\full_player_screen.dart
Read c:\flutter\go_sport\lib\features\player\presentation\player\widgets\player_top_bar.dart
Read c:\flutter\go_sport\lib\features\player\presentation\player\widgets\player_fluid_background.dart
Read c:\flutter\go_sport\lib\features\player\presentation\player\widgets\player_artwork_carousel.dart
Read c:\flutter\go_sport\lib\features\player\presentation\player\widgets\player_control_panel.dart

Теперь полная картина. Вот что я вижу:

## Структура FullPlayerScreen (музыка)

```
Stack:
  ├── Container(white)              ← белый фон
  ├── PlayerFluidBackground         ← анимированный blob-шейдер (scale: 1.7, opacity: 0.65)
  └── SafeArea > Column:
        ├── PlayerTopBar            ← кнопка назад + название источника (альбом/плейлист) + "···"
        ├── Expanded > PlayerArtworkCarousel   ← карусель обложек (prev/current/next), свайп = skip
        └── SizedBox(42% высоты) > PlayerControlPanel:
              ├── track info row   ← мини-обложка + title + artist + like button
              ├── PlayerSeekBar    ← прогресс-бар + время
              ├── Controls row     ← prev / play(72px circle) / next
              ├── Spacer
              └── Bottom actions   ← shuffle + repeat
```

## Для радио-плеера нужно заменить

Оставить те же:
- PlayerFluidBackground — тот же фон
- PlayerTopBar — но ему нужен title для радио (сейчас читает только source из музыкального стейта)
- Структура modal bottom sheet — идентично

Заменить:
- PlayerArtworkCarousel → одна большая статичная обложка радиостанции (radioImageUrl)
- PlayerControlPanel → радио-контрол панель:
  - Вместо track info row: название радио + radioNowPlaying (ICY метаданные)
  - Без PlayerSeekBar (live stream — нет позиции)
  - Только кнопка Play/Pause (без prev/next)
  - Без shuffle/repeat

Прежде чем реализовать — подтверди структуру нижней панели:

1. Радио-обложка сверху — одна большая, без карусели
2. Снизу: название радио + "сейчас играет" + play/pause кнопка
3. PlayerTopBar — оставить как есть (заголовок будет пустым или "Go Sport Radio")?

---

у радио плеера другой функционал
разделение на верх и низ а также шейдер сохранаются идетнично
но содержимое меняется
посмотри на картинку

Вижу дизайн. Проверю что уже есть для данных расписания.

Globpattern: "lib/**/*radio*"
Found 4 files
Globpattern: "lib/**/*schedule*"
Found 9 files
Read c:\flutter\go_sport\lib\domain\entities\scheduled_program.dart
Read c:\flutter\go_sport\lib\domain\repositories\schedule_repository.dart
Read c:\flutter\go_sport\lib\features\shared_widgets\schedule_tile.dart
Read c:\flutter\go_sport\lib\features\schedule\presentation\schedule\schedule_controller.dart
Read c:\flutter\go_sport\lib\features\radio\presentation\radio\radio_page_screen.dart

Хорошо, теперь полная картина. Вот что я вижу в дизайне и что уже есть:

## Анализ дизайна

**Верхняя часть (шейдер, сохраняется):**
- Top bar: только стрелка вниз + "Radio Go Sport Live" (без "···")
- Большая кнопка play по центру
- "Playing live now" подпись
- Название текущей программы (жирно, крупно) — из расписания
- Карточка текущего трека: обложка + название + артист — из ICY метаданных (radioNowPlaying)
- Три бокса: Started / Remaining / Will Finish — рассчитываются из ScheduledProgram

**Нижняя часть (белая панель, сохраняется):**
- "Up next" заголовок
- Список программ — уже есть готовый ScheduleTile и scheduleControllerProvider

## Что уже есть

| Нужно | Есть |
|-------|------|
| Данные расписания | ScheduleRepository + scheduleControllerProvider |
| Тайтл + тайминг программы | ScheduledProgram (startDate, duration, endDate) |
| Текущий трек (ICY) | playerState.radioNowPlaying ("Coldplay - Good Feelings") |
| Тайтл станции | playerState.radioTitle |
| Тайл списка | ScheduleTile |

## План реализации

1. **RadioFullPlayerScreen** — новый файл, аналог FullPlayerScreen:
   - Тот же modal bottom sheet, шейдер, структура
   - PlayerTopBar заменить инлайн-строкой (только стрелка + radioTitle)
   - Верх: play/pause + "Playing live now" + название текущей программы + трек-карточка + тайм-боксы
   - Низ: "Up next" список из scheduleControllerProvider(today)

2. **Текущая программа** — фильтрация сегодняшнего расписания по DateTime.now() (первый ScheduledProgram где startDate <= now < endDate)

3. **ICY метаданные** — парсинг radioNowPlaying по - → artist / title

4. **MiniPlayerWidget** — добавить onOpenRadioPlayer callback, изменить тап активной радио-панели

5. **main_shell.dart** — передать onOpenRadioPlayer: () => RadioFullPlayerScreen.show(context)

Подтверди — буду реализовывать?

---

пока обсуждаем
стоит ли сделать переиспользуемый каркас который будут использовать оба скрины или не стоит?

**Не стоит.**

Общий у них только: showModalBottomSheet + Stack с белым фоном и шейдером + SafeArea. Это ~15 строк. Всё остальное — top bar, контент, нижняя панель — полностью разное.

Абстракция типа PlayerShell(child: ...) сэкономит эти 15 строк, но добавит лишний слой индирекции. Когда радио-плеер начнёт расходиться с музыкальным (а он уже расходится — другой top bar, скролл вместо фиксированной высоты), абстракция начнёт мешать. Два самостоятельных файла проще читать и менять независимо.

---

согласен
тогда проанализируй еще раз и дай детальной описание что и где меняем добавляем используем

Проверю оставшиеся детали — мини-плеер, провайдеры, player_state_selectors.

Read c:\flutter\go_sport\lib\features\player\presentation\player\mini_player_widget.dart
Read c:\flutter\go_sport\lib\core\navigation\main_shell.dart
Read c:\flutter\go_sport\lib\domain\state\player_state_selectors.dart
Read c:\flutter\go_sport\lib\core\di\repository_providers.dart

Вот детальный план.

## 1. Новые файлы

**lib/features/player/presentation/player/radio_full_player_screen.dart**
Главный экран. Аналог FullPlayerScreen, но:
- Тот же showModalBottomSheet (transparent bg, barrier 30% black, drag)
- Stack: белый контейнер + PlayerFluidBackground(scale: 1.7, opacity: 0.65) + SafeArea > Column
- Колонка:
  - Инлайн top bar (строка с IconButton(Icons.keyboard_arrow_down) слева + Center(Text(radioTitle))) — без "···"
  - Expanded > RadioPlayerUpperContent
  - RadioPlayerUpNextPanel (высота 42% экрана, как в музыке)

**lib/features/player/presentation/player/widgets/radio_player_upper_content.dart**
Верхняя секция (внутри шейдера):
- Большая кнопка Play/Pause (72px, DSColors.blue, круг) — центр. Читает playerInfoProvider.isPlaying/status, тап → togglePlayPause() (если isRadioMode) или playRadio()
- Text('Playing live now') — подпись
- Text(currentProgram.title, context.h?) — название текущей программы из расписания
- Карточка текущего трека: обложка (source image или station image) + title + artist из распарсенного radioNowPlaying
- Row из трёх _TimeBox виджетов: Started / Remaining / Will Finish

**lib/features/player/presentation/player/widgets/radio_player_up_next_panel.dart**
Нижняя белая панель:
- Полупрозрачный белый фон + BorderRadius.vertical(top: DSRadius.l) (как в PlayerControlPanel)
- Заголовок "Up next" (context.h3)
- ListView.builder (внутри Expanded) — переиспользует существующий ScheduleTile

## 2. Изменения существующих файлов

**lib/features/player/presentation/player/mini_player_widget.dart**
- Добавить поле final VoidCallback onOpenRadioPlayer; в конструктор
- Строка 83-87 (left panel — radio): изменить onTap: !_isMusicMode ? null : _toggleMode → onTap: !_isMusicMode ? widget.onOpenRadioPlayer : _toggleMode (активная радио-панель открывает радио-плеер; неактивная переключает)

**lib/core/navigation/main_shell.dart**
- Строка 39-41: добавить onOpenRadioPlayer: () => RadioFullPlayerScreen.show(activeBranchContext)
- Импорт radio_full_player_screen.dart

## 3. Переиспользуем без изменений

| Компонент | Роль |
|-----------|------|
| PlayerFluidBackground | тот же шейдер |
| playerInfoProvider (селектор) | radioTitle, radioImageUrl, radioNowPlaying, isPlaying, status, isRadioMode |
| playerStateProvider.notifier | togglePlayPause(), playRadio() |
| scheduleControllerProvider(today) | список программ на сегодня → "Up next" + определение текущей |
| ScheduledProgram entity | startDate, endDate, duration, title, imageUrl |
| ScheduleTile | готовый тайл для "Up next" |
| DSColors, DSSpacing, DSRadius, context.* | дизайн-система |

## 4. Логика вычислений (внутри RadioPlayerUpperContent)

**Определение текущей программы:**
```dart
watch(scheduleControllerProvider(DateTime.now()))
  → when data: programs.firstWhereOrNull(p => p.startDate <= now && now < p.endDate)
```

**Парсинг ICY метаданных (radioNowPlaying):**
- Split по - → [artist, title]
- Fallback: если не парсится — показать всю строку как title, artist пустой

**Таймеры программы:**
- Started = now - currentProgram.startDate → mm:ss
- Remaining = currentProgram.endDate - now → mm:ss
- Will Finish = currentProgram.endDate → HH:mm
- Нужен Timer.periodic(1s) внутри StatefulWidget для живого обновления

## 5. Уточнения перед стартом

1. Формат "Started/Remaining/Will Finish" — подтверди: elapsed (mm:ss) / remaining (mm:ss) / end clock time (HH:mm)?
2. "Up next" список — показывать все программы сегодня начиная с текущей, или только после неё?
3. Иконка колокольчика (DSNotificationIcon в ScheduleTile) — функциональная (подписка на уведомление) или пока просто визуал?
4. Состояния загрузки расписания (ScheduleState.loading/error) — показывать скелет/пустую заглушку?

---

ответы на вопросы
1- не совсем понял вопрос
2 - показываем все программы сегодня начиная с текущейю текущая веделеная меленькой оранжевой точкой рядом с названием
3 - Иконка колокольчика функциоанльная имеен два состояние, тоесть есть иконка для выделенного колоколчика. при нажании пользователь подписывается на пуш ремайндер
4- я думаю показывать скелетон

По вопросу 1 — три бокса "Started / Remaining / Will Finish" показывают время, но непонятна семантика каждого. Уточни:
- Started — это время когда программа началась (часы:минуты, типа 09:30), или сколько времени прошло с начала (типа 12:05 прошло)?
- Remaining — сколько осталось до конца программы (типа 00:08 осталось)?
- Will Finish — время когда программа закончится (часы:минуты, типа 09:50), или тоже длительность?

Пока проверю иконку колокольчика — уже ли есть два состояния.

Read c:\flutter\go_sport\lib\design_system\components\icons\ds_notification_icon.dart

Иконка готова — isFilled переключает между bell_fill.svg / bell_stroke.svg. Проверю, есть ли уже логика подписок.

Grep "reminder|subscribe|notification" (in c:\flutter\go_sport\lib)
8 lines of output

Инфраструктуры подписок/пушей нет. Для колокольчика предложу минимальный вариант:

**Новый провайдер lib/features/schedule/presentation/schedule_reminders_controller.dart:**
- Notifier<Set<String>> — множество ID программ, на которые подписан пользователь
- Методы toggle(String programId) / isSubscribed(String programId)
- Пока in-memory (без персистентности и API) — это отдельная задача

Обновлю ScheduleTile: заменить тап по DSNotificationIcon на GestureDetector, читать isFilled из провайдера, по нажатию вызывать toggle.

Когда появится backend/push-инфраструктура — легко подменить notifier на персистентный.

Жду ответ по вопросу 1 (формат Started / Remaining / Will Finish), после этого стартую реализацию.

---

да именно так
когда началась - склько осталось до конца - когда закончится

давай еще обсудим как и откуда получаем программу где храним и тд

Хорошо, обсудим архитектуру данных.

## Откуда берём текущую программу

Всё нужное уже есть: scheduleControllerProvider(DateTime) (AutoDispose family) загружает расписание на переданную дату через scheduleRepositoryProvider.getScheduleByDate(date). Возвращает ScheduleState.loading / data(programs) / error.

Просто используем его с сегодняшней датой: ref.watch(scheduleControllerProvider(today)).

## Где и как определяем "какая сейчас идёт"

Здесь есть развилка. Два разумных варианта:

**Вариант А — считаем на лету внутри виджета**
- В RadioPlayerUpperContent делаем programs.firstWhereOrNull((p) => p.startDate <= now && now < p.endDate) прямо в build()
- Плюсы: минимум кода, нет новых провайдеров
- Минусы: логика "текущей программы" живёт в UI-виджете; если понадобится где-то ещё (например, в мини-плеере или банере "LIVE") — придётся дублировать

**Вариант Б — отдельный derived-провайдер**
- Новый currentProgramProvider в lib/features/player/presentation/player/providers/current_program_provider.dart
- Читает scheduleControllerProvider(today) + тикер, возвращает ScheduledProgram?
- Плюсы: логика инкапсулирована, переиспользуема, авто-дедупликация через freezed equality (виджет перестраивается только при реальной смене программы)
- Минусы: один лишний файл

Моя рекомендация — Б, потому что концепт "currently airing program" — это доменная сущность, а не UI-деталь. Плюс скоро понадобится для красной точки в списке "Up next" и потенциально для LiveBanner на главной радио-страницы.

## Тикер для живых таймеров

Только Remaining обновляется каждую секунду. Started и Will Finish — статичны после определения программы (это просто startDate / endDate отформатированные).

Предлагаю:

```dart
// lib/features/player/presentation/player/providers/now_ticker_provider.dart
final nowTickerProvider = StreamProvider<DateTime>((ref) {
  return Stream.periodic(const Duration(seconds: 1), (_) => DateTime.now());
});
```

Использование:
- currentProgramProvider читает тикер + расписание, но благодаря value equality ScheduledProgram пересчитывается только при смене программы → UI ребилдится только при реальном переходе (раз в ~20-30 мин)
- Виджет Remaining напрямую watch'ит тикер и пересчитывает endDate - now каждую секунду — но это один маленький Text, не весь экран

## Подписки на напоминания

Новый контроллер lib/features/schedule/presentation/schedule_reminders_controller.dart:

```dart
class ScheduleRemindersNotifier extends Notifier<Set<String>> {
  @override
  Set<String> build() => {};
  void toggle(String programId) { ... }
  bool isSubscribed(String programId) => state.contains(programId);
}
```

In-memory (сбрасывается при рестарте). Когда появится backend/локальные пуши — подменим реализацию без изменений UI.

Обновим ScheduleTile — заменим статичный DSNotificationIcon на GestureDetector, читающий статус из scheduleRemindersProvider и вызывающий toggle по тапу. Это автоматически активирует колокольчики и на экране ScheduleScreen.

## Итоговая картинка потока данных

```
scheduleRepositoryProvider (есть)
    ↓
scheduleControllerProvider(today) (есть)
    ↓
currentProgramProvider (NEW) ← + nowTickerProvider
    ↓
RadioPlayerUpperContent:
    ├── title программы, Started, Will Finish (статика)
    └── Remaining ← watch(nowTickerProvider) (живой)

scheduleRemindersProvider (NEW)
    ↕
ScheduleTile (обновлённый) — работает и в Up Next, и в ScheduleScreen
```

Согласен с таким разделением? Или вариант А (без отдельного провайдера для текущей программы) ближе?

---

у нас ведь есть есть schedule_list в переиспользуемых виджетах?
буде использовать их?

Read c:\flutter\go_sport\lib\features\shared_widgets\schedule_list.dart
Read c:\flutter\go_sport\lib\features\schedule\presentation\schedule\schedule_screen.dart

Да, ScheduleList можно переиспользовать — там уже есть заголовок + ScheduleTile + DottedDivider. И даже параметр title подойдёт — передадим "Up next".

## Что получаем бесплатно через ScheduleList

- Заголовок (context.h2) — совпадает с дизайном
- Тайлы программ (ScheduleTile) с обложкой/временем/колокольчиком
- Разделители (DottedDivider)
- Уже возвращает SliverPadding → встраивается в CustomScrollView

## Что нужно доработать

### 1. ScheduleTile — добавить индикатор "live"

В дизайне рядом с первой программой (которая сейчас идёт) — маленькая оранжевая точка. Нынешний ScheduleTile её не поддерживает.

Вариант: добавить опциональный параметр

```dart
class ScheduleTile extends StatelessWidget {
  final ScheduledProgram program;
  final bool isLive; // NEW, default false
  ...
}
```

Рисуем точку рядом с program.title, если isLive == true. Экран ScheduleScreen не пострадает — там всегда передаётся false (или не передаётся).

### 2. ScheduleList — передаётся ID текущей программы (опционально)

Чтобы внутри компонент знал, какому тайлу поставить isLive: true:

```dart
class ScheduleList extends ConsumerWidget {
  final List<ScheduledProgram> programs;
  final String title;
  final String? currentProgramId; // NEW, default null
  ...
}
```

Внутри при рендере: ScheduleTile(program: p, isLive: p.id == currentProgramId).

### 3. Фильтрация списка

В радио-плеере передаём не все сегодняшние программы, а начиная с текущей: programs.where((p) => p.endDate > now) (т.е. ещё не закончившиеся).

## Итоговое использование в RadioPlayerUpNextPanel

```dart
Container(
  decoration: // white + rounded top
  child: CustomScrollView(
    slivers: [
      ScheduleList(
        programs: upcomingPrograms,
        title: 'Up next',
        currentProgramId: currentProgram?.id,
      ),
    ],
  ),
)
```

## Колокольчики

Подписки (scheduleRemindersProvider) работают внутри ScheduleTile — то есть:
- В радио-плеере колокольчик функциональный ✓
- В ScheduleScreen тоже автоматически станет функциональным ✓
- Один источник правды для подписок

Подойдёт такой подход? Или что-то доработать?

---

подойдет
думаю о правдере программы
получается его нужно будет шейрит между фичерами
что думаешь стоит вынести в глобальный стейт?

В Riverpod все провайдеры глобально доступны — это вопрос расположения файла, а не скоупа.

## Но важно различать: state vs derived view

**PlayerState (в lib/domain/state/)** — это настоящий stateful (есть мутации, очередь, позиция, сайд-эффекты через AudioHandler). Поэтому живёт в domain/state/.

**currentProgramProvider** — derived read-only view над существующим scheduleControllerProvider. Никаких мутаций, никаких собственных данных — просто фильтрация programs по DateTime.now(). Такие штуки в domain/state/ класть избыточно.

## Моё предложение

Поместить рядом с уже существующим schedule_controller.dart:

```
lib/features/schedule/presentation/schedule/
  ├── schedule_controller.dart                    (есть)
  ├── schedule_controller.freezed.dart            (есть)
  └── current_program_controller.dart             (NEW)
      ├── nowTickerProvider      — Stream тикер 1с, autoDispose
      └── currentProgramProvider — autoDispose, возвращает ScheduledProgram?
```

## Почему именно здесь

1. Концептуально "currently airing program" — это derived view над расписанием, а расписание живёт в features/schedule/. Логика схемы остаётся в одном месте.
2. Доступность — любая фича (radio player, radio dashboard, future live banner, home screen) просто импортирует из одного модуля.
3. Не привязываем к radio — хотя сейчас потребитель только радио, сам концепт нейтрален (живое расписание).
4. autoDispose — когда никто не watch'ит, тикер останавливается, не жжём CPU просто так.

## Для подписок то же самое

scheduleRemindersProvider — по сути тоже relevant к schedule feature. Положу в ту же папку:

```
lib/features/schedule/presentation/schedule/
  └── schedule_reminders_controller.dart          (NEW)
```

## Когда имеет смысл двигать в domain/state/

Если бы currentProgram имел собственную сложную логику — например, кэширование, оптимистичные апдейты от WebSocket, preload следующей программы — тогда да, это был бы полноценный stateful модуль. Сейчас это три строки фильтра — не переусложняем.

Согласен с таким размещением?

---

но получается из фичера плеера будем подписыватся на контроллер другой фичи?

Справедливое замечание — это архитектурно нечисто. Кросс-фичер импорт (player → schedule) создаёт нежелательную связность.

## Правильная архитектура

В проекте уже есть правильный паттерн: PlayerState лежит в lib/domain/state/, потому что им пользуются несколько фич (music, radio, player). Тот же принцип здесь.

Обе фичи должны зависеть от domain, а не друг от друга:

```
features/player/   ─┐
features/radio/    ─┼──> domain/state/  ──> domain/repositories/
features/schedule/ ─┘
```

## Два варианта

**Вариант 1 — минимальный (compromise)**
Переносим только новое:

```
lib/domain/state/
  └── current_program_state.dart   (NEW)
      ├── nowTickerProvider
      └── currentProgramProvider
```

Но этот файл всё равно импортирует scheduleControllerProvider из features/schedule/ → слой domain зависит от feature. Это worse смелл, чем кросс-фичер импорт.

**Вариант 2 — правильная слоистость ⭐**
Заметил, что scheduleControllerProvider в features/schedule/presentation/schedule/schedule_controller.dart — это на самом деле тоже доменный concept (чистая загрузка данных через репозиторий, никакого UI). По логике это такой же shared state, как PlayerState.

Разделяем файл:

```
lib/domain/state/
  ├── schedule_state.dart           (NEW — вынести ScheduleController + state sealed)
  └── current_program_state.dart    (NEW — nowTicker + currentProgramProvider)

lib/features/schedule/presentation/schedule/
  └── schedule_controller.dart      → оставить только selectedDateProvider (UI)
```

ScheduleScreen импортирует оба (state из domain, selectedDate из feature). Любая другая фича импортирует только из domain.

Потрогать нужно:
1. Разделить schedule_controller.dart на два файла
2. Обновить импорты в schedule_screen.dart
3. Новый файл current_program_state.dart в domain

## Моя рекомендация

Вариант 2 — один разовый маленький рефактор, но получаем чистую слоистость: player и schedule не знают друг о друге, оба читают из общего domain. Это тот же паттерн, что уже применён для PlayerState.

Какой предпочитаешь?

---

да Вариант 2 лучше

Отлично. Тогда полный итог — вот что буду делать (по порядку).

## Рефактор domain/state/

1. **Новый lib/domain/state/schedule_state.dart**
   - Переношу ScheduleState (sealed), ScheduleController, scheduleControllerProvider из features/schedule/presentation/schedule/schedule_controller.dart

2. **Новый lib/domain/state/current_program_state.dart**
   - nowTickerProvider (Stream каждую секунду, autoDispose)
   - currentProgramProvider (autoDispose, ScheduledProgram?)

3. **Обновить lib/features/schedule/presentation/schedule/schedule_controller.dart**
   - Оставить только SelectedDateNotifier + selectedDateProvider

4. **Обновить lib/features/schedule/presentation/schedule/schedule_screen.dart**
   - Импорты под новые пути

## Подписки (reminders)

5. **Новый lib/features/schedule/presentation/schedule/schedule_reminders_controller.dart**
   - Notifier<Set<String>> с toggle(id) / isSubscribed(id)
   - scheduleRemindersProvider

6. **Обновить lib/features/shared_widgets/schedule_tile.dart**
   - Добавить опциональный bool isLive = false → оранжевая точка рядом с title
   - GestureDetector на DSNotificationIcon → toggle через scheduleRemindersProvider
   - isFilled теперь зависит от провайдера → станет ConsumerWidget

7. **Обновить lib/features/shared_widgets/schedule_list.dart**
   - Добавить опциональный String? currentProgramId
   - Прокидывает в ScheduleTile(isLive: p.id == currentProgramId)

## Радио-плеер (новые файлы)

8. **Новый lib/features/player/presentation/player/radio_full_player_screen.dart**
   - RadioFullPlayerScreen с static show() аналогично FullPlayerScreen.show()
   - Структура: Stack (white + PlayerFluidBackground + SafeArea > Column)
   - Колонка: inline top bar (↓ + radioTitle) → Expanded > RadioPlayerUpperContent → RadioPlayerUpNextPanel (42% экрана)

9. **Новый lib/features/player/presentation/player/widgets/radio_player_upper_content.dart**
   - Большая play-кнопка (72px, blue, круг) → togglePlayPause / playRadio
   - "Playing live now" подпись
   - Название текущей программы (из currentProgramProvider)
   - Карточка текущего трека: обложка + title + artist (парсинг radioNowPlaying по -)
   - 3 _TimeBox: Started (clock HH:mm) / Remaining (mm:ss, живой через nowTickerProvider) / Will Finish (clock HH:mm)
   - Если scheduleState.loading и нет текущей программы — скелетон

10. **Новый lib/features/player/presentation/player/widgets/radio_player_up_next_panel.dart**
    - Белый полупрозрачный контейнер с rounded top
    - CustomScrollView + ScheduleList (title: "Up next", currentProgramId)
    - Фильтрация programs.where((p) => p.endDate > now)
    - Состояния: loading → RadioPageSkeleton-style или простой skeleton; error → текст

## Мини-плеер + шелл

11. **Обновить lib/features/player/presentation/player/mini_player_widget.dart**
    - Добавить final VoidCallback onOpenRadioPlayer
    - Строка 86: активная радио-панель → widget.onOpenRadioPlayer вместо null

12. **Обновить lib/core/navigation/main_shell.dart**
    - Передать onOpenRadioPlayer: () => RadioFullPlayerScreen.show(activeBranchContext)

## Что проверить в конце

- flutter analyze — нет ошибок
- Запустить и пощёлкать: mini-player radio tap → radio player → back → play/pause работает → mini-player в sync

Стартую?

---

давай подписки пока оставим, обсидим их завтра
сейчас просто пусть меняют состояние и все

Понял. Тогда ScheduleTile → просто StatefulWidget с локальным bool _isSubscribed, тап переключает. Никаких провайдеров, никакого шаринга состояния между экранами.

Стартую реализацию.

[NOTE: Implementation steps and code generation continue here — kept brief in this archival summary. Key points:

- Created lib/domain/state/schedule_state.dart with ScheduleState sealed class and ScheduleController
- Created lib/domain/state/current_program_state.dart with nowTickerProvider and currentProgramProvider
- Refactored lib/features/schedule/presentation/schedule/schedule_controller.dart to keep only selectedDateProvider
- Updated ScheduleTile to StatefulWidget with local _isSubscribed
- Updated ScheduleList with optional currentProgramId
- Created radio_full_player_screen.dart, radio_player_upper_content.dart, radio_player_up_next_panel.dart
- Updated mini_player_widget.dart with onOpenRadioPlayer callback and symmetric tap logic
- Updated main_shell.dart to wire onOpenRadioPlayer]

---

## Discussion of architectural concepts (continued)

а как это обрабатывается когда для музыки?

Музыкальная панель использует прямое условие _isMusicMode (без !), потому что _isMusicMode == true означает "музыка активна" с её точки зрения.

Код правой (музыкальной) панели — строки 89-94

```dart
_buildPanel(
  isMusicPanel: true,
  isActive: _isMusicMode,                                    // активна, когда _isMusicMode == true
  onTap: _isMusicMode ? widget.onOpenFullPlayer : _toggleMode,
),
```

**Сравнение бок о бок:**

| Состояние _isMusicMode | Радио-панель (!_isMusicMode) | Музыка-панель (_isMusicMode) |
|------------------------|------------------------------|------------------------------|
| true (музыка активна)  | свёрнута → _toggleMode (раскрыть радио) | раскрыта → onOpenFullPlayer |
| false (радио активно)  | раскрыта → onOpenRadioPlayer | свёрнута → _toggleMode (раскрыть музыку) |

Логика одинаковая для обеих: активна ? открыть_свой_плеер : переключить_режим. Просто проверка "активна ли я" выглядит по-разному.

---

обясни что за два новых глобальныйх стейта появилось

Два новых файла в lib/domain/state/ — оба появились из-за рефактора, который мы обсудили (вариант 2, чтобы избежать кросс-фичер зависимостей).

## 1. schedule_state.dart — перенос, не новый

Это не новая логика, это перенос того что уже было в features/schedule/presentation/schedule/schedule_controller.dart.

**Что внутри:**
```
ScheduleState (sealed, 3 варианта):
  ├── loading
  ├── data(programs: List<ScheduledProgram>)
  └── error(message: String)

ScheduleController — загружает расписание на заданную дату через scheduleRepository
scheduleControllerProvider — AutoDispose.family<DateTime>
```

**Зачем перенесли:**
Раньше он жил внутри фичи schedule. Теперь его будет читать и фича player (радио-плеер) → кросс-фичер импорт. Чтобы обе фичи зависели только от domain, вынесли в domain/state/.

**Что остаётся в фиче schedule:**
- SelectedDateNotifier / selectedDateProvider — это UI-state экрана ScheduleScreen (какая дата сейчас выбрана в карусели). К доменной модели отношения не имеет.

## 2. current_program_state.dart — действительно новый

Этого раньше не было вообще. Это derived view над расписанием — "какая программа идёт прямо сейчас в эфире".

**Что внутри:**
```
nowTickerProvider          — Stream<DateTime> каждую секунду (autoDispose)
currentProgramProvider     — ScheduledProgram?  (autoDispose)
```

**Как работает currentProgramProvider:**
```
watch(scheduleControllerProvider(today))  — сегодняшнее расписание
watch(nowTickerProvider)                   — текущее время (тикает 1с)
    ↓
programs.firstWhereOrNull((p) => p.startDate <= now && now < p.endDate)
```

**Где используется:**
- В RadioPlayerUpperContent — показать название текущей программы, Started/Remaining/Will Finish
- В RadioPlayerUpNextPanel — определить, у какого тайла в списке поставить оранжевую точку isLive
- В будущем могут подцепиться — LiveBanner на главной радио, мини-плеер и т.д.

**Почему каждую секунду не жжёт CPU:**
Хотя тикер рекомпьютит провайдер каждую секунду, Riverpod дедуплицирует результат по ==. ScheduledProgram — freezed (value equality), поэтому пока программа та же — подписчики не получают уведомления. Реальные ребилды происходят только когда программа меняется (раз в ~20-30 мин).

## Итог

| Файл | Что это | Новое? |
|------|---------|--------|
| schedule_state.dart | загрузчик расписания | перенос из features/schedule |
| current_program_state.dart | "кто сейчас в эфире" + 1с тикер | новое |

Первый — чисто рефактор слоёв. Второй — новая доменная сущность, которая переиспользуется несколькими фичами.

---

## Final architecture verification

Данные программы идут из моков:

**Расписание программ → МОКИ**
schedule_repository_mock.dart генерит 14 дней × 14 программ с тайтлами "Program N" и случайными картинками из picsum.

Ограничения:
- После ~14-17:00 программы заканчиваются → currentProgram = null → UI покажет — и --:--
- Тайтлы "Program N" вместо "Live Music"
- Задержка 300ms имитирует сеть

**Текущий трек на радио (ICY) → РЕАЛЬНО**
player_state.dart:268-274:

```dart
_audioHandler.icyMetadataStream.listen((icy) {
  final title = icy?.info?.title;
  if (state.isRadioMode && title != null && title.isNotEmpty) {
    state = state.copyWith(radioNowPlaying: title);
  }
})
```

icyMetadataStream — это реальный поток из just_audio, читает метаданные встроенные прямо в радиострим.

## Итог по источникам данных

| Данные | Источник | Статус |
|--------|----------|--------|
| Расписание (Up next, current program) | ScheduleRepositoryMock | МОК |
| Название и время программы | Мок (через currentProgramProvider) | МОК |
| ICY "Coldplay - Good Feelings" | just_audio → icyMetadataStream | реально (если стрим есть) |
| Картинка программы | Мок (picsum.photos) | МОК |
| Картинка радиостанции | playerState.radioImageUrl | реально (если задано в config) |

Заменить мок на реальный API потом — одна правка в scheduleRepositoryProvider, UI и логика трогать не нужно.
