# Этап 5. ADT, атрибуты, операции и связи

## Преобразование понятий

| Исходное понятие | Элемент модели / владелец | Основание |
| --- | --- | --- |
| Плеер состоит из компонентов | `AudioPlayer.ui/controller/pluginManager/mediaLibrary/connections` | R1/R8, композиция при A1. |
| Подключаемый / установленный / загружаемый модуль | `Plugin`, атрибут `state` | Разные состояния одного объекта; наследование не нужно. |
| Загрузка / включение / выключение | `load/enable/disable` у `PluginManager` и `Plugin` | Менеджер ищет объект, объект выполняет переход своего состояния. |
| Установленный модуль | `Plugin.moduleId/installedVersion/packagePath` и `PluginManager.installed` | Установка предшествует загрузке; R6/R10, A3/A8. |
| Команда, ввод | `Command.requestId/kind/pluginId/source`, `UserInterface.submit` | Ввод становится типизированным запросом. |
| Асинхронность | `PlayerController.queue/busy`, `UserInterface.pending`, `Future<Outcome>` | Технический механизм, не класс предметной области. |
| Основная функциональность | `PlayerController.play/pause/stop`, состояние воспроизведения | Уточнение A9 к R3. |
| Взаимодействие компонентов и модулей | Ассоциации L14–L16, `Plugin.onEvent` | Ссылки доступны после загрузки, обработка событий — после включения. |
| Соединение | `ConnectionManager.state/serverUri/timeout`, `connect/disconnect` | Один канал на менеджер в принятой модели; отдельный `Connection` избыточен. |
| Проверка обновлений | `installedVersions`, `checkUpdates`, `requestUpdates`, `findUpdates` | R10 распределяется между владельцами реестра, транспорта и базы. |
| Доступный модуль / обновление | `PluginDescriptor.moduleId/version/downloadUri` | Сведения о версии; не загруженный серверный `Plugin`. |
| Пользователь | Актор `User` | Нет требований к профилю или учётной записи. |
| Компонент вообще | Конкретные классы и связи | Общий интерфейс в условии отсутствует; `PlayerComponent` не введён. |
| Система | Граница use cases; пакеты «Клиент» и «Сервер» | Ни класс, ни агрегат. |
| ADT, класс, операция, метод Аббота | Термины описания решения | Не элементы модели приложения. |

## Сигнатуры ADT

Поля закрыты; их изменение происходит только через операции владельца.
Коллекции и ссылки на диаграмме являются теми же свойствами, что и в этой таблице:
атрибут и подписанный конец ассоциации не означают два разных хранилища.

