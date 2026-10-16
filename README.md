# cleartosend — One gate before every text, email and call

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/cleartosend**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

cleartosend puts one gate in front of every text, email and call your software sends. Approval, opt-outs, quiet hours, calling hours and daily caps are checked before anything reaches a provider, and an audit finds the code that calls Twilio, SMTP or an email API without asking the gate. It can also hold a message for a later time and plan a text to a whole list before anyone is reached.

## Get it

Buy cleartosend at https://dominares.org/tools/cleartosend: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the cleartosend command.

Requirements: The Python standard library, with SQLite as the default store. Sending through Twilio or Resend needs your own account with them.

This page documents version 1.1.2. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

**Nothing reaches a customer's phone or inbox unless every rule you set says yes.**

```bash
cleartosend audit examples/reminders_before.py     # lists the send that skips the gate, as file:line
```

Run it from the root of your copy of the repository. The last line prints
`examples/reminders_before.py:10:9: twilio-sdk: Twilio send outside the gate ...` and exits 1: that is
the tool finding a text that would go out without asking the rules.

A text that goes out at 10:40 at night, a reminder to someone who replied STOP last week, a call
placed before 9 in the morning: each of these is one line of code that nobody reviewed, and each
can carry its own legal exposure. The usual cause is not a bug in your rules. It is a new button,
a cron job or a "quick" script that calls the provider directly and never asks the rules.

cleartosend puts one gate in front of every text, email and call:

- **Approval.** A message waits for a person to approve it, unless you turn on `auto_send` on purpose.
- **Opt-outs.** Anyone on the suppression list is never contacted, on any channel. STOP and its
  carrier siblings are read exactly as carriers define them, and so are plain sentences like
  "please take me off your list" or "quit texting me" (but not "stop the noise next door").
- **Quiet hours.** A message caught by them is *held*, not dropped, and goes out when they end.
- **Calling hours.** Voice calls are placed only between 09:00 and 20:00 (configurable).
- **Daily caps.** Per channel, counted on your clock.
- **A time zone you chose.** There is no default. A policy without one refuses to exist.
- **Fail closed.** If the suppression list cannot be read, nothing is sent to anyone until it can.
- **No double sends.** A task is marked `sending` and saved *before* the provider is called. If the
  process dies in between, the task is never sent again by accident.
- **Send later, once.** `send_at` holds a message until a time you choose ("7am tomorrow"), with a
  visible queue of what is waiting and a view of what got stuck.
- **A whole list, planned first.** `cleartosend broadcast plan` shows who a text to a list would reach,
  who it would not and why, what it costs in SMS segments and which messages quiet hours would hold,
  before anything is queued. A sweep from the command line stops after 5 failed sends in a row, so a
  provider with a dead key is not hammered (the Python call stops only when you ask).

And a check for the part a queue cannot protect on its own: `cleartosend audit` reads your Python
source and lists every place that calls Twilio, SMTP or an email API **without** going through the
gate, as `file:line`, and exits non-zero. Run it in CI.

**Who it is for:** developers at agencies, clinics, property managers and local-services software
companies who text, email or call customers and have per-message exposure if a rule is missed.

The runtime is the Python standard library only (3.10 or newer): SQLite for storage, `urllib` for
Twilio and Resend, `smtplib` for SMTP. No SDK, no account, no agent, no network calls of its own.

