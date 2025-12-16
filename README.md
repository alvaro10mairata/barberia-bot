import os
import json
import datetime
import asyncio
import gspread
from rapidfuzz import fuzz

from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    ConversationHandler,
    ContextTypes,
    filters
)

# =======================
# 📄 Información Barbería 1.0
# =======================
INFO_NEGOCIO = {
    "nombre": "Barbería 1.0",
    "direccion": "Calle Ejemplo 123, Centro Ciudad, España",
    "telefono": "+34 600 123 456",
    "whatsapp": "+34 600 123 456",
    "instagram": "@barberia1.0",
    "facebook": "Barbería 1.0",
    "horarios": {
        "Lunes": ("10:00", "19:00"),
        "Martes": ("10:00", "19:00"),
        "Miércoles": ("10:00", "19:00"),
        "Jueves": ("10:00", "20:00"),
        "Viernes": ("10:00", "20:00"),
        "Sábado": ("9:00", "14:00"),
        "Domingo": None
    },
    "servicios": {
        "Corte clásico de caballero": 30,
        "Corte degradado / Fade": 45,
        "Afeitado con toalla caliente": 30,
        "Arreglo de barba + perfilado": 30,
        "Corte + barba": 60,
        "Tinte o coloración": 60,
        "Cejas y detalles": 15,
        "Paquete Premium": 60
    },
    "precios": {
        "Corte clásico de caballero": 15,
        "Corte degradado / Fade": 18,
        "Afeitado con toalla caliente": 12,
        "Arreglo de barba + perfilado": 10,
        "Corte + barba": 25,
        "Tinte o coloración": 20,
        "Cejas y detalles": 5,
        "Paquete Premium": 35
    },
    "promociones": [
        "2x1 en corte + barba todos los martes 10:00–13:00",
        "10% de descuento a estudiantes con carnet",
        "1 corte gratis cada 10 visitas"
    ]
}

# =======================
# Estados para ConversationHandlers
# =======================
RESERVA_FECHA, RESERVA_SERVICIO, RESERVA_HORA, RESERVA_NOMBRE = range(4)
CANCELAR_FECHA, CANCELAR_HORA, CANCELAR_NOMBRE = range(4,7)

# =======================
# Conexión a Google Sheets
# =======================
credenciales_json = os.environ["GOOGLE_CREDENTIALS"]
credenciales_dict = json.loads(credenciales_json)
gc = gspread.service_account_from_dict(credenciales_dict)
SHEET_ID = os.environ["SHEET_ID"]
SHEET = gc.open_by_key(SHEET_ID).sheet1

# =======================
# Función segura para enviar mensajes
# =======================
async def safe_reply(update, text):
    try:
        await update.message.reply_text(text)
    except Exception as e:
        print(f"Error enviando mensaje a {update.effective_user.id}: {e}")

# =======================
# Funciones auxiliares
# =======================
def es_fecha_valida(fecha_str):
    try:
        fecha = datetime.datetime.strptime(fecha_str, "%d/%m/%Y").date()
        return fecha >= datetime.date.today()
    except ValueError:
        return False

DIA_ES = {
    "Monday": "Lunes",
    "Tuesday": "Martes",
    "Wednesday": "Miércoles",
    "Thursday": "Jueves",
    "Friday": "Viernes",
    "Saturday": "Sábado",
    "Sunday": "Domingo"
}

def es_hora_valida(fecha_str, hora_str, duracion_min=30):
    try:
        fecha = datetime.datetime.strptime(fecha_str, "%d/%m/%Y").date()
        hora = datetime.datetime.strptime(hora_str, "%H:%M").time()
        dia = DIA_ES[fecha.strftime("%A")]

        if dia not in INFO_NEGOCIO["horarios"] or INFO_NEGOCIO["horarios"][dia] is None:
            return False

        inicio_str, fin_str = INFO_NEGOCIO["horarios"][dia]
        inicio = datetime.datetime.strptime(inicio_str, "%H:%M").time()
        fin = datetime.datetime.strptime(fin_str, "%H:%M").time()

        hora_min = hora.hour*60 + hora.minute
        inicio_min = inicio.hour*60 + inicio.minute
        fin_min = fin.hour*60 + fin.minute

        return inicio_min <= hora_min and (hora_min + duracion_min) <= fin_min
    except ValueError:
        return False

