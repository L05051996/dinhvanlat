import telebot
import requests
import json
import os
import sqlite3
import logging
import time
import random
from datetime import datetime, timezone, timedelta
from textblob import TextBlob
from dotenv import load_dotenv
from tenacity import retry, stop_after_attempt, wait_fixed

# Load environment variables
load_dotenv()
TOKEN = os.getenv("TELEGRAM_TOKEN", "7109961635:AAHY8JfCuOxQedZZWFZOZNP_o1fzjOytDGQ")
WEATHER_API_KEY = os.getenv("WEATHER_API_KEY", "your_openweathermap_api_key")
ALPHA_VANTAGE_API_KEY = os.getenv("ALPHA_VANTAGE_API_KEY", "your_alpha_vantage_api_key")
LLM_API_KEY = os.getenv("LLM_API_KEY", "your_llm_api_key")
LLM_API_URL = os.getenv("LLM_API_URL", "https://api.x.ai/grok")

# Configure logging
logging.basicConfig(filename='bot.log', level=logging.INFO,
                    format='%(asctime)s - %(levelname)s - %(message)s')

# Initialize bot
bot = telebot.TeleBot(TOKEN)
weather_cache = {}
market_cache = {}
user_context = {}
PREF_FILE = "user_preferences.json"
VN_TIMEZONE = timezone(timedelta(hours=7))  # Vietnam timezone (+07)

