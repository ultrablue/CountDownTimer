# CountDownTimer — Requirements

## Purpose
Provide a small, client-side countdown timer app that allows users to create timers (from a time-of-day input), persist them locally, and display remaining time using one of multiple selectable display types.

## Scope
- In-scope: creating timers for the current day (with an explicit, user-confirmed option to allow a timer into the next day up to 24 hours), persisting timers in `localStorage`, selecting between multiple display representations, updating displays in real time (1s tick), and dismissing timers.
- Out-of-scope: server-side persistence, authentication, multi-user sync. Changing the display type after a Timer has been instantiated is allowed but should follow stored `displayType` semantics.

## Display Types (Identifiers)
Describe each display type the app must support. Each display type needs a short identifier, description, precision, visual/UX notes, and acceptance criteria.

1. `compact_mmhh` (current default)
   - Description: Shows hours and minutes as `HH:MM` and a trailing hourglass icon.
   - Precision: minutes (no seconds).
   - Visual: monospace `HH:MM ⏳` large display.
   

2. `precise_hhmmss`
   - Description: Shows hours, minutes, and seconds as `HH:MM:SS`.
   - Precision: seconds.
   - Visual: updates every second, no ambiguous `00:00` minute edge cases.
   

3. `block_minutes`
   - Description: Visual grid/row of blocks representing minutes remaining; blocks update as minutes elapse and highlight last 5 minutes differently.
   - Precision: minute-granular blocks, with last-minute transitions animated.
   - Visual: color-coded blocks, animations on removal.
   


   - Display behavior: All display implementations must accept and be able to render a timer's `name` (if present) alongside the timing representation, in a way that maintains readability at the chosen font size and layout.
## Selection Behavior
- ? Support a global display choice (default) and optional per-timer override. Default for existing timers is the global choice unless a timer explicitly stores a `displayType` value.
- ? UI must present an accessible control to switch the global display and an optional per-timer selector when editing/creating a timer.

## Data Model & Storage
- Timer object shape (required fields):
   - `id` (number): unique id (ms epoch or UUID)
   - `until` (number): epoch milliseconds when the timer expires
   - `name` (string, optional but recommended): short label for the timer, max 50 characters. Display types must have access to this field to render a timer's name. The UI must validate and trim input to 50 characters; if empty, a default (e.g., "Timer") may be used.
   - `displayType` (string, optional): which display to use for this timer; falls back to global if absent
- Storage: `localStorage` under a single key (e.g., `timers`). Persist both timers and global settings (like `globalDisplayType`).
- Migration note: if changing the shape later, provide a simple migration strategy (detect old format, map to new fields).

## Timing Rules & Edge Cases
- Timing input mapping and same-day rule: by default, `type="time"` maps the provided time to the current date. If the computed `until` is earlier than or equal to `now`, the app must prompt the user: "The entered time is earlier than the current time — did you mean tomorrow at HH:MM?" If the user confirms, create the timer with `until` advanced by 24 hours (i.e., tomorrow at the entered time). If the user declines, do not create the timer and allow the user to edit the entered time.
- 12/24-hour input handling: the UI must accept both 12-hour (with AM/PM) and 24-hour time entries. The default display and input parsing should use 24-hour format. If the user enters a 12-hour format, the app must correctly parse AM/PM and convert to the same-day mapping above.
- All display logic must base rendering on a single authoritative `now` value updated centrally (1s tick). Use that `now` consistently across expiry checks and formatting to avoid visual inconsistency.
- Expired vs nearly-expired rules: define thresholds (e.g., `almostExpired = remaining >0 && remaining < 5 * 60 * 1000`).