def hora_disponible(fecha_str, hora_str, duracion_min):
    reservas = SHEET.get_all_values()[1:]
    hora_inicio = datetime.datetime.strptime(hora_str, "%H:%M")
    hora_fin = hora_inicio + datetime.timedelta(minutes=duracion_min)
    for fila in reservas:
        if len(fila) >= 3:
            res_fecha, res_hora, res_servicio = fila[0], fila[1], fila[2]
            res_duracion = INFO_NEGOCIO["servicios"].get(res_servicio, 30)
            res_inicio = datetime.datetime.strptime(res_hora, "%H:%M")
            res_fin = res_inicio + datetime.timedelta(minutes=res_duracion)
            if fecha_str == res_fecha:
                latest_start = max(hora_inicio, res_inicio)
                earliest_end = min(hora_fin, res_fin)
                if (earliest_end - latest_start).total_seconds() > 0:
                    return False
    return True

def buscar_reserva(fecha, hora, nombre):
    valores = SHEET.get_all_values()
    for i, fila in enumerate(valores):
        if len(fila) >= 4 and fila[0]==fecha and fila[1]==hora and fila[3].lower()==nombre.lower():
            return i+1
    return None

# =======================
# Handlers
# =======================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await safe_reply(update, "👋 ¡Hola! Soy el asistente de Barbería 1.0. Usa /info, /reservar o /cancelar.")

async def info(update: Update, context: ContextTypes.DEFAULT_TYPE):
    texto = f"📍 {INFO_NEGOCIO['direccion']}\n📞 {INFO_NEGOCIO['telefono']}\n🕒 Horarios:\n"
    for dia, horario in INFO_NEGOCIO["horarios"].items():
        if horario: texto += f"{dia}: {horario[0]} – {horario[1]}\n"
        else: texto += f"{dia}: cerrado\n"
    texto += "\n💇 Servicios y precios:\n"
    for s, p in INFO_NEGOCIO["precios"].items():
        texto += f"{s} – {p} € ({INFO_NEGOCIO['servicios'][s]} min)\n"
    texto += "\n🪙 Promociones:\n" + "\n".join(INFO_NEGOCIO["promociones"])
    await safe_reply(update, texto)

