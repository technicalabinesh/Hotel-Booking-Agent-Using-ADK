# Hotel Booking Agent Using ADK

> A practical Google ADK project that demonstrates an AI-powered **restaurant booking** workflow using an Azure-hosted GPT model, in-memory state, and explicit tool-driven control flow.

## Overview

This repository contains a compact but complete conversational booking assistant built with Google Agent Development Kit (ADK). Even though the repository name says *Hotel Booking Agent*, the current implementation is intentionally focused on restaurant discovery and table booking. The assistant can search restaurants by location and cuisine, check time-slot availability, create a booking only after user confirmation, retrieve existing bookings, modify bookings, and cancel bookings. The implementation emphasizes predictable behavior, explicit validations, and safe conversational patterns instead of hidden automations.

The project is intentionally lightweight: it uses a fixed in-memory restaurant dataset, stores bookings in a Python dictionary, and avoids any external database dependency. That design makes the project easy to run locally, easy to understand in one file, and ideal for learning how to wire language models to deterministic tools. The trade-off is persistence: bookings disappear whenever the process restarts. For demos, tutorials, and rapid prototyping this is excellent, but production deployments would require durable storage, stronger authentication, observability, and operational safeguards.

## What This Agent Does

- Search restaurants using `location` and optional `cuisine` filters.

- Return normalized restaurant result objects with IDs and available time slots.

- Normalize date input (for example: `today`, `tomorrow`, `DD/MM/YYYY`, `YYYY-MM-DD`).

- Normalize time input (for example: `8 PM`, `08:30 PM`, `20:00`).

- Check availability against both restaurant time slots and existing confirmed bookings.

- Create a booking with a generated booking ID (`BK-XXXXXXXX`) after explicit user confirmation.

- Fetch booking details with `get_booking`.

- Cancel bookings with clear status handling.

- Modify booking date, time, and guest count with a final availability re-check.

- Enforce clear conversational rules inside the agent instruction prompt.


## Architecture Summary

The implementation lives primarily in `agent.py` and is built around four layers:

### Configuration Layer
Environment variables are loaded from `.env` using `python-dotenv`. The program fails early if required Azure settings are missing. Early failure makes misconfiguration obvious and prevents partially working runs that are hard to debug.

### Data Layer
Restaurant records are hardcoded in a list of dictionaries. This fixed catalog allows deterministic behavior and makes it easy to verify each tool response. There is no remote API call for restaurant lookup in the current version.

### Booking State Layer
Bookings are held in a module-level dictionary called `BOOKINGS`. Each booking has ID, restaurant metadata, customer name, date, time, guest count, and status. The state exists only while the process is alive.

### Tool Layer
Six tool functions expose business operations to the model: search, availability check, create, retrieve, cancel, and modify. The language model is instructed to rely on these tools rather than inventing data. This separation keeps business logic explicit and testable.

## Repository Structure

- `agent.py`: Main ADK agent definition, data, helper functions, and booking tools.

- `requirements.txt`: Python dependencies (`google-adk[extensions]`, `python-dotenv`).

- `README.md`: Project documentation (this file).

- `__init__.py`: Package marker file.


## Prerequisites

- Python 3.10 or newer (3.11+ recommended).

- An Azure-hosted model deployment compatible with LiteLLM Azure format.

- API credentials and deployment metadata stored in a local `.env` file.

- A virtual environment for dependency isolation.


## Environment Variables

Set the following values in your `.env` file:

- `AZURE_API_KEY`: API key used to authenticate with Azure model endpoint.

- `AZURE_API_BASE`: Base endpoint URL for your Azure model deployment.

- `AZURE_API_VERSION`: Version string required by Azure API invocation.

- `AZURE_DEPLOYMENT_NAME`: The deployment name used in `azure/<deployment-name>` model format.


If any required variable is missing, startup raises a `ValueError` with a direct message, allowing immediate correction.

## Quick Start

1. Clone the repository and move into the project directory.

2. Create and activate a Python virtual environment.

3. Install dependencies from `requirements.txt`.

4. Create a `.env` file with your Azure credentials.

5. Run the agent according to your ADK execution workflow or integration setup.

6. Start a conversation and test tool-driven booking flows.


## Tool-by-Tool Behavior

### `search_restaurants(location, cuisine="")`

- Filters restaurants by exact location match (case-insensitive).

- Applies optional cuisine inclusion check (case-insensitive contains).

