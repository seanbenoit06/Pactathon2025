# Seattle Messenger Response Service

**Technical Architecture Document**

---

## Overview

An intelligent auto-response system that powers the City of Seattle's Facebook Messenger page. When citizens DM @CityofSeattle, this service automatically responds with helpful information about service requests, city services, and routes complex inquiries to human agents.

**Goal:** Provide instant, accurate, 24/7 customer service through Facebook Messenger.

---

## Architecture

```
User DMs @CityofSeattle
         ↓
Facebook Messenger Platform
         ↓
POST /webhook (your server)
         ↓
┌─────────────────────────┐
│   Message Processor     │
│  1. Classify intent     │ ← OpenAI GPT-4
│  2. Query data          │ ← Seattle Open Data API
│  3. Format response     │
└─────────────────────────┘
         ↓
Send reply via Meta Graph API
         ↓
User receives response from @CityofSeattle
```

---

## Core Components

### 1. Webhook Handler
Receives messages from Facebook, verifies security signatures, and routes to the message processor.

### 2. Intent Classifier
Uses OpenAI GPT-4 to determine user intent:
- `STATUS_CHECK` - "What's the status of request 25-00105756?"
- `REPORT_ISSUE` - "There's a pothole on 5th Ave"
- `GENERAL_INQUIRY` - "When is garbage day?"
- `ESCALATE` - "I need to speak to someone"
- `GREETING` - "Hi" / "Hello"

### 3. Data Service
Queries Seattle Open Data APIs:
- **Dataset:** Customer Service Requests (`5ngg-rpne`)
- **Dataset:** Request Tracking Data (`43nw-pkdq`)
- **Base URL:** `https://data.seattle.gov/resource/`

### 4. Response Generator
Creates formatted replies with text, quick reply buttons, web links, and status information.

### 5. Context Manager
Tracks multi-turn conversations, remembers previous messages, and collects information step-by-step.

---

## Data Flow

```javascript
async function handleMessage(userId, messageText) {
  // 1. Classify intent using OpenAI
  const intent = await classifyIntent(messageText);
  
  // 2. Process based on intent
  if (intent === 'STATUS_CHECK') {
    const data = await seattleAPI.lookupRequest(requestNumber);
    response = formatStatusResponse(data);
  } 
  else if (intent === 'REPORT_ISSUE') {
    response = guideUserThroughReporting(issueType);
  }
  else if (intent === 'ESCALATE') {
    response = provideHumanContactInfo();
  }
  
  // 3. Send response back via Messenger
  await messengerAPI.sendMessage(userId, response);
}
```

---

## Key Integrations

### OpenAI API
```javascript
// Classify user intent
{
  "model": "gpt-4",
  "messages": [
    {"role": "system", "content": "You classify Seattle customer service intents..."},
    {"role": "user", "content": "What's the status of 25-00105756?"}
  ],
  "response_format": {"type": "json_object"}
}

// Returns:
{
  "intent": "STATUS_CHECK",
  "entities": {"request_number": "25-00105756"},
  "confidence": 0.95
}
```

### Seattle Open Data API
```javascript
// Look up service request
GET https://data.seattle.gov/resource/5ngg-rpne.json
    ?service_request_number=25-00105756

// Returns:
{
  "service_request_number": "25-00105756",
  "service_request_type": "Abandoned Vehicle",
  "status": "In Progress",
  "location": "1204 MINOR AVE",
  "city_department": "Seattle Police Department",
  "created_date": "2025-04-14"
}
```

### Meta Graph API
```javascript
// Send message back to user
POST https://graph.facebook.com/v18.0/me/messages
{
  "recipient": {"id": "USER_ID"},
  "message": {
    "text": "I found your request! Status: In Progress...",
    "quick_replies": [
      {"title": "Got it", "payload": "ACKNOWLEDGE"}
    ]
  }
}
```

---

## Example Conversations

### Conversation 1: Status Check

```
User: What's the status of 25-00105756?

Bot: I found your request! 🟡

📋 Abandoned Vehicle
📍 1204 Minor Ave, Seattle
📅 Reported: Apr 14, 2025
⏱️ Status: In Progress
🏛️ Seattle Police Department

Latest: Parking enforcement officers have located 
and marked the vehicle. If still in violation after 
72 hours, an impound will be ordered.

Track: https://seattle.gov/csr/25-00105756

[Got it, thanks] [Speak to someone]
```

