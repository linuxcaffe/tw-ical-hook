- Project: https://github.com/linuxcaffe/tw-ical-hook
- Issues:  https://github.com/linuxcaffe/tw-ical-hook/issues

# tw2ical

Keep your Taskwarrior tasks in sync with any calendar that speaks iCalendar.

## TL;DR

- Writes a standard `.ics` VTODO file for every task — automatically, on add and modify
- `due:` dates appear as `DUE`, `scheduled:` dates as `DTSTART` — tasks with both span a range
- Tags and project become `CATEGORIES`, annotations become `DESCRIPTION`
- Completed tasks stay in the calendar as `COMPLETED`; deleted tasks are removed
- Works with any CalDAV server (Radicale, Nextcloud, etc.) and any iCal client
- Pairs naturally with [khal](https://lostpackets.de/khal/) (CLI calendar) and [DAVx⁵](https://www.davx5.com/) (Android sync)
- Bulk export/repair with `tw-ical-export.py` — run once to bootstrap, re-run to fix drift
- Requires Taskwarrior 2.6.0+, Python 3.6+

## Why this exists

Taskwarrior is where the tasks live, but it has no calendar view. Due dates and scheduled
dates sit invisibly in the database — nothing reminds you about them at 9am, nothing shows
them alongside meetings, nothing syncs them to your phone. The only way to see them is to
run `task next` and remember to look.

CalDAV clients — khal on the desktop, any calendar app on Android via DAVx⁵ — understand
VTODO, the task component of the iCalendar standard. They can display deadlines, show
scheduled work blocks, send reminders, and sync across devices. But they need `.ics` files
to work from.

The gap between Taskwarrior and the calendar world is a directory of `.ics` files. tw2ical
fills it. Every time you add or modify a task, the corresponding `.ics` is written. Your
CalDAV server serves that directory. Your calendar client connects to your CalDAV server.

## What this means for you

Your due dates and scheduled blocks show up in your calendar alongside everything else —
on your laptop, on your phone — without any extra steps after the initial setup. Tasks you
complete disappear from the active view. You stop context-switching between Taskwarrior and
your calendar to get a full picture of your day.

## Core concepts

**VTODO** — the task component of the iCalendar standard (RFC 5545). Each `.ics` file
contains one VTODO. CalDAV servers and calendar clients understand this format natively.

**vdir** — a directory of `.ics` files, one per entry. This is the format Radicale serves
and khal reads directly. tw2ical writes to a vdir.

**CalDAV** — the protocol calendar clients use to sync with a server. Radicale is a
lightweight self-hosted CalDAV server that can serve your vdir over the network.

**DAVx⁵** — the standard Android app for syncing CalDAV and CardDAV. Connects your Android
calendar app directly to Radicale (or Nextcloud, or any CalDAV server).

## Installation

### Option 1 — Install script
```bash
curl -fsSL https://raw.githubusercontent.com/linuxcaffe/tw-ical-hook/main/tw2ical.install | bash
```
Installs hooks to `~/.task/hooks/`, scripts to `~/.task/scripts/`, config to `~/.task/config/`.

### Option 2 — Via [awesome-taskwarrior](https://github.com/linuxcaffe/awesome-taskwarrior)
```bash
tw -I tw2ical
```

### Option 3 — Manual
```bash
# hooks
cp on-add_tw-ical.py on-modify_tw-ical.py ~/.task/hooks/
chmod +x ~/.task/hooks/on-add_tw-ical.py ~/.task/hooks/on-modify_tw-ical.py

# shared library + export script
cp tw_ical_lib.py tw-ical-export.py ~/.task/scripts/

# config (edit before first run)
cp tw-ical.rc ~/.task/config/

# bootstrap existing tasks
python3 ~/.task/scripts/tw-ical-export.py
```

## Configuration

`~/.task/config/tw-ical.rc`:

```ini
# Directory for .ics files — serve this with Radicale or point khal here
ics_dir=~/.task/time/ics

# Keep completed tasks as COMPLETED VTODOs (yes/no)
# 'no' deletes the .ics when a task is marked done
keep_completed=yes

# Retain completed tasks for this many days (0 = keep forever)
# Requires running tw-ical-export.py --clean periodically
completed_days=90
```

## Usage

**Bootstrap — export all existing tasks:**
```bash
python3 ~/.task/scripts/tw-ical-export.py           # pending tasks only
python3 ~/.task/scripts/tw-ical-export.py --all     # include completed/deleted
python3 ~/.task/scripts/tw-ical-export.py project:work  # any task filter
```

**Repair — remove .ics files with no matching task:**
```bash
python3 ~/.task/scripts/tw-ical-export.py --clean
```

**After that, the hooks keep everything in sync automatically:**
```bash
task add "buy coffee" due:tomorrow scheduled:today   # → .ics written immediately
task 42 done                                         # → .ics updated to COMPLETED
task 43 delete                                       # → .ics removed
task 44 modify description:"new wording"             # → .ics rewritten
```

**Wire up Radicale to serve the vdir:**
```ini
# ~/.config/radicale/config
[storage]
filesystem_folder = ~/.task/time
```
Then point DAVx⁵ or khal at `http://localhost:5232`.

**khal direct (no CalDAV server needed for local use):**
```ini
# ~/.config/khal/config
[[calendars]]
[[[tasks]]]
path = ~/.task/time/ics/
type = discover
```

## Example workflow

1. Add a task with a scheduled block and a deadline:
   ```bash
   task add "write quarterly report" scheduled:monday due:friday project:work
   ```
2. The hook writes `~/.task/time/ics/<uuid>.ics` immediately.
3. Radicale picks up the new file; DAVx⁵ syncs to Android within minutes.
4. Your phone calendar shows "write quarterly report" starting Monday, due Friday.
5. When you finish: `task <id> done` — the VTODO status changes to `COMPLETED`.
6. The task disappears from your active calendar view (filtered by your CalDAV client).

## Project status

⚠️ Early release. Core hook functionality is working. CalDAV server setup and khal
integration require manual configuration. Recurrence and task dependencies are not yet
mapped to iCal equivalents.

## Further reading

- [iCalendar VTODO spec (RFC 5545)](https://www.rfc-editor.org/rfc/rfc5545)
- [Radicale — lightweight CalDAV server](https://radicale.org/)
- [khal — CLI calendar for vdir/CalDAV](https://lostpackets.de/khal/)
- [vdirsyncer — sync CalDAV to local vdir](https://vdirsyncer.pimutils.org/)
- [DAVx⁵ — Android CalDAV/CardDAV sync](https://www.davx5.com/)

## Metadata

- License: MIT
- Language: Python 3
- Requires: Taskwarrior 2.6.0+, Python 3.6+
- Platforms: Linux
- Version: 0.1.0
- Author: linuxcaffe + Claude Sonnet 4.6