| ADT | Данные и ссылки | Интерфейс |
| --- | --- | --- |
| `AudioPlayer` | Пять неизменных ссылок `ui`, `controller`, `pluginManager`, `mediaLibrary`, `connections` | Собственные операции не требуются: вход в приложение через UI. |
| `UserInterface` | `pending: Map<RequestId, Future<Outcome>>`; `controller: PlayerController` | `submit(command: Command): Future<Outcome>`; `onCompleted(requestId: RequestId, outcome: Outcome): void`. |
| `PlayerController` | `queue: Queue<Command>`; `busy: Bool`; `playbackState: PlaybackState`; `currentSource: URI?`; `activeHandle: MediaHandle?`; `plugins: PluginManager`; `media: MediaLibrary` | `enqueue(command): Future<Outcome>`; внутренний `processNext(): void`; `play(source): Outcome`; `pause(): Outcome`; `stop(): Outcome`. |
| `MediaLibrary` | `handles: Set<MediaHandle>` — технические ресурсы | `open(source: URI): MediaHandle`; `play(handle): void`; `pause(handle): void`; `close(handle): void`. Ошибки I/O преобразуются контроллером в `Outcome`. |
| `PluginManager` | `installed: Map<PluginId, Plugin>`; ссылки `connections`, `ui`, `controller`, `media` | `find(id): Plugin?`; `load(id): Outcome`; `enable(id): Outcome`; `disable(id): Outcome`; `installedVersions(): Map<PluginId, Version>`; `checkUpdates(): Future<Outcome>`. |
| `Plugin` | `moduleId: PluginId`; `installedVersion: Version`; `packagePath: Path`; `state: PluginState`; опциональные ссылки `ui`, `controller`, `media` | `load(ui, controller, media): Outcome`; `enable(): Outcome`; `disable(): Outcome`; `onEvent(event: Event): void`. Инициализация экземпляра восстанавливает запись об уже установленном пакете. |
| `ConnectionManager` | `serverUri: URI`; `state: ConnectionState`; `timeout: Duration`; `server: PluginServer?` | `connect(): Future<void>`; `requestUpdates(versions: Map<PluginId, Version>): Future<List<PluginDescriptor>>`; `disconnect(): void`. Сетевые ошибки завершают Future с ошибкой. |
| `PluginServer` | `catalog: ModuleCatalog` | `checkUpdates(versions: Map<PluginId, Version>): List<PluginDescriptor>`. |
| `ModuleCatalog` | `releases: Set<PluginDescriptor>` | `versions(id: PluginId): List<PluginDescriptor>`; `findUpdates(installed: Map<PluginId, Version>): List<PluginDescriptor>`. |
| `PluginDescriptor` | Неизменяемые `moduleId: PluginId`, `version: Version`, `downloadUri: URI` | `isNewerThan(installedVersion: Version): Bool`; сравнение версии применимо после совпадения `moduleId`. |
| `Command` | Неизменяемые `requestId: RequestId`, `kind: CommandKind`, `pluginId: PluginId?`, `source: URI?` | `validate(): Bool`; недопустимые сочетания аргументов отвергаются до постановки в очередь. |

`PluginState = {INSTALLED, LOADED, ENABLED}`;
`PlaybackState = {STOPPED, PLAYING, PAUSED}`;
`ConnectionState = {DISCONNECTED, CONNECTING, CONNECTED}`;
`CommandKind = {PLAY, PAUSE, STOP, LOAD_PLUGIN, ENABLE_PLUGIN, DISABLE_PLUGIN, CHECK_UPDATES}`.

Скалярные типы `PluginId`, `RequestId`, `Version`, `URI`, `Path`, `Duration`,
`MediaHandle`, `Event` — значения/API-типы, а не дополнительные классы этой модели.
`Version` сравнивается по согласованному порядку A6, не лексикографически как строка.
`Event` означает событие протокола расширения; его содержимое зависит от модуля.

`Outcome` — тип-сумма, не дополнительная сущность:

```text
Outcome = Success | Updates(List<PluginDescriptor>) | Failure(ErrorCode, String)
ErrorCode = InvalidCommand | UnknownPlugin | NotLoaded | LoadFailed
          | TransitionFailed | MediaError | NetworkError | ServerError
```

`Future<T>`, `Queue<T>`, `Map<K,V>`, `Set<T>`, `List<T>` — стандартные технические
типы. Создание и завершение promise, запись callback и пробуждение обработчика
очереди являются механизмом исполнения этих ADT, не отдельными классами.

## Именованные связи и роли

В записи **A (mA) — B (mB)** `mB` — число объектов B для одного A, `mA` — число
объектов A для одного B. Роль у конца B именует свойство A, ссылающееся на B.
Обратная роль не обязательно означает хранимую обратную ссылку.

