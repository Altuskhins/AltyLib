# AltyLib

Расширенная библиотека для плагинов exteraGram. Содержит DSL для настроек, типизированные заглушки для Telegram-контроллеров, менеджер безопасных хуков, форматирование сообщений, кеширование, готовые UI-компоненты и CLI-инструменты.

Документация актуальна для версии `1.0.4` (октябрь 2025).

---

## Содержание
1. [Быстрый старт](#быстрый-старт)
2. [Обзор API и структура библиотеки](#обзор-api-и-структура-библиотеки)
3. [CLI и генератор каркаса](#cli-и-генератор-каркаса)
4. [Хранилище и кеши](#хранилище-и-кеши)
5. [DSL настроек](#dsl-настроек)
6. [Системы высокого уровня](#системы-высокого-уровня)
   1. [Event Bus](#61-event-bus)
   2. [Управление командами](#62-управление-командами)
   3. [Планировщик задач](#63-планировщик-задач)
   4. [RPC-реестр](#64-rpc-реестр)
7. [Помощники Telegram API](#помощники-telegram-api)
8. [Хуки и жизненный цикл](#хуки-и-жизненный-цикл)
9. [Rate-limit и debounce](#rate-limit-и-debounce)
10. [Форматирование сообщений](#форматирование-сообщений)
11. [Медиа-хелперы](#медиа-хелперы)
12. [UI-компоненты](#ui-компоненты)
13. [Наблюдатели NotificationCenter](#наблюдатели-notificationcenter)
14. [Диагностика и хот-ридлоад](#диагностика-и-хот-ридлоад)
15. [Совместимость и feature flags](#совместимость-и-feature-flags)
16. [Cookbook: готовые рецепты](#cookbook-готовые-рецепты)
17. [Советы по разработке](#советы-по-разработке)
18. [FAQ и обратная связь](#faq-и-обратная-связь)

---

## Быстрый старт

```python
__id__ = "hello"
__name__ = "HelloPlugin"
__min_version__ = "11.9.1"

from base_plugin import BasePlugin, HookResult
import altylib as alty

settings = alty.create_settings_registry(__name__)

@settings.switch(default=True, summary="Отвечать на команду .hello")
def enabled(value: bool):
    alty.log(f"Hello command toggled to {value}")

class Hello(BasePlugin):
    def on_plugin_load(self):
        self.settings = settings
        alty.register_command("hello", self.cmd_hello, "Поздороваться")
        alty.events_subscribe("hello:ping", lambda _data: alty.bulletin_success(f"Pong from {__id__}!"))

    def cmd_hello(self, _self_pl, args, params):
        if not self.settings.get("enabled"):
            return "Команда выключена"
        alty.events_publish("hello:ping", None)
        return "Hello from AltyLib!"

    def on_send_message_hook(self, account, params):
        result = alty.handle_outgoing_command(self, account, params)
        return result or HookResult()
```

* `create_settings_registry()` создаёт DSL-реестр настроек и связанное хранилище.
* Все публичные API доступны через `altylib` после импорта (`import altylib as alty`).

---

## Обзор API и структура библиотеки

| Категория | API | Назначение |
|-----------|-----|------------|
| Логи | `log()/get_logs()/clear_logs()` | Унифицированный вывод и буфер последних 500 строк |
| UI | `bulletin_info/error/success`, `show_spinner/hide_spinner` | Всплывающие уведомления и «спиннер» |
| События | `events_subscribe/unsubscribe/publish` | Лёгкая шина между плагинами |
| Команды | `register_command/handle_outgoing_command/get_registered_commands` | Централизованный парсинг `.<command>` |
| Планировщик | `tasks_schedule/cancel/list` | Периодические фоновые задачи |
| RPC | `rpc_register/call` | Вызов процедур между плагинами |
| Данные | `JsonCacheFile`, `shared_*`, `KVStore`, `TTLCache` | Кеш и key-value хранилище |
| Прочее | `detect_client/language`, `get_plugins_dirs`, `get_current_activity`, Telegram-хелперы |

---

## CLI и генератор каркаса

AltyLib поставляется с CLI-командой `altylib`:

```bash
python -m altylib_cli init my_plugin --id hello --author @user
python -m altylib_cli validate my_plugin/hello.plugin
```

* `init` — создаёт минимальный плагин (шаблон с метаданными и точкой входа).
* `validate` — проверяет `.plugin`-файл на наличие обязательных полей.
* Встроенный валидатор сверяет схему и версии, чтобы легче следовать гайдам «First Plugin/Setup».

---

## Хранилище и кеши

### KVStore

```python
store = alty.KVStore("my_plugin")
store.set("counter", 1)
value = store.get("counter", 0)
```

* Значения сохраняются в JSON рядом с плагином, поддерживают примитивы и dict/list.
* Доступны готовые экземпляры: `alty.store_global` (общий namespace) и отдельные для DSL настроек.

### JsonCacheFile

```python
cache = alty.JsonCacheFile("history.json", default=[])
cache.content.append({"ts": time.time()})
cache.write()
```

* Автоматически создаёт папку `plugins/<your plugin>/cache/`.

### Shared store между плагинами

```python
alty.shared_set("counter", 42)
val = alty.shared_get("counter")          # 42
alty.shared_delete("counter")
```

* Хранится в `altylib_shared_store.json`.

### TTLCache

```python
cache = alty.TTLCache(ttl=30)
cache.set("dialog", data)
if cache.has("dialog"):
    use(cache.get("dialog"))
```

* Обновляет `updated_at` при каждом чтении, автоматически очищает просроченные ключи.
* Удобно для хранения результатов сетевых запросов и поиска сообщений.

---

## DSL настроек

`SettingsRegistry` описывает экран настроек через декораторы:

```python
settings = alty.create_settings_registry("hello", title="Настройки Hello")

@settings.switch(default=True, summary="Включить ответы")
def enabled(value: bool):
    alty.log(f"Enabled: {value}")

@settings.list(options=[("Быстро", "fast"), ("Медленно", "slow")], default="fast")
def mode(choice):
    alty.log(f"Mode set to {choice}")

@settings.slider(min_value=0, max_value=10, step=1, default=3)
def retries(value: float):
    alty.log(f"Retries: {value}")

# Показать экран настроек в UI
settings.open()
```

* Каждый декоратор возвращает `SettingHandle` с методами `.get()`, `.set()` и `.toggle()`.
* Значения автоматически сохраняются в `KVStore` и доступны через `settings.get()`.
* `settings.open()` строит AlertDialog/Preference UI с текущими значениями и колбэками `on_change`.

---

## Системы высокого уровня

### 6.1. Event Bus

```python
store = alty.KVStore("my_plugin")
store.set("counter", 1)
value = store.get("counter", 0)
```

* Значения сохраняются в JSON рядом с плагином, поддерживают примитивы и dict/list.
* Доступны готовые экземпляры: `alty.store_global` (общий namespace) и отдельные для DSL настроек.

### 6.2. Управление командами

1. Регистрация:
   ```python
   def cmd_echo(selfpl, args, params):      # сигнатура фиксирована
       return args or "Nothing to echo"
   alty.register_command("echo", cmd_echo, "Повторить текст")
   ```
2. Обработка: в своём `on_send_message_hook` первым делом вызываем
   ```python
   res = alty.handle_outgoing_command(self, account, params)
   if res:
       return res
   ```

*Если callback возвращает строку — она заменит исходящее сообщение; можно вернуть готовый `HookResult` для полного контроля.*

### 6.3. Планировщик задач

```python
settings = alty.create_settings_registry("hello", title="Настройки Hello")

@settings.switch(default=True, summary="Включить ответы")
def enabled(value: bool):
    alty.log(f"Enabled: {value}")

@settings.list(options=[("Быстро", "fast"), ("Медленно", "slow")], default="fast")
def mode(choice):
    alty.log(f"Mode set to {choice}")

@settings.slider(min_value=0, max_value=10, step=1, default=3)
def retries(value: float):
    alty.log(f"Retries: {value}")

# Показать экран настроек в UI
settings.open()
```

– Внутренний поток-планировщик пробуждается каждую секунду.
– Задачи выполняются в `externalNetworkQueue` через `run_on_queue`.

### 6.4. RPC-реестр

* `resolve_peer("@username" | 123456789 | "-100…") -> TL_inputPeer` — универсальный резолвер ID/username.
* `current_dialog()` — активный диалог (если доступен), `pick_dialog()` — UI-диалог выбора получателя.
* `send_text(peer, text, reply_to=None, entities=None, silent=False)` — безопасная отправка сообщений.
* Тайп-хелперы для `MessagesController`, `SendMessagesHelper`, `NotificationCenter` и др. доступны через прямые функции (`get_messages_controller`, `send_request`, `load_dialogs`, и т.п.).

---

## Хуки и жизненный цикл

Если функция не найдена — возбуждается `RuntimeError`.

---

## Помощники Telegram API

* `resolve_peer("@username" | 123456789 | "-100…") -> TL_inputPeer` — универсальный резолвер ID/username.
* `current_dialog()` — активный диалог (если доступен), `pick_dialog()` — UI-диалог выбора получателя.
* `send_text(peer, text, reply_to=None, entities=None, silent=False)` — безопасная отправка сообщений.
* Тайп-хелперы для `MessagesController`, `SendMessagesHelper`, `NotificationCenter` и др. доступны через прямые функции (`get_messages_controller`, `send_request`, `load_dialogs`, и т.п.).

---

## Хуки и жизненный цикл

* `safe_hook(target, method, hook, *, plugin=None, group=None, once=False)` — обёртка над Xposed-хуками.
  * Автоматически получает сигнатуры, предотвращает двойную регистрацию и логирует ошибки.
* `HookGroup` и `HookLifecycle` позволяют группировать хуки и автоматически удалять их в `on_plugin_unload`.
* `safe_unhook_all()` — аварийное снятие всех зарегистрированных хуков.
* Вспомогательные API: `add_tg_alias(path, cb)`, `detect_client()`, `detect_language()`, `get_current_activity()`, `get_plugins_dirs()`.

---

## Rate-limit и debounce

* `debounce(wait, leading=False)` — декоратор для отложенного выполнения.
* `throttle(interval)` — гарантирует вызов не чаще указанного интервала.
* `RateLimiter(rate, per_seconds)` — очереди anti-flood; метод `.acquire()` блокирует до разрешённой скорости.

Можно комбинировать с `send_text` и TL-запросами, чтобы избежать ограничений Telegram.

---

## Форматирование сообщений

* `md("**bold** [link](https://example.com)") -> (text, entities)` — надстройка над Markdown Parser.
* `split_text(long_text, limit=4096)` — разбивает текст на части, учитывая границы строк и слова.
* Все offsets нормализованы под UTF-16, что удобно при работе с `TLRPC.MessageEntity*`.

---

## Медиа-хелперы

* `download_media(message, blocking=False, priority=FileLoader.PRIORITY_NORMAL)` — возвращает путь к файлу после загрузки.
* `save_to_gallery(path)` — переносит файл в медиа-библиотеку пользователя.
* `with_image(path, fn)` — открывает изображение через Pillow, передаёт объект `Image` в колбэк и сохраняет результат.

---

## UI-компоненты

| Функция | Поведение |
|---------|-----------|
| `bulletin_info(text)` | Серый фон (не везде отображается) |
| `bulletin_success(text)` | Зелёный/синий фон |
| `bulletin_error(text)` | Красный фон |
| `snackbar_info/warn/error/undo` | Единый стиль Snackbar/Bulletin для уведомлений |
| `show_spinner(text?) → builder` | Модальное крутилко-окно |
| `hide_spinner()` | Скрыть |
| `quick_list_pick(title, options, on_select)` | Быстрые списки |
| `show_bottom_sheet(items, on_select)` | Модальные нижние листы |
| `copy_to_clipboard(label, text)` | Копирование в буфер с уведомлением |

Компоненты используют готовые контроллеры exteraGram и выдерживают единый визуальный стиль.

---

## Наблюдатели NotificationCenter

```python
@alty.on("didReceiveNewMessages")
def handle_new_messages(args):
    for dialog_id, messages, _ in args:
        process(dialog_id, messages)
```

* Декоратор автоматически подбирает сигнатуру делегата и отписывается при выгрузке плагина.
* Колбэк может принимать `(account, args)` или только `args` — сигнатура распознаётся по количеству параметров.

---

## Диагностика и хот-ридлоад

* `install_crash_reporter(path=None)` — перехватывает необработанные исключения, сохраняет лог и показывает уведомление.
* `enable_hot_reload(*modules)` + `reload_current_plugin()` — кнопка горячей перезагрузки модулей плагина.
* Логгер (`alty.log`, `alty.logcat`) дополняет отчёты, а панель диагностики показывает состояние очередей и хуков.

---

## Совместимость и feature flags

```python
alty.features_register("stories_effects", lambda: alty.detect_client() == "exteragram")
if alty.features_has("stories_effects"):
    enable_stories_effects()
```

* Регистрация детекторов через `features_register(name, detector)`.
* Кеш значений очищается `features_clear_cache()`.
* Полезно при различиях между версиями exteraGram/Telegram.

---

## Cookbook: готовые рецепты

`alty.list_recipes()` возвращает словарь с описанием типовых сценариев:

* **auto_responder** — автоответчик на новые сообщения (NotificationCenter + send_text).
* **media_autosave** — скачивание и сохранение медиа с контролем скорости.
* **quick_translate** — команда `.translate` с кешированием TTL и ответом в чат.
* **profanity_filter** — фильтр мата с подсветкой и заменой.
* **quick_reply_templates** — быстрые шаблоны ответов.

Рецепты служат отправной точкой для собственных плагинов: изучите шаги и адаптируйте под свои задачи.

---

## Советы по разработке

1. **Всегда** ставьте `handle_outgoing_command` перед своей логикой — тогда новые глобальные команды автоматически работают во всех плагинах.
2. Для «общения» между плагинами предпочитайте Event Bus; если нужен ответ — RPC.
3. Не держите долгие циклы в собственных потоках — используйте `tasks_schedule`.
4. Проверяйте UI-функции на конкретной сборке: если `bulletin_info` не видно, переходите на `bulletin_success`.
5. При отладке используйте `.altylogs` и `.clearaltylogs`, чтобы видеть логи библиотеки.
6. Воспользуйтесь DSL настроек вместо ручных AlertDialog — значения автоматически сохраняются.
7. Комбинируйте `RateLimiter` и `send_text`, если шлёте массовые сообщения или используете TL-запросы в цикле.

---

## FAQ и обратная связь

* Документация и примеры: [plugins.exteragram.app/docs](https://plugins.exteragram.app/docs)
* Вопросы и предложения — в ЛС канала **@AltyPlugins** или через issues репозитория.
