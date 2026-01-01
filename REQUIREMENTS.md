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
   - Acceptance: While >0 and <60s remaining, display must not show `00:00` if seconds would change the minute. If minute granularity is chosen, it must be clear to users (e.g., show `00:00` only when minute boundary crossed) — see Timing rules.

2. `precise_hhmmss`
   - Description: Shows hours, minutes, and seconds as `HH:MM:SS`.
   - Precision: seconds.
   - Visual: updates every second, no ambiguous `00:00` minute edge cases.
   - Acceptance: When <60s remaining, seconds should count down (e.g., `00:00:59`). Expired displays show `00:00:00` or an explicit expired indicator.

3. `block_minutes`
   - Description: Visual grid/row of blocks representing minutes remaining; blocks update as minutes elapse and highlight last 5 minutes differently.
   - Precision: minute-granular blocks, with last-minute transitions animated.
   - Visual: color-coded blocks, animations on removal.
   - Acceptance: blocks count equals total minutes remaining; transitions animate when a minute elapses.


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
   - Toggling `24-hour time` updates the `current-time` element and other formatters to 12/24-hour representation immediately and persists after reload.
   - Enabling `Update page title` causes `document.title` to reflect the time remaining of the soonest timer; disabling restores the default title.


## Acceptance Criteria / Tests
Provide simple examples and expected outcomes.

- Example 1: Add a timer 00:00:59 from now using `precise_hhmmss`. Expected: display shows `00:00:59` and counts down to `00:00:00`, then shows expired state.
- Example 2: Add a timer 00:00:59 using `compact_mmhh` (minute precision). Expected: display shows `00:01` (or clearly indicates minute-granularity) until the minute boundary is reached; it must not misleadingly show `00:00` while seconds remain.
 - Example 2: Add a timer 00:00:59 using `compact_mmhh` (minute precision). Expected: the display may show `00:00` when less than a minute remains; the UI must visually indicate near-expiration (for example via the `almost-expired` or `expired-bg` styles) so the user is not misled about remaining time.
- Example 3: Switch global display to `block_minutes`. Existing timers update their rendered representation according to selection unless they have a per-timer override.
- Example 4: It's 13:00 (1:00 PM). User enters `05:00` (5:00 AM). The app prompts: "Entered time is earlier than now. Create a timer for tomorrow at 05:00?" If the user confirms, the timer is created for the next day (within 24 hours); if the user declines, the timer is not created.
- Example 5: User enters `5:00 PM` (12-hour format) while current time is 16:00; the app correctly parses to 17:00 today and creates a timer for today at 17:00 without a rollover prompt.
 - Example 6: Create a timer with name "Lunch break" and end time 13:00. Expected: the timer appears using the selected display and includes the name (truncated to 50 characters if necessary) in a readable position.

## Notes / Implementation Guidance (non-normative)
- Maintain a single `now` tick and pass it to all formatters and state checks to keep rendering consistent.
- Prefer keeping formatters pure (input: `timer`, `now`, `displayOptions` -> output: string or UI data) to make multiple displays pluggable.


---

Please review and tell me any changes you want (additional displays, different rollover behavior, per-timer defaults, or stricter acceptance tests).