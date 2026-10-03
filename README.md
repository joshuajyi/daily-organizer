# Daily Organizer

<img src="docs/images/icon.png" width="96" alt="Daily Organizer icon: one vertical line, white above, indigo in the middle, grey below">

Daily Organizer is a Mac app I made for my own assignments, deadlines, events
and daily routines. I had given up on other organizers for the same three
reasons each time: adding things took too much effort, the app never felt like
mine, and I forgot it existed. I wanted to see whether a different design
could remove those problems instead of working around them.

The main idea is that **today is one line**. What I have finished sits above a
mark for the present moment. What is left hangs below it, with events at their
times.

The app can:

- Read a line typed the way I would say it, such as `essay due friday 5pm`
- Keep the day I plan to work on something separate from the day it is due
- Repeat tasks and events, including "every other Friday" and "the fourth
  Wednesday of each month"
- Show what is coming as rows, as seven day columns, or as a month
- Keep things I do every day in their own row, out of the day's count
- Hold a missed task until I decide where it goes, without moving its deadline
- Schedule reminders with macOS so they still arrive after the app is quit
- Store everything in a local SQLite database, with daily backups

There are no accounts and no network requests. All data stays on the Mac.

## Screenshots

![Today in the Nightfall theme](docs/images/today-nightfall.png)

Today in Nightfall. A square is a task and a diamond is an event. The "Every
day" row above the line holds routines. "Needs a day" under the line holds
anything still open that is not planned for today.

![Today in the Daylight theme](docs/images/today-daylight.png)

The same day in Daylight.

| Month | Columns | Details |
|---|---|---|
| ![Upcoming as a month](docs/images/upcoming-month.png) | ![Upcoming as seven columns](docs/images/upcoming-columns.png) | ![Details beside the day](docs/images/details.png) |

![Search and commands](docs/images/search.png)

Cmd+K finds any task or event, jumps to a view, or runs a command.

The screenshots use example data, not my own tasks. They were taken from the
app's browser preview on Linux, where a stand-in font replaces the Mac system
font, so the installed app looks slightly different.

## What typing looks like

Adding something is one field. The app reads the date, the time, the repeat and
the list out of the sentence and shows what it understood before I press
Return.

| I type | It becomes |
|---|---|
| `start essay monday` | A task planned for Monday |
| `essay due friday oct 16 5pm` | A task due Friday, October 16 at 5:00 PM |
| `essay tomorrow due friday` | A task planned for tomorrow, due Friday |
| `call dentist tomorrow 3pm` | An event tomorrow at 3:00 PM |
| `robotics club every wednesday 5pm` | An event that repeats every Wednesday |
| `club meeting every fourth wednesday` | A task on the fourth Wednesday of each month |
| `laundry every other sunday` | A task every two weeks on Sunday |
| `vitamins every day` | A routine in the Every day row |
| `lab report due sunday 11:59pm school` | A task with a deadline, filed in my School list |
| `SAT prep` | A task called "SAT prep" (not Saturday) |

The last row matters as much as the others. A parser that guesses too eagerly
is worse than no parser, so short weekday names only count after a word like
`due`, `on` or `next`, and "weekly" only counts next to a real day or time.

This uses plain rules. There is no AI model and no network call in the app.

## The rules behind the screen

Most of the work was deciding what the app should refuse to do.

1. **Planned and due are different dates.** Moving the day I plan to do
   something never changes its deadline.
2. **Today only shows what I chose for today.** A missed task does not pile
   onto the line. It waits in "Needs a day" with one question, "Move to", and
   three equal answers: Today, Tomorrow, or Anytime.
3. **Repeats do not create a backlog.** Finishing one sets up the next. A
   repeating event I never ticked off moves to its next date by itself. A
   daily routine simply comes back each morning.
4. **Only two marks.** A square is a task and a diamond is an event, everywhere
   in the app. A dashed mark is a later occurrence of something that repeats.
5. **Dates are words.** In the details, a date reads "Sunday, 11:59 PM" and is
   changed by typing `fri 5pm`. There are no date pickers.

[docs/DESIGN.md](docs/DESIGN.md) explains these choices and the complaints
about other apps that led to them.

## How it is built

- **Tauri 2** wraps the app. It uses the Mac's own WebKit instead of bundling
  a browser, which keeps the app small.