### Conversation 2: Report Issue (Multi-turn)

```
User: There's a huge pothole on 5th Ave

Bot: I can help you report that pothole!

📍 Where exactly is it located?

Please provide:
• Street address, OR
• Cross streets, OR  
• Nearby landmark

Example: "5th & Pine" or "1234 5th Ave"

---

User: 5th and Pine

Bot: Thanks! 📍 5th Ave & Pine St

📝 Can you describe what you're seeing?

Be specific - this helps crews address it quickly.

---

User: It's about 2 feet wide and pretty deep

Bot: Perfect! I have everything needed.

📍 Location: 5th Ave & Pine St
📝 Description: 2 feet wide, deep pothole

Ready to submit:
[Submit Request] [Start Over]
```

### Conversation 3: General Inquiry

```
User: When is my garbage day?

Bot: 🗑️ Garbage & Recycling Collection

Your collection day depends on your address.

To find your schedule:
1. Visit seattle.gov/utilities
2. Enter your address
3. View your personalized schedule

Missed pickup? Report it within 24 hours at the same link.

[Find My Schedule]
```

### Conversation 4: Escalation

```
User: This is ridiculous, I need to speak to a manager NOW

Bot: I understand you need additional assistance. 
Let me connect you with the right team.

📞 Customer Service Bureau
Call: 206-684-CITY (2489)
Hours: Mon-Fri, 8am-5pm

🌐 Online Support
Visit: seattle.gov/customer-service-bureau

📧 Email
customer.service@seattle.gov

A representative will follow up within 1 business day.
[Ticket #CSR-1699564800 created]

[Thanks]
```

### Conversation 5: Request Not Found

```
User: Status of 99-99999999

Bot: I couldn't find that request number. 
Please check the number and try again.

Request numbers are in this format: 25-00105756

You can also find your request number in the 
confirmation email you received.

[Try Again] [Report New Issue] [Get Help]
```

---

## System Components Detail

### Intent Classification with OpenAI

The system uses OpenAI's GPT-4 with a specialized system prompt to classify incoming messages into actionable intents and extract relevant entities (request numbers, locations, issue types).

**Key aspects:**
- JSON mode ensures structured output
- Conversation history maintained for context
- Confidence scores help determine escalation
- Entity extraction (request numbers, locations, issue types)

### Context Management

Tracks conversation state across multiple messages:
- Stores user ID and conversation history
- Maintains collected data (location, description, etc.)
- Tracks current intent and conversation state
- Cleans up stale conversations after 30 minutes

**Context structure:**
```javascript
{
  userId: "1234567890",
  currentIntent: "REPORT_ISSUE",
  collectedData: {
    issue_type: "pothole",
    location: "5th Ave and Pine St",
    description: "2 feet wide, deep"
  },
  conversationHistory: [...],
  lastInteraction: timestamp,
  state: "collecting_info"
}
```

### Data Integration

Connects to two primary Seattle Open Data datasets:

1. **Customer Service Requests** - Main request data
2. **Request Tracking Data** - Detailed status updates

Supports queries by:
- Request number (exact match)
- Location (geospatial search)
- Request type
- Date range

### Response Generation

Creates contextually appropriate responses based on:
- User intent
- Available data
- Conversation history
- Missing information needed

Formats include:
- Plain text messages
- Quick reply buttons (up to 13)
- Button templates with URLs
- Generic templates with structured data

---

## Tech Stack

**Backend:**
- Node.js + Express
- OpenAI SDK
- Axios for HTTP requests

**APIs:**
- OpenAI GPT-4 API
- Meta Graph API v18.0
- Seattle Open Data (Socrata)

**Infrastructure:**
- Hosting: Heroku, AWS Lambda, Google Cloud Run
- Context Storage: Redis (production)
- Logging: Winston
- Monitoring: Sentry

---

## Project Structure

