import asyncio
import datetime
import logging
import re
import sqlite3
from typing import Optional

import aiohttp
from aiogram import Bot, Dispatcher, html
from aiogram.enums import ChatType, ParseMode
from aiogram.filters import Command
from aiogram.types import BufferedInputFile, Message

# ==========================================
# ⚙️ НАСТРОЙКИ БОТА
# ==========================================
BOT_TOKEN = "8315469284:AAEOYRnSNO8Nh65N9Telsb7cx4cAXMHy7wU"
ADMIN_ID = 8990439952  # Твой ID

logging.basicConfig(level=logging.INFO)
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()


# ==========================================
# 🗄 РАБОТА С БАЗОЙ ДАННЫХ (SQLite)
# ==========================================
class Database:
    def __init__(self, db_file: str = "tracker_bot.db"):
        self.db_file = db_file
        self.init_db()

    def get_connection(self):
        return sqlite3.connect(self.db_file)

    def init_db(self):
        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            cursor.execute("""
                CREATE TABLE IF NOT EXISTS users (
                    user_id INTEGER PRIMARY KEY,
                    first_name TEXT,
                    last_name TEXT,
                    username TEXT,
                    total_msgs INTEGER DEFAULT 0,
                    voice_count INTEGER DEFAULT 0,
                    video_note_count INTEGER DEFAULT 0,
                    reply_count INTEGER DEFAULT 0,
                    last_active TIMESTAMP
                )
            """)

            cursor.execute("""
                CREATE TABLE IF NOT EXISTS user_chats (
                    user_id INTEGER,
                    chat_id INTEGER,
                    chat_title TEXT,
                    chat_username TEXT,
                    chat_type TEXT,
                    last_seen TIMESTAMP,
                    PRIMARY KEY (user_id, chat_id)
                )
            """)

            cursor.execute("""
                CREATE TABLE IF NOT EXISTS user_interactions (
                    user_id INTEGER,
                    target_id INTEGER,
                    target_name TEXT,
                    interaction_count INTEGER DEFAULT 1,
                    last_interaction TIMESTAMP,
                    PRIMARY KEY (user_id, target_id)
                )
            """)
            conn.commit()

    def process_activity(self, user, chat, msg: Message):
        now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        profile_changes = []

        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            cursor.execute("""
                SELECT first_name, last_name, username, total_msgs, voice_count, video_note_count, reply_count 
                FROM users WHERE user_id = ?
            """, (user.id,))
            row = cursor.fetchone()

            if row is None:
                cursor.execute("""
                    INSERT INTO users (user_id, first_name, last_name, username, total_msgs, voice_count, video_note_count, reply_count, last_active)
                    VALUES (?, ?, ?, ?, 0, 0, 0, 0, ?)
                """, (user.id, user.first_name, user.last_name, user.username, now))
                total_msgs = voice_cnt = video_cnt = reply_cnt = 0
            else:
                old_first, old_last, old_user, total_msgs, voice_cnt, video_cnt, reply_cnt = row
                
                if old_first != user.first_name:
                    profile_changes.append(f"<b>Имя:</b> <code>{html.quote(str(old_first))}</code> ➡️ <code>{html.quote(str(user.first_name))}</code>")
                if old_last != user.last_name:
                    profile_changes.append(f"<b>Фамилия:</b> <code>{html.quote(str(old_last))}</code> ➡️ <code>{html.quote(str(user.last_name))}</code>")
                if old_user != user.username:
                    old_u = f"@{old_user}" if old_user else "Отсутствовал"
                    new_u = f"@{user.username}" if user.username else "Удален"
                    profile_changes.append(f"<b>Юзернейм:</b> <code>{old_u}</code> ➡️ <code>{new_u}</code>")

            total_msgs += 1
            if msg.voice:
                voice_cnt += 1
            if msg.video_note:
                video_cnt += 1
            if msg.reply_to_message:
                reply_cnt += 1

            cursor.execute("""
                UPDATE users 
                SET first_name = ?, last_name = ?, username = ?, total_msgs = ?, 
                    voice_count = ?, video_note_count = ?, reply_count = ?, last_active = ?
                WHERE user_id = ?
            """, (user.first_name, user.last_name, user.username, total_msgs, voice_cnt, video_cnt, reply_cnt, now, user.id))

            if chat.type in [ChatType.GROUP, ChatType.SUPERGROUP]:
                c_type = "Группа / Чат"
                c_title = chat.title or "Частная группа"

                cursor.execute("""
                    INSERT INTO user_chats (user_id, chat_id, chat_title, chat_username, chat_type, last_seen)
                    VALUES (?, ?, ?, ?, ?, ?)
                    ON CONFLICT(user_id, chat_id) DO UPDATE SET
                        chat_title = excluded.chat_title,
                        chat_username = excluded.chat_username,
                        chat_type = excluded.chat_type,
                        last_seen = excluded.last_seen
                """, (user.id, chat.id, c_title, chat.username, c_type, now))

            if msg.forward_from_chat:
                f_chat = msg.forward_from_chat
                f_title = f_chat.title or "Канал"
                f_type = "ТГ-Канал" if f_chat.type == ChatType.CHANNEL else "Чат"
                
                cursor.execute("""
                    INSERT INTO user_chats (user_id, chat_id, chat_title, chat_username, chat_type, last_seen)
                    VALUES (?, ?, ?, ?, ?, ?)
                    ON CONFLICT(user_id, chat_id) DO UPDATE SET
                        chat_title = excluded.chat_title,
                        chat_username = excluded.chat_username,
                        last_seen = excluded.last_seen
                """, (user.id, f_chat.id, f_title, f_chat.username, f_type, now))

            target_user = None
            if msg.reply_to_message and msg.reply_to_message.from_user:
                target_user = msg.reply_to_message.from_user
            elif msg.forward_from:
                target_user = msg.forward_from

            if target_user and not target_user.is_bot and target_user.id != user.id:
                t_name = f"{target_user.first_name or ''} {target_user.last_name or ''}".strip()
                if target_user.username:
                    t_name += f" (@{target_user.username})"

                cursor.execute("""
                    INSERT INTO user_interactions (user_id, target_id, target_name, interaction_count, last_interaction)
                    VALUES (?, ?, ?, 1, ?)
                    ON CONFLICT(user_id, target_id) DO UPDATE SET
                        interaction_count = interaction_count + 1,
                        target_name = excluded.target_name,
                        last_interaction = excluded.last_interaction
                """, (user.id, target_user.id, t_name, now))

            conn.commit()

        return profile_changes

    def get_user_dossier(self, user_id: int):
        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            cursor.execute("""
                SELECT user_id, first_name, last_name, username, total_msgs, voice_count, video_note_count, reply_count, last_active 
                FROM users WHERE user_id = ?
            """, (user_id,))
            user_data = cursor.fetchone()

            if not user_data:
                return None

            cursor.execute("""
                SELECT chat_title, chat_username, chat_type, last_seen 
                FROM user_chats WHERE user_id = ? ORDER BY last_seen DESC
            """, (user_id,))
            chats_data = cursor.fetchall()

            cursor.execute("""
                SELECT target_name, interaction_count 
                FROM user_interactions WHERE user_id = ? ORDER BY interaction_count DESC LIMIT 5
            """, (user_id,))
            interactions_data = cursor.fetchall()

            return {
                "user": user_data,
                "chats": chats_data,
                "interactions": interactions_data
            }