# =======================
# Conversación Reservas
# =======================
async def reservar(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await safe_reply(update, "📅 ¿Qué fecha quieres reservar? (dd/mm/yyyy)")
    return RESERVA_FECHA

async def obtener_fecha(update: Update, context: ContextTypes.DEFAULT_TYPE):
    fecha = update.message.text
    if not es_fecha_valida(fecha):
        await safe_reply(update, "❌ Fecha inválida. Intenta de nuevo (dd/mm/yyyy).")
        return RESERVA_FECHA
    context.user_data["fecha"] = fecha
    servicios = "\n".join([f"- {s}" for s in INFO_NEGOCIO["servicios"].keys()])
    await safe_reply(update, f"💇 ¿Qué servicio quieres?\n{servicios}")
    return RESERVA_SERVICIO

async def obtener_servicio(update: Update, context: ContextTypes.DEFAULT_TYPE):
    servicio = update.message.text
    if servicio not in INFO_NEGOCIO["servicios"]:
        await safe_reply(update, "❌ Servicio no válido, escribe exactamente como aparece.")
        return RESERVA_SERVICIO
    context.user_data["servicio"] = servicio
    await safe_reply(update, "⏰ ¿A qué hora? (HH:MM, 24h)")
    return RESERVA_HORA

async def obtener_hora(update: Update, context: ContextTypes.DEFAULT_TYPE):
    hora = update.message.text
    fecha = context.user_data["fecha"]
    servicio = context.user_data["servicio"]
    duracion = INFO_NEGOCIO["servicios"][servicio]

    if not es_hora_valida(fecha, hora, duracion):
        await safe_reply(update, "❌ Hora no válida según horario de la barbería.")
        return RESERVA_HORA

    if not hora_disponible(fecha, hora, duracion):
        await safe_reply(update, "❌ Esa hora ya está ocupada. Elige otra.")
        return RESERVA_HORA

    context.user_data["hora"] = hora
    await safe_reply(update, "📝 Tu nombre completo para la reserva:")
    return RESERVA_NOMBRE

async def obtener_nombre(update: Update, context: ContextTypes.DEFAULT_TYPE):
    nombre = update.message.text
    context.user_data["nombre"] = nombre
    SHEET.append_row([context.user_data["fecha"], context.user_data["hora"],
                      context.user_data["servicio"], nombre])
    await safe_reply(update, f"✅ Reserva confirmada para {nombre} el {context.user_data['fecha']} a las {context.user_data['hora']} ({context.user_data['servicio']}).")
    return ConversationHandler.END

# =======================
# Conversación Cancelaciones
# =======================
async def cancelar(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await safe_reply(update, "📅 ¿Qué fecha quieres cancelar? (dd/mm/yyyy)")
    return CANCELAR_FECHA

async def cancelar_fecha(update: Update, context: ContextTypes.DEFAULT_TYPE):
    fecha = update.message.text
    if not es_fecha_valida(fecha):
        await safe_reply(update, "❌ Fecha inválida. Intenta de nuevo (dd/mm/yyyy).")
        return CANCELAR_FECHA
    context.user_data["fecha"] = fecha
    await safe_reply(update, "⏰ ¿A qué hora es la cita? (HH:MM)")
    return CANCELAR_HORA

async def cancelar_hora(update: Update, context: ContextTypes.DEFAULT_TYPE):
    hora = update.message.text
    context.user_data["hora"] = hora
    await safe_reply(update, "📝 Tu nombre completo para cancelar la cita:")
    return CANCELAR_NOMBRE

async def cancelar_nombre(update: Update, context: ContextTypes.DEFAULT_TYPE):
    nombre = update.message.text
    fila = buscar_reserva(context.user_data["fecha"], context.user_data["hora"], nombre)
    if fila:
        SHEET.delete_rows(fila)
        await safe_reply(update, f"✅ Cita de {nombre} el {context.user_data['fecha']} a las {context.user_data['hora']} cancelada.")
    else:
        await safe_reply(update, "❌ No encontramos esa cita.")
    return ConversationHandler.END

# =======================
# Respuesta inteligente
# =======================
async def responder_mensaje_inteligente(update: Update, context: ContextTypes.DEFAULT_TYPE):
    texto = update.message.text.lower()
    await update.message.chat.send_action(action="typing")
    await asyncio.sleep(1)

    acciones = {
        "horarios": ["horario", "abierto", "abrir", "hora", "cuando abren", "horas"],
        "servicios": ["servicio", "corte", "barba", "fade", "tinte", "arreglo"],
        "promociones": ["promoción", "descuento", "oferta", "promos", "rebaja"],
        "reservar": ["reservar", "cita", "quiero cita", "agendar", "hacer reserva"],
        "cancelar": ["cancelar", "eliminar cita", "quitar cita", "anular"]
    }

    acciones_detectadas = set()
    for accion, palabras in acciones.items():
        for palabra in palabras:
            if palabra in texto or fuzz.partial_ratio(texto, palabra) > 70:
                acciones_detectadas.add(accion)

    if "horarios" in acciones_detectadas:
        msg = "🕒 Horarios:\n"
        for dia, horario in INFO_NEGOCIO["horarios"].items():
            msg += f"{dia}: {' – '.join(horario) if horario else 'cerrado'}\n"
        await safe_reply(update, msg)

    if "servicios" in acciones_detectadas:
        msg = "💇 Servicios disponibles:\n"
        for s, p in INFO_NEGOCIO["precios"].items():
            msg += f"{s} – {p} € ({INFO_NEGOCIO['servicios'][s]} min)\n"
        await safe_reply(update, msg)

    if "promociones" in acciones_detectadas:
        await safe_reply(update, "🪙 Promociones actuales:\n" + "\n".join(INFO_NEGOCIO["promociones"]))

    if "reservar" in acciones_detectadas:
        await reservar(update, context)

    if "cancelar" in acciones_detectadas:
        await cancelar(update, context)

    if not acciones_detectadas:
        await safe_reply(update, "🤖 No entendí tu mensaje completamente. Usa /info /reservar /cancelar")

# =======================
# Main
# =======================
if __name__ == "__main__":
    TOKEN = os.environ["TELEGRAM_TOKEN"]
    app = ApplicationBuilder().token(TOKEN).build()

    conv_reserva = ConversationHandler(
        entry_points=[CommandHandler("reservar", reservar)],
        states={
            RESERVA_FECHA: [MessageHandler(filters.TEXT & ~filters.COMMAND, obtener_fecha)],
            RESERVA_SERVICIO: [MessageHandler(filters.TEXT & ~filters.COMMAND, obtener_servicio)],
            RESERVA_HORA: [MessageHandler(filters.TEXT & ~filters.COMMAND, obtener_hora)],
            RESERVA_NOMBRE: [MessageHandler(filters.TEXT & ~filters.COMMAND, obtener_nombre)],
        },
        fallbacks=[]
    )

    conv_cancelar = ConversationHandler(
        entry_points=[CommandHandler("cancelar", cancelar)],
        states={
            CANCELAR_FECHA: [MessageHandler(filters.TEXT & ~filters.COMMAND, cancelar_fecha)],
            CANCELAR_HORA: [MessageHandler(filters.TEXT & ~filters.COMMAND, cancelar_hora)],
            CANCELAR_NOMBRE: [MessageHandler(filters.TEXT & ~filters.COMMAND, cancelar_nombre)],
        },
        fallbacks=[]
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("info", info))
    app.add_handler(conv_reserva)
    app.add_handler(conv_cancelar)
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, responder_mensaje_inteligente))

    print("🤖 Bot Barbería 1.0 corriendo 24/7 en Railway")
    app.run_polling()
