## Backend API contract

All paths are relative to `API_URL`. Django requires the **trailing slash** on every path.

**Auth is Djoser + SimpleJWT. The `Authorization` prefix is `JWT`, not `Bearer`:** `Authorization: JWT <access_token>`.

Auth routes:

- `POST /auth/users/` — register (`username`, `email`, `password`, `first_name`, `last_name`, `phone_number`). All are required. The phone number is an optional `+` followed by 7 to 15 digits; the backend rejects surrounding whitespace rather than stripping it. The frontend is stricter and sends E.164; see [Phone numbers](#phone-numbers). Email is unique: a taken email comes back as an error on `email`.
- `GET` / `PATCH` / `DELETE /auth/users/me/` — current user. `email`, `first_name` and `last_name` are editable. `username` and `phone_number` are read-only: they can't be changed through the API. `phone_number` is `null` for an account without a student profile; it is displayed as described under [Phone numbers](#phone-numbers). `DELETE` deletes the account together with the student's weekly classes and trial lesson.
- `POST /auth/jwt/create/` — log in with `username` + `password`; returns `access` and `refresh`
- `POST /auth/jwt/refresh/`, `POST /auth/jwt/verify/`. `verify` checks only a token's signature and expiry, so it can't tell whether a session survived a password change; don't use it for that.
- `POST /auth/users/set_password/` — change the password while logged in.
- `POST /auth/users/reset_password/` — takes `email` and **always answers 204**, whether or not the email is registered (a missing or malformed email is a 400). It is rate limited: after 5 requests per hour from one IP address the next one gets 429.
- `POST /auth/users/reset_password_confirm/` — takes `uid`, `token` and `new_password` and answers 204. There is no retype field on the backend. A wrong `uid`, a wrong, expired or already used `token`, or a password that fails the backend's validators is a 400 on that field. The emailed link points to `/reset-password/{uid}/{token}` on this app, is valid for one hour and works once.

There is no email-activation step: registration is followed immediately by login.

**A password change logs the user out everywhere.** After a reset or `set_password`, every JWT issued before the change is rejected with 401, refresh tokens included. [Auth and data access](auth.md) describes what the frontend does about it.

Resources:

- **Subjects** (read-only, **public**, no auth needed): `GET /subjects/`, `GET /subjects/{slug}/`. A subject has `id`, `name`, `slug`, `description`, two USD prices (`price_40_min` for a 40-minute class, `price_60_min` for a 60-minute class) and `levels`: a list of one or more objects with `code` and `name`. There are four levels, always returned in this order: `o_level`, `a_level`, `all_levels`, `university`. Display the `name` the API returns; don't hardcode level labels. A price can be `0` (a free subject); show it, don't treat it as missing.
  - `slug` is always present and unique: lowercase letters and digits in groups separated by single hyphens (e.g. `a-level-maths`). It is the identifier in this site's URLs and does not change when the subject is renamed.
  - `description` is plain text exactly as typed in the Django admin, with line breaks, or `""` when blank. Render it as text, never as HTML.
  - One subject is retrieved by its slug (`GET /subjects/chemistry/`); an unknown slug is a 404. The numeric id does not work on this route (`/subjects/3/` is a 404). Bookings still send the subject's `id`.
- **Weekly classes** (full CRUD, scoped to the logged-in student): `/classes/`, `/classes/{id}/`. A booking has a subject, a level, a day of the week, a start time, and a duration of 40 or 60 minutes.
  - Requests send `subject` as the subject's id and `level` as the level's code (e.g. `"a_level"`). Responses return `subject` as an object with `id`, `name`, `slug` and `levels` (no description and no prices) and `level` as an object with `code` and `name`.
  - The day is a code, `monday` to `sunday`, returned as `day`, with its label as `day_display`. The day and time are UTC.
  - `/classes/` returns **no price**.
- **Trial lessons** (full CRUD, scoped to the logged-in student): `/trial-lessons/`, `/trial-lessons/{id}/`. A trial lesson has a subject and a level, sent and returned exactly as on a weekly class, and `starts_at`: one ISO 8601 date-time in UTC (e.g. `"2026-10-13T02:00:00Z"`), not a separate date and time. There is no duration or price field. It also carries a read-only completed flag, which only an admin can set. Editing or deleting a locked trial lesson (see [Booking rules](#booking-rules)) answers 403 with a message in `detail`.
- **My schedule**: `GET /schedule/` — read-only, with three parts:
  - `weekly_classes`: a list (empty when there are none). Each item has the fields it has on `/classes/` plus `price`, its own price for its duration.
  - `trial_lesson`: one object with the fields it has on `/trial-lessons/`, or `null` when the student has none. It is included even when it is completed or its date and time have passed.
  - `weekly_cost`: the sum of the `price` values; zero when there are no weekly classes.
  - The backend orders `weekly_classes` by UTC day and time; see [Timezones](#timezones) for why the frontend sorts again.

**No student profile.** Only a registered student has a student profile. A logged-in user without one (e.g. an admin account created in the Django admin) gets 403 on every `/classes/`, `/trial-lessons/` and `/schedule/` route. [Auth and data access](auth.md) describes how the UI handles it.

Only the field names above are documented. Everything else (e.g. the time and duration fields of a weekly class, the completed flag of a trial lesson, the payloads of `set_password` and of deleting the account, whether `price` and `weekly_cost` on `/schedule/` are numbers like the subject prices) is confirmed against the live API when the feature is specced. Never guess a field name. Define each type once in `lib/` and its zod schema once in `app/validationSchemas.ts` (see [Code organization](code-organization.md)) and reuse them.

### Money

Prices arrive as **JSON numbers** (`20.0`), not strings, so they don't carry two decimal places and are floating-point values in JavaScript. Never add, multiply or compare them as they are: convert each price to whole cents as soon as it is used in a calculation (`Math.round(price * 100)`) and do all arithmetic in cents. Prefer the backend's own figures: the price of a booked class and the weekly total both come from `/schedule/` (`price` on each class, `weekly_cost`). `/classes/` returns no price, so a live preview while booking uses the chosen subject's price for the chosen duration. If a client-side sum is unavoidable, sum the cents. Format for display with `Intl.NumberFormat` (USD), which always shows two decimal places (`$20.00`); never print the raw number.

### Booking rules

The backend enforces these. The UI should guide users toward valid choices, but **the backend is the source of truth: always display its validation errors clearly** (field errors next to the field, `non_field_errors`/`detail` in a form-level alert) and never assume a client-side check is enough.

- Classes and trial lessons start only on the full hour (4:00, 5:00, 6:00…). Time pickers offer full hours only.
- A timeslot (weekday + hour) holds only one booking **across all students**. Taken slots are rejected. There is no availability endpoint, so the UI can only pre-disable slots the student already holds; a clash with another student surfaces as a backend error on submit.
- A student can book several weekly classes of the same subject.
- A weekly class and a trial lesson each have exactly **one level**, and it must be one of the chosen subject's levels. The level picker offers only that subject's levels and is reset when the subject changes. The backend rejects a mismatch with an error on `level`, and checks it on every write, including an edit that changes only the subject or only the level.
- Trial lessons are 60 minutes and free. Each student gets **one** trial lesson. It can be edited or deleted only while it is not completed and its start time has not been reached; after that it is locked but still visible (read-only). An edit or delete of a locked trial lesson answers 403 with a message, which the UI shows. A locked trial lesson still counts as the student's one trial, so they can't book another; deleting an unlocked one frees them to book again.
- A trial lesson can't be booked in the past: a date and time that have already passed are rejected, on a new booking and on an edit. The picker offers future slots only.
- A weekly class can't take a weekday + hour occupied by an upcoming trial lesson (any student's; one whose start time has passed no longer blocks the slot), and a trial lesson can't take a time occupied by a weekly class or another trial lesson. Clashes are compared in UTC.
- Weekly cost = the sum of each weekly class's price for its duration. Trial lessons add nothing.

### Timezones

The backend stores times in UTC; the UI shows and collects them in the **student's local timezone** and converts at the boundary. Keep all conversion in one helper module in `lib/` rather than scattering `Date` maths through components. Things to get right:

- A weekly slot is a weekday + hour (the `day` code and the time), so converting can move it to the **previous or next weekday**. Convert the pair together, never the hour alone. A trial lesson is different: `starts_at` is one instant, converted directly.
- The backend orders the schedule's `weekly_classes` by UTC day and time. After converting, that order can be wrong locally (a class late on Sunday UTC can be Monday for the student), so **sort by local day and time** after converting; never rely on the order the API returns.
- "Full hour" is the backend's rule, in UTC. For students in a half-hour-offset zone (e.g. UTC+5:30) valid slots display as `:30`; offer the backend-valid slots converted to local time, not local full hours.
- A weekly slot fixed in UTC shifts by an hour locally across a DST change. Show the timezone next to times so this is not a surprise.
- Timezone-dependent output differs between server and browser. Render local times in Client Components (or after mount) to avoid hydration mismatches.

### Phone numbers

The backend's check is minimal (an optional `+` and 7 to 15 digits), so the frontend validates phone numbers strictly itself. The backend stays the source of truth: its errors on `phone_number` are still shown next to the field.

**Not installed yet; added in the `auth` feature.** Use the existing solution below; don't hand-roll country data, formatting or validation.

- **Component**: ReUI's phone input, from its shadcn registry. Add the registry to `components.json` (`"@reui": "https://reui.io/r/{style}/{name}.json"`; confirm the URL against ReUI's docs when installing), then run `npx shadcn@latest add @reui/phone-input`. It generates `components/reui/phone-input.tsx`, adds the shadcn `combobox`, `scroll-area` and `input` primitives, and installs `react-phone-number-input`. Its `base-vega` build depends only on those Base UI primitives. Don't use a phone input built on Radix Popover or `cmdk`.
- **Library**: `react-phone-number-input`, which wraps `libphonenumber-js`. Add `libphonenumber-js` as a direct dependency too, because the zod schema imports it.
- **Metadata**: use the **max** metadata everywhere (`react-phone-number-input/max`, `libphonenumber-js/max`). The default min set checks little more than length, which is not strict validation. Max is about 145 kB against 80 kB, so keep phone imports out of shared layouts; they should load only on routes that have a phone field.

**Input.**

- A country selector on the left of the number input. Its button shows the country's flag **and** dialling code (e.g. the US flag and `+1`). The list shows flag, country name and dialling code, and is searchable by country name **and** by dialling code.
- The default country is the United States (`US`, +1). No locale or IP detection.
- Flags are the library's SVG flags, never emoji: Windows doesn't render flag emoji.
- The generated component shows only the flag on the button and searches by name only. Make exactly these edits to `components/reui/phone-input.tsx`: show the dialling code on the button, match the dialling code in search, give the button an accessible name (e.g. "Country: United States, +1"), and import from `react-phone-number-input/max`. Note the edits in a comment at the top of the file so re-adding it through the CLI doesn't silently undo them.
- Design and accessibility rules apply as everywhere: a real `<label>` on the number input, the error tied to it with `aria-invalid` / `aria-describedby`, the selector fully keyboard operable, 44px touch targets, both themes checked.

**Validation.** One phone schema in `app/validationSchemas.ts`, reused by every form with a phone field.

- `isValidPhoneNumber` from `libphonenumber-js/max` is the check: the number must have a correct length and a valid format for its country, not just a plausible digit count. Any valid number is accepted, mobile or landline.
- `validatePhoneNumberLength` picks the message. Empty → "Enter your phone number"; too short or too long → say which; anything else → "Enter a valid phone number for the selected country".

**Wire format.** The form value is E.164 (e.g. `+923001234567`) and is sent unchanged as `phone_number`. A valid E.164 number always passes the backend's check.

**Display.** Wherever a phone number is shown read-only (e.g. `/account`), show the flag and the number in international format (`formatPhoneNumberIntl`, e.g. `+92 300 1234567`) with `tabular-nums`. The flag has a text alternative (the country name). `null` shows as "Not set". A stored value that can't be parsed (an admin can enter digits without a `+`) is shown as it is, without a flag. This is one small display component, placed by the rules under [Code organization](code-organization.md).
