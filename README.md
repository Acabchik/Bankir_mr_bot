# bot.py

import telebot
from telebot import types
import json
import os

TOKEN = "8762441973:AAFwE0R2adyayO7YjHB9FF06jTNbIbHCKtE"

bot = telebot.TeleBot(TOKEN)

SETTINGS_FILE = "settings.json"

# --- завантаження ---
def load_settings():
    if os.path.exists(SETTINGS_FILE):
        with open(SETTINGS_FILE, "r") as f:
            return json.load(f)
    return {"mr": 5, "Шум": 17}

# --- збереження ---
def save_settings(data):
    with open(SETTINGS_FILE, "w") as f:
        json.dump(data, f)

user_settings = load_settings()
user_data = {}
user_state = {}

# --- старт ---
@bot.message_handler(commands=['start'])
def start(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("Шум", "mr")
    markup.add("⚙️ Налаштування")
    bot.send_message(message.chat.id, "Обери варіант:", reply_markup=markup)

# --- вибір режиму ---
@bot.message_handler(func=lambda m: m.text in ["Шум", "mr"])
def choose_mode(message):
    mode = message.text
    k = user_settings[mode]

    user_data[message.chat.id] = {"k": k, "mode": mode}

    bot.send_message(message.chat.id, f"Режим: {mode}\nВведи суму в грн:")

# --- налаштування ---
@bot.message_handler(func=lambda m: m.text == "⚙️ Налаштування")
def settings(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("Змінити %")
    markup.add("Назад")
    bot.send_message(message.chat.id, "Налаштування:", reply_markup=markup)

# --- назад ---
@bot.message_handler(func=lambda m: m.text == "Назад")
def back(message):
    start(message)

# --- зміна % ---
@bot.message_handler(func=lambda m: m.text == "Змінити %")
def change_percent(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("Шум", "mr")
    markup.add("Назад")

    user_state[message.chat.id] = "choose_mode_for_percent"
    bot.send_message(message.chat.id, "Для якого режиму змінити %?", reply_markup=markup)

# --- вибір режиму ---
@bot.message_handler(func=lambda m: user_state.get(m.chat.id) == "choose_mode_for_percent")
def choose_mode_for_percent(message):
    if message.text not in ["Шум", "mr"]:
        return

    user_state[message.chat.id] = "enter_new_percent"
    user_data[message.chat.id] = {"mode_to_change": message.text}

    bot.send_message(message.chat.id, f"Введи новий % для {message.text}:")

# --- новий % ---
@bot.message_handler(func=lambda m: user_state.get(m.chat.id) == "enter_new_percent")
def set_new_percent(message):
    try:
        new_k = float(message.text)
        mode = user_data[message.chat.id]["mode_to_change"]

        user_settings[mode] = new_k
        save_settings(user_settings)  # 🔥 збереження

        user_state[message.chat.id] = None

        bot.send_message(message.chat.id, f"% для {mode} змінено на {new_k}%")
        start(message)

    except:
        bot.send_message(message.chat.id, "Введи число")

# --- розрахунок ---
@bot.message_handler(func=lambda message: True)
def calculate(message):
    chat_id = message.chat.id

    try:
        a = float(message.text)
        k = user_data[chat_id]["k"]

        b = a / 46.35
        v = b * (k / 100)
        g = b - (b * (k / 100))

        result = (
            f"a = {a} грн\n"
            f"b = {b:.2f} $\n"
            f"k = {k}%\n\n"
            f"b * k% = {v:.2f} $\n"
            f"b - k% = {g:.2f} $"
        )

        bot.send_message(chat_id, result)

    except:
        bot.send_message(chat_id, "Введи число")

import threading
from flask import Flask

app = Flask('')

@app.route('/')
def home():
    return "I'm alive"

def run():
    app.run(host='0.0.0.0', port=8080)

def keep_alive():
    t = threading.Thread(target=run)
    t.start()

if __name__ == "__main__":
    keep_alive()  # Запускає сервер у фоні
    bot.infinity_polling() # Бот працює без зупинок