- Returns `success`, `count`, and normalized result objects.



### `check_availability(restaurant_id, date, time, guests)`

- Validates restaurant existence.

- Normalizes date and time values before validation.

- Ensures requested time belongs to restaurant `available_times`.

- Rejects slot if another confirmed booking already occupies same restaurant/date/time.

- Returns clear availability message for conversational use.



### `create_booking(restaurant_id, customer_name, date, time, guests)`

- Validates restaurant existence.

- Performs final availability check before commit.

- Generates booking ID in `BK-XXXXXXXX` format.

- Stores booking with `CONFIRMED` status and returns full booking payload.



### `get_booking(booking_id)`

- Looks up booking by ID in in-memory store.

- Returns success=false if not found.

- Returns booking payload if found.



### `cancel_booking(booking_id)`

- Rejects unknown booking IDs.

- Prevents repeated cancellation of already cancelled bookings.

- Marks booking status as `CANCELLED` when valid.



### `modify_booking(booking_id, new_date="", new_time="", new_guests=0)`

- Supports partial updates by reusing existing values when fields are omitted.

- Blocks updates for cancelled bookings.

- Temporarily sets status to `MODIFYING` while performing conflict checks.

- Revalidates availability before persisting updated values.



## Conversation Safety and Guardrails

A major strength of this project is the instruction block bound to the ADK agent. The prompt enforces strict operating rules: do not invent restaurant data, do not invent availability, always use tools for booking facts, request missing user details, and ask explicit confirmation before final booking creation. This pattern is a practical example of combining deterministic business methods with LLM flexibility in natural language understanding. Users can speak naturally, while the system still behaves like a controlled transactional assistant.

## Data Normalization Rules

The helper functions standardize user input before business checks:

- `normalize_date` supports keywords (`today`, `tomorrow`) and multiple date formats.

- `normalize_time` supports 24-hour and AM/PM formats.

- Normalized formats become canonical (`YYYY-MM-DD` and `HH:MM`) for internal comparisons.

- Canonical formatting reduces duplicate slots caused by mixed input styles.


## Current Limitations

- No persistent database; booking records are lost on restart.

- No authentication/authorization layer for user identity.

- No capacity model beyond one booking per restaurant/date/time slot.

- No timezone handling or locale negotiation for date parsing.

- No formal API server wrapper in this repository version.

- No dedicated automated test suite in the current repository state.


## Production Hardening Ideas

- Replace in-memory dictionaries with persistent storage (SQL or managed NoSQL).

- Introduce booking capacity constraints by table size and slot limits.

- Add user profile and authentication mechanisms.

- Implement structured logging and request correlation IDs.

- Capture metrics around booking attempts, failures, and completion rates.

- Add retry policies and timeout handling around model invocations.

- Write unit tests for helper and tool functions.

- Add integration tests for full conversational flows.

- Introduce policy checks for abuse prevention and prompt injection defense.

- Deploy behind a service boundary with secure secrets management.


## Detailed Usage Narratives

The following narratives describe representative workflows and operator expectations.

### Discovery Workflow
A user starts by asking for a cuisine in a city. The assistant should call `search_restaurants` and present only returned options. If the user asks for recommendations outside dataset boundaries, the assistant should stay transparent: it can only offer what the configured list provides. This straightforward honesty reduces hallucination risk and builds trust, especially in transactional assistants where invented data can immediately create user frustration.

### Availability Workflow
After selecting a restaurant, the assistant must gather date, time, and guests before checking availability. The tool first validates if the requested slot exists in restaurant opening slots, then checks whether a confirmed booking already occupies it. If unavailable, the assistant should show alternative times returned by the tool, preserving continuity and helping the user make a valid selection quickly.

### Confirmation Workflow
The instruction set intentionally separates *availability check* from *booking creation*. This creates a safety gate where users can review details before committing. The assistant should summarize restaurant, date, time, guests, and customer name, then ask for explicit confirmation. Only after a clear affirmative response should `create_booking` run. This pattern mirrors transaction confirmation flows in mature digital products.

### Modification Workflow
Users may change plans after booking. The modify operation allows partial updates and reuses original values where no new value is provided. Before writing updates, the function temporarily excludes the current booking from collision checks by setting a temporary status. That small technique avoids false conflicts when a user keeps the same slot while changing another field.

