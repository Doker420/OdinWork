# OdinWork
# Как подключить обязательную подписку к своему боту за 15 минут: код на aiogram 3

Если у вашего бота есть живая аудитория, он уже зарабатывает — просто деньги пока получает кто-то другой. Самый быстрый способ монетизации Telegram-бота — блок обязательной подписки (ОП): перед доступом к функциям пользователь подписывается на спонсоров, а вам платят за каждое подтверждённое действие.

В этой статье — рабочий код на aiogram 3, который подключается к бирже трафика за один файл и 15 минут. Без абстрактных «схем интеграции»: всё, что ниже, запускается как есть.

Полный исходник: [`op_quickstart.py`](https://odinwork.sbs/api/docs) — код целиком продублирован в статье.

---

## Что происходит под капотом

Механика у всех сетей ОП одинаковая, отличается только форма запросов:

1. Бот спрашивает у биржи: **«что показать этому пользователю?»** — передаёт `user_id`, язык, премиум-статус.
2. Биржа возвращает список спонсоров, подобранных под конкретного человека (уже подписанные не показываются).
3. Бот рисует кнопки и **блокирует доступ** до выполнения.
4. Пользователь жмёт «Я подписался» → бот дёргает **подтверждение**, биржа проверяет членство в каналах и начисляет деньги.

Три метода API, больше ничего не нужно:

| Метод | Зачем |
| --- | --- |
| `POST /api/v1/get-sponsors` | Получить спонсоров под пользователя |
| `POST /api/v1/confirm-subscription` | Подтвердить выполнение и получить начисление |
| `GET /api/v1/stats?api_token=…` | Статистика: выдано, подтверждено, заработано |

Авторизация — заголовок `Auth: <ваш_ключ>`. Лимит — 120 запросов в минуту. Документация: [odinwork.sbs/api/docs](https://odinwork.sbs/api/docs).

---

## Шаг 1. Получить ключ (2 минуты)

1. Откройте [@OdinWorksBot](https://t.me/OdinWorksBot) → **💸 Продать трафик** → **Подключить площадку**.
2. Выберите тип: бот, чат, канал или короткая ссылка.
3. Для бота — пришлите его `@username`; бот сразу выдаст API-ключ.

Ключ — это и авторизация, и идентификатор площадки: отдельные ключи на каждый ресурс, статистика считается раздельно.

---

## Шаг 2. Клиент биржи (5 минут)

Главное правило интеграции — **fail-open**. Если биржа не ответила, упала или тупит — пользователь всё равно должен пройти дальше. Чужая авария не имеет права ронять ваш продукт, а потеря одного показа стоит копейки по сравнению с потерей пользователя.

```python
import aiohttp

API_BASE = "https://odinwork.sbs"
TIMEOUT = aiohttp.ClientTimeout(total=3)  # ждём недолго: бот не должен «задумываться»


class OpClient:
    """Тонкий клиент биржи трафика. Любая ошибка = пустой список спонсоров."""

    def __init__(self, api_key: str, base: str = API_BASE) -> None:
        self._headers = {"Auth": api_key}
        self._base = base.rstrip("/")
        self._session: aiohttp.ClientSession | None = None

    async def _http(self) -> aiohttp.ClientSession:
        if self._session is None or self._session.closed:
            self._session = aiohttp.ClientSession(timeout=TIMEOUT)
        return self._session

    async def _post(self, path: str, payload: dict) -> dict:
        try:
            http = await self._http()
            async with http.post(f"{self._base}{path}", json=payload, headers=self._headers) as r:
                data = await r.json()
                return data if isinstance(data, dict) else {}
        except Exception:      # сеть, таймаут, кривой JSON — всё сюда
            return {}          # fail-open: пустой ответ = пропускаем пользователя
```

### Запрос спонсоров

Чем больше контекста передадите, тем дороже трафик: биржа умеет таргетировать по языку и премиум-статусу, а такие показы стоят дороже.

```python
    async def sponsors(self, user, chat_id: int) -> list[dict]:
        data = await self._post("/api/v1/get-sponsors", {
            "user_id": user.id,
            "chat_id": chat_id,
            "first_name": user.first_name or "",
            "username": user.username or "",
            "language_code": user.language_code or "",
            "is_premium": bool(user.is_premium),
            "max_sponsors": 3,
        })
        if data.get("status") != "warning":
            return []
        return (data.get("additional") or {}).get("sponsors") or []
```

Поле `status` в ответе:

- `ok` — пользователь уже всё выполнил, **пропускайте**;
- `warning` — есть спонсоры, показывайте блок;
- `error` — проблема на нашей стороне, **тоже пропускайте**.

Каждый спонсор приходит так:

```json
{
  "resource_id": "campaign:317",
  "link": "https://t.me/example_channel",
  "resource_name": "Канал про Telegram",
  "button_text": "➕ Подписаться",
  "type": "channel",
  "status": "not_subscribed",
  "reward": "1.26"
}
```

`reward` — ваша доля за подтверждение, **в рублях с копейками** (60 % от цены задания). Можно показывать её пользователю, можно нет — это ваше поле.

### Подтверждение

```python
    async def confirm(self, user_id: int, chat_id: int) -> tuple[bool, list[str]]:
        data = await self._post("/api/v1/confirm-subscription",
                                {"user_id": user_id, "chat_id": chat_id})
        if not data:
            return True, []          # биржа молчит — не держим пользователя
        pending = data.get("pending") or []
        return not pending, [str(x) for x in pending]
```

В ответе приходят `confirmed` (что зачтено), `pending` (что осталось) и `earned` — сколько вы заработали этим запросом.

---

## Шаг 3. Middleware-шлюз (5 минут)

Вешать проверку в каждый хендлер — путь в ад. Правильное место — **outer middleware на `update`**: он ловит сразу всё, включая колбэки и инлайн-режим.

```python
from aiogram import BaseMiddleware
from aiogram.types import CallbackQuery, InlineKeyboardButton, InlineKeyboardMarkup, Update

GATE_TEXT = (
    "🔒 <b>Доступ к боту</b>\n\n"
    "Подпишитесь на спонсоров ниже и нажмите «Я подписался».\n"
    "Это бесплатно и занимает 10 секунд — так бот остаётся бесплатным для вас."
)


def sponsors_keyboard(sponsors: list[dict]) -> InlineKeyboardMarkup:
    rows = [
        [InlineKeyboardButton(text=s.get("button_text") or "Подписаться", url=s["link"])]
        for s in sponsors if s.get("link")
    ]
    rows.append([InlineKeyboardButton(text="✅ Я подписался", callback_data="op:check")])
    return InlineKeyboardMarkup(inline_keyboard=rows)


class OpMiddleware(BaseMiddleware):
    def __init__(self, client: OpClient) -> None:
        self._client = client

    async def __call__(self, handler, event, data):
        user = data.get("event_from_user")
        chat = data.get("event_chat")
        # Кнопку проверки пропускаем всегда, иначе пользователь застрянет в цикле.
        if user is None or chat is None or _is_check_press(event):
            return await handler(event, data)

        sponsors = await self._client.sponsors(user, chat.id)
        if not sponsors:
            return await handler(event, data)

        await data["bot"].send_message(chat.id, GATE_TEXT,
                                       reply_markup=sponsors_keyboard(sponsors))
        return None  # апдейт дальше не идёт


def _is_check_press(event) -> bool:
    if isinstance(event, Update) and event.callback_query is not None:
        return (event.callback_query.data or "") == "op:check"
    return isinstance(event, CallbackQuery) and (event.data or "") == "op:check"
```

Обработчик кнопки:

```python
@dp.callback_query(F.data == "op:check")
async def on_check(call: CallbackQuery) -> None:
    done, pending = await client.confirm(call.from_user.id, call.message.chat.id)
    if done:
        await call.answer("Спасибо! Доступ открыт ✅")
        await call.message.edit_text("✅ Проверка пройдена. Пользуйтесь ботом: /start")
        return
    await call.answer(f"Осталось подписок: {len(pending)}. Проверьте ещё раз.", show_alert=True)
```

Подключение — одна строка:

```python
dp.update.outer_middleware(OpMiddleware(client))
```

> ⚠️ В aiogram 3.7+ `parse_mode` в конструкторе `Bot` больше не принимается:
> `Bot(TOKEN, default=DefaultBotProperties(parse_mode="HTML"))`. Пример проверен на aiogram 3.31.

Всё. Весь файл — около 180 строк вместе с импортами и комментариями.

---

## Шаг 4. Проверить, что деньги капают

```bash
curl "https://odinwork.sbs/api/v1/stats?api_token=ВАШ_КЛЮЧ"
```

```json
{
  "status": "ok",
  "bot": {"username": "my_cool_bot", "status": "active"},
  "stats": {
    "requests": 1840, "issued": 1512, "confirmed": 967,
    "earned": "1218.42", "today_confirmed": 73, "today_earned": "91.98",
    "currency": "RUB"
  }
}
```

То же самое — в кабинете бота на экране **📊 Статистика**, там же кнопки вывода: картой от 10 000 ₽, чеком CryptoBot от 500 ₽, звёздами от 500 ⭐.

---

## Пять ошибок, на которых теряют деньги

**1. Fail-closed вместо fail-open.** Провайдер прилёг на 10 минут — у вас лёг весь бот. Оборачивайте запросы в `try/except`, ставьте таймаут 2–3 секунды и пропускайте пользователя при любой ошибке.

**2. Проверка внутри каждого хендлера.** Через месяц вы забудете добавить её в новый хендлер, и половина аудитории пройдёт мимо блока. Middleware на `update` — единственное надёжное место.

**3. Показ блока при каждом действии.** Пользователь подписался, а бот снова просит. Биржа кэширует список спонсоров на пользователя, но и вы не запрашивайте его на каждое нажатие — достаточно на входные точки и раз в N минут.

**4. Блокировка кнопки проверки.** Классика: middleware не пускает колбэк `op:check`, пользователь жмёт «Я подписался» и ничего не происходит. Явно исключайте эту кнопку — в примере это `_is_check_press`.

**5. Один провайдер.** Доход = ставка × заполняемость. Если сеть не вернула спонсоров — показ пустой, деньги потеряны. У нас на стороне биржи каскад из семи источников: не дал первый — подхватывает следующий, и вам для этого ничего не нужно делать, ключ остаётся один.

---

## Сколько это приносит

Считаем честно, без «заработай миллион». Бот с **10 000 активных пользователей в месяц**, блок ОП на входе:

| Показатель | Значение |
| --- | --- |
| Показов блока | ~10 000 |
| Доходят до подтверждения (конверсия 45 %) | 4 500 |
| Оплачиваемых из них (без бесплатных слотов) | ~3 600 |
| Средняя выплата за подтверждение | ~1.2 ₽ |
| **Итого в месяц** | **~4 300 ₽** |

Цифры зависят от тематики и гео: боты с кино, инструментами и заработком конвертят выше среднего, узкоспециализированные — ниже. Важно, что это пассивный доход с уже существующей аудитории, а вложение — 15 минут работы.

Ещё один момент, о котором стоит знать заранее: в блоке ОП есть **бесплатные слоты** — собственный бот площадки и офферы партнёрских сетей, за которые платит провайдер, а не биржа. Подтверждение засчитывается, а ставка по нему нулевая. Это нормально (они держат заполняемость), но сервис обязан показывать это отдельной строкой, а не прятать в общем счётчике. У нас в кабинете так и сделано: видно, сколько подтверждений были оплачиваемыми, сколько — бесплатными, и какая ставка по активным кампаниям прямо сейчас.

---

## Что дальше

- **Не только бот.** Точно так же подключаются чат, канал и короткая ссылка — источники разные, ключ и код одинаковые.
- **Таргетинг.** Передавайте `language_code` и `is_premium` — за таргетированные показы платят больше.
- **Обзор рынка.** Если хотите сравнить биржи перед тем, как подключаться, — [мы разобрали шесть сервисов с ценами и механикой](https://telegra.ph/6-servisov-pokupki-i-prodazhi-trafika-v-Telegram-2026-09-27).

**Подключиться:** [@OdinWorksBot](https://t.me/OdinWorksBot) · **Документация API:** [odinwork.sbs/api/docs](https://odinwork.sbs/api/docs) · **Поддержка:** [@Odinworks](https://t.me/Odinworks?direct)

Вопросы по интеграции пишите в поддержку — отвечаем разработчику, а не скриптом.
