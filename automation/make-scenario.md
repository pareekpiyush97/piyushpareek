# WhatsApp AI Appointment Booking — Make.com Scenario Spec

Website: DONE. All `wa.me/918949498980` CTAs in `index.html` already open WhatsApp. No site changes needed — the entire flow below runs in Make.com.

## MISSING DEPENDENCIES — flow cannot run until these exist
1. **WhatsApp Business Cloud API** (developers.facebook.com → Meta app → WhatsApp): Phone Number ID + permanent access token for +91 89494 98980. A plain personal WhatsApp number CANNOT deliver incoming messages to Make. **This is the blocker.**
2. Make.com account, Core plan+ (Data Stores required).
3. Zoho CRM connection authorized inside Make.
4. Google Calendar connection authorized inside Make.
5. Gemini API key (aistudio.google.com). Store it in the Make connection/keychain — never inline in a module.

## Data Store
Name: `wa_sessions` · Data structure: `phone` (text, key), `history` (text).

## Scenario modules, in order

**1. WhatsApp Business Cloud → Watch Events** (instant trigger)
Fires on every incoming message. Sender = `{{1.from}}`, text = `{{1.text.body}}`.

**2. Data Store → Get a Record**
Key: `{{1.from}}`. Missing record = new conversation (empty history).

**3. HTTP → Make a request** (Gemini)
- URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- Method: POST · Header: `x-goog-api-key: <key from keychain>` · `Content-Type: application/json`
- Body:
```json
{
  "system_instruction": {"parts": [{"text": "<paste automation/gemini-system-prompt.txt, with {{today}} = {{formatDate(now; 'YYYY-MM-DD')}}, {{history}} = {{2.history}}, {{message}} = {{1.text.body}}>"}]},
  "contents": [{"role": "user", "parts": [{"text": "{{1.text.body}}"}]}]
}
```
- Parse response: yes. AI reply = `{{3.data.candidates[1].content.parts[1].text}}` (call it REPLY below).

**4. Router — two branches**

### Branch A: conversation continues
Filter: REPLY **does not contain** `"booking_complete":true`
- **A1. WhatsApp → Send a Message**: to `{{1.from}}`, body = REPLY.
- **A2. Data Store → Add/Replace a Record**: key `{{1.from}}`, history = `{{2.history}}` + newline + `User: {{1.text.body}}` + newline + `Assistant: REPLY`.

### Branch B: booking complete
Filter: REPLY **contains** `"booking_complete":true`
- **B1. JSON → Parse JSON**: input REPLY. Structure: booking_complete, name, phone, property_type, city, budget, preferred_date, preferred_time.
- **B2. Zoho CRM → Create a Record** (module: Leads)
  - Last Name: `{{B1.name}}` · Phone: `{{B1.phone}}` · City: `{{B1.city}}`
  - Lead Source: `WhatsApp AI Assistant`
  - Description: `Type: {{B1.property_type}} | Budget: {{B1.budget}} | Appointment: {{B1.preferred_date}} {{B1.preferred_time}}`
- **B3. Google Calendar → Create an Event**
  - Calendar: primary · Event name: `Consultation — {{B1.name}} ({{B1.property_type}}, {{B1.city}})`
  - Start: `{{parseDate(B1.preferred_date + " " + B1.preferred_time; "YYYY-MM-DD HH:mm"; "Asia/Kolkata")}}` · Duration: 30 min
  - Description: `Phone: {{B1.phone}} | Budget: {{B1.budget}} | Booked via WhatsApp AI`
- **B4. WhatsApp → Send a Message**: to `{{1.from}}`, body:
  `✅ Confirmed, {{B1.name}}! Your consultation is booked for {{B1.preferred_date}} at {{B1.preferred_time}}. You'll get a reminder before the call. — Piyush Pareek`
- **B5. Data Store → Delete a Record**: key `{{1.from}}` (resets the conversation).

## Error handling
- On B2/B3 failure: add a Break error handler (retry ×3, 5-min interval) so a Zoho/Calendar hiccup doesn't lose the booking.
- Set scenario sequential processing ON so two rapid messages from one user don't race the data store.

## Self-check (run once after wiring)
Send "Hi" from a test number → assistant asks for name → answer all 7 → reply "yes" → verify: Zoho lead exists, Calendar event at right IST time, WhatsApp confirmation received, data store record deleted.