## UX & Accessibility
- Focus: After adding or removing timers, focus must return to the time input.
- Confirmations: Dismissing an active (non-expired) timer requires confirmation.
- Accessibility: Buttons and dynamic regions must have ARIA labels; color-only states must be accompanied by text or icons for color-blind users.
- Contrast: Ensure color choices meet WCAG AA for foreground/background contrast.
 - Title update option: Provide an app-wide setting to optionally update the page `<title>` to reflect the remaining time of the timer that will end soonest. When enabled the title should show a compact, unambiguous representation of the shortest remaining timer (format chosen by global display precision rules) and fall back to the default app title when there are no timers. The option must be toggleable and persisted as part of global settings.
 - Current time display: Show the current local time in the upper-left area of the page. It should display hours and minutes only (no seconds) formatted according to the global 12/24-hour setting. The element must be easily style-able by developers (provide a stable DOM hook such as an id `current-time` or a class `current-time`) and be accessible (e.g., appropriate `aria-label`).

 - Title update option: Provide an app-wide setting to optionally update the page `<title>` to reflect the remaining time of the timer that will end soonest. When enabled the title MUST use the format: `Timer name - formatted time` whenever there is an active (non-expired) timer, or when the most recently dismissed timer is still visible in the UI. When no timers are visible, the title MUST read `CountDown Timer`. The time portion must obey the global precision/display rules (compact, precise, or block_minutes). The option must be toggleable and persisted as part of global settings.

 - Current time display: Show the current local time in the upper-left area of the page. It should display hours and minutes only (no seconds) formatted according to the global 12/24-hour setting. The element must be easily style-able by developers (provide a stable DOM hook such as an id `current-time` or a class `current-time`) and be accessible (e.g., appropriate `aria-label`).

## Settings UI (app-wide preferences)
The app must expose an easily discoverable settings control that allows users to change global preferences. Suggested pattern: a gear icon button that opens a modal dialog containing toggles and selectors for global settings.

- Entry point & DOM hooks:
   - Provide a gear button with id `settings-button` and `aria-label="Settings"`.
   - The dialog should be a focus-trapped modal with role `dialog` and id `settings-dialog`.

- Controls to include (initial set):
   - Checkbox `setting-time24` (label: "24-hour time") — default: checked (24-hour).
   - Checkbox `setting-update-title` (label: "Update page title with soonest timer") — default: unchecked.
   - Selector `setting-default-display` (label: "Default display") — optional: allows choosing the global default display type.

- Persistence and model:
   - Store settings under a single `localStorage` key (e.g., `countdownSettings` or `globalSettings`). Suggested shape:

```json
{
   "time24": true,
   "updateTitle": false,
   "defaultDisplay": "compact_mmhh"
}
```

   - Load settings at app initialization and expose them reactively to formatters and UI components.

- Accessibility & UX details:
   - The modal must be keyboard-accessible (focus trap, Esc to close, Save and Cancel buttons).
   - Each control must have an associated label and a help text where necessary (use `aria-describedby`).
   - Save applies changes and closes the dialog; Cancel reverts unsaved changes.

- Integration notes:
   - Formatters and current-time display must reference the global `time24` setting to format `HH:MM` consistently.
   - If `updateTitle` is enabled, the app should update `document.title` to reflect the soonest timer using the selected precision/format rules. When disabled, the app should restore the default title.
   - Use the stable DOM hooks (`settings-button`, `settings-dialog`, `setting-time24`, `setting-update-title`, `setting-default-display`) so developers can style or script the controls.

- Acceptance tests / examples:
  


## Acceptance Criteria / Tests
Each test below is a pass/fail criterion. Follow the Given/When/Then pattern and mark the test as PASS when the stated condition is met.

1. Given a timer created for 59 seconds from now using `precise_hhmmss`, when the timer runs, then the displayed value counts down each second from `00:00:59` to `00:00:00` and then shows an explicit expired indicator. Pass if the UI updates every second and the expired state is shown when remaining <= 0.

2. Given a timer created for 59 seconds from now using `compact_mmhh` (minute precision), when the timer runs, then the UI must not misleadingly present a full-minute zero remaining while seconds remain without an additional near-expiration indicator. Pass if either (a) the display holds the previous minute until the minute boundary (e.g., shows `00:01` until it becomes `00:00`), or (b) it shows `00:00` but also renders a clear near-expiration visual indicator (e.g., `almost-expired` style) while seconds remain.