| ID | A (mA) | B (mB) | Тип и имя | Роли у концов A / B | Основание |
| --- | --- | --- | --- | --- | --- |
| L1 | AudioPlayer (1) | UserInterface (1) | Композиция «владеет UI» | player / ui | R1, A1. |
| L2 | AudioPlayer (1) | PlayerController (1) | Композиция «владеет контроллером» | player / controller | R1, A1. |
| L3 | AudioPlayer (1) | PluginManager (1) | Композиция «владеет менеджером модулей» | player / pluginManager | R1, A1. |
| L4 | AudioPlayer (1) | MediaLibrary (1) | Композиция «владеет библиотекой» | player / mediaLibrary | R1, A1/A2: runtime-экземпляр, не файл общей библиотеки. |
| L5 | AudioPlayer (1) | ConnectionManager (1) | Композиция «владеет менеджером соединений» | player / connections | R8, A1. |
| L6 | UserInterface (1) | PlayerController (1) | Ассоциация «передаёт команды» | input / controller | R2/R7, A1/A5. |
| L7 | PlayerController (1) | PluginManager (1) | Двунаправленная ассоциация «координирует модули» | controller / plugins | R3/R6; обратная ссылка нужна для `Plugin.load`. |
| L8 | PlayerController (1) | MediaLibrary (1) | Ассоциация «воспроизводит аудио через» | playback / media | R3/A2/A9. |
| L9 | PluginManager (1) | Plugin (0..*) | Ассоциация «учитывает установленные» | manager / installed | R6/R10, A3; не композиция с пакетами на диске. |
| L10 | PluginManager (1) | ConnectionManager (1) | Ассоциация «запрашивает обновления через» | requester / connections | R10, A1. |
| L11 | PluginManager (1) | UserInterface (1) | Ассоциация «передаёт модулям UI» | provider / ui | R4/A7: передача контекста в `Plugin.load`. |
| L12 | PluginManager (1) | MediaLibrary (1) | Ассоциация «передаёт модулям библиотеку» | provider / media | R4/A7. Контроллер передаётся через L7. |
| L13 | PlayerController (0..1) | Command (0..*) | Ассоциация «ожидает или исполняет» | executor / commands | R7/A5: очередь и максимум одна текущая команда; ещё не принятый запрос имеет 0 исполнителей. |
| L14 | Plugin (0..*) | UserInterface (0..1) | Ассоциация «использует UI» | extensions / ui | R4/A7. |
| L15 | Plugin (0..*) | PlayerController (0..1) | Ассоциация «использует управление» | extensions / controller | R4/A7. |
| L16 | Plugin (0..*) | MediaLibrary (0..1) | Ассоциация «использует мультимедиа» | extensions / media | R4/A7. |
| L17 | ConnectionManager (0..*) | PluginServer (0..1) | Ассоциация «подключён к» | clients / server | R10/A6: 0 при отсутствии соединения, 1 при `CONNECTED`. |
| L18 | PluginServer (1) | ModuleCatalog (1) | Ассоциация «читает доступные версии» | service / catalog | R9/A6. База имеет независимый жизненный цикл. |
| L19 | ModuleCatalog (0..*) | PluginDescriptor (0..*) | Ассоциация «хранит описания» | catalogs / releases | R9: пустой каталог допустим; значение может быть скопировано в ответ или использовано несколькими каталогами. В рассматриваемой системе каталог один. |
| D1 | UserInterface | Command | Зависимость «принимает значение» | consumer / argument | R7. Кратности неприменимы к зависимости. |
| D2 | ConnectionManager | PluginDescriptor | Зависимость «возвращает сведения» | transport / updateInfo | R10. Кратности неприменимы. |
| D3 | PluginManager | PluginDescriptor | Зависимость «передаёт результат проверки» | coordinator / updateInfo | R10. Кратности неприменимы. |
| D4 | PluginServer | PluginDescriptor | Зависимость «формирует ответ» | responder / updateInfo | R10. Кратности неприменимы. |

Все структурные кратности определены в пределах A1/A3/A5–A7. Текст задания сам
по себе не задаёт эти числа: без предположений остаются неустановленными
совместное использование компонентов, число серверов и соединений. Отдельное
наследование и UML-агрегация с пустым ромбом не нужны: отношение «вид» не задано,
а обычные ссылки не требуют слабой агрегации.

## Контракты модуля

Во всех строках пакет уже установлен; отсутствующий ID отклоняется менеджером
с `UnknownPlugin`, не создавая пустую запись. Все обращения к реестру ищут объект
по ключу, а не по текущему состоянию.

