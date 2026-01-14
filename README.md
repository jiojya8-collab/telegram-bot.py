import os
import json
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, CallbackQueryHandler, filters, ContextTypes

TOKEN = os.environ.get('8439676471:AAHX65e07XNLqMOsT0g5AYCPhFGh7GehKSc')

# Datei zum Speichern der Weiterleitungen
CONFIG_FILE = 'forward_config.json'

def load_config():
    """Lädt die Weiterleitungskonfiguration"""
    try:
        with open(CONFIG_FILE, 'r') as f:
            return json.load(f)
    except FileNotFoundError:
        return {}

def save_config(config):
    """Speichert die Weiterleitungskonfiguration"""
    with open(CONFIG_FILE, 'w') as f:
        json.dump(config, f, indent=2)

# Globale Config
forward_config = load_config()

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Startnachricht mit Anleitung"""
    willkommen = """
🤖 **Telegram Forward Bot**

Ich leite Videos, Bilder und Dateien automatisch weiter!

📋 **Befehle:**
/start - Diese Nachricht
/neue_regel - Neue Weiterleitungsregel erstellen
/regeln - Alle Regeln anzeigen
/loeschen - Regel löschen
/hilfe - Detaillierte Anleitung

🔧 **Schnellstart:**
1. `/neue_regel` eingeben
2. Quelle-Chat ID eingeben
3. Ziel-Chat ID eingeben
4. Fertig! 🎉

💡 **Tipp:** Um Chat-IDs zu finden, füge mich zu einem Chat hinzu und sende /chat_info
    """
    await update.message.reply_text(willkommen, parse_mode='Markdown')

async def hilfe(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Detaillierte Hilfe"""
    hilfe_text = """
📖 **Detaillierte Anleitung**

**1️⃣ Chat-IDs herausfinden:**
- Für Gruppen: Füge mich zur Gruppe hinzu und sende `/chat_info`
- Für Kanäle: Mach mich zum Admin und sende dort `/chat_info`
- Für private Chats: Sende mir `/chat_info` im privaten Chat

**2️⃣ Weiterleitungsregel erstellen:**
```
/neue_regel
```
Dann folge den Anweisungen:
- Gib die Quell-Chat-ID ein (wo die Nachrichten herkommen)
- Gib die Ziel-Chat-ID ein (wo sie hinkommen sollen)

**3️⃣ Beispiel:**
```
Quelle: -1001234567890 (eine Gruppe)
Ziel: -1009876543210 (ein Kanal)
```

**4️⃣ Wichtige Hinweise:**
- Ich muss in BEIDEN Chats Mitglied/Admin sein
- Negative IDs sind normal für Gruppen/Kanäle
- Positive IDs sind für private Chats

**5️⃣ Berechtigungen:**
- In Gruppen: Normales Mitglied reicht
- In Kanälen: Ich muss Admin sein (zum Posten)

❓ **Probleme?**
- Überprüfe ob ich in beiden Chats bin
- Überprüfe die Chat-IDs (mit `/chat_info`)
- Stelle sicher ich habe Rechte zum Senden
    """
    await update.message.reply_text(hilfe_text, parse_mode='Markdown')

