import os
import json
import asyncio
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command, StateFilter
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    InlineKeyboardMarkup, InlineKeyboardButton,
    ReplyKeyboardMarkup, KeyboardButton, ReplyKeyboardRemove
)
from aiogram.utils.keyboard import InlineKeyboardBuilder

# ===================== НАСТРОЙКИ =====================
BOT_TOKEN = os.getenv("BOT_TOKEN", "ВСТАВЬ_СВОЙ_ТОКЕН_СЮДА")
ADMIN_IDS = [int(x) for x in os.getenv("ADMIN_IDS", "ВСТАВЬ_СВОЙ_TELEGRAM_ID").split(",")]
DB_FILE = "products.json"

# ===================== БАЗА ДАННЫХ (JSON файл) =====================
def load_db():
    if os.path.exists(DB_FILE):
        with open(DB_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {"categories": {"parfum": "🌸 Парфюм", "auto": "🚗 Автопарфюм"}, "products": []}

def save_db(data):
    with open(DB_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def get_next_id(data):
    if not data["products"]:
        return 1
    return max(p["id"] for p in data["products"]) + 1

# ===================== СОСТОЯНИЯ FSM =====================
class AddProduct(StatesGroup):
    category = State()
    name = State()
    description = State()
    photo = State()
    prices = State()
    confirm = State()

class EditProduct(StatesGroup):
    choose_field = State()
    enter_value = State()

# ===================== КЛАВИАТУРЫ =====================
def main_menu_kb():
    kb = ReplyKeyboardMarkup(keyboard=[
        [KeyboardButton(text="🌸 Парфюм"), KeyboardButton(text="🚗 Автопарфюм")],
        [KeyboardButton(text="📋 Все товары"), KeyboardButton(text="ℹ️ О нас")]
    ], resize_keyboard=True)
    return kb

def admin_menu_kb():
    kb = ReplyKeyboardMarkup(keyboard=[
        [KeyboardButton(text="🌸 Парфюм"), KeyboardButton(text="🚗 Автопарфюм")],
        [KeyboardButton(text="📋 Все товары"), KeyboardButton(text="ℹ️ О нас")],
        [KeyboardButton(text="➕ Добавить товар"), KeyboardButton(text="📦 Управление товарами")]
    ], resize_keyboard=True)
    return kb

def category_kb():
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🌸 Парфюм", callback_data="cat_parfum")],
        [InlineKeyboardButton(text="🚗 Автопарфюм", callback_data="cat_auto")]
    ])
    return kb

def product_card_kb(product_id, is_admin=False, in_stock=True):
    builder = InlineKeyboardBuilder()
    if in_stock:
        builder.button(text="📩 Заказать", callback_data=f"order_{product_id}")
    if is_admin:
        status_text = "🔴 Закрыть (нет в наличии)" if in_stock else "🟢 Открыть (есть в наличии)"
        builder.button(text=status_text, callback_data=f"toggle_{product_id}")
        builder.button(text="✏️ Редактировать", callback_data=f"edit_{product_id}")
        builder.button(text="🗑 Удалить", callback_data=f"delete_{product_id}")
    builder.adjust(1)
    return builder.as_markup()

def confirm_kb():
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Подтвердить", callback_data="confirm_add"),
         InlineKeyboardButton(text="❌ Отмена", callback_data="cancel_add")]
    ])
    return kb

def edit_fields_kb(product_id):
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📝 Название", callback_data=f"editfield_{product_id}_name")],
        [InlineKeyboardButton(text="📄 Описание", callback_data=f"editfield_{product_id}_description")],
        [InlineKeyboardButton(text="🖼 Фото", callback_data=f"editfield_{product_id}_photo")],
        [InlineKeyboardButton(text="💰 Цены", callback_data=f"editfield_{product_id}_prices")],
        [InlineKeyboardButton(text="❌ Отмена", callback_data="cancel_edit")]
    ])
    return kb

def delete_confirm_kb(product_id):
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Да, удалить", callback_data=f"confirm_delete_{product_id}"),
         InlineKeyboardButton(text="❌ Нет", callback_data="cancel_delete")]
    ])
    return kb

