# AltyLib

Расширенная библиотека для плагинов exteraGram. Содержит DSL для настроек, типизированные заглушки для Telegram-контроллеров, менеджер безопасных хуков, форматирование сообщений, кэширование, готовые UI-компоненты и CLI-инструменты.

Документация актуальна для версии `1.1.0+` (октябрь 2025).

---

## Содержание
1. [Быстрый старт](#быстрый-старт)
2. [CLI и генератор каркаса](#cli-и-генератор-каркаса)
3. [Хранилище и кеши](#хранилище-и-кеши)
4. [DSL настроек](#dsl-настроек)
5. [Помощники Telegram API](#помощники-telegram-api)
6. [Хуки и жизненный цикл](#хуки-и-жизненный-цикл)
7. [Rate-limit и debounce](#rate-limit-и-debounce)
8. [Форматирование сообщений](#форматирование-сообщений)
9. [Медиа-хелперы](#медиа-хелперы)
10. [UI-компоненты](#ui-компоненты)
11. [Наблюдатели NotificationCenter](#наблюдатели-notificationcenter)
12. [Диагностика и хот-ридлоад](#диагностика-и-хот-ридлоад)
13. [Совместимость и feature flags](#совместимость-и-feature-flags)
14. [Cookbook: готовые рецепты](#cookbook-готовые-рецепты)
15. [FAQ и обратная связь](#faq-и-обратная-связь)

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

    def cmd_hello(self, _self_pl, args, params):
        if not self.settings.get("enabled"):
            return "Команда выключена"
        return "Hello from AltyLib!"

    def on_send_message_hook(self, account, params):
        result = alty.handle_outgoing_command(self, account, params)
        return result or HookResult()
```

* `create_settings_registry()` создаёт DSL-реестр настроек и связанное хранилище.
* Все публичные API доступны через `altylib` после импорта (`import altylib as alty`).

---

## CLI и генератор каркаса

AltyLib поставляется с CLI-командой `altylib`:

```bash
python -m altylib_cli init my_plugin --id hello --author @user
python -m altylib_cli validate my_plugin/hello.plugin
```

* `init` — создаёт минимальный плагин (шаблон с метаданными и точкой входа).
* `validate` — проверяет `.plugin`-файл на наличие обязательных полей.

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

* `snackbar_info/warn/error/undo` — единый стиль Snackbar/Bulletin для уведомлений.
* `quick_list_pick(title, options, on_select)` — быстрые списки.
* `show_bottom_sheet(items, on_select)` — модальные нижние листы.
* `copy_to_clipboard(label, text)` — копирование в буфер с уведомлением.

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
* Кэш значений очищается `features_clear_cache()`.

---

## Cookbook: готовые рецепты

`alty.list_recipes()` возвращает словарь с описанием типовых сценариев:

* **auto_responder** — автоответчик на новые сообщения (NotificationCenter + send_text).
* **media_autosave** — скачивание и сохранение медиа с контролем скорости.
* **quick_translate** — команда `.translate` с кешированием TTL и ответом в чат.

Рецепты служат отправной точкой для собственных плагинов: изучите шаги и адаптируйте под свои задачи.

---

## FAQ и обратная связь

* Документация и примеры: [plugins.exteragram.app/docs](https://plugins.exteragram.app/docs)
* Вопросы и предложения — в ЛС канала **@AltyPlugins** или через issues репозитория.
