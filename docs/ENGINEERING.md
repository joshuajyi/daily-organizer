# Engineering notes

These notes describe how Daily Organizer works underneath. They are written so
that someone who has not seen the source can follow the decisions.

## Shape of the app

```text
Interface (React, TypeScript)
  screens, rows, the details column, Cmd+K
        |
Rules (plain TypeScript, no React, no Tauri)
  dates and time zones, repeats, the typed-line parser, validation
        |
Save queue (TypeScript)
  one save at a time, newest state wins, retry on failure
        |
Tauri commands (Rust)
  load, save, export, notifications, open at login
        |                         |
SQLite store (Rust)        macOS bridge (Objective-C)
  one row, a revision,       UNUserNotificationCenter,
  recovery history,          login item
  daily backup files
```

The middle layer is the important one. Every rule about dates, repeats and
parsing is a plain function that takes values and returns values. None of them
touch the screen or the database, so they can be tested by calling them.

## One document, not many tables

The whole workspace (items, lists, day notes, settings) is saved as one JSON
document in a single SQLite row, next to a revision number.

A personal organizer holds a few thousand items at most, so there is nothing
to gain from normalizing it into tables, and a single document makes three
things simple:

- **Backup and restore** are the same document written to a file.
- **Undo** is keeping the previous value of an item in memory.
- **Adding a field** needs no migration. A new field is optional, and loading
  an older document fills in its default.

The cost is that every save writes the whole document. At this size that takes
well under a frame, and the store rejects anything over 10 MB.

## Saving safely

Three things can go wrong with saving, and each has its own guard.

**Two windows writing over each other.** Every save carries the revision the
window last saw. The store runs the save in one immediate transaction: read the
current revision, refuse if it differs, otherwise write and add one. An
out-of-date window gets an error and cannot overwrite newer data.

**Saves overlapping.** The interface never runs two saves at once. Changes go
into a queue that holds only the newest state. If a save is in flight when
another change arrives, the next save simply sends the latest state. If a save
fails, the state stays in the queue and the screen shows a Retry button. It
does not fall back to another storage method.

**A bad state reaching the disk.** The document is validated before every save
and after every load: types, lengths, date formats, references from items to
lists, and rules such as "a repeating task has no fixed deadline". The same
check runs on an imported backup, so a damaged file is refused before it can
replace anything.

The store also keeps the previous 20 states in a history table, and writes one
backup file per day, keeping seven. Backup files are written to a temporary
name, synced, then renamed, so a crash cannot leave a half-written file.

Quitting waits for the queue. Cmd+Q is intercepted, the interface finishes
saving, and only then does the app exit.

## Dates and times

This was the part with the most ways to be subtly wrong.

**Days and instants are different things.** "Planned for Monday" is a calendar
day. It should stay Monday if I fly to another time zone. "Due at 5 PM" is an
instant. It should stay the same moment. So a date in the app is a calendar
day, plus, only if it has a time, an absolute instant and the time zone it was
entered in.

**Day arithmetic uses calendar days.** Adding seven days by adding
7 x 24 hours breaks twice a year at daylight saving. All day arithmetic is done
on whole calendar days at noon UTC, where no day is ever 23 or 25 hours long.

**Daylight saving has two edge cases.** In spring, a time such as 2:30 AM does
not exist. The app refuses it and says why. In autumn, 1:30 AM happens twice.
The app takes the first one. When a repeating reminder lands in the spring
gap, it moves to the next hour that exists.

**Editing must not move things.** If I change the name of an item, its stored
instant is kept exactly. It is only recomputed when the date or time field
itself changes. Otherwise editing a title after travelling would shift the
time.

**An untimed deadline lasts all day.** "Due Friday" becomes overdue when
Saturday starts, not when Friday starts.

## The typed-line parser

The add field turns a sentence into a title, a kind (task or event), a planned
day, a deadline, a repeat and a list.

It works in passes over the sentence:

1. Find a list name at the end, or a `#tag` anywhere.
2. Find a repeat phrase that starts with `every`.
3. Find every date and time. Each one is either attached to the word before it
   (`due`, `by`, `on`) or stands alone.
4. Whatever was matched is cut out. What remains is the title.
5. Decide the kind. A time with no `due` makes an event. Otherwise it is a
   task.

The hard part is not recognizing dates. It is **not** recognizing things that
only look like dates:

- `SAT prep`, `sun screen`, `may need a calculator`: short weekday and month
  names are only dates after `due`, `by`, `on`, `next` or `this`, or when a day
  number follows.
- `read chapter 3`, `buy 2 notebooks`: a bare number is never a time. A time
  needs `am`, `pm` or a colon.
- `weekly review`: one-word repeats only count beside a real day or time.
- `watch the 3rd episode`: "the 3rd" is only a date after a weekday or `due`.