# Initialize SQLite database
def init_db():
    conn = sqlite3.connect('chat_history.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS messages
                 (user_id TEXT, message TEXT, response TEXT, timestamp TEXT)''')
    conn.commit()
    conn.close()

init_db()

# Cleanup old messages
def cleanup_old_messages(days=30):
    conn = sqlite3.connect('chat_history.db')
    c = conn.cursor()
    cutoff = (datetime.now(VN_TIMEZONE) - timedelta(days=days)).strftime("%Y-%m-%d %H:%M:%S")
    c.execute("DELETE FROM messages WHERE timestamp < ?", (cutoff,))
    conn.commit()
    conn.close()

# Load/save user preferences
def load_preferences():
    if os.path.exists(PREF_FILE):
        with open(PREF_FILE, 'r') as f:
            return json.load(f)
    return {}

def save_preferences(prefs):
    with open(PREF_FILE, 'w') as f:
        json.dump(prefs, f)

# Save/retrieve conversation history
def save_message(user_id, message, response):
    conn = sqlite3.connect('chat_history.db')
    c = conn.cursor()
    timestamp = datetime.now(VN_TIMEZONE).strftime("%Y-%m-%d %H:%M:%S")
    c.execute("INSERT INTO messages (user_id, message, response, timestamp) VALUES (?, ?, ?, ?)",
              (user_id, message, response, timestamp))
    conn.commit()
    conn.close()

def get_recent_messages(user_id, limit=5):
    conn = sqlite3.connect('chat_history.db')
    c = conn.cursor()
    c.execute("SELECT message, response FROM messages WHERE user_id = ? ORDER BY timestamp DESC LIMIT ?",
              (user_id, limit))
    messages = c.fetchall()
    conn.close()
    return messages

# Intent detection with Vietnamese support
def extract_intent(text):
    text = text.lower()
    if any(word in text for word in ["time", "clock", "hour", "gio", "thoi gian"]):
        return "time"
    if any(word in text for word in ["weather", "forecast", "thoi tiet", "du bao"]):
        return "weather"
    if any(word in text for word in ["joke", "funny", "laugh", "vui", "cuoi"]):
        return "joke"
    if any(word in text for word in ["market", "stock", "crypto", "forex", "price", "thi truong", "co phieu", "tien dien tu", "gia"]):
        return "market"
    return "general"

# LLM integration
@retry(stop=stop_after_attempt(3), wait=wait_fixed(2))
def get_llm_response(prompt, user_id):
    headers = {
        "Authorization": f"Bearer {LLM_API_KEY}",
        "Content-Type": "application/json"
    }
    context = get_recent_messages(user_id, limit=3)
    context_text = "\n".join([f"User: {msg}\nBot: {resp}" for msg, resp in context])
    full_prompt = f"Context:\n{context_text}\n\nUser: {prompt}\nBot:"
    
    payload = {
        "prompt": full_prompt,
        "max_tokens": 500,
        "temperature": 0.7
    }
    
    try:
        response = requests.post(LLM_API_URL, headers=headers, json=payload)
        response.raise_for_status()
        result = response.json()
        return result.get("choices", [{}])[0].get("text", "Xin lỗi, tui không hiểu lắm! 😅")
    except Exception as e:
        logging.error(f"LLM API error: {str(e)}")
        return "Ôi, tui hơi lag! 😴 Thử lại nha!"

# Weather API
@retry(stop=stop_after_attempt(3), wait=wait_fixed(2))
def get_weather(city):
    if city in weather_cache and time.time() - weather_cache[city][1] < 600:
        return weather_cache[city][0]
    
    url = f"http://api.openweathermap.org/data/2.5/weather?q={city}&appid={WEATHER_API_KEY}&units=metric"
    try:
        response = requests.get(url).json()
        if response.get("cod") != 200:
            logging.warning(f"Weather API error for city {city}: {response.get('message')}")
            return f"Không tìm thấy thời tiết cho {city}. 😅"
        weather = response["weather"][0]["description"]
        temp = response["main"]["temp"]
        result = f"{weather}, {temp}°C"
        weather_cache[city] = (result, time.time())
        return result
    except Exception as e:
        logging.error(f"Error fetching weather data: {str(e)}")
        return f"Lỗi lấy dữ liệu thời tiết! 😴 Thử lại nha!"

# Market API
@retry(stop=stop_after_attempt(3), wait=wait_fixed(2))
def get_market_data(symbol, market_type="stock"):
    cache_key = f"{symbol}_{market_type}"
    if cache_key in market_cache and time.time() - market_cache[cache_key][1] < 300:
        return market_cache[cache_key][0]
    
    try:
        if market_type == "crypto":
            url = f"https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&ids={symbol.lower()}"
            response = requests.get(url).json()
            if not response:
                return f"Không tìm thấy dữ liệu cho {symbol}. 😅"
            data = response[0]
            result = (
                f"Tiền điện tử: {data['name']} ({data['symbol'].upper()})\n"
                f"Giá: ${data['current_price']:.2f}\n"
                f"Thay đổi 24h: {data['price_change_percentage_24h']:.2f}%\n"
                f"Vốn hóa: ${data['market_cap']:,.0f}"
            )
        else:
            if market_type == "forex":
                url = f"https://www.alphavantage.co/query?function=FX_DAILY&from_symbol={symbol[:3]}&to_symbol={symbol[3:]}&apikey={ALPHA_VANTAGE_API_KEY}"
            else:
                url = f"https://www.alphavantage.co/query?function=GLOBAL_QUOTE&symbol={symbol}&apikey={ALPHA_VANTAGE_API_KEY}"
            
            response = requests.get(url).json()
            if "Error Message" in response:
                return f"Không tìm thấy dữ liệu cho {symbol}. 😅"
            if "Note" in response and "rate limit" in response["Note"].lower():
                return f"API bận rồi! 😴 Thử lại sau vài phút nha!"
            
            if market_type == "forex":
                data = response["Time Series FX (Daily)"][list(response["Time Series FX (Daily)"].keys())[0]]
                result = (
                    f"Cặp ngoại hối: {symbol[:3]}/{symbol[3:]}\n"
                    f"Đóng cửa: {float(data['4. close']):.4f}\n"
                    f"Cao: {float(data['2. high']):.4f}\n"
                    f"Thấp: {float(data['3. low']):.4f}"
                )
            else:
                data = response["Global Quote"]
                result = (
                    f"Cổ phiếu: {symbol}\n"
                    f"Giá: ${float(data['05. price']):.2f}\n"
                    f"Thay đổi: {float(data['09. change']):.2f} ({float(data['10. change percent'].strip('%')):.2f}%)\n"
                    f"Khối lượng: {int(data['06. volume']):,}"
                )
        
        market_cache[cache_key] = (result, time.time())
        return result
    except Exception as e:
        logging.error(f"Error fetching market data for {symbol}: {str(e)}")
        return f"Ôi, lỗi rồi! 😅 Thử lại sau nha!"

# Joke function
def get_joke():
    jokes = [
        "Sao thị trường chứng khoán lại sụp? Vì nó say margin! 📉",
        "Tại sao đám mây đi trị liệu tâm lý? Vì nó có quá nhiều mối quan hệ 'giông bão'! 🌩️",
        "Lập trình viên thích sáng hay tối? Tối, vì sáng thu hút bọ! 🐞",
        "Bitcoin mà nghèo thì gọi là gì? Dân ăn xin điện tử! 😜"
    ]
    return random.choice(jokes)

# Sentiment analysis
def analyze_sentiment(text):
    blob = TextBlob(text)
    sentiment = blob.sentiment.polarity
    if sentiment > 0.1:
        return "positive", random.choice([
            "Trời ơi, bạn vui thế! Có gì hay ho đang xảy ra hả? 😄",
            "Hôm nay bạn phấn khởi ghê! Kể tui nghe đi! 🚀"
        ])
    elif sentiment < -0.1:
        return "negative", random.choice([
            "Ôi, sao nghe buồn vậy? Muốn tui kể chuyện cười cho vui không? 😊",
            "Hôm nay hơi xuống mood hả? Thử hỏi tui về thời tiết hoặc thị trường xem! 📈"
        ])
    else:
        return "neutral", random.choice([
            "Bình thường thôi hả? Muốn tui khuấy động tí không? 😎",
            "Chill thế! Hỏi tui gì đi, thời tiết hay cổ phiếu? 🤙"
        ])

# Bot commands
@bot.message_handler(commands=['start'])
def start(message):
    user_name = message.from_user.first_name
    response = f"Xin chào {user_name}! Tui là bot thông minh, biết từ thời tiết, chứng khoán đến trả lời mọi câu hỏi! Gõ /help để khám phá nha! 😎"
    save_message(str(message.from_user.id), message.text, response)
    bot.reply_to(message, response)

@bot.message_handler(commands=['help', 'tro_giup'])
def help_command(message):
    help_text = (
        "Tui là bot siêu xịn! 😎 Có thể làm:\n"
        "/start - Chào bạn\n"
        "/help hoặc /tro_giup - Xem hướng dẫn\n"
        "/weather hoặc /thoitiet <thành phố> - Xem thời tiết\n"
        "/market hoặc /thitruong <mã> [stock|crypto|forex] - Xem thị trường\n"
        "/joke hoặc /cuoi - Nghe chuyện cười\n"
        "/history hoặc /lich_su - Xem lịch sử trò chuyện\n"
        "/feedback hoặc /phan_hoi <text> - Gửi phản hồi\n"
        "/preferences hoặc /ca_nhan - Xem sở thích\n"
        "/reset_preferences hoặc /xoa_ca_nhan - Xóa sở thích\n"
        "/status hoặc /trang_thai - Kiểm tra trạng thái bot\n"
        "Hỏi gì cũng được, tui trả lời hết! 😄"
    )
    save_message(str(message.from_user.id), message.text, help_text)
    bot.reply_to(message, help_text)

@bot.message_handler(commands=['weather', 'thoitiet'])
def weather_command(message):
    user_id = str(message.from_user.id)
    try:
        city = message.text.split(maxsplit=1)[1]
        user_context[user_id] = {"last_city": city}
        preferences = load_preferences()
        if user_id not in preferences:
            preferences[user_id] = {"favorites": []}
        favorites = preferences[user_id].get("favorites", [])
        if city not in favorites:
            favorites.append(city)
        preferences[user_id]["favorites"] = favorites
        save_preferences(preferences)
        
        weather_info = get_weather(city)
        response = f"{city}: {weather_info}. Cool không? 😄"
        save_message(user_id, message.text, response)
        bot.reply_to(message, response)
    except IndexError:
        preferences = load_preferences()
        if user_id in preferences and preferences[user_id].get("favorites", []):
            city = preferences[user_id]["favorites"][-1]
            weather_info = get_weather(city)
            response = f"Chưa chọn thành phố, tui check {city} nè: {weather_info}. Thử cái khác? 😎"
        else:
            response = "Cho tui một thành phố đi, ví dụ: /thoitiet Hà Nội! 😄"
        save_message(user_id, message.text, response)
        bot.reply_to(message, response)

@bot.message_handler(commands=['market', 'thitruong'])
def market_command(message):
    user_id = str(message.from_user.id)
    try:
        parts = message.text.split(maxsplit=2)
        if len(parts) < 2:
            raise IndexError
        symbol = parts[1].upper()
        market_type = parts[2].lower() if len(parts) > 2 else "stock"
        
        if market_type not in ["stock", "crypto", "forex"]:
            response = "Sai loại rồi! Chỉ có 'stock', 'crypto', hoặc 'forex' thôi nha! 😅"
        else:
            market_data = get_market_data(symbol, market_type)
            response = f"{market_data}. Xem cái khác không? 📈"
            
            preferences = load_preferences()
            if user_id not in preferences:
                preferences[user_id] = {"favorites": []}
            favorites = preferences[user_id].get("favorites", [])
            if {"symbol": symbol, "type": market_type} not in favorites:
                favorites.append({"symbol": symbol, "type": market_type})
            preferences[user_id]["favorites"] = favorites
            save_preferences(preferences)
            user_context[user_id] = {"last_favorite": {"symbol": symbol, "type": market_type}}
        
        save_message(user_id, message.text, response)
        bot.reply_to(message, response)
    except IndexError:
        preferences = load_preferences()
        if user_id in preferences and preferences[user_id].get("favorites", []):
            last_favorite = preferences[user_id]["favorites"][-1]
            market_data = get_market_data(last_favorite["symbol"], last_favorite["type"])
            response = f"Chưa chọn mã, tui check {last_favorite['symbol']} ({last_favorite['type']}) nè: {market_data}. Thử cái khác? 😎"
        else:
            response = "Cho tui mã với loại đi, ví dụ: /thitruong AAPL stock hoặc /thitruong BTC crypto! 😄"
        save_message(user_id, message.text, response)
        bot.reply_to(message, response)

@bot.message_handler(commands=['joke', 'cuoi'])
def joke_command(message):
    joke = get_joke()
    response = f"{joke}. Muốn nghe thêm không? 😜"
    save_message(str(message.from_user.id), message.text, response)
    bot.reply_to(message, response)

@bot.message_handler(commands=['history', 'lich_su'])
def history_command(message):
    user_id = str(message.from_user.id)
    messages = get_recent_messages(user_id)
    if messages:
        response = "Lịch sử trò chuyện của bạn:\n"
        for msg, resp in messages:
            response += f"Bạn: {msg}\nTui: {resp}\n\n"
    else:
        response = "Chưa có lịch sử trò chuyện nè! 😅"
    save_message(user_id, message.text, response)
    bot.reply_to(message, response)

@bot.message_handler(commands=['feedback', 'phan_hoi'])
def feedback_command(message):
    user_id = str(message.from_user.id)
    try:
        feedback = message.text.split(maxsplit=1)[1]
        logging.info(f"Feedback from {user_id}: {feedback}")
        response = "Cảm ơn phản hồi của bạn! Tui sẽ cố gắng tốt hơn nha! 😄"
    except IndexError:
        response = "Cho tui ý kiến đi, ví dụ: /phan_hoi Bot xịn lắm! 😎"
    save_message(user_id, message.text, response)
    bot.reply_to(message, response)

@bot.message_handler(commands=['preferences', 'ca_nhan'])
def preferences_command(message):
    user_id = str(message.from_user.id)
    preferences = load_preferences()
    if user_id in preferences:
        favorites = preferences[user_id].get("favorites", [])
        response = "Cá nhân của bạn:\n"
        response += f"- Yêu thích: {', '.join([f'{m['symbol']} ({m['type']})' if isinstance(m, dict) else m for m in favorites]) if favorites else 'Chưa có'}\n"
        response += "Muốn xóa hết? Gõ /xoa_ca_nhan nha! 😄"
    else:
        response = "Chưa có thông tin cá nhân nào! Thử /thoitiet hoặc /thitruong để lưu nha! 😎"
    save_message(user_id, message.text, response)
    bot.reply_to(message, response)

@bot.message_handler(commands=['reset_preferences', 'xoa_ca_nhan'])
def reset_preferences_command(message):
    user_id = str(message.from_user.id)
    preferences = load_preferences()
    if user_id in preferences:
        del preferences[user_id]
        save_preferences(preferences)
        response = "Xóa hết thông tin cá nhân rồi nha! 😄 Bắt đầu lại nào!"
    else:
        response = "Chưa có gì để xóa cả! 😅"
    save_message(user_id, message.text, response)
    bot.reply_to(message, response)

@bot.message_handler(commands=['status', 'trang_thai'])
def status_command(message):
    user_id = str(message.from_user.id)
    response = "Tui vẫn sống khỏe! 😎 API thời tiết, thị trường và LLM đang hoạt động. Muốn thử hỏi gì không?"
    save_message(user_id, message.text, response)
    bot.reply_to(message, response)

@bot.message_handler(func=lambda message: True)
def handle_all_messages(message):
    user_id = str(message.from_user.id)
    user_name = message.from_user.first_name
    text = message.text.lower()
    sentiment, sentiment_response = analyze_sentiment(message.text)
    intent = extract_intent(message.text)
    
    logging.info(f"User: {user_name}, Message: {message.text}, Sentiment: {sentiment}, Intent: {intent}")
    
    if intent == "time":
        current_time = datetime.now(VN_TIMEZONE).strftime("%H:%M:%S")
        response = f"Chào {user_name}, bây giờ là {current_time}. Làm gì tiếp nào? 😎 {sentiment_response}"
    elif intent == "weather":
        city = user_context.get(user_id, {}).get("last_city") or (load_preferences().get(user_id, {}).get("favorites", [None])[-1] if isinstance(load_preferences().get(user_id, {}).get("favorites", [None])[-1], str) else None)
        if city:
            weather_info = get_weather(city)
            response = f"Thời tiết {city}: {weather_info}. Muốn xem nơi khác không? 🌤️ {sentiment_response}"
        else:
            response = f"Chưa chọn thành phố nè! Thử /thoitiet Hà Nội đi! 😄 {sentiment_response}"
    elif intent == "joke":
        response = f"{get_joke()}. Muốn thêm không vui? 😜 {sentiment_response}"
    elif intent == "market":
        last_favorite = user_context.get(user_id, {}).get("last_favorite") or load_preferences().get(user_id, {}).get("favorites", [None])[0]
        if last_favorite:
            market_data = get_market_data(last_favorite["symbol"], last_favorite["type"])
            response = f"Thị trường {last_favorite['symbol']}: {market_data}. Xem cái khác xem! 📊 {sentiment_response}"
        else:
            response = f"Muốn xem cổ phiếu hay crypto? Thử /thitruong BTC crypto nha! 😎 {sentiment_response}"
    else:
        llm_response = get_llm_response(message.text, user_id)
        response = f"{llm_response} \nHỏi tiếp đi nha, {user_name}! 😄 {sentiment_response}"
    
    save_message(user_id, message.text, response)
    bot.reply_to(message, response)

if __name__ == "__main__":
    print("Bot is running...")
    cleanup_old_messages()
    try:
        bot.polling()
    except Exception as e:
        logging.error(f"Bot polling error: {str(e)}")
        print(f"Error: {str(e)}. Check bot.log for details.")