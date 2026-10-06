# Apex Home Services — AI Voice Receptionist

A live voice AI agent that answers calls for a (fictional) home services company, collects everything a dispatcher needs, classifies emergencies, answers pricing/policy questions from a knowledge base, and books real appointments on a shared calendar in real time, all without a human in the loop.

**Stack:** Retell AI (voice agent + LLM + function calling) · Make.com (automation/orchestration) · Google Sheets (lead log) · Gmail (emergency alerting) · Google Calendar (live scheduling)


---

## Why this project

Most voice AI demos stop at "transcribe the call and log it somewhere." That's a useful pattern, but it's the same pattern repeated for every integration. I wanted to build something that also showed the harder, more valuable case: an agent that takes a real-time action mid-call and changes its next sentence based on the result. That's the difference between a call logger and something that behaves like a dispatcher.

So this project has two tracks:

1. **Post-call automation** — every call logs structured data to a spreadsheet, and emergency calls additionally trigger an alert email. This is the "safe" pattern: nothing blocks the conversation, failures are easy to retry.
2. **Live function calling** — mid-call, the agent checks real calendar availability and books the appointment on the spot, then tells the caller whether it worked, in the same breath. This is harder: it has to be fast (the caller is waiting), it has to handle both outcomes gracefully, and a bug here is immediately audible to the caller instead of silently failing in a log somewhere.

## What the agent does

A caller reaches "Alex," the intake agent, and the conversation collects:

1. Full name
2. A valid 10-digit callback number (re-prompts digit by digit if it's incomplete or mumbled)
3. A complete service address (street number, street name, city — won't accept a vague answer)
4. The issue or service needed
5. Urgency: classified strictly as `Emergency` or `Routine` based on explicit criteria (active leak/flooding, no heat/AC in extreme conditions, gas smell, sewage backup, exposed wiring, full power loss → Emergency; everything else → Routine)
6. A preferred appointment day and time

Along the way, the agent can also answer questions about pricing, service area, hours, cancellation policy, and warranties, pulled from an attached knowledge base document rather than guessed.

Once all 6 details are confirmed back to the caller, two things happen automatically:

- The call is logged to a spreadsheet (every call, regardless of urgency), and if it's an Emergency, an alert email fires immediately.
- The requested appointment time is checked against a live Google Calendar. If it's free, the agent books it on the spot and tells the caller. If it's taken, the agent asks for another time and tries again, in the same call.

## Architecture

```
Caller
  │
  ▼
Retell AI Agent ("Alex")
  │
  ├── Knowledge Base (pricing, policy, FAQ — retrieval, not hallucination)
  │
  ├── Function: book_appointment ──────► Make.com: Lead Logging & Emergency Alert
  │                                         │
  │                                         ├── Google Sheets (every call)
  │                                         └── Gmail (Emergency/Urgent only, filtered)
  │
  └── Function: check_and_book_appointment ──► Make.com: Check & Book Appointment
                                                 │
                                                 ├── Google Calendar: Search Events
                                                 │     (is this 2-hour slot free?)
                                                 ├── Router
                                                 │     ├── Free  → Create Event → "confirmed" response
                                                 │     └── Taken → "unavailable" response
                                                 └── Response spoken back to caller live
```

Both Make.com scenarios run independently and continuously ("immediately as data arrives"), so the agent never has to wait on anything but the one function call it's actively making.

![Apex - Check & Book Appointment scenario in Make.com](https://github.com/user-attachments/assets/51caf111-0d02-442a-aad9-11fcbd849ccd)


*The live calendar-booking automation: Webhook → Search Events → Router → Create Event / "slot taken" response.*

## Design decisions (and why)

**Single shared calendar, fixed 2-hour appointment blocks.** A real dispatch system would route by technician, skill, and job-type duration. I scoped that out deliberately. The goal here was to prove the hard part, real-time availability checking and booking inside a live conversation, without building a full scheduling engine around it. This is a documented simplification, not a gap I didn't notice.

**Two separate Make.com scenarios instead of one.** Lead logging and calendar booking have different failure characteristics. Logging a lead can retry silently in the background; booking a calendar slot has to resolve before the agent's next sentence. Splitting them keeps the latency-sensitive path (calendar) lean, and means an outage in one doesn't take down the other.

**Knowledge base over hard-coded prompt facts.** Pricing, policy, and service-area details live in a separate markdown document the agent retrieves from, not in the system prompt. That keeps the prompt focused on *how to run the call* and makes updating a price or policy a content change, not a prompt-engineering change.

## What broke, and what I learned fixing it

This is the part I think is actually worth reading, the real engineering work in a project like this is almost entirely in these failure modes, not in the happy path.

**"Search found nothing" and "search found a conflict" look identical if you're not careful.** Google Calendar's Search Events module, when it finds zero matching events, still emits one bundle, with blank fields, rather than zero bundles. My first version ran that output through an Array Aggregator and checked its length to decide free vs. busy. Since the aggregator saw "1 item" even when the calendar was empty, every single request came back "unavailable," regardless of the actual calendar. The fix was to drop the aggregator entirely and check the search module's own output field (its event ID) directly: *does an ID exist, or not*. Simpler, and correct. The lesson: when a "no results" case is silently indistinguishable from a "one result" case, build an index-independent check, not something you expect to have to infer from counting.

**Function parameters aren't where you tested them.** I built and validated the calendar-check webhook using manual browser requests with flat query parameters (`?requested_start_time=...`). It worked. Then I wired it to Retell's actual function-calling and it broke, every request came back "unavailable" no matter the date. The real cause: Retell nests custom function arguments inside an `args` object in the webhook payload, so the field is `args.requested_start_time`, not `requested_start_time`. My manual tests never exercised that shape, so the mapping looked right and was actually silently resolving to nothing, which meant the calendar search ran with no date filter at all and just matched whatever event already existed. The fix was simple once diagnosed (prefix every mapped field with `args.`), but finding it took walking through Make's execution history module by module to see the literal payload Retell actually sent, rather than trusting what I'd built against a hand-crafted test request. The lesson: test the integration with its real caller, not a stand-in for it, especially once function-calling or any wrapper layer is involved.

![Make.com module inspector showing the args-nested payload](https://github.com/user-attachments/assets/45ef61a0-3627-4168-8d0f-c8c3af383b50)

*The module inspector view that revealed Retell nests function parameters inside `args`, the root cause of the "always unavailable" bug.*

![Retell custom Function configuration](https://github.com/user-attachments/assets/5a84506e-6eac-4b90-bdc6-993460657a63)
![Retell custom Function parameters and execution settings](https://github.com/user-attachments/assets/14aa976f-4793-4d83-aa97-a1a28e57538e)


*The `check_and_book_appointment` custom Function, configured with its 5 parameters and pointed at the Make.com webhook.*

**An instruction that's "obviously implied" isn't.** The lead-logging function originally fired on its own, based purely on its own description ("call this after the caller confirms their details"), with no explicit mention of it in the prompt. That worked, until I added an explicit multi-step instruction for what happens after confirmation (the calendar booking sequence). The moment the prompt gave the model one clear, sequential thing to do next, it stopped independently invoking the other function, even though nothing in the new prompt told it to stop. The lesson: once you give an LLM an explicit procedure, every action you need to happen at that point has to be named in the procedure. Relying on "it'll still do the other thing because its description says so" is a bet you'll eventually lose.

![A real appointment created on the live calendar from a test call](https://github.com/user-attachments/assets/864f7dbb-ce06-4329-87e2-395e8d800f9c)

*A real event, created live during a test call, sitting on the actual Google Calendar.*

## Status

Live and tested end to end: intake validation, knowledge base retrieval, emergency classification and alerting, lead logging, and real-time calendar booking (both the "confirmed" and "slot taken, try again" paths).

Not built, by design: per-technician routing, variable appointment lengths, CRM/ticketing lookups. These are natural next steps for a production version, scoped out here to keep the project focused on demonstrating live function calling well rather than building a full field-service platform.