# ===================== ФОРМАТИРОВАНИЕ ТОВАРА =====================
def format_product(product):
    status = "✅ В наличии" if product.get("in_stock", True) else "❌ Нет в наличии"
    cat_name = "🌸 Парфюм" if product["category"] == "parfum" else "🚗 Автопарфюм"

    text = f"<b>{product['name']}</b>\n"
    text += f"📂 {cat_name}\n"
    text += f"{status}\n\n"

    if product.get("description"):
        text += f"📝 {product['description']}\n\n"

    text += "💰 <b>Цены:</b>\n"
    for volume, price in product.get("prices", {}).items():
        text += f"  • {volume} — {price} ₽\n"

    return text

# ===================== ПОКАЗ КАТАЛОГА =====================
async def show_catalog(message: types.Message, category=None):
    data = load_db()
    is_admin = message.from_user.id in ADMIN_IDS

    products = data["products"]
    if category:
        products = [p for p in products if p["category"] == category]

    if not is_admin:
        products = [p for p in products if p.get("in_stock", True)]

    if not products:
        await message.answer("😔 Товаров пока нет в этой категории.")
        return

    for product in products:
        caption = format_product(product)
        kb = product_card_kb(product["id"], is_admin, product.get("in_stock", True))

        if product.get("photo_id"):
            await message.answer_photo(
                photo=product["photo_id"],
                caption=caption,
                parse_mode="HTML",
                reply_markup=kb
            )
        else:
            await message.answer(caption, parse_mode="HTML", reply_markup=kb)

# ===================== ОСНОВНЫЕ КОМАНДЫ =====================
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(storage=MemoryStorage())

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    is_admin = message.from_user.id in ADMIN_IDS
    kb = admin_menu_kb() if is_admin else main_menu_kb()

    welcome = (
        "🌿 <b>Noble Essence</b>\n\n"
        "NO – нет жизни без парфюма 🤍\n\n"
        "Здесь ты найдёшь:\n"
        "🌸 Парфюм — распив от 3ml\n"
        "🚗 Автопарфюм\n\n"
        "Выбирай категорию 👇"
    )
    if is_admin:
        welcome += "\n\n🔑 <i>Ты вошёл как администратор</i>"

    await message.answer(welcome, parse_mode="HTML", reply_markup=kb)

@dp.message(F.text == "🌸 Парфюм")
async def show_parfum(message: types.Message):
    await show_catalog(message, "parfum")

@dp.message(F.text == "🚗 Автопарфюм")
async def show_auto(message: types.Message):
    await show_catalog(message, "auto")

@dp.message(F.text == "📋 Все товары")
async def show_all(message: types.Message):
    await show_catalog(message)

@dp.message(F.text == "ℹ️ О нас")
async def about(message: types.Message):
    text = (
        "🌿 <b>Noble Essence</b>\n\n"
        "Мы продаём оригинальные масла и парфюм.\n"
        "Распив от 3ml — попробуй перед покупкой!\n\n"
        "📱 Instagram: @_noble_essence_\n"
        "📢 Канал: @NobleEssenceNO\n\n"
        "По вопросам заказа нажми кнопку <b>«Заказать»</b> на карточке товара."
    )
    await message.answer(text, parse_mode="HTML")

# ===================== ЗАКАЗ =====================
@dp.callback_query(F.data.startswith("order_"))
async def order_product(callback: types.CallbackQuery):
    product_id = int(callback.data.split("_")[1])
    data = load_db()
    product = next((p for p in data["products"] if p["id"] == product_id), None)

    if not product:
        await callback.answer("Товар не найден")
        return

    text = (
        f"📩 <b>Заказ: {product['name']}</b>\n\n"
        f"Напиши администратору и укажи:\n"
        f"• Товар: <b>{product['name']}</b>\n"
        f"• Желаемый объём\n"
        f"• Способ получения\n\n"
        f"✍️ Написать: @NobleEssenceNO"
    )
    await callback.message.answer(text, parse_mode="HTML")
    await callback.answer()

