# Incident 2026-09-15 — "confirmed" demo that was never booked

**Lead:** Enrico, +393200382838 (lead `a52d2d90-8dd0-4c7b-8088-c8d9e9c8f62a`)
**Conversation:** `67de808f-6639-49ee-86ea-edfa5a6310ce` (campaign `74f8f9a4` "Whappy Demo & Booking ITA", tenant `sw8M9aBAsWO2qVph7xeH7wDy2ge2`)
**Appointment doc:** `2d1f04a6-c68d-4460-a13d-4ddd3ebd730c` — status `REQUESTED`, never `BOOKED`.

## What the lead experienced

07:29 AI (template `primo_contatto`) → 08:06 "Yess" → 08:07 "Marketing per aziende di affitti brevi airbnb"
→ 08:09 "Ti andrebbe di fissare una breve demo…" → 08:12 "Si va bene" → 08:14 "Oggi alle 15"
→ 08:15 "Ti confermo quindi oggi alle 15:00?" → 08:15 "Si"
→ 08:16 **"Perfetto Enrico! Ti confermo l'appuntamento per oggi alle 15:00. A brevissimo ti invieremo tutti i dettagli per collegarti."**

No booking exists on Cal.com. Nothing on team@whappy.ai's calendar. No details were ever sent.

## Root cause #1 — Cal.com rejected the booking: missing attendee email

Prod log, `whappy-processor`, 2026-09-15T08:16:46.745993Z:

```
Cal.com API v2 error: 400 - {"status":"error","path":"/v2/bookings",
 "error":{"code":"BAD_REQUEST","message":"responses - {email}error_required_field, "}}
Error creating booking: Client error '400 Bad Request' for url 'https://api.cal.com/v2/bookings'
Failed Booking appointment for user sw8M9aBAsWO2qVph7xeH7wDy2ge2
```

The lead document has `email: ""`. `step_appointment_processor.py` passes it straight through
(`attendee={"name": lead.name, "email": lead.email, ...}`) and the prod log confirms
`appointment.attendee.email == ""`. Cal.com requires `email` on every booking, so the POST 400s.

**Why the lead has no email:** campaign `74f8f9a4` has **zero `info_definitions`** and **no INFO step**
(steps are START → TALK → TALK → APPOINTMENT → CLOSE). Email is never asked for, never collected,
and nothing in the appointment step checks for it before promising a slot. The lead arrived by
campaign transition from `e7786bff` ("International Real Estate Agent ITA"), which also collected
no email — only `purchase_timeline`, `estimated_budget`, `target_location`, `property_type`.

## Root cause #2 — we say "confirmed" even when the booking failed

`step-processor/controllers/step_controllers/appointment/step_appointment_processor.py` (~line 265-300):

```python
if result:
    await db_list.lead_db.update_appointment_status(..., status=AppointmentStatus.BOOKED)
else:
    logger.warning("Failed Booking appointment ...")
    next_step_id = current_step_id          # <-- only consequence of failure

reply = await run_llm(llm_helper.thanks_appointment, ...)   # <-- runs UNCONDITIONALLY
...
return ConversationProcessResult(status=WAITING_FOR_REPLY, next_step_id=next_step_id,
                                 response_text=reply)
```

On failure the code re-arms the same step but still sends the *thank-you-your-appointment-is-confirmed*
message. There is no failure-path reply. The lead is told a slot is booked that does not exist.

The GHL branch has the same shape (identical `next_step_id = current_step_id` fallthrough).

**Same family, one block earlier:** the tenant WhatsApp notification (`appointment_scheduled_notification`)
fires *before* `create_booking` is ever called. Any tenant with that setting on gets an
"appointment scheduled" ping on their real phone for bookings that then 400. The in-code comment
already flags the synthetic-lead case; the failed-booking case is the same hole. This tenant has the
setting off, so no bogus ping went out here.

The "we'll send you the details" line is LLM improvisation. `ThanksAppointmentPrompt.TASK[CALCOM]`
says *"do not talk about links or invitation if not already provided in the conversation"* — the model
routed around it by promising "dettagli" instead of "link". A soft guardrail, not an enforced one.

