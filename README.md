# FoydaHisobBot
import os, sqlite3, asyncio, pandas as pd
from aiogram import Bot, Dispatcher, F
from aiogram.types import Message, FSInputFile
from aiogram.filters import Command

TOKEN = os.getenv("BOT_TOKEN")
ADMIN_ID = None  # Botni 1-marta ishga tushirganda sizning ID avtomatik saqlanadi

bot = Bot(TOKEN)
dp = Dispatcher()

# --- Database setup ---
conn = sqlite3.connect("sales.db")
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS sales(
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id TEXT,
    cost REAL,
    months INTEGER,
    percent REAL,
    plan TEXT,
    profit REAL,
    real_profit REAL
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS admin(
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id TEXT
)
""")
conn.commit()

# --- Helpers ---
def get_admin():
    a = cursor.execute("SELECT user_id FROM admin LIMIT 1").fetchone()
    return a[0] if a else None

def set_admin(uid):
    cursor.execute("DELETE FROM admin")
    cursor.execute("INSERT INTO admin(user_id) VALUES(?)", (uid,))
    conn.commit()

def save_sale(uid, cost, months, percent, plan, profit, real_profit):
    cursor.execute("""
    INSERT INTO sales(user_id, cost, months, percent, plan, profit, real_profit)
    VALUES(?,?,?,?,?,?,?)
    """, (uid, cost, months, percent, plan, profit, real_profit))
    conn.commit()

def real_profit_calc(cost, profit, months):
    inflation_year = 24
    inflation_quarter = inflation_year / 4
    quarter = months / 3
    inflation = cost * (inflation_quarter / 100) * quarter
    return round(profit - inflation, 2)

# --- Bot Commands ---
@dp.message(Command("start"))
async def start(message: Message):
    global ADMIN_ID
    if get_admin() is None:
        set_admin(message.from_user.id)
        ADMIN_ID = message.from_user.id
        await message.answer("Siz bot admini bo‘ldingiz ✅")

    await message.answer(
        "Assalomu alaykum! FoydaHisobBot ishlayapti 🤖\n"
        "Hisoblash: /calc\nOptimal ustama: /optimal\nAdmin panel: /admin"
    )

@dp.message(Command("calc"))
async def calc(message: Message):
    await message.answer(
        "Format:\nTannarx,muddat(oy),ustama%,reja(50/50 yoki 30/70)\n\n"
        "Misol:\n1000,3,15,50/50"
    )

@dp.message(F.text.contains(","))
async def calculate(message: Message):
    try:
        cost, months, percent, plan = message.text.split(",")
        cost = float(cost)
        months = int(months)
        percent = float(percent)

        profit = round(cost * percent / 100, 2)
        price = cost + profit

        first, later = (0.5, 0.5) if plan.strip() == "50/50" else (0.3, 0.7)

        pay_now = round(price * first, 2)
        pay_later = round(price * later, 2)
        month_pay = round(pay_later / months, 2)

        real = real_profit_calc(cost, profit, months)
        save_sale(message.from_user.id, cost, months, percent, plan, profit, real)

        result = f"""
📦 Tannarx: {cost}
📈 Ustama: {percent}% → {profit}
💰 Sotuv narxi: {price}

💳 To‘lov rejasi ({plan.strip()}):
- Hozir: {pay_now}
- Keyin: {pay_later}  ({month_pay}/oy)

📊 Real sof foyda: {real}
✅ {'FOYDADASIZ' if real > 0 else 'ZARARDASIZ'}
"""
        await message.answer(result)
    except:
        await message.answer("Xatolik! Misol: 1000,3,15,50/50")

@dp.message(Command("optimal"))
async def optimal(message: Message):
    await message.answer("3 oy: 12–18%\n6 oy: 25–35%\n12 oy: 45–60%")

# --- ADMIN PANEL ---
@dp.message(Command("admin"))
async def admin_panel(message: Message):
    if str(message.from_user.id) != str(get_admin()):
        return await message.answer("Siz admin emassiz!")

    all_sales = cursor.execute("SELECT COUNT(*), SUM(real_profit) FROM sales").fetchone()
    count, total_profit = all_sales[0], all_sales[1] or 0

    await message.answer(
        f"📊 Admin Panel\n"
        f"🧾 Umumiy savdolar: {count}\n"
        f"💰 Jami real foyda: {round(total_profit,2)}\n\n"
        f"Buyruqlar:\n"
        f"/history – savdolar\n"
        f"/export – Excel yuklash"
    )

@dp.message(Command("history"))
async def history(message: Message):
    if str(message.from_user.id) != str(get_admin()):
        return await message.answer("Ruxsat yo‘q!")
    rows = cursor.execute("SELECT cost, months, percent, plan, profit, real_profit FROM sales").fetchall()
    text = "📜 Savdolar:\n"
    for r in rows:
        text += f"{r}\n"
    await message.answer(text[:4000])

@dp.message(Command("export"))
async def export(message: Message):
    if str(message.from_user.id) != str(get_admin()):
        return await message.answer("Ruxsat yo‘q!")

    rows = cursor.execute("SELECT * FROM sales").fetchall()
    df = pd.DataFrame(rows, columns=["id","user","cost","months","percent","plan","profit","real_profit"])
    path = "sales.xlsx"
    df.to_excel(path, index=False)
    await message.answer_document(FSInputFile(path))

async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