# ===================== АДМИН: ДОБАВЛЕНИЕ ТОВАРА =====================
@dp.message(F.text == "➕ Добавить товар")
async def admin_add_start(message: types.Message, state: FSMContext):
    if message.from_user.id not in ADMIN_IDS:
        return
    await message.answer("📂 Выбери категорию товара:", reply_markup=category_kb())
    await state.set_state(AddProduct.category)

@dp.callback_query(AddProduct.category)
async def add_category(callback: types.CallbackQuery, state: FSMContext):
    cat = callback.data.replace("cat_", "")
    await state.update_data(category=cat)
    await callback.message.answer("✏️ Введи <b>название</b> товара:", parse_mode="HTML",
                                   reply_markup=ReplyKeyboardRemove())
    await state.set_state(AddProduct.name)
    await callback.answer()

@dp.message(AddProduct.name)
async def add_name(message: types.Message, state: FSMContext):
    await state.update_data(name=message.text)
    await message.answer("📄 Введи <b>описание</b> товара (или напиши «-» чтобы пропустить):",
                         parse_mode="HTML")
    await state.set_state(AddProduct.description)

@dp.message(AddProduct.description)
async def add_description(message: types.Message, state: FSMContext):
    desc = "" if message.text == "-" else message.text
    await state.update_data(description=desc)
    await message.answer("🖼 Отправь <b>фото</b> товара (или напиши «-» чтобы пропустить):",
                         parse_mode="HTML")
    await state.set_state(AddProduct.photo)

@dp.message(AddProduct.photo, F.photo)
async def add_photo(message: types.Message, state: FSMContext):
    photo_id = message.photo[-1].file_id
    await state.update_data(photo_id=photo_id)
    await message.answer(
        "💰 Введи <b>цены</b> в формате:\n"
        "<code>3ml - 500\n5ml - 800\n10ml - 1500</code>\n\n"
        "Каждый объём с новой строки:",
        parse_mode="HTML"
    )
    await state.set_state(AddProduct.prices)

@dp.message(AddProduct.photo, F.text == "-")
async def add_photo_skip(message: types.Message, state: FSMContext):
    await state.update_data(photo_id=None)
    await message.answer(
        "💰 Введи <b>цены</b> в формате:\n"
        "<code>3ml - 500\n5ml - 800\n10ml - 1500</code>\n\n"
        "Каждый объём с новой строки:",
        parse_mode="HTML"
    )
    await state.set_state(AddProduct.prices)

@dp.message(AddProduct.prices)
async def add_prices(message: types.Message, state: FSMContext):
    prices = {}
    for line in message.text.strip().split("\n"):
        if "-" in line:
            parts = line.split("-", 1)
            volume = parts[0].strip()
            price = parts[1].strip()
            prices[volume] = price

    if not prices:
        await message.answer("❌ Неверный формат. Попробуй ещё раз:\n<code>3ml - 500</code>",
                             parse_mode="HTML")
        return

    await state.update_data(prices=prices)
    data = await state.get_data()

    # Показываем превью
    preview_product = {
        "name": data["name"],
        "category": data["category"],
        "description": data.get("description", ""),
        "prices": prices,
        "in_stock": True
    }

    preview_text = "👀 <b>Предпросмотр товара:</b>\n\n" + format_product(preview_product)
    preview_text += "\n\nВсё верно?"

    if data.get("photo_id"):
        await message.answer_photo(photo=data["photo_id"], caption=preview_text,
                                   parse_mode="HTML", reply_markup=confirm_kb())
    else:
        await message.answer(preview_text, parse_mode="HTML", reply_markup=confirm_kb())

    await state.set_state(AddProduct.confirm)

