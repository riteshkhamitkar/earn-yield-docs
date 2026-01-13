# Chatbot API Documentation

## Overview
This API handles the AI feedback chatbot flow. Users answer questions about feature preferences, and responses are stored in the database.

**Base URL:** `/api/v1/ai/chat`  
**Authentication:** Required (Bearer token in Authorization header)

---

## Endpoint

### `POST /ai/chat`

Handles chatbot interactions - starting a new chat, answering questions, and resuming incomplete surveys.

**Headers:**
```
Authorization: Bearer <your-jwt-token>
Content-Type: application/json
```

---

## Request Format

### Start New Chat / Resume
```json
{
  "sessionId": null,
  "message": null
}
```

### Answer a Question (Select Option)
```json
{
  "sessionId": "any-string-or-null",
  "message": {
    "type": "option",
    "questionId": "q1_card_benefits",
    "optionId": "higher_cashback"
  }
}
```

### Submit Text Feedback (Final Question)
```json
{
  "sessionId": "any-string-or-null",
  "message": {
    "type": "text",
    "text": "I'd love to see more DeFi integrations!"
  }
}
```

**Note:** `sessionId` is optional - you can send `null` or any value. The backend uses `userId` from the auth token to track progress.

---

## Response Format

```json
{
  "sessionId": "uuid-string",
  "status": "active" | "completed",
  "messages": [
    {
      "id": "m1",
      "role": "assistant",
      "type": "text",
      "text": "Question text here..."
    },
    {
      "id": "m2",
      "role": "assistant",
      "type": "options",
      "questionId": "q1_card_benefits",
      "options": [
        { "id": "higher_cashback", "label": "Higher Cashback (2-5%)..." },
        { "id": "zero_fx_fees", "label": "0% Foreign Exchange Fees..." }
      ]
    }
  ]
}
```

---

## Message Types

### Text Message
```json
{
  "id": "m_intro",
  "role": "assistant",
  "type": "text",
  "text": "Welcome message..."
}
```

### Options Message
```json
{
  "id": "m_q1_options",
  "role": "assistant",
  "type": "options",
  "questionId": "q1_card_benefits",
  "options": [
    { "id": "option1", "label": "Option 1 label" },
    { "id": "option2", "label": "Option 2 label" }
  ]
}
```

**Important:** Use the `questionId` from the options message in your next request.

---

## Question Flow

The chatbot asks questions in this order:

1. **Q1: Crypto Card** (`q1_card_benefits`)
   - Follow-ups: `q1_spend_volume` → `q1_card_fee`
   - Skip: Select "not_interested" to skip follow-ups

2. **Q2: Prediction Markets** (`q2_platform`)
   - Follow-up: `q2_market_types`
   - Skip: Select "not_interested" to skip follow-up

3. **Q3: On/Off-Ramp** (`q3_regions`)
   - Follow-ups: `q3_payment_methods` → `q3_speed_cost`

4. **Q4: Perpetuals** (`q4_tokens`)
   - Follow-ups: `q4_platform` → `q4_leverage`
   - Skip: Select "not_interested" in platform to skip leverage

5. **Q5: Networks/Assets** (`q5_networks`)
   - Follow-ups: `q5_asset_types` → `q5_use_cases`

6. **Q6: Savings** (`q6_products`)
   - Follow-ups: `q6_risk_tolerance` → `q6_protocols` → `q6_withdrawal`

7. **Closing: Ranking** (`closing_ranking`)
   - User must select 3 features
   - Backend will re-prompt if less than 3 selected

8. **Closing: Feedback** (`closing_feedback`)
   - Optional text input
   - Survey completes after this

---

## Usage Flow

### Step 1: Start Chat
```javascript
POST /ai/chat
Body: { "sessionId": null, "message": null }
```

**Response:** Returns intro message + first question with options

### Step 2: User Selects Option
```javascript
POST /ai/chat
Body: {
  "sessionId": null, // or previous sessionId
  "message": {
    "type": "option",
    "questionId": "q1_card_benefits", // from previous response
    "optionId": "higher_cashback" // user's selection
  }
}
```