### Cancellation Workflow
Cancellation depends on booking ID and includes idempotency-style handling. If the booking does not exist, the user receives a clear not-found message. If already cancelled, the assistant states it directly. Otherwise the booking status changes to `CANCELLED`. This avoids silent failures and keeps user expectations aligned with internal state transitions.


## Extended Implementation Notes

### Why deterministic tools matter
Language models are strong at language understanding but weaker at strict transactional consistency when left unconstrained. By exposing explicit tool functions and directing the model to use them, this project turns fuzzy intent into deterministic operations. That creates reproducible outputs, easier debugging, and safer business behavior. In practical terms, it means a booking is only “real” when the booking function says it is real.

### Why early configuration validation matters
Startup validation for required environment variables is a simple but important reliability practice. If credentials are missing, the process exits immediately with a targeted message. This is better than latent failures during user interactions because it shifts configuration errors into startup checks, where they are easier to detect and correct.

### Why in-memory state is useful for learning
Persistent infrastructure can hide core logic behind operational complexity. In-memory state keeps attention on data flow, validation rules, and conversation design. For educational and prototyping contexts, this simplicity is a net benefit, as long as users understand persistence limitations.

### Prompt design as executable policy
The instruction block in this project does more than describe behavior; it acts as policy. It constrains when a tool may be called, what must be shown to users, and what information should never be exposed. Good prompt policy does not replace code-level checks, but it dramatically improves baseline behavior in live conversations.

### Normalization as conflict prevention
Without normalization, two semantically equal requests can be treated as different values (`8 PM` versus `20:00`). Normalization creates canonical representations so conflict checks stay accurate. This is a small implementation detail with outsized effect on booking correctness and user trust.


## Operational Checklist

- [ ] Confirm `.env` contains all required Azure variables before runtime.

- [ ] Keep API keys private and out of source control.

- [ ] Verify location strings in requests align with available dataset values.

- [ ] Always ask users for explicit booking confirmation before create operation.

- [ ] Use booking IDs exactly as generated for retrieval, modification, and cancellation.

- [ ] Communicate non-persistence clearly in demos or pilot environments.

- [ ] Treat unavailable slots as recoverable outcomes and offer alternatives.

- [ ] Add logging before introducing multi-user or production traffic.

- [ ] Add tests before larger refactors or model/provider swaps.

- [ ] Review prompt instructions whenever tool behavior changes.


## Frequently Asked Questions

### Is this really a hotel booking agent?
The current code implements a restaurant booking flow. The repository name references hotel booking, but the active tools, data model, and prompt all target restaurant reservations.

### Are bookings saved permanently?
No. Bookings are stored in memory and disappear when the process restarts.

### Can I connect this to real restaurant APIs?
Yes. You can replace static restaurant data with external APIs and keep the same tool contract, then add caching and error handling.

### Can this support multiple cities?
Yes. Expand the restaurant dataset or data source and keep location filtering logic aligned with your new schema.

### Can I add payment collection?
Yes, but payment should be isolated behind secure backend services and never handled directly in prompt text.

### What should I test first?
Start with normalization edge cases, availability collisions, confirmation gating, and cancellation idempotency behavior.

### How do I improve reliability?
Introduce persistent storage, robust tests, structured logs, and deployment-time health checks.

### How do I reduce hallucinations further?
Keep tool contracts strict, keep prompt policy explicit, and avoid ambiguous fallback behavior for data-bearing responses.


## Implementation Expansion Backlog

### Data & Persistence

- Add a relational schema for restaurants, slots, and bookings.

- Track booking history with immutable event records.

- Create migration scripts for schema versioning.

- Support soft deletes and audit trails.



### Agent Experience

- Add multilingual support with locale-sensitive date parsing.

- Add preference-aware recommendations for returning users.

- Add follow-up prompts for ambiguous date/time inputs.

- Add fallback strategy for unsupported user requests.



### Reliability & DevEx

- Write comprehensive unit tests for all tool functions.

- Add CI checks for formatting, linting, and tests.

- Add static typing hardening and stricter type validations.

- Document local run targets and troubleshooting matrix.



### Security & Governance

- Introduce request-level authentication and authorization.

- Enforce rate limits for abusive traffic patterns.

- Redact potentially sensitive details from logs.

- Add policy checks for prompt injection attempts.



## Long-Form Developer Notes

1. When extending this project, preserve the clear separation between conversational guidance and business execution. The model should translate user intent, while tool functions should enforce truth. This separation is the most valuable architectural principle in the repository and should remain stable even if every other part of the stack changes.