@dp.callback_query(AddProduct.confirm, F.data == "confirm_add")
async def confirm_add(callback: types.CallbackQuery, state: FSMContext):
    data = await state.get_data()
    db = load_db()

    new_product = {
        "id": get_next_id(db),
        "category": data["category"],
        "name": data["name"],
        "description": data.get("description", ""),
        "photo_id": data.get("photo_id"),
        "prices": data["prices"],
        "in_stock": True
    }

    db["products"].append(new_product)
    save_db(db)

    is_admin = callback.from_user.id in ADMIN_IDS
    kb = admin_menu_kb() if is_admin else main_menu_kb()

    await callback.message.answer(f"✅ Товар <b>{new_product['name']}</b> добавлен!",
                                  parse_mode="HTML", reply_markup=kb)
    await state.clear()
    await callback.answer()

@dp.callback_query(F.data == "cancel_add")
async def cancel_add(callback: types.CallbackQuery, state: FSMContext):
    await state.clear()
    is_admin = callback.from_user.id in ADMIN_IDS
    kb = admin_menu_kb() if is_admin else main_menu_kb()
    await callback.message.answer("❌ Добавление отменено.", reply_markup=kb)
    await callback.answer()

# ===================== АДМИН: УПРАВЛЕНИЕ ТОВАРАМИ =====================
@dp.message(F.text == "📦 Управление товарами")
async def admin_manage(message: types.Message):
    if message.from_user.id not in ADMIN_IDS:
        return

    data = load_db()
    if not data["products"]:
        await message.answer("😔 Товаров пока нет.")
        return

    text = "📦 <b>Все товары:</b>\n\n"
    for p in data["products"]:
        status = "✅" if p.get("in_stock", True) else "❌"
        cat = "🌸" if p["category"] == "parfum" else "🚗"
        text += f"{status} {cat} ID:{p['id']} — {p['name']}\n"

    text += "\n💡 Нажми кнопку на карточке товара чтобы управлять им.\nИли используй /product_ID (например /product_1)"
    await message.answer(text, parse_mode="HTML")

@dp.message(Command(commands=["product"]))
async def show_product_by_id(message: types.Message):
    if message.from_user.id not in ADMIN_IDS:
        return
    try:
        product_id = int(message.text.split("_")[1])
    except:
        await message.answer("Используй: /product_1")
        return

    data = load_db()
    product = next((p for p in data["products"] if p["id"] == product_id), None)
    if not product:
        await message.answer("Товар не найден")
        return

    caption = format_product(product)
    kb = product_card_kb(product["id"], True, product.get("in_stock", True))

    if product.get("photo_id"):
        await message.answer_photo(photo=product["photo_id"], caption=caption,
                                   parse_mode="HTML", reply_markup=kb)
    else:
        await message.answer(caption, parse_mode="HTML", reply_markup=kb)

# ===================== АДМИН: TOGGLE НАЛИЧИЯ =====================
@dp.callback_query(F.data.startswith("toggle_"))
async def toggle_stock(callback: types.CallbackQuery):
    if callback.from_user.id not in ADMIN_IDS:
        await callback.answer("Нет доступа")
        return

    product_id = int(callback.data.split("_")[1])
    data = load_db()
    product = next((p for p in data["products"] if p["id"] == product_id), None)

    if not product:
        await callback.answer("Товар не найден")
        return

    product["in_stock"] = not product.get("in_stock", True)
    save_db(data)

    status = "✅ В наличии" if product["in_stock"] else "❌ Нет в наличии"
    await callback.answer(f"Статус изменён: {status}")

    caption = format_product(product)
    kb = product_card_kb(product["id"], True, product["in_stock"])

    try:
        if callback.message.photo:
            await callback.message.edit_caption(caption=caption, parse_mode="HTML", reply_markup=kb)
        else:
            await callback.message.edit_text(caption, parse_mode="HTML", reply_markup=kb)
    except:
        pass

# ===================== АДМИН: УДАЛЕНИЕ =====================
@dp.callback_query(F.data.startswith("delete_"))
async def delete_product(callback: types.CallbackQuery):
    if callback.from_user.id not in ADMIN_IDS:
        await callback.answer("Нет доступа")
        return

    product_id = int(callback.data.split("_")[1])
    data = load_db()
    product = next((p for p in data["products"] if p["id"] == product_id), None)

    if not product:
        await callback.answer("Товар не найден")
        return

    await callback.message.answer(
        f"🗑 Удалить товар <b>{product['name']}</b>?\nЭто действие нельзя отменить.",
        parse_mode="HTML",
        reply_markup=delete_confirm_kb(product_id)
    )
    await callback.answer()