## Root cause #3 — the campaign timezone is America/Chicago

The appointment doc stores `timezone: "America/Chicago"`, `utc_datetime: 2026-09-15 20:00:00`.
Enrico said "oggi alle 15" meaning 15:00 Europe/Rome. **We recorded 15:00 Chicago = 22:00 Rome —
7 hours off.**

This is not an LLM hallucination: `process_appointment`'s JSON schema has no timezone field at all.
`llm_helper.process_appointment` sets `appointment_timezone=settings.timezone`, and the per-campaign
`settings` row for `74f8f9a4` genuinely stores `America/Chicago`.

Source: `platform-backend/controllers/settings_controller.py:34` hardcodes `timezone="America/Chicago"`
when creating default funnel settings.

Blast radius across prod `settings`:

| timezone | campaigns |
|---|---|
| America/Chicago | 174 |
| Europe/Paris | 15 |
| Europe/Madrid | 5 |
| America/Los_Angeles | 4 |
| Etc/UTC | 3 |
| Asia/Jakarta | 1 |
| Etc/GMT+12 | 1 |

174 of 203 campaigns carry the Chicago default. Every appointment they book lands in the wrong
timezone unless the tenant changed it manually. The same value drives two more things, both verified:

- `base_prompt.py:64` → `_get_formatted_date(settings.timezone)` renders the `today: {date}` header, so
  every prompt in this campaign states the Chicago wall-clock. Between ~01:00 and ~07:00 Rome the
  model is told it is still *yesterday*.
- `_list_availabilities` passes `timezone=settings.timezone` to both the Cal.com and GHL availability
  calls, so the free slots shown to the model are Chicago-local too.

Had the email been present, this booking would have gone through — slot permitting — at **22:00 Rome**,
not 15:00.

## Blast radius of the booking failure

`appointments` collection, all time, by (status, integration):

| status | integration | count |
|---|---|---|
| REQUESTED | calendly | 50 |
| BOOKED | calendly | 39 |
| CREATED | (empty) | 7 |
| CREATED | calendly | 3 |
| REQUESTED | (empty) | 2 |
| REQUESTED | **calcom** | **2** |
| BOOKED | calcom | 1 |
| CREATED | (none) | 1 |

`REQUESTED` is set the moment the LLM says `confirmed=true`; only the CALCOM and GHL branches promote
it to `BOOKED`. **Calendly has no booking branch at all** — it is link-based, so its 50 `REQUESTED`
rows are the normal resting state, not failures. Ignore them.

The real figure is the Cal.com column: **2 REQUESTED vs 1 BOOKED.** Cal.com bookings fail more often
than they succeed. The two failures are 2025-08-03 and today. Cal.com is barely used in prod yet —
which is exactly why this is worth fixing now, before campaign `74f8f9a4` scales.

No GHL appointments exist at all.

## Fixes, in priority order

1. **Gate the confirmation message on booking success.** On `result is None`, generate a recovery
   reply (ask for the missing email / offer another slot), do **not** run `thanks_appointment`.
   Applies to both the Cal.com and GHL branches.
2. **Require an email before the APPOINTMENT step can confirm.** Either block `confirmed` when
   `not lead.email` and have the step ask for it, or synthesize a placeholder only where the tenant
   has explicitly opted in. Right now every emailless lead on a Cal.com campaign hits this.
3. **Fix the timezone default.** Change `settings_controller.py:34` away from `America/Chicago`
   (derive from tenant locale / WhatsApp number country, or force an explicit choice at campaign
   creation), and backfill the 174 affected campaigns.
4. **Add the missing INFO step / info_definition for email** to campaign `74f8f9a4` (and audit other
   Cal.com campaigns for the same gap).
5. **Move the tenant notification below the booking call**, so it only fires on a real booking.
6. **Alert on Cal.com/GHL `REQUESTED` appointments older than N minutes** — this failure was completely silent to
   operations; only a log line recorded it.

## Immediate action for this lead

Enrico's 15:00 demo does not exist on any calendar. As of 10:30 Rome there are ~4.5 hours left —
book it manually for 15:00 Europe/Rome and message him, or tell him it needs rescheduling.
