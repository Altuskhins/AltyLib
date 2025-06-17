# AltyLib
Плагин для экстераграма, облегчающий разработку ваших плагинов


# AltyLib — техническая документация  
*(актуально для версии `1.0.3+`, июнь 2025)*  

---

## Содержание
1. Быстрый старт  
2. Структура библиотеки  
3. Системы высокого уровня  
   3.1. Event Bus  
   3.2. Управление командами  
   3.3. Планировщик задач  
   3.4. RPC-реестр  
4. UI-утилиты  
5. Работа с данными  
6. Хуки и вспомогательные функции  
7. Общие советы по разработке  

---

## 1. Быстрый старт

```python
# пример minimal_plugin.plugin
__id__ = "hello"
__name__ = "HelloPlugin"
__min_version__ = "11.9.1"

from base_plugin import BasePlugin, HookResult
import altylib as alty      # импорт библиотеки

class Hello(BasePlugin):
    def on_plugin_load(self):
        alty.register_command("hello", self.cmd_hello, "Поздороваться")
        alty.events_subscribe("hello:ping", lambda d: alty.bulletin_success(f"Pong from {__id__}!"))

    # -- команды ----------------------------------------------------
    def cmd_hello(self, _self_pl, args, params):
        alty.events_publish("hello:ping", None)
        return "Hello from AltyLib!"

    # -- перехват исходящих сообщений -------------------------------
    def on_send_message_hook(self, account, params):
        res = alty.handle_outgoing_command(self, account, params)
        return res or HookResult()
```

---

## 2. Структура библиотеки

| Категория | API | Назначение |
|-----------|-----|------------|
| Логи | `log()/get_logs()/clear_logs()` | Унифицированный вывод и буфер последних 500 строк |
| UI | `bulletin_info/error/success`, `show_spinner/hide_spinner` | Всплывающие уведомления и «спиннер» |
| События | `events_subscribe/unsubscribe/publish` | Лёгкая шина между плагинами |
| Команды | `register_command/handle_outgoing_command/get_registered_commands` | Централизованный парсинг `.<command>` |
| Планировщик | `tasks_schedule/cancel/list` | Периодические фоновые задачи |
| RPC | `rpc_register/call` | Вызов процедур между плагинами |
| Данные | `JsonCacheFile`, `shared_*` | Кеш и key-value хранилище |
| Прочее | `detect_client/language`, `get_plugins_dirs`, `get_current_activity`, TG-alias |

---

## 3. Системы высокого уровня

### 3.1. Event Bus

| Функция | Описание |
|---------|----------|
| `events_subscribe(event, callback)` | Подписка. Callback(data) |
| `events_unsubscribe(event, callback)` | Отписка |
| `events_publish(event, data, *, async_call=True)` | Рассылка события |

> Внутри асинхронный вызов идёт через `run_on_queue`, поэтому обработчики не блокируют UI.

### 3.2. Управление командами

1. Регистрация:  
   ```python
   def cmd_echo(selfpl, args, params):      # сигнатура фиксирована
       return args or "Nothing to echo"
   alty.register_command("echo", cmd_echo, "Повторить текст")
   ```
2. Обработка: в своём `on_send_message_hook` первым делом вызоваем  
   ```python
   res = alty.handle_outgoing_command(self, account, params)
   if res: return res
   ```

*Если callback возвращает строку — она заменит исходящее сообщение;  
можно вернуть готовый `HookResult` для полного контроля.*

### 3.3. Планировщик задач

```python
def my_job():
    alty.log("Tick!")

# каждые 300 с, без немедленного старта
alty.tasks_schedule("my_job", my_job, 300)

# отмена
alty.tasks_cancel("my_job")
```

– Внутренний поток-планировщик пробуждается каждую секунду  
– Задачи выполняются в `externalNetworkQueue` через `run_on_queue`.

### 3.4. RPC-реестр

```python
# — в плагине-провайдере —
alty.rpc_register("math.add", lambda a, b: a + b)

# — в другом плагине —
res = alty.rpc_call("math.add", 5, 7)   # 12
```

Если функция не найдена - возбуждается `RuntimeError`.

---

## 4. UI-утилиты

| Функция | Поведение |
|---------|-----------|
| `bulletin_info(text)` | Серый фон (не везде отображается) |
| `bulletin_success(text)` | Зелёный/синий фон |
| `bulletin_error(text)` | Красный фон |
| `show_spinner(text?) → builder` | Модальное крутилко-окно |
| `hide_spinner()` | Скрыть |

> На некоторых кастомных клиентах `BulletinHelper.show` скрыт; используйте `bulletin_success/error` для гарантии отображения.

---

## 5. Работа с данными

### 5.1. JsonCacheFile

```python
cache = alty.JsonCacheFile("history.json", default=[])
cache.content.append({"ts": time.time()})
cache.write()
```

Автоматически создаёт папку `plugins/<your plugin>/cache/`.

### 5.2. Shared-store между плагинами

```python
alty.shared_set("counter", 42)
val = alty.shared_get("counter")          # 42
alty.shared_delete("counter")
```

Хранится в `altylib_shared_store.json`.

---

## 6. Хуки и вспомогательные функции

| API | Кратко |
|-----|--------|
| `add_tg_alias(path, cb)` | Свой `tg://<path>` |
| `detect_client()` | `exteragram` / `ayugram` |
| `detect_language()` | `en`, `ru`, … |
| `get_current_activity()` | Текущая `Activity` или `None` |
| `get_plugins_dirs()` | Список путей с плагинами |

---

## 7. Советы по разработке

1. **Всегда** ставьте `handle_outgoing_command` перед своей логикой — тогда новые глобальные команды автоматически работают во всех плагинах.  
2. Для «общения» между плагинами предпочитайте Event Bus; если нужен ответ — RPC.  
3. Не держите долгие циклы в собственных потоках — используйте `tasks_schedule`.  
4. Проверяйте UI-функции на конкретной сборке: если `bulletin_info` не видно, переходите на `bulletin_success`.  
5. При отладке используйте `.altylogs` и `.clearaltylogs`, чтобы видеть логи библиотеки.

---

### Обратная связь

Если у вас есть идеи или баг-репорты — публикуйте их в канале **@AltyPlugins** или открывайте issue в репозитории.
