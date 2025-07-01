---
title: Routerlar, ko‘p fayllilik va bot tuzilmasi
description: Routerlar, ko‘p fayllilik va bot tuzilmasi
---

# Routerlar, ko‘p fayllilik va bot tuzilmasi

!!! info ""
    Foydalanilayotgan aiogram versiyasi: 3.7.0

Ushbu bobda biz aiogram 3.x ning yangi imkoniyati — routerlar bilan tanishamiz, kodimizni alohida komponentlarga ajratishni o‘rganamiz va keyingi boblar hamda amaliyot uchun qulay bo‘ladigan botning asosiy tuzilmasini shakllantiramiz.

## Dasturga kirish nuqtasi {: id="entrypoint" }

Teatr garderobdan boshlangani kabi, bot ham kirish nuqtasidan boshlanadi. Bu faylni `bot.py` deb ataymiz. Unda kerakli obyektlarni yaratib, pollingni ishga tushiradigan asinxron `main()` funksiyasini yozamiz. Qanday obyektlar kerak? Avvalo, albatta, botning o‘zi. Bir nechta bot bo‘lishi ham mumkin, lekin bu haqida keyinroq. Ikkinchidan, dispatcher kerak bo‘ladi. Dispatcher Telegramdan keladigan hodisalarni qabul qilib, ularni handlerlar orqali filtrlab va middleware'lar yordamida tarqatadi.

```python title="bot.py"
import asyncio
from aiogram import Bot, Dispatcher


# Botni ishga tushirish
async def main():
    bot = Bot(token="TOKEN")
    dp = Dispatcher()

    # Botni ishga tushiramiz va barcha to'plangan xabarlarni o'tkazib yuboramiz
    # Ha, bu metod pollingda ham ishlaydi
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
```

Lekin xabarlarni qayta ishlash uchun faqat shu yetarli emas, yana handlerlar kerak bo'ladi. Ularni boshqa fayllarga joylashtirishni xohlaymiz, chunki minglab qatorli koddan qochish kerak. Oldingi boblarda barcha handlerlar dispatcherga biriktirilgan edi, lekin hozir dispatcher funksiyaning ichida va uni global qilishni istamaymiz.
Nima qilish kerak? Bu yerda yordamga...

## Routerlar {: id="routers" }