```
seattle-messenger-bot/
├── server.js                    # Main entry point
├── .env                         # Environment variables
├── package.json
│
├── handlers/
│   └── message.handler.js       # Core processing logic
│
├── services/
│   ├── openai.service.js        # Intent classification
│   ├── seattle.service.js       # Data queries
│   └── messenger.service.js     # Send messages
│
└── generators/
    └── response.generator.js    # Format responses
```

---

## Environment Variables

```bash
FACEBOOK_PAGE_ACCESS_TOKEN=    # From Facebook Developer Console
FACEBOOK_VERIFY_TOKEN=         # Custom token for webhook verification
FACEBOOK_APP_SECRET=           # For signature verification
OPENAI_API_KEY=                # OpenAI API key
PORT=3000                      # Server port
```

---

## Installation

```bash
# Install dependencies
npm install express body-parser axios openai dotenv

# Run server
npm start

# Expose for testing (separate terminal)
ngrok http 3000
```

---

## Demo Strategy

### Live Demo Setup
1. Open Facebook Messenger (phone or web)
2. Show @CityofSeattle page
3. Split screen: Messenger conversation + Server logs

### Demo Scenarios

**1. Status Check (Success)**
- Type: "Check status of 25-00105756"
- Highlight: Real data from Seattle API, instant response

**2. Status Check (Not Found)**
- Type: "Status of 99-99999999"
- Highlight: Helpful error handling

**3. Report Issue (Multi-turn)**
- Type: "There's graffiti at 3rd and Pine"
- Highlight: Conversation flow, data collection

**4. Escalation**
- Type: "I need to speak to someone NOW"
- Highlight: Human handoff with ticket

**5. FAQ**
- Type: "When is my garbage day?"
- Highlight: Quick informational response

### Key Demo Points

✅ **Instant responses** - No waiting for office hours  
✅ **Real data** - Connected to actual Seattle systems  
✅ **Natural language** - Understands variations and typos  
✅ **Smart escalation** - Knows when humans are needed  
✅ **Accessible** - Works on any device, no app required  
✅ **Conversational** - Multi-turn interactions feel natural

---

## Hackathon Alignment

### Challenge: Customer Service Chatbot ⭐
Primary match - This is a production-ready chatbot for Seattle's customer service

### Challenge: Better Status Tracker
Provides real-time status lookups via conversational interface

### Challenge: No-UI Dashboard
Conversation-based access to data, screen reader friendly

### Challenge: Improving Customer Service
Reduces response time from days to seconds, available 24/7

---

## Success Metrics

- **Response Rate:** % of messages successfully answered
- **Resolution Rate:** % resolved without human escalation
- **Response Time:** Average time to reply (target: <3 seconds)
- **User Satisfaction:** Conversation ratings
- **Top Intents:** Most common user requests
- **Escalation Rate:** % requiring human intervention

---

## Security

**Critical Requirements:**
- Verify webhook signatures (prevent spoofing)
- Use HTTPS only (required by Meta)
- Never log tokens or secrets
- Validate all user inputs
- Rate limit to prevent abuse

**Signature Verification:**
```javascript
const crypto = require('crypto');

function verifySignature(signature, body) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', process.env.FACEBOOK_APP_SECRET)
    .update(body)
    .digest('hex');
  
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}
```

---

## Future Enhancements

1. **Proactive Notifications** - Alert users when requests are updated
2. **Photo Support** - Accept images for issue reports
3. **Location Sharing** - Use GPS coordinates for easier reporting
4. **Multi-language** - Spanish, Vietnamese, Chinese support
5. **Voice Messages** - Transcribe and process audio
6. **Analytics Dashboard** - Track usage patterns and trends
7. **Sentiment Analysis** - Detect frustrated users for priority escalation

---

## Key Advantages

**For Citizens:**
- No app to download
- 24/7 availability
- Instant responses
- Familiar interface (Messenger)
- Works on any device

**For the City:**
- Reduces call center volume
- Handles routine inquiries automatically
- Provides consistent information
- Tracks all interactions
- Easy to update and maintain

**For Equity:**
- Free to use (no SMS charges)
- Accessible to users with disabilities
- Works on older phones
- Available in multiple languages (future)
- Low barrier to entry

---

**Version:** 1.0.0  
**Last Updated:** November 2025
