# Design notes

## The idea: one day, one line

Today is a single vertical line down the page.

- What I have finished sits above a **Now** mark, and the line is solid there.
- What is left hangs below it. Events sit at their times.
- The last thing on the line is the field where I add something.

The line is the only decoration. Everything else is type and space.

This is the third design. The first used warm cards, and the second tried to
look like a native Mac list. An in-between attempt at a code-editor look, with
panels and a monospace font, was thrown out because it was cluttered and hard
to read. What survived from each was a rule about what to leave out.

## Rules the interface follows

**Flat and sharp.** No cards, no rounded corners, no shadows. Regions are
separated by space or by one hairline.

**The Mac's own font, in four sizes.** 13px for labels, times and buttons.
15px for anything read or typed. 22px for a side heading. 34px for the page
title. Weight and colour do the rest. An earlier version used twelve sizes and
felt uneven without it being obvious why. The right column goes one step
further and uses a single size: a label, a day or a time is told apart from
what it describes by its colour, so every line of a list sits on the same
measure.

**One accent.** Indigo marks the present moment, the selection and keyboard
focus. Red appears only for something overdue. Everything else is ink and grey.

**Two marks.** A square is a task. A diamond is an event. They mean the same
thing everywhere: on the line, in the month, in search results. The add field's
mark turns into a square or a diamond as words are typed, so what is about to
be added is visible before Return. A third shape was considered for deadlines
and rejected, because three shapes need a legend and two do not.

**Words, not form fields.** A date reads "Sunday, 11:59 PM" and is changed by
typing `fri 5pm` or picking from a short list. There are no date pickers and
no boxed buttons in the details.

**Placeholders look like placeholders.** Prompt text is much fainter than real
text. In an early build the two were close enough that an empty field looked
filled in.

**It explains itself.** Each view says in one line what it holds. An empty page
says what would fill it. A one-page guide covers the rest.

## Two themes

**Nightfall** is true black with charcoal for the selected row.
**Daylight** is warm paper with dark brown ink. Both use the same indigo
accent. Daylight first used a caramel accent, which sat too close to the brown
text to stand out.

## Where things are

| Place | What it holds |
|---|---|
| Left column | Today, Upcoming, Anytime, then lists |
| Centre | The day's line, then "Needs a day" |
| Right column | The daily checklist, the week ahead and a note for the day, or the details of the selected item |
| Upcoming | The same dates as Rows, as seven Columns, or as a Month |
| Any day | The same line as Today, for any date. Clicking the date goes straight to any day; arrows beside it step a day. |
| Bottom | Save state, how much of today is done, and the next fixed thing |
| Cmd+K | Find anything, go anywhere, run any command |

Today, Upcoming and Anytime answer one question, *when*: today, a later date,
or no date yet. A task moves between them as it gets a day and the day arrives.

## Complaints that shaped it

Before settling the rules, complaints from two long online discussions about
to-do apps were collected. These five shaped the rules. Each one maps to
something the app does.

| Complaint, in short | What the app does |
|---|---|
| The backlog is in my face every day | Today shows only what was chosen for today. A missed task is one row under "Needs a day". |
| It shows me things I cannot do right now | The planned day and the deadline are separate. Anything not planned for today stays off the line. |
| I stopped looking at it and started missing appointments | The month writes out events and deadlines in full. Planned tasks are only a count. |
| Finishing something just hides it | Finished things stay on the day, crossed out, and the line fills in above Now. |
| The app wants me to work its way | Nothing is required. Type a line and press Return. |

One more came from my own reason for giving up on other apps: adding things
took effort. A syllabus already lists a term's deadlines, so several lines
pasted anywhere in the app are read as a list of dates. A syllabus or a
Canvas calendar file adds a term at once, after showing what was understood
and what was not.

## Small decisions that took a while

**"Let it go" became "Anytime".** The choices for a missed task used to be
"Do today", "Tomorrow" and "Let it go", with the first one in bold. It was
not clear what "Let it go" did, and the bold one looked like a different kind
of button. They are now three equal answers to one labelled question, each
named after where the task goes.

**Month sits inside Upcoming.** It used to be a separate page, which made the
Rows / Columns / Month switch feel like it sometimes navigated away. Now it is
a third layout of the same tab.

**An intro, but only on a real start.** I first decided against any opening
screen, because the app is ready in well under a second. I changed my mind
after noticing how rarely it actually starts: closing the window only hides
it. So starting it from nothing now shows two lines, "Your life." and then
"Organized.", each coming into focus on the bare page. They leave before the
day comes through, so words are never read over other words, and the page is
only put together at that moment so its own arrival is seen. Bringing the
window back shows the day at once, a click or a key press skips the intro,
and it does not play when macOS is set to reduce motion.

**Asking for a code without asking for trust.** The first screen a new person
sees asks for an access code. It holds a heading, the code, and one sentence:
"Only to confirm your copy, and only this once. Checked on this Mac. Nothing
is sent." That is the question anyone has when an app asks for something on
first launch. There is no button. A complete code is checked as it lands.

**Closing the window hides the app.** The Dock count, the reminders and the
global shortcut only work while the app is running, so the red button hides the
window and Cmd+Q quits.
