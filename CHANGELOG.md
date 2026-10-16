# Changelog

## 1.1.2 (2026-10-05)

Tests only. No code change.

- Buyer test suite restored; no code change. More of the README-checking tests now ship and run in your copy. Version number only otherwise.

## 1.1.1 (2026-10-04)

Fixes from review of 1.1.0. No change to a queue file.

- A sweep from the command line now stops after 5 failed sends in a row by default (`--halt-after N`
  to change it, `--halt-after 0` to turn it off), as the README said it did. `Queue.sweep` still halts
  only when `halt_after_failures` is passed.
- New: a failed task can be tried again by a named person: `queue retry ID --by NAME` and
  `Queue.retry`; for a broadcast, `broadcast retry-failed ID --by NAME` and `retry_failed_broadcast`.
  Ids do not change, so nobody is texted twice, and every rule is judged again at send time.
- The sweep leaves a task waiting for a later `send_at` without re-reading it or looking up opt-outs
  and the daily count. 50,000 waiting tasks plus one due text: one sweep from 17 s to under a second.
  An opt-out that arrives before the time is now honoured when the time comes (skipped, never sent)
  rather than at the first sweep after the reply.
- `stale_after_hours` now counts from when a task could first have gone (its `send_at`, or the end
  of the quiet or calling hours covering it), so a text scheduled inside quiet hours is no longer
  held as stale for having waited out the night. New `sendable_from(task, policy)`.
- `queue stuck` lists a task with an unreadable `send_at` (`bad_send_at`).
- `queue sweep --dry-run` leaves out tasks waiting for a later time and says how many.
- `QueueStore.add` is documented to raise the new `TaskExists` (a `ValueError`) for a duplicate id and
  never overwrite; both bundled stores do, and `submit_broadcast` relies on that type rather than on
  error text.
- `submit_broadcast` and the README now say what it checks: the plan it is handed, so hand it one made
  at submit time (the command line does).
- README: examples that took a fixed date use a future time; the package metadata names no publisher.
- The README tests also pass on a README that has had a header and a trailing access section added.

## 1.1.0 (2026-10-04)

Send later, and a text to a whole list. Everything from 1.0.x works as before, and an existing
queue file is upgraded in place the first time it is opened.

Send later
- `Queue.enqueue(..., send_at=...)` and `cleartosend queue add --send-at`: hold a task until a time
  ("7am tomorrow, once"). An ISO string or an aware `datetime`; no offset means the policy's clock.
  A time in the past is refused, and so is one more than 90 days ahead (a typo guard).
- The gate gains `scheduled` (its time has not come), `bad_send_at` and `stale`. The sweep leaves a
  scheduled task untouched and counts it as `waiting`, so a long list scheduled for tomorrow is not
  rewritten by every sweep and does not crowd out tasks that are due.
- New policy key `stale_after_hours` (off by default): a task still unsent that long after its
  `send_at` is held for a person instead of going out late.
- `Queue.scheduled()` (the visible queue, soonest first) and `Queue.stuck()` (left in `sending`,
  failed, or stale, each with why), with `cleartosend queue scheduled` and `queue stuck`.
  `check --send-at` judges a task as if scheduled.
- `Queue.enqueue(task_id=...)` lets the caller choose the id, so adding the same task twice is an error.

A text to a whole list
- `plan_broadcast(queue, contacts, template)` and `cleartosend broadcast plan`: who would be reached,
  who would not and why (no number, not a number, opted out, duplicate, missing merge field, too long,
  unknown time zone), SMS segment counts (GSM-7 and UCS-2) from the finished messages, which messages
  quiet hours would hold at a given moment (on each contact's own `timezone` when given) and how many
  the daily cap would push to tomorrow. A suppression list that cannot be read blocks the whole plan.
- `submit_broadcast` and `broadcast submit`: refuses unless the plan's digest is the one a person
  approved, then queues one approved task per recipient. Running it again after a crash adds only the
  missing tasks; a broadcast id cannot be reused for a different list or message.
- `broadcast_report` and `broadcast status`: the run journal, read from the queue's tasks.
- `Queue.sweep(halt_after_failures=N)` and `queue sweep --halt-after N`: stop after N provider
  failures in a row; the rest stay approved for the next sweep. `SweepResult` gains `waiting` and `halted`.
- `sms_segments`, `merge_fields`, `render_body`, `read_contacts` are public.

Also
- The README opens with the pitch and a three-line example, and installs from inside the repository
  (`pip install .`). The 1.0.2 note above said this was already so; it is now.
- A queue-file upgrade test, and the CLI test that depended on the time of day no longer does.

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show cleartosend`, `cleartosend.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated (version 1.3).
- One public contact address in the licence and the README.

## 1.0.0 (2026-09-30)

First release.

- `Policy`: one explicit object for the rules (enabled channels, approval or `auto_send`, daily
  caps per channel, quiet hours that wrap midnight, calling hours for voice). The time zone is
  required and has no default. Built in code, or read from JSON (TOML on Python 3.11+); an unknown
  key is an error that names it.
- `check(task, policy, now)`: a pure gate that returns a `Decision` naming every rule that fails
  (`not_approved`, `channel_off`, `suppressed`, `suppression_unreadable`, `quiet_hours`,
  `calling_hours`, `daily_cap` and the unusable-task reasons). An unreadable suppression list fails
  closed.
- `Queue` over a pluggable store (SQLite by default, in memory for tests): `enqueue`, `approve`,
  `reject`, `list`, `resolve` and `sweep`. Statuses `pending`, `approved`, `held`, `sending`,
  `sent`, `failed`, `skipped`, each with a reason. A task is saved as `sending` before the provider
  is called, the sweep is locked, and status changes are compare-and-set, so a crash or two
  overlapping sweeps never send twice. Quiet hours hold a message instead of dropping it, and are
  judged at the time of each provider call. A provider that is not set up holds the task
  (`sender_not_configured`) instead of failing it. `enqueue` refuses a body that would be cut short.
- Suppression list: phone numbers normalised to digits with the country code, emails lowercased,
  one list for every channel, your own database through the `SuppressionList` protocol.
- Opt-out detection: carrier keywords as a standalone message (STOP, STOPALL, UNSUBSCRIBE, CANCEL,
  END, QUIT, OPT OUT; START, UNSTOP, OPT IN, YES; HELP, INFO) and sentence-level requests ("please
  take me off your list", "quit texting me") without false positives on sentences like "stop the
  noise next door" or "remove my late fee". Short messages ("please stop", "Stop it", "STOP STOP")
  and REVOKE / "I revoke my consent" count as stops. `handle_reply` updates the suppression list,
  accepts `Pat <pat@example.com>` as an address, and raises `UnusableAddress` rather than report a
  stop it could not record. A bare YES does not lift an opt-out unless asked to.
- Senders that never raise and return `(ok, error)`: Twilio SMS and voice over REST (no SDK), email
  through Resend's HTTP API or stdlib SMTP, `Router`, `StdoutSender`, and a `Sender` protocol for
  any other provider. Twilio SMS can send through a Messaging Service.
- `cleartosend audit PATH` and `assert_no_ungated_sends(...)` for pytest: a static (`ast`) scan for
  direct provider sends that skip the gate, reported as file:line, exit 1. `# cleartosend: sender`
  and `# cleartosend: allow <reason>` mark the sends that are meant to be there.
- Command line (`queue sweep --at` is refused with a real provider; `check` refuses a missing queue
  file instead of judging against an empty suppression list): `check`, `queue` (add, list, approve, reject, resolve, sweep), `suppress` (add,
  remove, list), `reply`, `audit`.