> **cleartosend is a tool. It is not legal advice and it does not make your messages lawful.**
> Getting and recording consent, registering your numbers (for example A2P 10DLC in the US), and
> following telemarketing, anti-spam and privacy law such as the TCPA stay with you. What it does is
> make the rules you decide on impossible to skip by accident. See [What it does not do](#what-it-does-not-do).

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

Installing it adds the `cleartosend` command and the `cleartosend` Python package. The commands below run
from the same folder. On Windows or a slim container with no time zone database, add the `tz` extra:
`pip install '.[tz]'`.

### 60-second quickstart

Nothing here sends anything real: `--sender stdout` prints instead of calling a provider. Run these
in the folder you installed from, one after another.

<!-- quickstart:start -->
```bash
export CLEARTOSEND_POLICY=examples/policy.json CLEARTOSEND_DB=demo.db

# Queue a text. It waits for a person. (This also creates demo.db, which holds the queue and the suppression list.)
ID=$(cleartosend queue add -q --channel sms --to "(555) 123-0000" --body "Your appointment is tomorrow at 10:00.")

# Would it be allowed at 10:30 at night? No: quiet hours, and nobody has approved it.
cleartosend check --channel sms --to "(555) 123-0000" --body "Your appointment is tomorrow at 10:00." --status pending --at 2026-06-22T22:30

cleartosend queue sweep --sender stdout --at 2026-06-23T10:00     # sends nothing: not approved

# Approve it, then sweep late at night: it is held, not dropped.
cleartosend queue approve "$ID" --by dana
cleartosend queue sweep --sender stdout --at 2026-06-22T22:30     # held: quiet hours

# Sweep in the morning: it goes.
cleartosend queue sweep --sender stdout --at 2026-06-23T10:00     # prints: -> sms to (555) 123-0000 ...

# The customer writes back. From now on every rule says no to them.
cleartosend reply --from "(555) 123-0000" --text "please take me off your list"
cleartosend check --channel sms --to "555-123-0000" --at 2026-06-23T10:00    # NOT clear to send: suppressed

# Send later, once: hold a text until a time (a day from now, 9am on the policy's clock).
WHEN=$(python3 -c "import datetime as d; print((d.datetime.now()+d.timedelta(days=1)).strftime('%Y-%m-%dT09:00'))")
LATER=$(cleartosend queue add -q --channel sms --to "(555) 123-1111" --body "Doors open at 9." --send-at "$WHEN")
cleartosend queue approve "$LATER" --by dana
cleartosend queue scheduled                                        # waiting, with its send time
cleartosend queue sweep --sender stdout                            # sends nothing: its time has not come

# A text to a whole list: who would be reached, who would not and why, before anything is queued.
cleartosend broadcast plan --contacts examples/contacts.csv --template "Hi {first_name}, your table is ready." --fallback first_name=there --at 2026-06-23T10:00

# Find the sends that skip the gate in a codebase.
cleartosend audit examples/reminders_before.py        # file:line for the Twilio call
cleartosend audit examples/reminders_after.py         # nothing left
```
<!-- quickstart:end -->

`check` exits 0 when the message is clear and 1 when it is not, naming **every** rule that failed;
`audit` exits 1 when it finds a send outside the gate. So a few lines above exit non-zero on
purpose. The same flow from Python is in `examples/app.py`:

```python
from cleartosend import Policy, Queue, Router, TwilioSMS

policy = Policy(timezone="America/Chicago", channels={"sms", "email"})
queue = Queue.sqlite("outbound.db", policy)          # the queue and the suppression list, one file

task = queue.enqueue("sms", "+15551230000", "Your appointment is tomorrow at 10:00.")
queue.approve(task.id, by="dana")                     # a person said yes
queue.sweep(Router(sms=TwilioSMS.from_env()))         # the only call that sends; run it from cron
```

### How a message moves

```
enqueue --> pending --approve--> approved --sweep--> sending --> sent
               |                  |    ^                |
             reject              held --+                +--> failed
               v             (a rule that clears
            skipped           with time said not yet)
```

| Status | Meaning |
|---|---|
| `pending` | Waiting for a person. Under `auto_send`, the next sweep treats it as approved. |
| `approved` | Approved. The next sweep sends it if every rule passes. |
| `held` | Approved, but a rule that clears with time said not yet (quiet hours, calling hours, a daily cap, a channel that is off, an unreadable suppression list, a provider that is not set up yet). The next sweep looks again. |
| `sending` | A sweep claimed it and saved that **before** calling the provider. |
| `sent` | The provider accepted it. |
| `failed` | The provider refused or errored. **Never retried automatically.** |
| `skipped` | Rejected by a person, or stopped by a rule that time cannot clear (the recipient opted out, no usable address). |

Every status carries a `reason` in plain words, and `failed` tasks carry the provider's `error`.
`enqueue` refuses a body longer than the channel holds (1,600 characters for sms, 4,000 for voice,
100,000 for email; 200 for a subject) instead of cutting it: what the approver read is what is sent.

**Only `Queue.sweep` sends.** It is safe to run every few minutes and safe to run twice at once:

1. A sweep takes a lock in the store; a second sweep that finds it taken returns at once. A lock
   left behind by a crashed sweep is taken over after ten minutes.
2. Each task is re-read just before it is judged, so a rejection made while the sweep runs is honoured.
3. The move to `sending` is a compare-and-set that is written before the provider is called, so two
   sweeps cannot both claim a task.
4. Quiet hours, calling hours and the daily cap are judged at the moment of each provider call, not
   when the sweep started: a sweep that runs long stops at 21:00 or 20:00 instead of carrying on.
   (`sweep(sender, now=...)` judges everything at that one time, for a dry run or a test.)
5. `limit` (default 500) is the most tasks one sweep hands to the provider. Tasks that stay held are
   looked at once per sweep, so a pile of held calls cannot keep newer texts from being reached.
6. A crash between "claimed" and "sent" leaves the task in `sending`, and no sweep ever picks it up
   again. The failure mode is "maybe not sent", never "sent twice". Look at the provider's log and
   settle it: `cleartosend queue resolve ID --sent --by dana` (or `--failed`), or `Queue.resolve`.

### Send later

`send_at` keeps a task from going before a time you choose, and it still has to be approved and pass
every rule when the time comes.

```python
from datetime import datetime, timedelta
tomorrow_7am = (datetime.now() + timedelta(days=1)).replace(hour=7, minute=0, second=0, microsecond=0)
task = queue.enqueue("sms", "+15551230000", "Doors open at 9.", send_at=tomorrow_7am)   # no offset: the policy's clock
queue.approve(task.id, by="dana")
queue.sweep(router)         # before 07:00: leaves it alone, counts it as `waiting`; after: sends it once
```

```bash
cleartosend queue add --channel sms --to "(555) 123-1111" --body "Doors open at 9." --send-at "$WHEN"   # $WHEN: a future time, as in the quickstart
cleartosend queue scheduled        # everything waiting for a later time, soonest first, with its send time
cleartosend queue stuck            # what needs a person; exit 1 when there is anything
```

- **The time.** An ISO 8601 string or an aware `datetime`. A time with no offset is read on the
  policy's clock. A time that has passed is refused (leave `send_at` out to send as soon as it is
  approved), and so is one more than 90 days away: that is a typo guard, not a limit of the tool.
  (`Task.new(send_at=...)` is lower level: give it UTC with a `Z`, as the queue stores it.)