db = Database()


# ==========================================
# 📊 КОМАНДЫ БОТА
# ==========================================
@dp.message(Command("stats", "info"))
async def cmd_show_dossier(message: Message):
    target_user = message.reply_to_message.from_user if message.reply_to_message else message.from_user
    dossier = db.get_user_dossier(target_user.id)

    if not dossier:
        await message.reply("❌ Данные об этом пользователе ещё не собраны.")
        return

    u = dossier["user"]
    chats = dossier["chats"]
    interactions = dossier["interactions"]

    user_id, first_name, last_name, username, total_msgs, voice_cnt, video_cnt, reply_cnt, last_active = u

    reply_pct = round((reply_cnt / total_msgs * 100), 1) if total_msgs > 0 else 0.0
    voice_pct = round((voice_cnt / total_msgs * 100), 1) if total_msgs > 0 else 0.0
    video_pct = round((video_cnt / total_msgs * 100), 1) if total_msgs > 0 else 0.0

    full_name = f"{first_name or ''} {last_name or ''}".strip()
    username_str = f"@{username}" if username else "Отсутствует"

    public_chats = []
    private_chats = []

    if chats:
        for title, chat_uname, c_type, _ in chats:
            if chat_uname:
                public_chats.append(f'  • <a href="https://t.me/{chat_uname}">{html.quote(title)}</a> (<code>@{chat_uname}</code>)')
            else:
                private_chats.append(f'  • 🔒 <b>{html.quote(title)}</b> <i>(Закрытая группа)</i>')

    public_str = "\n".join(public_chats) if public_chats else "  <i>Нет данных о публичных чатах</i>"
    private_str = "\n".join(private_chats) if private_chats else "  <i>Нет данных о закрытых группах</i>"

    interactions_list = []
    if interactions:
        for name, count in interactions:
            interactions_list.append(f'  • <b>{html.quote(name)}</b> — <code>{count}</code> раз(а)')
    interactions_str = "\n".join(interactions_list) if interactions_list else "  <i>Взаимодействия не зафиксированы</i>"

    dossier_card = (
        f"┌───────────────────────────────┐\n"
        f"│       📊 <b>ДОСЬЕ ПОЛЬЗОВАТЕЛЯ</b>        │\n"
        f"└───────────────────────────────┘\n"
        f"👤 <b>Имя:</b> {html.quote(full_name)}\n"
        f"🏷 <b>Юзернейм:</b> {username_str}\n"
        f"🆔 <b>ID:</b> <code>{user_id}</code>\n"
        f"🕒 <b>Активность:</b> {last_active}\n"
        f"───────────────────────────────\n"
        f"┌───────────────────────────────┐\n"
        f"│      📈 <b>АКТИВНОСТЬ И МЕДИA</b>       │\n"
        f"└───────────────────────────────┘\n"
        f"💬 Сообщений: <code>{total_msgs}</code>\n"
        f"🎙 Голосовых: <code>{voice_cnt}</code> (<b>{voice_pct}%</b>)\n"
        f"📹 Кружков:   <code>{video_cnt}</code> (<b>{video_pct}%</b>)\n"
        f"↩️ Реплаев:   <code>{reply_cnt}</code> (<b>{reply_pct}%</b>)\n"
        f"───────────────────────────────\n"
        f"┌───────────────────────────────┐\n"
        f"│     👥 <b>ОБНАРУЖЕННЫЕ СВЯЗИ</b>       │\n"
        f"└───────────────────────────────┘\n"
        f"{interactions_str}\n"
        f"───────────────────────────────\n"
        f"┌───────────────────────────────┐\n"
        f"│     🌐 <b>ОТКРЫТЫЕ ЧАТЫ И ТГК</b>     │\n"
        f"└───────────────────────────────┘\n"
        f"{public_str}\n"
        f"───────────────────────────────\n"
        f"┌───────────────────────────────┐\n"
        f"│     🔒 <b>ЗАКРЫТЫЕ ГРУППЫ (НАЗВАНИЯ)</b> │\n"
        f"└───────────────────────────────┘\n"
        f"{private_str}\n"
        f"───────────────────────────────"
    )

    await message.reply(dossier_card, parse_mode=ParseMode.HTML, disable_web_page_preview=True)


