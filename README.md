# TailorTalk
from fastapi import FastAPI, Request
from app.langgraph_agent import agent_response

app = FastAPI()

@app.post("/chat")
async def chat_endpoint(request: Request):
    data = await request.json()
    user_input = data.get("message", "")
    response = agent_response(user_input)
    return {"response": response}

# File: app/langgraph_agent.py

from app.utils import parse_intent, extract_time, format_response
from app.calendar import check_availability, book_slot

def agent_response(message):
    # Step 1: Detect user's intent and extract time-related info
    intent, time_info = parse_intent(message)

    # Step 2: Handle the intent
    if intent == "check_availability":
        slots = check_availability(time_info)
        return format_response(slots)

    elif intent == "book_appointment":
        result = book_slot(time_info)
        return result

    # Step 3: Fallback for unclear input
    return "Sorry, I didn’t understand that. Could you rephrase?"


from datetime import datetime, timedelta
from google.oauth2 import service_account
from googleapiclient.discovery import build

# Define scopes and credentials
SCOPES = ['https://www.googleapis.com/auth/calendar']
SERVICE_ACCOUNT_FILE = 'credentials.json'  # Make sure this file exists in the root

# Load credentials and initialize the Calendar API client
credentials = service_account.Credentials.from_service_account_file(
    SERVICE_ACCOUNT_FILE, scopes=SCOPES
)
calendar_service = build('calendar', 'v3', credentials=credentials)

# The calendar ID – 'primary' for default calendar
CALENDAR_ID = 'primary'

def check_availability(time_window):
    """
    Checks the next 7 days for existing events and returns start times.
    """
    now = datetime.utcnow().isoformat() + 'Z'
    later = (datetime.utcnow() + timedelta(days=7)).isoformat() + 'Z'

    events_result = calendar_service.events().list(
        calendarId=CALENDAR_ID,
        timeMin=now,
        timeMax=later,
        singleEvents=True,
        orderBy='startTime'
    ).execute()

    events = events_result.get('items', [])
    return [event['start']['dateTime'] for event in events]

def book_slot(time_info):
    """
    Books a new event with the given start and end datetime strings.
    """
    event = {
        'summary': 'Scheduled Appointment',
        'start': {
            'dateTime': time_info['start'],
            'timeZone': 'Asia/Kolkata',
        },
        'end': {
            'dateTime': time_info['end'],
            'timeZone': 'Asia/Kolkata',
        },
    }

    created_event = calendar_service.events().insert(
        calendarId=CALENDAR_ID, body=event
    ).execute()

    return f"✅ Booking confirmed at {created_event['start']['dateTime']}"

# File: app/utils.py

from datetime import datetime, timedelta

def parse_intent(message):
    """
    Determines user intent and extracts time window.
    Returns: (intent_type, time_info_dict)
    """
    message = message.lower()

    if "book" in message or "schedule" in message:
        return "book_appointment", extract_time(message)
    elif "available" in message or "free" in message or "slots" in message:
        return "check_availability", extract_time(message)
    
    return "unknown", {}

def extract_time(message):
    """
    Parses time phrases from message and returns start/end ISO strings.
    This version uses simple keywords, but can be extended with NLP.
    """
    now = datetime.utcnow()

    if "tomorrow" in message:
        start = (now + timedelta(days=1)).replace(hour=14, minute=0, second=0, microsecond=0)
        end = start + timedelta(hours=1)
    elif "next week" in message:
        start = (now + timedelta(days=7)).replace(hour=15, minute=0, second=0, microsecond=0)
        end = start + timedelta(hours=1)
    elif "afternoon" in message:
        start = now.replace(hour=14, minute=0, second=0, microsecond=0)
        end = start + timedelta(hours=1)
    elif "morning" in message:
        start = now.replace(hour=10, minute=0, second=0, microsecond=0)
        end = start + timedelta(hours=1)
    else:
        # Default to one hour from now
        start = now + timedelta(hours=1)
        end = start + timedelta(hours=1)

    return {
        "start": start.isoformat(),
        "end": end.isoformat()
    }

def format_response(slots):
    """
    Takes a list of booked slots and returns a friendly string.
    """
    if not slots:
        return "✅ You have no events scheduled in the next 7 days."
    
    return f"📅 Existing events:\n" + "\n".join(f"• {slot}" for slot in slots[:5])

import streamlit as st
import requests

# Streamlit UI title
st.title("🧵 TailorTalk – AI Appointment Agent")

# Store messages in session
if 'messages' not in st.session_state:
    st.session_state.messages = []

# Text input field
user_input = st.text_input("Type your message to book or check availability:", key="input")

# On send
if st.button("Send") and user_input.strip():
    # Show user message
    st.session_state.messages.append(("User", user_input))

    # Send to FastAPI backend
    try:
        response = requests.post("http://localhost:8000/chat", json={"message": user_input})
        reply = response.json().get("response", "Error: No response from agent.")
    except Exception as e:
        reply = f"🚫 Error contacting backend: {e}"

    # Show agent reply
    st.session_state.messages.append(("Agent", reply))

# Display conversation history
st.markdown("---")
for role, message in st.session_state.messages:
    st.markdown(f"**{role}:** {message}")