@dp.callback_query(F.data.startswith("confirm_delete_"))
async def confirm_delete(callback: types.CallbackQuery):
    if callback.from_user.id not in ADMIN_IDS:
        return

    product_id = int(callback.data.split("_")[2])
    data = load_db()
    product = next((p for p in data["products"] if p["id"] == product_id), None)

    if product:
        data["products"] = [p for p in data["products"] if p["id"] != product_id]
        save_db(data)
        await callback.message.edit_text(f"✅ Товар <b>{product['name']}</b> удалён.", parse_mode="HTML")
    await callback.answer()

@dp.callback_query(F.data == "cancel_delete")
async def cancel_delete(callback: types.CallbackQuery):
    await callback.message.edit_text("❌ Удаление отменено.")
    await callback.answer()

# ===================== АДМИН: РЕДАКТИРОВАНИЕ =====================
@dp.callback_query(F.data.startswith("edit_"))
async def edit_product(callback: types.CallbackQuery, state: FSMContext):
    if callback.from_user.id not in ADMIN_IDS:
        await callback.answer("Нет доступа")
        return

    product_id = int(callback.data.split("_")[1])
    await callback.message.answer("✏️ Что хочешь изменить?", reply_markup=edit_fields_kb(product_id))
    await state.set_state(EditProduct.choose_field)
    await callback.answer()

@dp.callback_query(EditProduct.choose_field, F.data.startswith("editfield_"))
async def edit_choose_field(callback: types.CallbackQuery, state: FSMContext):
    parts = callback.data.split("_")
    product_id = int(parts[1])
    field = parts[2]

    await state.update_data(product_id=product_id, field=field)

    prompts = {
        "name": "✏️ Введи новое название:",
        "description": "📄 Введи новое описание (или «-» чтобы убрать):",
        "photo": "🖼 Отправь новое фото:",
        "prices": "💰 Введи новые цены:\n<code>3ml - 500\n5ml - 800</code>"
    }

    await callback.message.answer(prompts[field], parse_mode="HTML",
                                  reply_markup=ReplyKeyboardRemove())
    await state.set_state(EditProduct.enter_value)
    await callback.answer()

@dp.message(EditProduct.enter_value)
async def edit_enter_value(message: types.Message, state: FSMContext):
    data = await state.get_data()
    product_id = data["product_id"]
    field = data["field"]

    db = load_db()
    product = next((p for p in db["products"] if p["id"] == product_id), None)

    if not product:
        await message.answer("Товар не найден")
        await state.clear()
        return

    if field == "photo":
        if message.photo:
            product["photo_id"] = message.photo[-1].file_id
        else:
            await message.answer("Пожалуйста, отправь фото.")
            return
    elif field == "prices":
        prices = {}
        for line in message.text.strip().split("\n"):
            if "-" in line:
                parts = line.split("-", 1)
                prices[parts[0].strip()] = parts[1].strip()
        if not prices:
            await message.answer("❌ Неверный формат цен.")
            return
        product["prices"] = prices
    elif field == "description":
        product["description"] = "" if message.text == "-" else message.text
    else:
        product[field] = message.text

    save_db(db)

    is_admin = message.from_user.id in ADMIN_IDS
    kb = admin_menu_kb() if is_admin else main_menu_kb()
    await message.answer(f"✅ Товар обновлён!", reply_markup=kb)
    await state.clear()

@dp.callback_query(F.data == "cancel_edit")
async def cancel_edit(callback: types.CallbackQuery, state: FSMContext):
    await state.clear()
    await callback.message.answer("❌ Редактирование отменено.")
    await callback.answer()

# ===================== ЗАПУСК =====================
async def main():
    print("🤖 Noble Essence Bot запущен!")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