@dp.message(Command("export_dossier", "export"))
async def cmd_export_dossier(message: Message):
    target_user = message.reply_to_message.from_user if message.reply_to_message else message.from_user
    dossier = db.get_user_dossier(target_user.id)

    if not dossier:
        await message.reply("❌ Нет данных для экспорта.")
        return

    u = dossier["user"]
    chats = dossier["chats"]
    interactions = dossier["interactions"]

    user_id, first_name, last_name, username, total_msgs, voice_cnt, video_cnt, reply_cnt, last_active = u
    reply_pct = round((reply_cnt / total_msgs * 100), 1) if total_msgs > 0 else 0.0
    voice_pct = round((voice_cnt / total_msgs * 100), 1) if total_msgs > 0 else 0.0
    video_pct = round((video_cnt / total_msgs * 100), 1) if total_msgs > 0 else 0.0

    full_name = f"{first_name or ''} {last_name or ''}".strip()

    report = [
        "==================================================",
        "           ПОЛНОЕ ДОСЬЕ ПОЛЬЗОВАТЕЛЯ             ",
        "==================================================",
        f"Имя Пользователя : {full_name}",
        f"Юзернейм         : @{username if username else 'НЕТ'}",
        f"Telegram ID      : {user_id}",
        f"Последняя запись : {last_active}",
        "--------------------------------------------------",
        "          СТАТИСТИКА СООБЩЕНИЙ И МЕДИА            ",
        "--------------------------------------------------",
        f"Всего сообщений : {total_msgs}",
        f"Голосовых (Voice): {voice_cnt} ({voice_pct}%)",
        f"Кружков (Video)  : {video_cnt} ({video_pct}%)",
        f"Реплаев (Ответов): {reply_cnt} ({reply_pct}%)",
        "--------------------------------------------------",
        "               ОБНАРУЖЕННЫЕ СВЯЗИ                 ",
        "--------------------------------------------------"
    ]

    if interactions:
        for idx, (name, count) in enumerate(interactions, 1):
            report.append(f"{idx}. {name} — {count} взаимодействий")
    else:
        report.append("Взаимодействия не зафиксированы.")

    report.extend([
        "--------------------------------------------------",
        "         ИСТОРИЯ ЧАТОВ И КАНАЛОВ (ОБНАРУЖЕНО)      ",
        "--------------------------------------------------"
    ])

    if chats:
        for idx, (title, chat_uname, c_type, last_seen) in enumerate(chats, 1):
            uname_formatted = f"@{chat_uname}" if chat_uname else "Закрытый/Без юзернейма"
            report.append(f"{idx}. [{c_type}] {title} | Юзернейм: {uname_formatted} | Виден: {last_seen}")
    else:
        report.append("Данных о чатах не найдено.")

    report.append("==================================================")

    file_bytes = "\n".join(report).encode("utf-8")
    document = BufferedInputFile(file_bytes, filename=f"dossier_user_{user_id}.txt")

    await message.reply_document(
        document=document,
        caption=f"📄 <b>Полное досье экспортировано для ID:</b> <code>{user_id}</code>",
        parse_mode=ParseMode.HTML
    )


@dp.message()
async def process_all_messages(message: Message):
    user = message.from_user
    if not user or user.is_bot:
        return

    changes = db.process_activity(user, message.chat, message)

    if changes and ADMIN_ID != 0:
        changes_text = "\n".join(changes)
        alert_card = (
            f"🚨 <b>ОБНАРУЖЕНО ИЗМЕНЕНИЕ ПРОФИЛЯ!</b>\n"
            f"━━━━━━━━━━━━━━━━━━━━━━━\n"
            f"👤 <b>Пользователь:</b> {html.quote(user.full_name)}\n"
            f"🆔 <b>ID:</b> <code>{user.id}</code>\n"
            f"━━━━━━━━━━━━━━━━━━━━━━━\n"
            f"<b>Изменения:</b>\n{changes_text}"
        )
        try:
            await bot.send_message(chat_id=ADMIN_ID, text=alert_card, parse_mode=ParseMode.HTML)
        except Exception:
            pass


# ==========================================
# 🚀 ЗАПУСК БОТА
# ==========================================
async def main():
    print("Бот успешно запущен и следит за активностью!")
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
