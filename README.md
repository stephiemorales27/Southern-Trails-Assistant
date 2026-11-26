# Southern Trails Tours: AI Virtual Assistant (RAG with n8n and Gemini)

An AI messaging assistant for a **fictional** tourism agency in Dunedin, New Zealand. It answers questions about prices, schedules and policies instantly, using only the information in its own knowledge base, and hands the conversation to a person when it doesn't know, without inventing data.

> **Practice project.** Fictional business, Meta test number, in-memory knowledge base and Google's free tier.

## Problem

Small businesses lose time and sales answering the same questions over and over. A generic chatbot can make up answers, which is worse than not answering.

## What it does

- Answers in the customer's language (Spanish or English).
- Queries a knowledge base (RAG) and answers **only** with that information.
- Classifies the customer's intent (greeting, tours, prices, schedules, location, cancellation, other).
- Hands off to a person when the information is not in the base: it gives the business hours and asks for the customer's name and query.
- Greets the customer with the main options when they say hello.

## Frequent questions it covers

1. Available tours and prices.
2. Departure times.
3. Meeting point.
4. Cancellation policy.
5. Price for children.

Anything outside the base (discounts, private tours, prices for other destinations) is handed off to a person.

## Architecture

```
Customer → Meta messaging API → Webhook (n8n)
                                     │
                             Edit Fields + If (message has text)
                                     │
                                  AI Agent ── Google Gemini (chat model)
                                     │  ├── base_tours (Simple Vector Store, RAG)
                                     │  │       └── Google Gemini embeddings (gemini-embedding-001)
                                     │  └── Structured Output Parser (JSON)
                                     │
                              Send message (Meta API) → Customer
```

- **Orchestration:** n8n.
- **Chat model:** Google Gemini.
- **Embeddings:** `models/gemini-embedding-001`.
- **Knowledge base:** 9 short chunks (one fact per chunk), loaded into a Simple Vector Store.
- **Structured output:** `respuesta` (reply), `intencion` (intent), `derivar` (hand off), `motivo_derivacion` (handoff reason).

## Knowledge base

The chunks are in [`knowledge-base.md`](knowledge-base.md): business hours, meeting point, guide languages, three tours (Heritage Walk, Coastal Viewpoints, Museum & Art Half-Day), cancellation, children's policy and other destinations on request.

The base was split by hand into one-fact chunks because it is small: each chunk answers a complete question on its own.

## Tests

Six tests, all correct (see [`images/03-test-results-table.png`](images/03-test-results-table.png)):

| # | Question | Result |
|---|----------|--------|
| 1 | Hola | Welcome message with options, no handoff |
| 2 | ¿Tienen tours a las Islas Cook? (Do you have tours to the Cook Islands?) | Says they are organized on request, gives business hours, asks for name and query, hands off |
| 3 | ¿Hacen tours a Milford Sound? ¿Y tienen descuento para estudiantes? (Do you run tours to Milford Sound? Any student discount?) | The team confirms the destination; no discount data; hands off without inventing |
| 4 | ¿Cuánto cuesta el tour Coastal Viewpoints y a qué hora sale? (How much is the Coastal Viewpoints tour and when does it leave?) | NZD 95 per person, departs at 9:00 |
| 5 | What time does the Museum & Art tour start and how much is it? | Answers in English: 1:00 PM, NZD 60, entrance fees included |
| 6 | How much is the Heritage Walk for a 8-year-old child? | Answers in English: NZD 22.50 (half of NZD 45) |

During development, 2 failures were fixed by adjusting the agent's instructions: it claimed a destination was not offered when it was simply missing from the base, and it set a handoff incorrectly. The assistant was re-tested after every change.

## Limitations

- **In-memory base:** the Simple Vector Store loses its data if n8n restarts, so the load block must be run again. In production I would use a persistent vector database.
- **Real incoming messages:** the Meta API only delivers them to production when the app is published. With the app in development mode, tests were run from the n8n chat and the replies were sent to a phone through the API.
- **Free tier:** the free Google key has a daily request limit.
- **No bookings:** the assistant does not manage reservations; it hands off to a person.

## How to use it

1. Import [`workflow.json`](workflow.json) into n8n.
2. Create your own credentials (Google Gemini and Meta messaging API). The file **does not include** credentials.
3. In Meta, configure the webhook with your n8n production URL and your own verify token.
4. Run the load block ("When clicking 'Execute workflow'") to load the knowledge base.
5. Test from the n8n chat.

To change tours, prices or schedules, edit the chunks in the **Chunks** node and run the load again.

## Screenshots

- [`images/01-whatsapp-conversation-test.png`](images/01-whatsapp-conversation-test.png)
- [`images/02-n8n-workflow-southern-trails-assistant.png`](images/02-n8n-workflow-southern-trails-assistant.png)
- [`images/03-test-results-table.png`](images/03-test-results-table.png)
- [`images/04-n8n-rag-tool-call.png`](images/04-n8n-rag-tool-call.png)

## Tech stack

n8n · Google Gemini · RAG · Meta messaging API · JSON