3. Given existing timers and the user switches the global display to `block_minutes`, when the switch is applied, then timers without a per-timer `displayType` override must render using `block_minutes`. Pass if all unaffected timers change representation immediately and per-timer overrides remain unchanged.

4. Given the current time is later than an entered time-of-day (same-day mapping), when the user attempts to create a timer for that earlier time, then the app prompts: "The entered time is earlier than the current time — did you mean tomorrow at HH:MM?". Pass if the prompt appears and, upon confirmation, the timer is created with `until` advanced by 24 hours; if the user declines, the timer is not created.

5. Given a 12-hour formatted input like `5:00 PM`, when parsed while the clock is 16:00 (4:00 PM), then the app must interpret the input as 17:00 today and create a timer for today without a rollover prompt. Pass if the created timer's `until` corresponds to today at 17:00.

6. Given a user creates a timer with a `name` field (e.g., "Lunch break") and end time 13:00, when the timer is displayed, then the timer's rendered card must include the name (trimmed to 50 characters if necessary). Pass if the UI shows the provided name and truncation occurs when the length exceeds 50 characters.

7. Given the global setting `setting-update-title` is enabled and multiple timers exist, when the app updates titles, then `document.title` must reflect the remaining time of the soonest non-expired timer using the global precision rules (compact or precise). Pass if `document.title` matches the formatted remaining time for the soonest timer while it is not expired.

8. Given `setting-update-title` is enabled and the soonest timer expires while other timers remain, when the soonest timer reaches expiry, then the app must update `document.title` to reflect the next soonest non-expired timer (and must not display the expired timer). Pass if `document.title` changes from the expired timer's value to the next timer's formatted remaining time and never shows the expired timer as the current title once expired.

9. Given a timer rendered using `block_minutes`, when the timer has N whole minutes remaining, then the UI must display N minute blocks and visually distinguish the last five minutes. Pass if the number of rendered blocks equals the integer minute count of remaining time and the last five blocks are styled/highlighted differently; minute-elapse transitions must animate.

10. Given the settings control `setting-time24` is toggled, when the setting is saved and the page is reloaded, then the `current-time` element and other time formatters must reflect the chosen 12/24-hour representation immediately and persist across reloads. Pass if formatting updates immediately on save and persists after a full reload.

11. Given a user adds or removes a timer, when the operation completes, then focus must return to the time input control. Pass if focus is on the time input after add or remove completes (keyboard users can continue entering times without extra clicks).

12. Given an active (non-expired) timer and the user initiates a dismiss action, when the dismiss is requested, then the UI must require an explicit confirmation before removing the timer. Pass if a confirmation dialog appears and the timer is only removed after the user confirms.

13. Given `setting-update-title` is enabled and there is an active (non-expired) timer with the name "My Timer", when the app updates the page title, then `document.title` must equal `My Timer - <formatted time>` where `<formatted time>` matches the global display precision. Pass if `document.title` exactly equals the timer's name, a hyphen, a space, and the formatted remaining time.

14. Given `setting-update-title` is enabled and no active non-expired timers exist but the most recently dismissed timer is still rendered in the UI, when the app updates the page title, then `document.title` must equal `TimerName - <formatted time>` for that most recently dismissed visible timer. Pass if the title matches that timer's name and formatted time.

15. Given `setting-update-title` is enabled and no timers (active or visible) exist, when the app updates the page title, then `document.title` must equal `CountDown Timer`. Pass if the exact text `CountDown Timer` appears in the document title.

## Notes / Implementation Guidance (non-normative)
- Maintain a single `now` tick and pass it to all formatters and state checks to keep rendering consistent.
- Prefer keeping formatters pure (input: `timer`, `now`, `displayOptions` -> output: string or UI data) to make multiple displays pluggable.


---

Please review and tell me any changes you want (additional displays, different rollover behavior, per-timer defaults, or stricter acceptance tests).