- **Once.** It is the same claim as for any task: saved as `sending` before the provider is called, so
  two sweeps and a crash cannot send it twice.
- **Rules at that time.** If `send_at` falls in quiet hours the task is *held* and goes out when they
  end; the CLI says so when you add it. An opt-out that arrives before the time is honoured when the
  time comes: the task is skipped, never sent.
  `check` reports `scheduled` until the time comes, and the sweep leaves such a task exactly as it
  was, without re-reading it or looking anything up, so a long list scheduled for tomorrow neither
  slows every sweep (50,000 waiting tasks add well under a second) nor is rewritten by it.
  `queue sweep --dry-run` leaves such tasks out and says how many.
- **Cancel** a scheduled task with `queue reject` (or `Queue.reject`). It is never sent.
- **Too late.** Set `stale_after_hours` in the policy and a task still unsent that long after it
  could first have gone (because the sweep did not run, say) is held for a person (`stale`) and
  listed by `queue stuck`, instead of an "open at 9" text arriving at noon. "Could first have gone"
  is its `send_at`, or the end of the quiet hours or calling hours that cover it: a text scheduled
  for 22:00 that waits out the night is not late at 08:00 for having waited. It is off by default,
  because a task held by quiet hours is meant to go out late.
- **What needs a person.** `Queue.stuck()` lists tasks left in `sending` (they may or may not have
  gone), tasks that failed, stale ones, and any whose `send_at` cannot be read (`bad_send_at`), each
  with why. Nothing is retried for you. To try a failed one again, after you have looked at the
  provider's log, a person approves it again: `queue retry ID --by dana` (`Queue.retry`).

### A text to a whole list

`examples/contacts.csv` has a name, a phone and an optional `timezone` per row. In the plan below, the
10:00 New York moment is 07:00 for the contact in Los Angeles, so that one text is held until quiet hours end.
Planning and sending are two steps on purpose. The planner only reads, and the sending is the
ordinary queue, so every rule above applies to every recipient.

```bash
cleartosend broadcast plan   --contacts examples/contacts.csv --template "Hi {first_name}, your table is ready." --fallback first_name=there --at 2026-06-23T10:00
cleartosend broadcast submit --contacts examples/contacts.csv --template "Hi {first_name}, your table is ready." --fallback first_name=there --at 2026-06-23T10:00 \
                             --id spring --digest 739c5ab31a39af54 --by dana     # the digest the plan printed
cleartosend queue sweep                                                           # the only step that sends
cleartosend broadcast status spring
```