async def chat_info(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Zeigt die Chat-ID an"""
    chat = update.effective_chat
    user = update.effective_user
    
    info = f"""
ℹ️ **Chat Informationen**

**Chat ID:** `{chat.id}`
**Chat Typ:** {chat.type}
**Chat Titel:** {chat.title if chat.title else 'Privater Chat'}

**Deine User ID:** `{user.id}`
**Dein Name:** {user.full_name}

💡 Kopiere die Chat ID für deine Weiterleitungsregel!
    """
    await update.message.reply_text(info, parse_mode='Markdown')

async def neue_regel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Startet das Erstellen einer neuen Regel"""
    await update.message.reply_text(
        "📝 **Neue Weiterleitungsregel**\n\n"
        "Schritt 1/2: Sende mir die **Quell-Chat-ID** (von wo Nachrichten weitergeleitet werden sollen)\n\n"
        "💡 Tipp: Nutze `/chat_info` in dem Chat um die ID zu bekommen",
        parse_mode='Markdown'
    )
    context.user_data['erstelle_regel'] = 'warte_auf_quelle'

async def regeln_anzeigen(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Zeigt alle aktiven Regeln"""
    if not forward_config:
        await update.message.reply_text(
            "❌ Noch keine Regeln vorhanden!\n\n"
            "Erstelle eine mit `/neue_regel`"
        )
        return
    
    text = "📋 **Aktive Weiterleitungsregeln:**\n\n"
    for i, (quelle, ziel) in enumerate(forward_config.items(), 1):
        text += f"{i}. Von `{quelle}` → Nach `{ziel}`\n"
    
    text += "\n💡 Lösche Regeln mit `/loeschen`"
    await update.message.reply_text(text, parse_mode='Markdown')

async def regel_loeschen(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Löscht eine Regel"""
    if not forward_config:
        await update.message.reply_text("❌ Keine Regeln zum Löschen vorhanden!")
        return
    
    keyboard = []
    for quelle, ziel in forward_config.items():
        keyboard.append([
            InlineKeyboardButton(
                f"❌ {quelle} → {ziel}",
                callback_data=f"delete_{quelle}"
            )
        ])
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text(
        "🗑️ **Regel löschen**\n\nWähle die Regel die gelöscht werden soll:",
        reply_markup=reply_markup
    )

async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Behandelt Button-Klicks"""
    query = update.callback_query
    await query.answer()
    
    if query.data.startswith('delete_'):
        quelle = query.data.replace('delete_', '')
        if quelle in forward_config:
            ziel = forward_config[quelle]
            del forward_config[quelle]
            save_config(forward_config)
            await query.edit_message_text(
                f"✅ Regel gelöscht!\n\n"
                f"Von `{quelle}` → Nach `{ziel}` wird nicht mehr weitergeleitet.",
                parse_mode='Markdown'
            )
        else:
            await query.edit_message_text("❌ Regel nicht gefunden!")

async def text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Verarbeitet Textnachrichten für Regelkonfiguration"""
    if 'erstelle_regel' not in context.user_data:
        return
    
    status = context.user_data['erstelle_regel']
    text = update.message.text.strip()
    
    if status == 'warte_auf_quelle':
        try:
            quelle_id = int(text)
            context.user_data['quelle_id'] = str(quelle_id)
            context.user_data['erstelle_regel'] = 'warte_auf_ziel'
            await update.message.reply_text(
                f"✅ Quelle gespeichert: `{quelle_id}`\n\n"
                f"Schritt 2/2: Sende mir jetzt die **Ziel-Chat-ID** (wohin weitergeleitet werden soll)",
                parse_mode='Markdown'
            )
        except ValueError:
            await update.message.reply_text(
                "❌ Ungültige Chat-ID! Bitte sende eine Zahl (z.B. `-1001234567890`)"
            )
    
    elif status == 'warte_auf_ziel':
        try:
            ziel_id = int(text)
            quelle_id = context.user_data['quelle_id']
            
            # Speichere Regel
            forward_config[quelle_id] = str(ziel_id)
            save_config(forward_config)
            
            await update.message.reply_text(
                f"🎉 **Regel erfolgreich erstellt!**\n\n"
                f"Von: `{quelle_id}`\n"
                f"Nach: `{ziel_id}`\n\n"
                f"✅ Medien werden jetzt automatisch weitergeleitet!\n\n"
                f"💡 Stelle sicher dass ich in beiden Chats Mitglied bin.",
                parse_mode='Markdown'
            )
            
            # Reset Status
            del context.user_data['erstelle_regel']
            del context.user_data['quelle_id']
            
        except ValueError:
            await update.message.reply_text(
                "❌ Ungültige Chat-ID! Bitte sende eine Zahl (z.B. `-1009876543210`)"
            )

async def media_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Leitet Medien weiter (Bilder, Videos, Dateien)"""
    chat_id = str(update.effective_chat.id)
    
    # Prüfe ob eine Regel für diesen Chat existiert
    if chat_id not in forward_config:
        return
    
    ziel_chat = forward_config[chat_id]
    
    try:
        # Leite die Nachricht weiter
        if update.message.photo:
            # Foto weiterleiten
            photo = update.message.photo[-1]  # Höchste Auflösung
            await context.bot.send_photo(
                chat_id=ziel_chat,
                photo=photo.file_id,
                caption=update.message.caption if update.message.caption else None
            )
        
        elif update.message.video:
            # Video weiterleiten
            await context.bot.send_video(
                chat_id=ziel_chat,
                video=update.message.video.file_id,
                caption=update.message.caption if update.message.caption else None
            )
        
        elif update.message.document:
            # Datei weiterleiten
            await context.bot.send_document(
                chat_id=ziel_chat,
                document=update.message.document.file_id,
                caption=update.message.caption if update.message.caption else None
            )
        
        elif update.message.animation:
            # GIF weiterleiten
            await context.bot.send_animation(
                chat_id=ziel_chat,
                animation=update.message.animation.file_id,
                caption=update.message.caption if update.message.caption else None
            )
        
        print(f"✅ Medien weitergeleitet: {chat_id} → {ziel_chat}")
        
    except Exception as e:
        print(f"❌ Fehler beim Weiterleiten: {e}")
        # Optional: Admin benachrichtigen
        # await context.bot.send_message(
        #     chat_id=ADMIN_ID,
        #     text=f"❌ Fehler beim Weiterleiten von {chat_id} nach {ziel_chat}:\n{e}"
        # )

def main():
    """Startet den Bot"""
    if not TOKEN:
        print("❌ FEHLER: BOT_TOKEN Umgebungsvariable fehlt!")
        return
    
    print("🤖 Forward Bot startet...")
    
    app = Application.builder().token(TOKEN).build()
    
    # Befehle
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("hilfe", hilfe))
    app.add_handler(CommandHandler("chat_info", chat_info))
    app.add_handler(CommandHandler("neue_regel", neue_regel))
    app.add_handler(CommandHandler("regeln", regeln_anzeigen))
    app.add_handler(CommandHandler("loeschen", regel_loeschen))
    
    # Callbacks für Buttons
    app.add_handler(CallbackQueryHandler(button_callback))
    
    # Text für Regelkonfiguration
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, text_handler))
    
    # Medien Handler (Bilder, Videos, Dateien, GIFs)
    app.add_handler(MessageHandler(
        filters.PHOTO | filters.VIDEO | filters.Document.ALL | filters.ANIMATION,
        media_handler
    ))
    
    print("✅ Bot läuft! Bereit zum Weiterleiten...")
    app.run_polling(drop_pending_updates=True)

if __name__ == '__main__':
    main()