- **React 19 and TypeScript** draw the interface. TypeScript runs in strict
  mode.
- **Rust with SQLite** stores the data. Every save is one transaction that
  first checks a revision number, so an out-of-date window cannot overwrite
  newer data. The previous 20 states are kept for recovery, and one backup
  file is written per day for seven days.
- **An Objective-C bridge** talks to macOS for notifications and opening at
  login. Reminders are handed to the system, so they fire after the app quits.
- **Node's built-in test runner** runs the unit tests with no extra framework.
- **Playwright** drives a real browser through the interface for the
  end-to-end check.

The date rules, the parser and the repeat rules are plain TypeScript functions
with no dependency on React or Tauri. That is what makes them testable without
opening a window.

[docs/ENGINEERING.md](docs/ENGINEERING.md) goes through the data model, the
time handling, the parser, repeats, storage and testing in more detail.

## Testing

| What | Count | Covers |
|---|---:|---|
| Date and repeat rules | 18 tests | Planned, due and reminder staying independent; daylight-saving gaps and repeats; month-end and leap-day repeats; intervals and "Nth weekday"; skipping; rejecting bad imports |
| Typed-line parser | 18 tests | Dates, times, lists and repeats in different word orders; words that must not be read as dates |
| Summaries and day files | 3 tests | The morning summary sentence and the Markdown page for a day |
| SQLite store (Rust) | 4 tests | Reopening, refusing a stale save, keeping the old state when a save is rejected, backup rotation |
| Interface check | 1 scripted run | A real browser is driven through adding, completing, repeats, all three Upcoming layouts, backup and restore, and layout at two window sizes |

The interface check also guards things that are easy to break without
noticing. It fails if a type size outside the app's fixed scale appears on
screen, or if the Rows / Columns / Month switch moves by even one pixel
between layouts.

Reminders are different. Whether macOS delivers an alert while the app is
closed can only be checked by hand on a Mac, so that is a written checklist
rather than an automated test.

## Problems I ran into

**A date that was read as two dates.** I typed `essay due friday oct 16` and
the preview said "Planned Oct 16, Due Oct 9". The parser had matched `friday`
and `oct 16` as two separate dates: it took "due friday" as the coming Friday
and treated "oct 16" as the day I planned to start. The fix was to treat a
weekday said together with a date as one date, and let the date decide when
the two disagree. That exact sentence is now a test.

**A heading that jumped sideways.** Switching Upcoming between Rows and
Columns moved the buttons at the top. There were two causes. The selected
word turns bold and gets wider, which nudged its neighbours, and a long Rows
list showed a scrollbar that took 15 pixels the short Columns view did not.
The fix reserves the bold width for every word and keeps the scrollbar's
space whether or not one is showing. The second cause only appears when macOS
is set to always show scrollbars, which it does when a mouse is connected, so
it never showed up in the automated check until I measured for it.

**An update that did not seem to arrive.** Closing the window hides the app
so that reminders and the Dock count keep working. That also meant installing
a new build kept showing the old one, because the old copy was still running.
The install script now asks the running copy to quit before replacing it.

## Limitations

- It runs on macOS only and is a personal build. It is not notarized, so it is
  not ready to hand to other people yet.
- There is no sync, no phone version and no calendar import.
- An alert at an exact time needs the Mac to be on and awake, and Focus modes
  can silence it.
- The parser understands English and US-style dates.
- "The last Friday of the month" is not supported yet. The first through
  fourth are.
- Opening at login, the global shortcut and the per-day Markdown files have
  had less testing than the rest of the app.

## Source

The source code is in a private repository. I am happy to walk through any
part of it. This repository holds the project description, the design and
engineering notes, and the screenshots.

## Development note

I built this with Claude as a coding assistant. My part was the product: 
deciding what the app is for, setting the rules above, going through two 
earlier designs before this one, and using each build on my Mac to find what 
was confusing or broken. The three problems described above all came from 
that testing. I am now working through the code myself so that I can extend 
it without help.

## References

- [Tauri 2 documentation](https://v2.tauri.app/)
- [SQLite: write-ahead logging](https://www.sqlite.org/wal.html)
- [Apple: UNUserNotificationCenter](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter)
- Two Hacker News discussions about why people give up on to-do apps:
  [one](https://news.ycombinator.com/item?id=44864134) and
  [two](https://news.ycombinator.com/item?id=28029809)