```
would be reached: 6
segments: 1 per message at most, 6 in all (GSM-7)
held at 2026-06-23T10:00:00-04:00: 1 by quiet_hours (they go out when that clears)
left out: 3
  1 x bad_number: The value is not a phone number (too short, too long or not digits).
  1 x duplicate_number: Another record in this list has the same number.
  1 x no_phone: There is no phone number on the record.
digest: 739c5ab31a39af54
```

- **The plan** (`plan_broadcast(queue, contacts, template)`) lists every recipient with the exact text
  they would get and every person left out with a reason: no number, not a phone number, on the
  suppression list, a duplicate of another record, a merge field the record lacks, a message that
  is too long, or a time zone that does not exist. It counts SMS segments from the *finished*
  messages (GSM-7 or UCS-2, extended characters and emoji counted as the carrier counts them), so
  one emoji that doubles the bill shows up before the send.
- **Fields.** `{first_name}` is the first word of `name` unless the record has its own `first_name`;
  any other `{field}` is the column of that name. A field with no value and no `--fallback` leaves that
  person out: "Hi !" is worse than no text. A field no record has at all blocks the plan (a typo).
- **Quiet hours** are named, not hidden: the plan says how many messages the gate would hold at the
  moment you ask (`--at`), judged on each person's own `timezone` column when there is one, and how many
  would wait for tomorrow because of the daily cap. They are held and go out later; nobody is
  dropped for them.
- **Fail closed.** If the suppression list cannot be read for even one number, the *whole* plan is
  blocked, not shortened.
- **The digest** is a fingerprint of exactly who gets exactly what. Show the plan to a person, keep
  the digest they approved, plan again at the moment you submit, and `submit_broadcast` refuses unless
  *that* plan has the digest. The Python function checks the plan it is handed (so pass a new one;
  the old object proves nothing); `broadcast submit` plans afresh every time.
  The list moves between approval and send; the person approved a specific message to specific people,
  not a query.
- **Submit** creates one approved task per recipient, signed with `--by`, and from there it is the
  queue: an opt-out that arrives after the plan is honoured at send time, and a task is saved as
  `sending` before the provider is called. Run `submit` again after a crash and it adds only the
  missing tasks (their ids come from the broadcast id and the number), so nobody is queued twice. A
  broadcast id cannot be reused for a different list or message. `--send-at` holds the whole
  broadcast until a time.
- **A provider that is down.** `queue sweep` stops after 5 failed sends in a row and says so
  (`--halt-after N` changes the number, `--halt-after 0` turns it off). From Python,
  `Queue.sweep(halt_after_failures=5)` does the same, and without it a sweep tries everything. The
  tasks it did not reach stay approved for the next sweep; they are not marked failed for a problem
  that is not theirs. The few that did fail before the halt stay `failed`, and nothing retries them
  by itself, since a provider can take a message and lose the answer.
- **Trying the failed ones again.** After the outage is fixed (and the provider's log shows those
  messages did not go), `broadcast retry-failed spring --by dana` (`retry_failed_broadcast`) approves
  again only that broadcast's failed tasks and then `queue sweep` sends them. Their ids do not change,
  so nobody is texted twice through the broadcast, and every rule, an opt-out since then included, is
  judged again at send time.
- **The run journal** is the queue itself: `broadcast status ID` (`broadcast_report`) counts the
  tasks by status, one per number, so you can see who was sent, who is waiting, who failed and
  whether a crash left any in doubt.

This module cannot send: it imports no provider and makes no network call, and a test keeps it that
way. One text per recipient, SMS only.

### The gate on its own

`check` is a pure function. It reads nothing and sends nothing.

```python
from datetime import datetime, timezone
from cleartosend import Policy, Task, check

policy = Policy(timezone="America/Chicago", channels={"sms"})
task = Task.new("sms", "+15551230000", "Your order shipped.", status="approved")
decision = check(task, policy, datetime(2026, 6, 23, 3, 30, tzinfo=timezone.utc), suppressed=False, sent_today=0)

decision.ok          # False
decision.reasons     # ("quiet_hours",)   10:30 pm in Chicago
decision.explain()   # ["quiet_hours: it is quiet hours for the recipient"]
```

`now` must carry a time zone. `suppressed` is `True`, `False`, or `None` when the list could not be
read (which fails closed). `sent_today` is how many messages the channel has sent today on the
policy's clock. `Queue.check(task)` looks both up for you.