aiogram 3.x [rasmiy hujjatlari](https://docs.aiogram.dev/en/dev-3.x/dispatcher/router.html) ga murojaat qilamiz va quyidagi rasmga qaraymiz:

![Bir nechta routerlar](https://docs.aiogram.dev/en/dev-3.x/_images/nested_routers_example.png)

Nimani ko'ryapmiz?

1. Dispatcher — ildiz router.
2. Handlerlar routerlarga biriktiriladi.
3. Routerlar ichma-ich bo'lishi mumkin, lekin ular orasida faqat bir yo'nalishli bog'lanish bo'ladi.
4. Routerlarni ulash (va tekshirish) tartibi aniq belgilanadi.

Quyidagi rasmda esa update'ning kerakli handlerni topish tartibi ko'rsatilgan:

![update orqali handler qidirish tartibi](https://docs.aiogram.dev/en/dev-3.x/_images/update_propagation_flow.png)

Keling, ikki funksiyali oddiy bot yozamiz:

1. Botga `/start` yuborilsa, u savol va ikkita tugma ("Ha" va "Yo'q") yuboradi.
2. Boshqa har qanday matn, stiker yoki gif yuborilsa, bot xabar turini javob qiladi.

Avval klaviaturani tayyorlaymiz: `bot.py` yonida `keyboards` papkasini va unda `for_questions.py` faylini yaratamiz. Unda "Ha" va "Yo'q" tugmalari bir qatorda bo'ladigan oddiy klaviatura funksiyasini yozamiz:

```python title="keyboards/for_questions.py"
from aiogram.types import ReplyKeyboardMarkup
from aiogram.utils.keyboard import ReplyKeyboardBuilder

def get_yes_no_kb() -> ReplyKeyboardMarkup:
    kb = ReplyKeyboardBuilder()
    kb.button(text="Ha")
    kb.button(text="Yo'q")
    kb.adjust(2)
    return kb.as_markup(resize_keyboard=True)
```

Hech qanday murakkablik yo'q, klaviaturalarni [oldingi bobda](buttons.md) batafsil ko'rganmiz. Endi `bot.py` yonida `handlers` papkasini va unda `questions.py` faylini yaratamiz.

```python title="handlers/questions.py" hl_lines="7 9"
from aiogram import Router, F
from aiogram.filters import Command
from aiogram.types import Message, ReplyKeyboardRemove

from keyboards.for_questions import get_yes_no_kb

router = Router()  # [1]

@router.message(Command("start"))  # [2]
async def cmd_start(message: Message):
    await message.answer(
        "Ishingizdan mamnunmisiz?",
        reply_markup=get_yes_no_kb()
    )

@router.message(F.text.lower() == "ha")
async def answer_yes(message: Message):
    await message.answer(
        "Zo'r!",
        reply_markup=ReplyKeyboardRemove()
    )

@router.message(F.text.lower() == "yo'q")
async def answer_no(message: Message):
    await message.answer(
        "Afsus...",
        reply_markup=ReplyKeyboardRemove()
    )
```

[1] va [2] bandlariga e'tibor bering. Birinchidan, har bir modulda o'z routerini yaratdik va uni keyin ildiz routerga (dispatcherga) ulaymiz. Ikkinchidan, handlerlar endi lokal routerga biriktiriladi.

Xuddi shu tarzda `different_types.py` faylini ham yaratamiz, unda xabar turini chiqaruvchi handlerlar bo'ladi:

```python title="handlers/different_types.py"
from aiogram import Router, F
from aiogram.types import Message

router = Router()

@router.message(F.text)
async def message_with_text(message: Message):
    await message.answer("Bu matnli xabar!")

@router.message(F.sticker)
async def message_with_sticker(message: Message):
    await message.answer("Bu stiker!")

@router.message(F.animation)
async def message_with_gif(message: Message):
    await message.answer("Bu GIF!")

```

Endi `bot.py` ga qaytamiz, router va handler fayllarini import qilib, dispatcherga ulaymiz:

```python title="bot.py" hl_lines="3 11 12"
import asyncio
from aiogram import Bot, Dispatcher
from handlers import questions, different_types


# Botni ishga tushirish
async def main():
    bot = Bot(token="TOKEN")
    dp = Dispatcher()

    dp.include_routers(questions.router, different_types.router)

    # Routerlarni har birini alohida ham ulash mumkin
    # dp.include_router(questions.router)
    # dp.include_router(different_types.router)

    # Botni ishga tushiramiz va barcha to'plangan xabarlarni o'tkazib yuboramiz
    # Ha, bu metod pollingda ham ishlaydi
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
```

Biz shunchaki `handlers/` papkasidagi fayllarni import qilib, ulardagi routerlarni dispatcherga ulaymiz. Bu yerda importlar tartibi muhim! Agar routerlarni ulash tartibini o'zgartirsak, `/start` komandasi uchun bot birinchi mos kelgan handlerni (masalan, `message_with_text()`) ishlatadi va "Bu matnli xabar!" deb javob beradi. Filtrlar haqida keyinroq batafsil gaplashamiz, hozir esa yana bir savolga to'xtalamiz.

## Xulosa {: id="conclusion" }

Botimizni turli fayllarga toza va tartibli ajratdik, ishlashiga xalaqit bermadik. Fayl va papkalar taxminan quyidagicha ko'rinishda bo'ladi (bu yerda ba'zi ahamiyatsiz fayllar ko'rsatilmagan):

```
├── bot.py
├── handlers
│   ├── different_types.py
│   └── questions.py
├── keyboards
│   └── for_questions.py
```

Kelgusida ham shu tuzilmani saqlaymiz, unga filtrlar, middleware'lar, ma'lumotlar bazasi bilan ishlash uchun fayllar va boshqalar qo'shiladi.