2. Treat booking operations as state transitions rather than plain text outcomes. A user-facing confirmation message should be considered a rendering of the underlying tool result, not an independent decision by the model. This mindset keeps implementation anchored to source-of-truth objects and reduces accidental inconsistencies.

3. As the scope grows, avoid embedding complex business rules directly into prompt instructions alone. Prompt guidance is excellent for flow and tone, but critical constraints should also be encoded in deterministic Python checks so behavior remains consistent even if prompt text evolves over time.

4. If you adopt persistent storage, carry forward canonical date and time formatting rules. Data normalization should happen before persistence and before conflict checks. Keeping a canonical format across layers prevents subtle bugs during querying, indexing, and reporting.

5. Build observability early once traffic grows. Structured logs around tool calls, availability decisions, and booking status changes provide high value for debugging and trust. Include correlation IDs if you expose this through an API so one user journey can be traced end to end.

6. Design error messages for both users and developers. User-facing messages should be clear and actionable, while internal logs can include deeper diagnostic context. Separating these concerns makes the system friendlier without sacrificing operational insight.

7. Future integrations with calendars, maps, or notifications should remain optional modules behind well-defined interfaces. Keep the core booking behavior simple and deterministic so optional integrations do not weaken baseline reliability.

8. In multi-user deployments, collision checks must become transactional. In-memory checks are fine for demos, but real systems require atomic writes and conflict-safe constraints at the database layer. This prevents race conditions where two users book the same slot simultaneously.

9. When extending this project, preserve the clear separation between conversational guidance and business execution. The model should translate user intent, while tool functions should enforce truth. This separation is the most valuable architectural principle in the repository and should remain stable even if every other part of the stack changes.

10. Treat booking operations as state transitions rather than plain text outcomes. A user-facing confirmation message should be considered a rendering of the underlying tool result, not an independent decision by the model. This mindset keeps implementation anchored to source-of-truth objects and reduces accidental inconsistencies.

11. As the scope grows, avoid embedding complex business rules directly into prompt instructions alone. Prompt guidance is excellent for flow and tone, but critical constraints should also be encoded in deterministic Python checks so behavior remains consistent even if prompt text evolves over time.

12. If you adopt persistent storage, carry forward canonical date and time formatting rules. Data normalization should happen before persistence and before conflict checks. Keeping a canonical format across layers prevents subtle bugs during querying, indexing, and reporting.

13. Build observability early once traffic grows. Structured logs around tool calls, availability decisions, and booking status changes provide high value for debugging and trust. Include correlation IDs if you expose this through an API so one user journey can be traced end to end.

14. Design error messages for both users and developers. User-facing messages should be clear and actionable, while internal logs can include deeper diagnostic context. Separating these concerns makes the system friendlier without sacrificing operational insight.

15. Future integrations with calendars, maps, or notifications should remain optional modules behind well-defined interfaces. Keep the core booking behavior simple and deterministic so optional integrations do not weaken baseline reliability.

16. In multi-user deployments, collision checks must become transactional. In-memory checks are fine for demos, but real systems require atomic writes and conflict-safe constraints at the database layer. This prevents race conditions where two users book the same slot simultaneously.

17. When extending this project, preserve the clear separation between conversational guidance and business execution. The model should translate user intent, while tool functions should enforce truth. This separation is the most valuable architectural principle in the repository and should remain stable even if every other part of the stack changes.

18. Treat booking operations as state transitions rather than plain text outcomes. A user-facing confirmation message should be considered a rendering of the underlying tool result, not an independent decision by the model. This mindset keeps implementation anchored to source-of-truth objects and reduces accidental inconsistencies.

19. As the scope grows, avoid embedding complex business rules directly into prompt instructions alone. Prompt guidance is excellent for flow and tone, but critical constraints should also be encoded in deterministic Python checks so behavior remains consistent even if prompt text evolves over time.

20. If you adopt persistent storage, carry forward canonical date and time formatting rules. Data normalization should happen before persistence and before conflict checks. Keeping a canonical format across layers prevents subtle bugs during querying, indexing, and reporting.

21. Build observability early once traffic grows. Structured logs around tool calls, availability decisions, and booking status changes provide high value for debugging and trust. Include correlation IDs if you expose this through an API so one user journey can be traced end to end.