Reasons, all named when they apply:

| Reason | Means | Kind |
|---|---|---|
| `not_approved` | Nobody approved it and `auto_send` is off. | waits for a person |
| `channel_off` | The channel is not in the policy's `channels`. | held |
| `suppressed` | The recipient is on the suppression list. | final |
| `suppression_unreadable` | The list could not be read, so the recipient is treated as suppressed. | held |
| `quiet_hours` | Inside quiet hours on the recipient's clock. | held |
| `calling_hours` | A voice call outside calling hours. | held |
| `daily_cap` | The channel has reached its cap for today. | held |
| `scheduled` | `send_at` has not come yet. The sweep leaves the task as it is. | held |
| `stale` | Unsent for longer than the policy's `stale_after_hours` after its `send_at`. A person decides. | held |
| `bad_send_at` | The stored `send_at` is not a readable time, so the task is held, not sent at a guess. | held |
| `sender_not_configured` | Added by the sweep, not by `check`: no provider is set up for the channel yet. | held |
| `no_recipient`, `bad_recipient`, `unknown_channel`, `bad_timezone` | The task itself is unusable. | final |
| `closed` | Already sending, sent, failed or skipped. | |

### The policy

One explicit object, built in code or read from a JSON file (TOML on Python 3.11+). Unknown keys are
errors that name the key, so a typo cannot quietly switch a rule off. `examples/policy.json` and
`examples/policy.toml` are the same policy.

```json
{
  "timezone": "America/New_York",
  "channels": ["sms", "email", "voice"],
  "auto_send": false,
  "daily_caps": {"sms": 200, "email": 500, "voice": 50},
  "quiet_hours": {"start": "21:00", "end": "08:00", "channels": ["sms", "voice"]},
  "calling_hours": {"start": "09:00", "end": "20:00"}
}
```

| Key | Meaning | Default |
|---|---|---|
| `timezone` | **Required.** An IANA name such as `America/Chicago`. Whose clock quiet hours, calling hours and the daily cap run on. | none: refuses to start |
| `channels` | The channels that may send: `sms`, `email`, `voice`. | none: nothing is enabled |
| `auto_send` | `true` lets the sweep send pending tasks without a person approving each one. Turn it off and any task that was only held under `auto_send` goes back to `pending`, to wait for a person. | `false` |
| `daily_caps` | Messages per channel per day on the policy's clock. `0` pauses the channel. | 100 each |
| `quiet_hours` | `start`/`end` in 24-hour time; `start` later than `end` wraps midnight. `channels` limits which channels it applies to. `null` turns it off. | 21:00 to 08:00, every channel |
| `calling_hours` | When a voice call may be placed. Must not wrap midnight. | 09:00 to 20:00 |
| `stale_after_hours` | Hours after a task could first have gone (its `send_at`, or the end of quiet or calling hours covering it) beyond which it is held for a person instead of sent. `null` turns it off. | off |

In code: `Policy(timezone="America/Chicago", channels={"sms"}, daily_caps={"sms": 200})`,
or `load_policy("policy.json")`, or `Policy.from_dict({...})`.

**The recipient's own clock.** Quiet hours and calling hours belong to the person receiving the
message. If you know their zone, pass it: `queue.enqueue("sms", to, body, timezone="America/Denver")`.
Without it, the policy's zone is used. cleartosend does not guess a zone from an area code.

### The suppression list

- Phone numbers are stored as digits with the country code, so `+1 (555) 123-0000`,
  `555-123-0000` and `15551230000` are one number. Ten bare digits get country code `1`
  (change it with `default_country_code=`). Email addresses are lowercased.
- One list covers every channel, but it is keyed by address and knows nothing about who owns one.
  A number that texted STOP is not texted or called; an address that unsubscribed is not emailed.
  To stop email to someone who texted STOP, add their email address too.
- A display name is ignored: `Pat <Pat@Example.com>` is `pat@example.com`.
- **An unreadable list is not an empty list.** If it cannot be read the gate says
  `suppression_unreadable` and the queue holds the task.

```bash
cleartosend suppress add "(555) 123-0000" --reason "asked by phone"
cleartosend suppress list
cleartosend suppress remove pat@example.com        # only with the person's consent
```

