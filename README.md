from flask import Flask, request
from twilio.twiml.messaging_response import MessagingResponse
import anthropic
import os

app = Flask(__name__)

# ============================================================
# ВСТАВЬТЕ СВОИ ПРОДУКТЫ СЮДА
# ============================================================
PRODUCTS = """
Наши продукты (БАДы и витамины):

1. Витамин C 1000мг — 450 сом
   Укрепляет иммунитет, мощный антиоксидант.
   Приём: 1 таблетка в день после еды.

2. Витамин D3 2000 МЕ — 380 сом
   Для здоровья костей, иммунитета и настроения.
   Приём: 1 капсула в день с едой.

3. Омега-3 (рыбий жир) — 650 сом
   Поддерживает сердце, мозг и суставы.
   Приём: 2 капсулы в день во время еды.

4. Магний B6 — 520 сом
   Снимает стресс, улучшает сон и работу нервной системы.
   Приём: 2 таблетки в день.

5. Цинк + Селен — 420 сом
   Поддерживает иммунитет и щитовидную железу.
   Приём: 1 таблетка в день.

(Замените эти продукты на ваш реальный ассортимент)
"""
# ============================================================

SYSTEM_PROMPT = f"""Ты — профессиональный консультант фармацевтической компании в WhatsApp.
Твоя задача: помогать клиентам выбрать подходящие БАДы и витамины, консультировать и принимать заказы.

{PRODUCTS}

📋 ПРАВИЛА РАБОТЫ:
- Отвечай коротко и понятно (это WhatsApp, не длинные тексты)
- Будь вежливым, дружелюбным и профессиональным
- Давай точную информацию только о наших продуктах
- Если клиент хочет купить — уточни: какой товар, количество, имя и адрес доставки
- Когда получил все данные заказа — подтверди заказ и напиши: "✅ Заказ принят! Менеджер свяжется с вами в ближайшее время."
- Если вопрос вне твоей компетенции — скажи: "Для уточнения свяжитесь с нашим менеджером."
- Всегда отвечай на русском языке
- Используй эмодзи умеренно для дружелюбности
"""

# Хранение истории диалогов (по номеру телефона)
conversations = {}

client = anthropic.Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))


def get_bot_response(user_number: str, user_message: str) -> str:
    """Генерирует ответ бота с учётом истории диалога."""

    if user_number not in conversations:
        conversations[user_number] = []

    conversations[user_number].append({
        "role": "user",
        "content": user_message
    })

    # Берём последние 10 сообщений (5 пар вопрос-ответ)
    history = conversations[user_number][-10:]

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=500,
        system=SYSTEM_PROMPT,
        messages=history
    )

    bot_reply = response.content[0].text

    conversations[user_number].append({
        "role": "assistant",
        "content": bot_reply
    })

    return bot_reply


@app.route("/webhook", methods=["POST"])
def webhook():
    """Основной вебхук для приёма сообщений от Twilio."""
    incoming_msg = request.values.get("Body", "").strip()
    from_number = request.values.get("From", "")

    if not incoming_msg:
        return str(MessagingResponse())

    try:
        reply = get_bot_response(from_number, incoming_msg)
    except Exception as e:
        reply = "Извините, произошла ошибка. Попробуйте ещё раз или свяжитесь с менеджером."
        print(f"Ошибка: {e}")

    resp = MessagingResponse()
    resp.message(reply)
    return str(resp)


@app.route("/", methods=["GET"])
def home():
    return "✅ WhatsApp бот запущен и работает!"


if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port, debug=False)