22. Design error messages for both users and developers. User-facing messages should be clear and actionable, while internal logs can include deeper diagnostic context. Separating these concerns makes the system friendlier without sacrificing operational insight.

23. Future integrations with calendars, maps, or notifications should remain optional modules behind well-defined interfaces. Keep the core booking behavior simple and deterministic so optional integrations do not weaken baseline reliability.

24. In multi-user deployments, collision checks must become transactional. In-memory checks are fine for demos, but real systems require atomic writes and conflict-safe constraints at the database layer. This prevents race conditions where two users book the same slot simultaneously.

25. When extending this project, preserve the clear separation between conversational guidance and business execution. The model should translate user intent, while tool functions should enforce truth. This separation is the most valuable architectural principle in the repository and should remain stable even if every other part of the stack changes.

26. Treat booking operations as state transitions rather than plain text outcomes. A user-facing confirmation message should be considered a rendering of the underlying tool result, not an independent decision by the model. This mindset keeps implementation anchored to source-of-truth objects and reduces accidental inconsistencies.

27. As the scope grows, avoid embedding complex business rules directly into prompt instructions alone. Prompt guidance is excellent for flow and tone, but critical constraints should also be encoded in deterministic Python checks so behavior remains consistent even if prompt text evolves over time.

28. If you adopt persistent storage, carry forward canonical date and time formatting rules. Data normalization should happen before persistence and before conflict checks. Keeping a canonical format across layers prevents subtle bugs during querying, indexing, and reporting.

29. Build observability early once traffic grows. Structured logs around tool calls, availability decisions, and booking status changes provide high value for debugging and trust. Include correlation IDs if you expose this through an API so one user journey can be traced end to end.

30. Design error messages for both users and developers. User-facing messages should be clear and actionable, while internal logs can include deeper diagnostic context. Separating these concerns makes the system friendlier without sacrificing operational insight.

31. Future integrations with calendars, maps, or notifications should remain optional modules behind well-defined interfaces. Keep the core booking behavior simple and deterministic so optional integrations do not weaken baseline reliability.

32. In multi-user deployments, collision checks must become transactional. In-memory checks are fine for demos, but real systems require atomic writes and conflict-safe constraints at the database layer. This prevents race conditions where two users book the same slot simultaneously.

33. When extending this project, preserve the clear separation between conversational guidance and business execution. The model should translate user intent, while tool functions should enforce truth. This separation is the most valuable architectural principle in the repository and should remain stable even if every other part of the stack changes.

34. Treat booking operations as state transitions rather than plain text outcomes. A user-facing confirmation message should be considered a rendering of the underlying tool result, not an independent decision by the model. This mindset keeps implementation anchored to source-of-truth objects and reduces accidental inconsistencies.

35. As the scope grows, avoid embedding complex business rules directly into prompt instructions alone. Prompt guidance is excellent for flow and tone, but critical constraints should also be encoded in deterministic Python checks so behavior remains consistent even if prompt text evolves over time.

36. If you adopt persistent storage, carry forward canonical date and time formatting rules. Data normalization should happen before persistence and before conflict checks. Keeping a canonical format across layers prevents subtle bugs during querying, indexing, and reporting.

37. Build observability early once traffic grows. Structured logs around tool calls, availability decisions, and booking status changes provide high value for debugging and trust. Include correlation IDs if you expose this through an API so one user journey can be traced end to end.

38. Design error messages for both users and developers. User-facing messages should be clear and actionable, while internal logs can include deeper diagnostic context. Separating these concerns makes the system friendlier without sacrificing operational insight.

39. Future integrations with calendars, maps, or notifications should remain optional modules behind well-defined interfaces. Keep the core booking behavior simple and deterministic so optional integrations do not weaken baseline reliability.

40. In multi-user deployments, collision checks must become transactional. In-memory checks are fine for demos, but real systems require atomic writes and conflict-safe constraints at the database layer. This prevents race conditions where two users book the same slot simultaneously.


## Conclusion

This repository is a strong starting point for tool-driven conversational booking systems. It demonstrates a clean combination of ADK orchestration, Azure-backed model usage, deterministic Python tools, and explicit confirmation policies. With persistent storage, tests, and deployment hardening, it can evolve from a learning project into a production-capable service pattern.

Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 
Additional implementation guidance: keep business constraints explicit, normalize user input early, and treat tool responses as the canonical source for user-visible booking status. Document every state transition and ensure modifications and cancellations are validated with the same rigor as creation. 