`Queue.sqlite(path, policy)` keeps the suppression list in the same file as the queue. To use your
own database, implement `SuppressionList` (four methods: `is_suppressed`, `add`, `remove`,
`entries`; `is_suppressed` must raise `SuppressionUnavailable` rather than answer `False` when it
cannot tell) and pass it to `Queue(store, policy, suppression)`.

### Reading replies

Call `handle_reply` from the webhook that receives your customers' inbound texts or email replies.

```python
queue.handle_reply("+15551230000", "STOP")                          # "stop": now suppressed
queue.handle_reply("+15551230000", "please take me off your list")  # "stop"
queue.handle_reply("+15551230000", "Can you stop the noise next door?")  # "": ordinary message
queue.handle_reply("+15551230000", "START")                         # "start": opt-out lifted
queue.handle_reply("+15551230000", "HELP")                          # "help": you answer with who you are
```

- **Carrier keywords** are exactly the standalone words carriers treat as commands: STOP, STOPALL,
  UNSUBSCRIBE, CANCEL, END, QUIT, OPT OUT (stop); START, UNSTOP, OPT IN, YES (start); HELP, INFO.
  REVOKE counts as a stop too: the FCC's 2024 revocation rules list it with the others.
  "Please stop by the office" is not a keyword.
- **Short messages.** A stop word plus politeness is a stop: "please stop", "Stop it", "STOP STOP",
  "stop all", "UNSUBSCRIBE please", "not interested, stop". Six words or fewer, and nothing else in them.
