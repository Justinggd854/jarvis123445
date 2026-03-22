# ---------------- Jarvis Ultimate: All-in-One ----------------
import os, json, datetime, subprocess, pyautogui, threading
import speech_recognition as sr, pyttsx3, requests
from websocket import create_connection
from flask import Flask, request, jsonify
from flask_socketio import SocketIO, emit
from openai import OpenAI
from PyQt5.QtWidgets import QApplication, QWidget, QLabel
from PyQt5.QtCore import Qt
import sys

# ---------------- CONFIG ----------------
WAKE_WORD = "jarvis"
MEMORY_FILE = "jarvis_memory.json"
SERVER_IP = "127.0.0.1"  # Server IP für REST/WebSocket
REST_URL = f"http://{SERVER_IP}:5000/command"
WS_URL = f"ws://{SERVER_IP}:5000/socket.io/?EIO=4&transport=websocket"

client = OpenAI(api_key="DEIN_API_KEY")

# ---------------- MEMORY ----------------
try:
    with open(MEMORY_FILE, "r") as f:
        memory = json.load(f)
except:
    memory = {"history":[]}

def remember(text):
    memory["history"].append(text)
    with open(MEMORY_FILE, "w") as f:
        json.dump(memory, f, indent=2)

def get_context():
    return memory["history"][-10:]

# ---------------- FLASK SERVER + WebSocket ----------------
app = Flask(__name__)
socketio = SocketIO(app, cors_allowed_origins="*")

def ai_response(text):
    context = get_context()
    response = client.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {"role":"system","content":"Du bist Jarvis, weiblich, sarkastisch, kompetent."},
            {"role":"system","content":f"Kontext: {context}"},
            {"role":"user","content":text}
        ]
    )
    return response.choices[0].message.content

@app.route("/command", methods=["POST"])
def command():
    data = request.json
    text = data.get("text","")
    remember(text)
    response = ai_response(text)
    return jsonify({"response":response})

@socketio.on("voice_command")
def handle_voice_command(data):
    text = data.get("text","")
    remember(text)
    response = ai_response(text)
    emit("voice_response", {"response":response})

def run_server():
    socketio.run(app, host="0.0.0.0", port=5000)

# ---------------- CLIENT ----------------
engine = pyttsx3.init()
engine.setProperty('rate', 175)
recognizer = sr.Recognizer()

def speak(text):
    # Neural TTS via OpenAI
    try:
        response = client.audio.speech.create(model="gpt-4o-mini-tts",voice="alloy_female",input=text)
        filename="speech.mp3"
        with open(filename,"wb") as f: f.write(response.audio)
        os.system(f"start {filename}") if os.name=="nt" else os.system(f"afplay {filename}")
        os.remove(filename)
    except:
        engine.say(text)
        engine.runAndWait()

def listen():
    with sr.Microphone() as source:
        audio = recognizer.listen(source)
    try: return recognizer.recognize_google(audio,language="de-DE").lower()
    except: return ""

def detect_intent(text):
    if any(w in text for w in ["öffne","starte"]): return "app"
    if any(w in text for w in ["wie spät","uhrzeit"]): return "time"
    if any(w in text for w in ["suche","recherchiere","finde"]): return "ai_search"
    if any(w in text for w in ["zeige bildschirm","was siehst du"]): return "vision"
    if any(w in text for w in ["smarthome","licht","temperatur","geräte"]): return "smarthome"
    if any(w in text for w in ["öffne datei","lese datei","durchsuche"]): return "file"
    return "ai_chat"

# ---------------- APP / FILE / VISION / SMARTHOME ----------------
def handle_app(text):
    if "browser" in text: subprocess.Popen("start chrome",shell=True); return "Browser wird geöffnet."
    if "spotify" in text: subprocess.Popen("start spotify",shell=True); return "Spotify gestartet."
    return "App-Befehl unbekannt."

def handle_file(text):
    try:
        if "lese datei" in text:
            filename=text.split("lese datei")[-1].strip()
            with open(filename,"r",encoding="utf-8") as f: return f.read(500)+"..."
        return "Datei nicht gefunden oder Befehl unklar."
    except Exception as e: return f"Fehler: {e}"

def handle_vision(): 
    pyautogui.screenshot("screen.png"); return "Screenshot erstellt."

def handle_smarthome(text):
    if "licht" in text: return "Lichtsteuerung ausgelöst."
    if "temperatur" in text: return "Thermostat angepasst."
    return "Smarthome-Befehl ausgeführt."

# ---------------- GUI / HUD ----------------
class HUD(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowFlags(Qt.FramelessWindowHint | Qt.WindowStaysOnTopHint | Qt.Tool)
        self.setAttribute(Qt.WA_TranslucentBackground)
        self.setGeometry(50,50,500,250)
        self.label=QLabel("Jarvis Online",self)
        self.label.setStyleSheet("color:#00ff00;font-size:28px;")
        self.label.setGeometry(50,50,400,150)
        self.show()

# ---------------- PROCESS COMMAND ----------------
def process_command(text):
    if WAKE_WORD not in text: return None
    cleaned=text.replace(WAKE_WORD,"").strip()
    if not cleaned: return "Ja?"
    remember(cleaned)
    intent=detect_intent(cleaned)
    if intent=="time": return f"Es ist {datetime.datetime.now().strftime('%H:%M')} Uhr."
    if intent=="app": return handle_app(cleaned)
    if intent=="file": return handle_file(cleaned)
    if intent=="vision": return handle_vision()
    if intent=="smarthome": return handle_smarthome(cleaned)
    if intent in ["ai_search","ai_chat"]: return ai_response(cleaned)

# ---------------- RUN EVERYTHING ----------------
def run_client():
    speak("Jarvis Ultimate ist online und bereit.")
    while True:
        text=listen()
        if not text: continue
        resp=process_command(text)
        if resp: speak(resp)

# ---------------- THREADING SERVER + GUI + CLIENT ----------------
if __name__=="__main__":
    # HUD GUI
    app_qt = QApplication(sys.argv)
    hud = HUD()
    # Server Thread
    t_server = threading.Thread(target=run_server, daemon=True)
    t_server.start()
    # Client Thread
    t_client = threading.Thread(target=run_client, daemon=True)
    t_client.start()
    sys.exit(app_qt.exec_())
