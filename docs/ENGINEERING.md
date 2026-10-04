# Engineering notes

These notes describe how Daily Organizer works underneath. They are written so
that someone who has not seen the source can follow the decisions.

## Shape of the app

```text
Interface (React, TypeScript)
  one file per page, rows, the details column, Cmd+K
        |
App state (React hooks)
  where you are, every action with its Undo, the keyboard
        |
Rules (plain TypeScript, no React, no Tauri)
  dates and time zones, repeats, the typed-line parser, the importer,
  undo history, validation
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
2. Find a repeat phrase that starts with `every`, and a last day after
   `until` if one follows it.
3. Find a stretch of time, `6-8:45pm` or `2 to 3:30pm`, and keep its end.
4. Find every date and time. Each one is either attached to the word before it
   (`due`, `by`, `on`) or stands alone.
5. Whatever was matched is cut out. What remains is the title.
6. Decide the kind. A time with no `due` makes an event. Otherwise it is a
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
- In `6-8:45pm` the "pm" belongs to both ends. In `11-1pm` it cannot, because
  that would end before it starts, so the start is taken as the morning.

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

**A last day.** A repeat can be given the last day it may fall on. Because
only one occurrence is stored, ending a series is a comparison made in the
four places that compute a next date: showing later occurrences, finishing
one, skipping one, and the app moving a passed event forward. Each asks the
same one-line question, "is the next date past the last day?", and stops if
so. A repeating event that has run out is marked done by itself, so it leaves
the day without anything being deleted.

**What happens when one is missed** depends on what it is:

| Kind | If I do nothing |
|---|---|
| Repeating event (a class) | Moves to its next date by itself |
| Daily checklist item (vitamins) | Comes back the next morning |
| Repeating task (vacuum weekly) | Waits under "Needs a day", where I can do it or skip it |

The reasoning is that nobody ticks off every class they attended, and nobody
wants yesterday's vitamins twice, but a chore that was skipped usually still
needs doing.

## The daily checklist

A checklist item is not a new kind of thing. It is a task that repeats every
day, and the interface lists such tasks beside the line instead of on it. That
reuse meant the checklist needed only two additions to the repeat rules.

**Unticking.** Finishing a repeating task normally cannot be undone, because
the next occurrence already exists and reopening would leave two. A checklist
gets ticked by mistake often enough that this was not acceptable. Unticking
finds tomorrow's copy and removes it, so there is exactly one item again.
Renaming or deleting a ticked item reaches that copy too. Otherwise the old
name, or the deleted item, would be back the next morning.

**Order.** Each day's copy keeps the moment the item was first added. The
list is sorted by that, so it stays in the order it was written no matter
which items were ticked or when.

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

## Bringing dates in

Pasting a syllabus, or choosing a calendar file, adds a term's deadlines at
once. Both readers are plain functions that take text and return a list of
what was found and a list of what was not understood. The dialog only shows
those two lists.

**Pasted text** is read a line at a time with the same parser as the add
field, after each line is cleaned up: bullets and numbering removed, brackets
opened, "Due:" turned into "due", a time zone abbreviation dropped. Then the
import adds three rules of its own, because a schedule is not a line typed
now.

- A date with no year means the nearest one. Typed today, "Sep 15" means next
  September. In a syllabus pasted in October it means three weeks ago.
- A bare date is a deadline if the line names work ("homework", "report",
  "due"), and an event if it names somewhere to be ("exam", "quiz"). The words
  decide, not the position of the date.
- "Sep 15: Homework 1 due 11:59 PM" has the time after "due" and the date
  before it. In the add field a lone "due 11:59 PM" means today. In a
  schedule it belongs to the date on the same line.

Two shapes need more than one line, and I only learned that from pasting a
real syllabus.

- **A schedule copied out of a table.** Each cell arrives on its own line: a
  date alone, then the topic, a paragraph, and perhaps "HW4 – Oct 16". The
  reader collects consecutive lines that are nothing but a date, and takes
  the first line after them as what those dates are for. An exam or a day
  off on two dates becomes two events. A chapter taught over three days is
  one topic, on the first.
- **A weekly class.** "Monday, Wednesday, 10:30 AM to 11:45 AM, Art Building
  133" has no date at all. Days followed by a time range are read as a
  weekly meeting, one event for each day, named by the short heading above
  ("Lecture", "Office Hours") and carrying what follows the times as its
  place. "2-3pm" starts in the afternoon and "11-1pm" in the morning.

- **The term's own dates.** A syllabus usually says once, near the top, when
  the term runs: "08/19/2026 to 12/07/2026". That line is not a thing to add,
  but it answers two questions nothing else on the page does. Weekly classes
  start no earlier than the first day and repeat until the last. And a date
  written without a year is placed inside the term, so "May 18" in a spring
  syllabus read the October before is next May, where the nearest-date rule
  alone would have called it five months ago. A span of a few days with
  years on it is a break, not a term, and is left alone.

A time with no day of its own means today in the add field. In a pasted page
it is a footer clock, so the import passes it over. And of the lines that
were not understood, only those that mention a date or a time are listed.
Listing every line of prose buried the two that mattered under two hundred
that did not.

A course code near the top of the text is put in front of each name and
offered as a new list, made in the same change as the items so that one Undo
removes both.

**Calendar files** follow the iCalendar format, which I parse directly: long
lines are unfolded, each property is split into its name, parameters and
value, and blocks nested inside an event (alarms) are skipped. The parts that
took care:

- **Three kinds of time.** A date alone is an all-day entry. A time ending in
  Z is UTC. A time with a named zone is wall-clock time there. Each becomes
  this Mac's day and time. Converting a wall-clock time in another zone to an
  instant uses the zone's offset at that moment, computed twice so a time just
  after a daylight-saving change is right.
- **Repeats the app can keep, and ones it cannot.** Weekly, monthly and yearly
  rules map onto the app's own. A class on Monday and Wednesday becomes two
  weekly events. "The last Friday" is not something the app can represent, so
  the first date is brought in alone and its note says it repeats in the
  original. A series that has already ended is left out.
- **The anchor.** Rent on the 31st, first paid in January, must be measured
  from January 31 even though the next one is in October. The import keeps
  the original first date as the anchor and only moves the visible occurrence.

Everything found is checked against what the app already holds, by name and
day, so the same file can be brought in again every week.

## Undo

The whole workspace is one value that is never changed in place. Every change
builds a new value that shares whatever did not change. That makes undo
almost free: a step back is the value from before, and a hundred steps cost
little more than one.

The work was in deciding what a step is. Details save on every keystroke, so
a naive history would undo one letter at a time. A change counts as typing
only if it altered nothing but the words of one thing: its name, its notes or
its place. More typing in the same thing within a second and a half joins
the same step. Anything else, such as finishing, moving, deleting or
importing, is always its own step, however fast it follows the last.

One kind of change is deliberately not recorded: the app moving a repeating
event on to its next date by itself. If it were, Undo would land on a state
the app immediately changes again, and Cmd+Z would appear to do nothing.

## Access codes

A copy that is handed out asks for a code the first time it opens. I wanted
that without breaking the promise that the app never uses the network, so the
check had to work with no server.

When I make codes, a script writes the codes themselves to a file that stays
on my Mac, and writes only their SHA-256 fingerprints into the app. A typed
code is hashed and looked up in that list. A fingerprint cannot be turned back
into a code, so having the app is not enough to make one. Each code is sixteen
characters from a 32-letter alphabet, which is 80 bits, far too many to guess.

The hash function is written out in the app, about sixty lines, instead of
calling the system's. That way the same code runs in the Mac app, in the
browser preview and in the tests, with no dependence on which web view
features are available. The tests compare it with Node's implementation for
every message length from 0 to 200 bytes, because the mistakes in a
hand-written SHA-256 are almost always in the padding at 55, 56, 63 and 64
bytes.

The alphabet leaves out I, L, O and U, and a typed O or I is read as 0 or 1,
so a code read aloud or copied by hand still works.

What it does not do: it cannot tell who is typing a code, so a code can be
shared, and I cannot cancel one without a new build. Someone determined could
also alter the app to skip the check. Fixing either would need a server, and
for an app whose point is that it stays on your Mac, I decided that was the
wrong trade.

## How the interface is organized

The interface began as one component and grew to 3,063 lines: every page,
every action and about forty pieces of state. It is now three folders with
one rule each.

| Folder | Rule | Holds |
|---|---|---|
| `app/` | Knows things, draws nothing | Where you are, every action with its Undo, the keyboard, the macOS hooks, and plain functions that work out what each page shows |
| `views/` | One file per page | Today, a day, Upcoming as rows, columns and month, a list, Reminders, Trash, the two side columns |
| `components/` | Knows nothing about the app | A row, the add field, the details, the month grid, Cmd+K. Everything arrives as arguments |

One hook builds a single object holding the workspace, the clock, the
navigation state and the actions, and hands it to every page through React
context. A page asks for what it needs by name. This is not finer-grained
than before, since any change still redraws the page, but the workspace is a
few hundred items and that was never the cost. The cost was that nobody
could find anything.

Two details kept behaviour the same. The right column is one element whose
contents change, as it was, so its width and scroll survive moving between
pages. And the functions that decide what a page holds (what is above and
below "now", the order of a day in Upcoming, the week ahead) take the
workspace and the time and return lists, so they are tested by calling them.

## Testing approach

The rules layer has 77 unit tests. They run in about a tenth of a second
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
- Read a syllabus written as paragraphs. The current reader needs a date on
  the same line as the thing it dates. Prose is the one place plain rules run
  out, and the only place I would consider a language model.
- Run the scripted browser check automatically on every change. The unit
  tests, the type check and the build are quick to run anywhere; the browser
  check measures layout, which depends on the fonts of the machine it runs
  on, so it is still run by hand.
- Sign and notarize the app so other students can install it.