- **Sentences.** A person who wants out usually writes a sentence. It counts when the verb takes the
  sender or the messages as its object ("stop texting me", "take my number off", "don't contact me
  again", "I revoke my consent"), not when it is about something else ("end the lease early", "stop
  the leak", "remove my late fee", "don't call me before 5").
- **A bare YES does not lift an opt-out** unless you pass `yes_lifts_opt_out=True`. YES is also the
  commonest answer to "reply YES to confirm", and treating it as a fresh opt-in would re-subscribe
  a number that had opted out.
- Where it is unsure it records the opt-out: a wrongly recorded one costs one person their texts
  (they can reply START), a missed one messages someone who said stop.
- **A stop that cannot be recorded raises** `UnusableAddress` (a `ValueError`) naming the sender: a
  short code, or an address with no domain, cannot be put on the list. `handle_reply` never returns
  `"stop"` for an opt-out it did not write down, and `cleartosend reply` exits 2. A display name in
  an email `From` header is fine. Do not carry on as if the stop had been recorded.

The functions are also available alone: `classify_keyword(text)` and `asks_to_be_removed(text)`.

### Senders

A sender has one method, `send(task) -> (ok, error)`, and **never raises**: a network error, a
refusal or a bug comes back as `(False, "why")`, which the sweep records on the task.

| Sender | Provider | Needs |
|---|---|---|
| `TwilioSMS` | Twilio Messages API over HTTPS | account SID, auth token, and a from number or a Messaging Service SID |
| `TwilioVoice` | Twilio Calls API over HTTPS | the same. The task body is spoken, or is TwiML (starts with `<`), or an `https://` URL that returns TwiML. |
| `ResendEmail` | Resend HTTP API | API key, from address |
| `SMTPEmail` | Any SMTP server (STARTTLS on 587, or implicit TLS on 465) | host, from address, optional login |
| `StdoutSender` | Prints instead of sending | nothing |
| `Router(sms=, email=, voice=)` | Sends each task to the sender for its channel | |

Each provider sender has `from_env()`, which reads these variables:

| Variable | Used by |
|---|---|
| `CLEARTOSEND_TWILIO_ACCOUNT_SID`, `CLEARTOSEND_TWILIO_AUTH_TOKEN`, `CLEARTOSEND_TWILIO_FROM` | `TwilioSMS`, `TwilioVoice` |
| `CLEARTOSEND_TWILIO_MESSAGING_SERVICE_SID` | `TwilioSMS`, optional: send through a Messaging Service (the usual setup for a registered A2P 10DLC sender) instead of a single number. A `CLEARTOSEND_TWILIO_FROM` that starts with `MG` is treated the same way. Voice calls still need a from number. |
| `CLEARTOSEND_TWILIO_TWIML_URL` | `TwilioVoice`, optional: what to play when a call has no body |
| `CLEARTOSEND_RESEND_API_KEY`, `CLEARTOSEND_EMAIL_FROM` | `ResendEmail` |
| `CLEARTOSEND_SMTP_HOST`, `_PORT`, `_USERNAME`, `_PASSWORD`, `_SECURITY` (`starttls`, `ssl` or `none`), and `CLEARTOSEND_EMAIL_FROM` | `SMTPEmail` |
| `CLEARTOSEND_DB`, `CLEARTOSEND_POLICY` | the command line: the queue file and the policy file |

`cleartosend queue sweep` (without `--sender stdout`) builds its senders from these variables: Twilio
if the account SID is set, Resend if its key is set, otherwise SMTP if its host is set.

**A provider that is not set up does not burn your queue.** Before it claims a task, the sweep asks
the sender `can_send(channel)`. `Router`, the Twilio, Resend and SMTP senders answer no while their
credentials are missing, and the task is *held* with the reason `sender_not_configured` (not
`failed`), so the next sweep sends it once the variables are set. `queue sweep` prints a note and
exits 1 when that happens. Your own sender can have a `can_send(channel)` method too; without one it
is assumed ready.

Any other provider is a class with a `send` method:

```python
class MyProvider:
    def send(self, task):                    # task.channel, task.to, task.body, task.subject, task.id
        ok = my_client.deliver(task.to, task.body, idempotency_key=task.id)
        return (True, None) if ok else (False, "my provider said no")

queue.sweep(Router(sms=MyProvider(), email=ResendEmail.from_env()))
```

Pass `task.id` to your provider as an idempotency key where it supports one (`ResendEmail` does).
Calling a sender's `send` yourself skips every rule; give senders to `Queue.sweep` and let it decide.

For email, pass `headers={"List-Unsubscribe": "<mailto:stop@example.com>"}` to `ResendEmail` or
`SMTPEmail` if your mail needs one. cleartosend does not add unsubscribe links or postal addresses
to message bodies.

### Finding the sends that skip the gate

```bash
cleartosend audit src/                      # exit 1 and file:line for each one; 0 when clean
cleartosend audit . --exclude tests --exclude scripts
cleartosend audit src/ --json
```

```
src/jobs/reminders.py:10:9: twilio-sdk: Twilio send outside the gate: client.messages.create(...)
1 send(s) outside the gate in 14 file(s)
```

It parses your Python with `ast`; it never imports or runs it. It recognises direct calls to the
Twilio (including `create_async` and Verify), Plivo, Telnyx, Bandwidth, Vonage and MessageBird SDKs; `smtplib`/`aiosmtplib` `sendmail` and
`send_message`; Django `send_mail`, `mail_admins` and `EmailMessage(...).send()`, and Flask-Mail,
fastapi-mail and yagmail sends; Amazon SES, SNS (texts) and
Pinpoint; the Resend, SendGrid, Postmark and Mailchimp Transactional SDKs; and strings holding the
send endpoint of those and of a few more REST APIs (Mailgun, Mailjet, Brevo, SparkPost, Sinch),
for code that posts with `requests`, `httpx` or `urllib`. SDK method names are matched only in a
file that imports that SDK, so an unrelated `client.messages.create` (an AI SDK, say) is not reported.

Two comments say a send is meant to be there. Both show up in a code review diff, which is the point:

```python
def send(self, task):  # cleartosend: sender
    ...                # a sender, reached only through Queue.sweep; everything inside is accepted

client.calls.create(...)  # cleartosend: allow paging our own on-call phone
```

An `allow` needs a reason; one without a reason is still reported. Exit codes: 0 nothing found, 1 a
send outside the gate, 2 a file could not be parsed or a path does not exist.

In your test suite:

```python
from cleartosend import assert_no_ungated_sends

def test_every_send_goes_through_the_gate():
    assert_no_ungated_sends("myapp", exclude=["tests"])     # fails with every file:line
```

To move a call to the gate, compare `examples/reminders_before.py` with `examples/reminders_after.py`:
the job stops calling the provider and calls `queue.enqueue` instead.

### Command line

```
cleartosend check    --channel sms|email|voice --to ADDRESS [--body ..] [--status approved] [--tz ZONE] [--at TIME] [--send-at TIME]
cleartosend queue    add [--send-at TIME] | list | scheduled | stuck | approve ID --by NAME | reject ID --by NAME
                     | resolve ID --sent|--failed --by NAME | retry ID --by NAME | sweep [--halt-after N]
cleartosend broadcast plan | submit --id ID --digest D --by NAME [--send-at TIME] | status ID
                     | retry-failed ID --by NAME
                     (plan and submit take --contacts FILE and --template TEXT or --template-file FILE)
cleartosend suppress add ADDRESS | remove ADDRESS | list
cleartosend reply    --from ADDRESS --text "..."
cleartosend audit    PATH... [--exclude GLOB]
```

`--db` (or `CLEARTOSEND_DB`, default `cleartosend.db`), `--policy` (or `CLEARTOSEND_POLICY`) and
`--json` work on every command. `--at` takes an ISO 8601 time; without an offset it is read on the
policy's clock. On `queue sweep`, `--at` works only with `--sender stdout` or `--dry-run`: a real
sweep is always judged at the current time, so the flag cannot be used to text people during quiet
hours. `queue sweep --dry-run` shows what would happen and changes nothing. `check` needs the queue
file, because that is where the suppression list is; if `--db` points at a file that does not exist
it exits 2 instead of judging against an empty list. Exit 2 is a
usage or configuration error, printed with the key at fault.

### Your own storage

`Queue.sqlite(path, policy)` is the default and is enough for one machine. To keep the queue in
another database, implement `QueueStore` (`add`, `get`, `transition`, `list`, `count_sent`,
`try_lock`, `release_lock`; the docstrings say what each must do) and pass it to
`Queue(store, policy, suppression)`. Two properties to keep. `transition` is a compare-and-set: it
changes a task only if its status is still one the caller expects, which stops two sweeps, or a sweep
and a person, from both winning. And `add` must raise `TaskExists` (a `ValueError`) for an id it
already holds and never overwrite it: an upserting `add` could turn a task a sweep has already
claimed back into a new pending one. `MemoryStore` and
`MemorySuppressionList` are in-memory versions for tests.

### What it does not do

Read this part before you rely on it.

- **It is not legal advice and not a promise of lawful sending.** Whether a message is lawful depends on
  what it says, who it is to, and what they agreed to. cleartosend checks the rules you configure;
  it cannot know whether they are the right rules for your business or your country.
- **It does not get or record consent.** It has no record of who agreed to what. Only put
  people on the queue you have permission to contact.
- **It does not register your numbers or brand** (A2P 10DLC or any local equivalent), check a
  do-not-call registry, or add opt-out wording, unsubscribe links or postal addresses to messages.
- **"Sent" means the provider accepted the message.** It does not track delivery, bounces or
  receipts, and it does not retry. A `failed` task stays failed until you queue it again.
- **Only what goes through the queue is protected.** The audit finds direct provider calls in
  Python source it can read. It cannot see other languages, calls built up dynamically, a provider it
  does not know, or a send hidden behind your own wrapper. Treat a clean audit as a good sign, not a proof.
- **Sentence-level opt-out detection is a set of English patterns.** It reads the common ways of
  saying stop, and leaves alone sentences about something else, but it will now and then record an
  opt-out it should not, or miss an unusual phrasing. Keep a person
  reading replies, and keep carrier-level STOP handling switched on at your provider as well.
- **Phone numbers are normalised, not validated.** Ten bare digits are assumed to be in the default
  country; a number that looks plausible can still be undeliverable.
- **Time zones are the ones you give it.** Without a per-task zone, the policy's zone stands in for
  the recipient's.
- **SQLite suits one machine.** The sweep lock and compare-and-set are safe across processes on the
  same file. For several machines, write a `QueueStore` on a shared database.
- **A broadcast is SMS to people you already hold the numbers and consent for.** There is no area-code
  guessing: quiet hours run on the policy's zone unless a record has its own `timezone`. Phone numbers
  are normalised, not looked up, so a landline is still planned and the provider refuses it later.
  Segment counts are the carrier's rules as published; a carrier can count differently.
- **`broadcast status` reads every task and keeps the ones tagged with the broadcast.** That suits
  queues of thousands, not millions. A sweep that finds tens of thousands of tasks waiting for a later
  time still reads past them to reach the ones that are due.
- **A scheduled task is only as punctual as your sweep.** It goes out at the first sweep after its time;
  run the sweep every few minutes. A broadcast that is interrupted is finished by running `submit`
  again (and sweeping), not by a background process.
- **A crash can leave a task in `sending`.** That is deliberate (it is how a double send is
  prevented), and it needs a person to settle it with `queue resolve`.

### Development

```bash
python3 -m pytest
```

The tests use fakes for every provider: no network, no account.

---

Docs mirror of the private repository Dominares-Tools/cleartosend; updated with each release.