| Операция | INSTALLED | LOADED | ENABLED |
| --- | --- | --- | --- |
| `load` | Проверить/загрузить пакет, получить доступ к компонентам; при успехе LOADED. | Success, без изменений. | Success, без изменений. |
| `enable` | Failure(NotLoaded), без изменений. | Активировать обработчики; при успехе ENABLED. | Success, без изменений. |
| `disable` | Success, без изменений: уже выключен. | Success, без изменений. | Остановить обработчики; при успехе LOADED. |

При ошибке загрузки или смены активности состояние и ранее действовавшие
регистрации сохраняются: операция откатывает свои частичные действия.
`onEvent` разрешён только в `ENABLED`. После успешного `disable` новые вызовы
`onEvent` не допускаются, выполнявшиеся обработчики завершены. Реализация
должна согласовать этот барьер с доставкой событий; конкретный механизм не задан.

`moduleId`, `installedVersion`, `packagePath` не меняются при этих операциях.
Саму установку обновления пользователь не запрашивал.

## Асинхронный контракт

1. `Command.validate` требует непустой `source` только для `PLAY`, `pluginId`
   только для трёх операций над модулем; для остальных команд оба аргумента пусты.
2. `submit` получает Future от `enqueue`, записывает его в `pending` и подписывает
   `onCompleted` на завершение. Подписка корректно работает и для уже завершённого
   Future. `requestId` уникален в пределах сеанса UI.
3. `enqueue` проверяет команду, создаёт promise результата, ставит запрос в FIFO
   и пробуждает исполнителя. Недопустимая команда получает `Failure(InvalidCommand)`.
4. `processNext` запускается только при `busy = false`. Пока команда выполняется,
   `busy = true`; следующая команда не меняет состояние того же плеера.
5. При сетевом запросе контроллер ожидает Future асинхронно; UI продолжает принимать
   ввод. При успехе или ошибке promise завершается ровно один раз, `busy` сбрасывается,
   запускается следующая команда. Ошибка не оставляет очередь заблокированной.
6. `onCompleted` получает тот же `requestId`, удаляет его из `pending` и отображает
   результат. Внутренние вызовы модулей, изменяющие состояние контроллера, также
   маршрутизируются через очередь; обработчики не ждут синхронно свою новую команду.

`play/pause/stop` и переходы состояния модулей доступны контроллеру как внутренний
протокол выполнения команды; пользователь не обходит очередь прямым вызовом.

## Проверка обновлений и инварианты

`checkUpdates` делает снимок `installedVersions()` по **всем** записям реестра,
вызывает `connect`, `requestUpdates`, затем `disconnect` в завершении, включая
ветку ошибки. Ошибка транспорта/сервера преобразуется в `Failure`, успешный
ответ — в `Updates`, в том числе с пустым списком. Пустой локальный реестр
допускает немедленный `Updates([])` без соединения.

Для каждого `(id, installedVersion)` каталог выбирает `max(versions(id))` и
включает описание в ответ лишь при `version > installedVersion`. Неизвестный
серверу ID просто не даёт обновления. В базе уникальна пара `(moduleId, version)`;
в ответе не больше одного описания на `moduleId`. Чтение сравнивает согласованный
снимок каталога. Модель не обещает, что найденная версия не устареет после ответа.

Управляющий компонент поддерживает инвариант: `STOPPED` означает отсутствие
`activeHandle` и `currentSource`; при `PLAYING/PAUSED` оба заданы. `play` открывает
и запускает ресурс через библиотеку, `pause` приостанавливает, `stop` закрывает.
Пауза уже приостановленного/остановленного плеера и повторная остановка — успешные
пустые операции. Ошибка нового воспроизведения освобождает созданные ресурсы и
оставляет контроллер в согласованном `STOPPED`.

Ни один `Plugin` не ссылается на компоненты другого плеера. При завершении
runtime-плеера ссылки расширений на его компоненты освобождаются; это не удаляет
установленные пакеты. Типы коллекций в интерфейсе не раскрывают изменяемые
внутренние контейнеры: клиент получает снимки либо доступ только для чтения.