**Response:** Returns next question(s) or follow-up

### Step 3: Continue Through Questions
Repeat Step 2, using the `questionId` from each options message.

### Step 4: Final Text Feedback (Optional)
```javascript
POST /ai/chat
Body: {
  "sessionId": null,
  "message": {
    "type": "text",
    "text": "User's feedback text"
  }
}
```

**Response:** `status: "completed"` with thank you messages

---

## Resume Behavior

If a user starts a new chat (sends `sessionId: null, message: null`):

- **Completed Survey:** Returns "You've already completed the survey" message
- **Incomplete Survey:** Returns the next unanswered question
- **New User:** Returns intro + first question

The backend automatically tracks progress using the authenticated `userId`.

---

## Status Values

- `"active"` - Survey in progress, more questions to answer
- `"completed"` - Survey finished (thank you message shown)

---

## Example: Complete Flow

### 1. Start
```json
Request: { "sessionId": null, "message": null }
Response: {
  "status": "active",
  "messages": [
    { "type": "text", "text": "Thanks for being part of Avvio!..." },
    { "type": "options", "questionId": "q1_card_benefits", "options": [...] }
  ]
}
```

### 2. Answer Q1
```json
Request: {
  "message": {
    "type": "option",
    "questionId": "q1_card_benefits",
    "optionId": "higher_cashback"
  }
}
Response: {
  "status": "active",
  "messages": [
    { "type": "text", "text": "How much do you typically spend..." },
    { "type": "options", "questionId": "q1_spend_volume", "options": [...] }
  ]
}
```

### 3. Continue...
Keep sending option selections until you reach the closing questions.

### 4. Ranking (Select 3 times)
```json
Request: {
  "message": {
    "type": "option",
    "questionId": "closing_ranking",
    "optionId": "crypto_card"
  }
}
// Backend will ask for 2 more selections if less than 3
```

### 5. Final Feedback
```json
Request: {
  "message": {
    "type": "text",
    "text": "Great features!"
  }
}
Response: {
  "status": "completed",
  "messages": [
    { "type": "text", "text": "Thank you for your feedback!..." }
  ]
}
```

---

## Error Responses

### 401 Unauthorized
```json
{
  "statusCode": 401,
  "message": "Invalid token"
}
```
**Fix:** Ensure valid Bearer token in Authorization header

### 400 Bad Request
```json
{
  "statusCode": 400,
  "message": ["message.type must be a string"]
}
```
**Fix:** Check request body matches the format above

### 429 Too Many Requests
```json
{
  "statusCode": 429,
  "message": "ThrottlerException: Too Many Requests"
}
```
**Fix:** Rate limit is 30 requests/minute. Wait 60 seconds.

---

## Important Notes

1. **sessionId is Optional:** You can always send `null` - backend tracks by `userId` from auth token
2. **questionId is Required:** Always use the `questionId` from the options message in your next request
3. **Upsert Behavior:** If a user answers the same question twice, the backend updates the previous answer
4. **Skip Logic:** Some questions have "not_interested" options that skip follow-up questions
5. **Ranking:** Must select exactly 3 features - backend validates and re-prompts if needed
6. **Text Feedback:** Only the final `closing_feedback` question accepts text input

---

## Quick Reference

| Action | Request Body |
|--------|-------------|
| Start/Resume | `{ "sessionId": null, "message": null }` |
| Select Option | `{ "message": { "type": "option", "questionId": "...", "optionId": "..." } }` |
| Submit Text | `{ "message": { "type": "text", "text": "..." } }` |

---

## Frontend Implementation Tips

1. **Store questionId:** When you receive an options message, save the `questionId` for the next request
2. **Handle Status:** Check `status: "completed"` to show completion UI
3. **Resume Logic:** You can always send `sessionId: null, message: null` to resume - backend handles it
4. **Error Handling:** Show user-friendly messages for 401, 400, 429 errors
5. **Loading States:** Show loading while waiting for API response

---

## Support

For questions or issues, contact the backend team.