When the parser is unsure, it leaves the words in the title. A wrong guess
would silently schedule something on the wrong day, which is worse than making
me set the date by hand. The preview next to the field shows what was
understood, and clicking it adds the line exactly as typed.

A few decisions were about how people actually talk:

- A bare weekday means its next occurrence. Typed on a Friday, `friday` means
  a week from now.
- `3:30` with no am or pm means the afternoon for hours 1 through 7.
- `due midnight` means the end of that day, not its first minute.
- A weekday said with a date (`friday oct 16`) is one date, and the date wins
  if they disagree.

## Repeats

A repeat is stored as a pattern plus an **anchor**: the day the pattern is
measured from. The anchor supplies everything else. Its weekday gives "every
Wednesday". Its day of the month gives "monthly on the 28th". Its position in
the month gives "the fourth Wednesday".

**Month ends.** A monthly repeat anchored on January 31 goes to February 28,
then back to March 31. It does this by always computing from the anchor, never
from the previous occurrence. Computing from the previous one would drift to
the 28th forever.

**Only one occurrence is stored.** The app stores the current occurrence and
computes the rest when a screen needs them. Later occurrences are drawn with a
dashed mark, and nothing is saved for them. This avoids generating hundreds of
rows for "every day, forever", and it means changing the pattern changes every
future date at once.

**What happens when one is missed** depends on what it is:

| Kind | If I do nothing |
|---|---|
| Repeating event (a class) | Moves to its next date by itself |
| Daily routine (vitamins) | Comes back the next morning |
| Repeating task (vacuum weekly) | Waits under "Needs a day", where I can do it or skip it |

The reasoning is that nobody ticks off every class they attended, and nobody
wants yesterday's vitamins twice, but a chore that was skipped usually still
needs doing.

## One line for every day

Today was originally its own screen. It is now one case of a more general one:
a page that draws any date as a line. The three cases differ only in how the
line is drawn.

| Day | The line | Can add |
|---|---|---|
| Today | Solid above the Now mark, faint below | Yes |
| Ahead | Faint all the way, ending in the add field | Yes |
| Behind | Solid all the way, ending at the last thing | No, it is a record |

A past day shows what was finished that day and what was planned but not done.
Nothing is stored per day to make this work. The page is computed from the
same items, filtered by date, which is why a day page is always consistent
with Today, Upcoming and the month.

## Dragging to another day

In the Columns layout a task or an event can be dragged to another day. This
uses pointer events directly instead of the browser's drag-and-drop API. The
drag-and-drop API behaves differently between browsers and inside a desktop
web view, and it cannot be driven reliably by an automated test. With pointer
events the same code runs in the Mac app and in the test browser.

The details that make it feel right:

- A press only becomes a drag after the pointer has moved five pixels, so a
  click still opens the item.
- The row captures the pointer, so the drag keeps working when the pointer
  leaves the row or moves quickly.
- The click that follows a drag is swallowed. Otherwise dropping an item would
  also open it.
- Escape cancels and puts the item back.

What a drop means depends on what was dragged. An event keeps its time of day
and takes its reminder with it. A task keeps its deadline, because moving a
plan must never move a deadline. A deadline shown in a column cannot be
dragged at all, and neither can a later occurrence of something that repeats,
since it does not exist yet.

## Reminders

Reminders are handed to macOS rather than run by a timer in the app. After
each save, the app computes the full list of alerts that should exist and
gives it to the system. macOS then delivers them even if the app has quit.

Sending the whole list each time, instead of adding and removing single
alerts, means the system's list cannot drift from the saved data. Completing
or deleting an item removes its alert on the next save without any special
case.

The app keeps at most 60 alerts pending at a time. If more are needed it shows
an error instead of claiming they are all scheduled.

## Testing approach

The rules layer has 40 unit tests. They run in about a tenth of a second
because they call functions directly. Tests that involve dates set a fixed
time zone and a fixed "today", so they give the same result on any machine on
any day.

The store has 4 tests in Rust against a real SQLite file in a temporary
folder.

The interface has one long scripted run in a real browser. It adds, edits,
completes, repeats, exports and restores, then checks the layout at two window
sizes. It also checks a few things that are design promises rather than
features:

- only the app's fixed type sizes appear on screen
- placeholder text is a different colour from real text
- the layout switch does not move when it is used
- nothing overflows sideways at the smallest window size

The interface check runs against a browser preview of the app, which stores
data in the browser instead of SQLite. That keeps it fast and independent of
macOS, but it means the native parts are outside it. Those are covered by a
written checklist that is followed by hand on the Mac.

## What I would do next

- Support "the last Friday of the month".
- Import deadlines from a pasted syllabus. This is the one job that plain
  rules cannot do well, and the only place I would consider a language model.
- Sign and notarize the app so other students can install it.
