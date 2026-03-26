- Project: https://github.com/linuxcaffe/tw-ical-hook
- Issues:  https://github.com/linuxcaffe/tw-ical-hook/issues

# tw2ical

Keep your Taskwarrior tasks in sync with any calendar that speaks iCalendar.

## TL;DR

- Writes a VEVENT `.ics` for every task with a `due:` or `scheduled:` date — automatically, on add and modify
- Each date field generates its own calendar event: `[due] buy milk`, `[sched] write report`
- Event duration comes from `uda.duration` per task, or `default_duration` in config (default 1h)
- Tags and project become `CATEGORIES`, annotations become `DESCRIPTION`
- Completed tasks stay in the calendar as cancelled events; deleted tasks are removed
- Works with khal directly, or any CalDAV server (Radicale, Nextcloud, etc.) + DAVx⁵ on Android
- Bulk export/repair with `tw-ical-export.py` — run once to bootstrap, re-run to fix drift
- Requires Taskwarrior 2.6.0+, Python 3.6+

## Why this exists

Taskwarrior is where the tasks live, but it has no calendar view. Due dates and scheduled
dates sit invisibly in the database — nothing reminds you about them at 9am, nothing shows
them alongside meetings, nothing syncs them to your phone. The only way to see them is to
run `task next` and remember to look.

Calendar clients — khal on the desktop, any calendar app on Android via DAVx⁵ — understand
VEVENT, the calendar event component of the iCalendar standard. They display events by date,
send reminders, and sync across devices. But they need `.ics` files to work from.

The gap between Taskwarrior and the calendar world is a directory of `.ics` files. tw2ical
fills it. Every time you add or modify a task, the hook writes (or removes) the corresponding
`.ics`. Your calendar client picks it up immediately.

## What this means for you

Your due dates and scheduled blocks show up in `khal list` alongside everything else —
on your laptop, on your phone — without any extra steps after the initial setup. Tasks you
complete disappear from the active view. You stop context-switching between Taskwarrior and
your calendar to get a full picture of your day.

## Design

**One VEVENT per date field.** A task with `scheduled:monday` produces a `[sched]` event on
Monday. A task with `due:friday` produces a `[due]` event on Friday. A task with both
produces two events in the same `.ics` file. Tasks with neither date produce no `.ics`.

**Duration.** iCal events need a time span. tw2ical uses `uda.duration` if set on the task,
otherwise falls back to `default_duration` in the config (default `PT1H`). The `duration` UDA
is a general-purpose work-estimate field — useful beyond tw2ical for time tracking and
planning. Set it per-task: `task 42 modify duration:PT30M`.

**X-properties** carry Taskwarrior metadata for round-tripping: `X-TW-UUID`, `X-TW-DATE-TYPE`
(`SCHEDULED` or `DUE`), `X-TW-URGENCY`.

## Core concepts

**VEVENT** — the calendar event component of the iCalendar standard (RFC 5545). Each `.ics`
file contains one VCALENDAR with up to two VEVENTs (one per date field). All calendar clients
understand this format natively.

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

# register the duration UDA in taskwarrior
echo "uda.duration.type=duration" >> ~/.taskrc
echo "uda.duration.label=Duration" >> ~/.taskrc

# bootstrap existing tasks
python3 ~/.task/scripts/tw-ical-export.py
```

## Configuration

`~/.task/config/tw-ical.rc`:

```ini
# Directory for .ics files — serve this with Radicale or point khal here
ics_dir=~/.task/time/ics

# Default event duration (ISO 8601) — used when task has no uda.duration
# PT1H = 1 hour, PT30M = 30 min, P1D = all day
default_duration=PT1H

# Keep completed tasks as cancelled events (yes/no)
# 'no' deletes the .ics when a task is marked done
keep_completed=yes

# Retain completed tasks for this many days (0 = keep forever)
# Requires running tw-ical-export.py --clean periodically
completed_days=90
```

**khal config** (`~/.config/khal/config`):

```ini
[calendars]

[[tasks]]
path = ~/.task/time/ics
type = calendar

[locale]
local_timezone = America/Toronto   # set to your timezone
timeformat = %H:%M
dateformat = %m/%d/%Y
datetimeformat = %m/%d/%Y %H:%M

[default]
default_calendar = tasks
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
task add "buy coffee" due:tomorrow                   # → [due] event written
task add "write report" scheduled:monday due:friday  # → two events written
task 42 modify duration:PT2H                         # → event lengthened to 2h
task 43 done                                         # → event marked cancelled
task 44 delete                                       # → .ics removed
```

**Wire up Radicale to serve the vdir:**
```ini
# ~/.config/radicale/config
[storage]
filesystem_folder = ~/.task/time
```
Then point DAVx⁵ or khal at `http://localhost:5232`.

## Example workflow

1. Add a task with a scheduled block and a deadline:
   ```bash
   task add "write quarterly report" scheduled:monday due:friday project:work
   ```
2. The hook writes `~/.task/time/ics/<uuid>.ics` with two VEVENTs immediately.
3. `khal list` shows `[sched] write quarterly report` on Monday and `[due] write quarterly report` on Friday.
4. Radicale picks up the new file; DAVx⁵ syncs to Android within minutes.
5. When you finish: `task <id> done` — both events become cancelled (hidden from active view).

## Project status

Early release. Core hook functionality is working. khal integration works out of the box.
CalDAV server setup (Radicale + DAVx⁵) requires manual configuration.
Recurrence and task dependencies are not yet mapped to iCal equivalents.

## Further reading

- [iCalendar VEVENT spec (RFC 5545)](https://www.rfc-editor.org/rfc/rfc5545)
- [Radicale — lightweight CalDAV server](https://radicale.org/)
- [khal — CLI calendar for vdir/CalDAV](https://lostpackets.de/khal/)
- [vdirsyncer — sync CalDAV to local vdir](https://vdirsyncer.pimutils.org/)
- [DAVx⁵ — Android CalDAV/CardDAV sync](https://www.davx5.com/)

## Metadata

- License: MIT
- Language: Python 3
- Requires: Taskwarrior 2.6.0+, Python 3.6+
- Platforms: Linux
- Version: 0.1.1
- Author: linuxcaffe + Claude Sonnet 4.6
