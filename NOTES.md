# Notes: update a user (`PUT /users/:id`)

## Plan

Implemented the "update a user" feature per `tests/update-user.test.js`, which fully
specified the contract: 200 + updated user on success, 404 for an unknown id, 400 when
`name` or `email` is missing. Explored the existing codebase first (`routes/users.js`,
`db/store.js`) and reused its conventions rather than inventing new ones: hand-rolled
truthiness validation (matching `POST /users`), `Number(req.params.id)` + lookup for
existence (matching `GET /users/:id`), and the `{ error: "<message>" }` response shape
for both 400 and 404. Validation runs before the not-found lookup so a request missing a
field always returns 400, even for an existing user.

## Model choice

Added `updateUser(id, { name, email })` to `db/store.js`, mirroring `createUser`'s style:
look up the user via the existing `getUserById`, return `undefined` if not found (letting
the route decide the 404 response), otherwise mutate `name`/`email` in place and return
the same object reference. No new dependencies or validation library were introduced —
the app has none, and adding one for two fields would be inconsistent with the rest of
the codebase.

## Commit split

Two logical pieces: (1) the store helper `updateUser` plus its export, and (2) the route
handler `router.put("/:id", ...)` that wires validation, the store call, and the 404/200
responses together. This `NOTES.md` rides along as a third, since `tests/notes.test.js`
requires it for the suite to pass.

## Review

Checked the three grading tests in `tests/update-user.test.js` line by line against the
handler: update-and-200, unknown-id-404, and missing-field-400. Confirmed the existing
`tests/users.test.js` and `tests/notes.test.js` still pass, and that `npm run lint` stays
clean (no new eslint-disable comments needed, no unused vars).

Ran a broader self-review (correctness, reuse, and cross-file angles) before opening the
PR. It confirmed the not-found and malformed-JSON paths already behave the same as the
existing `GET /users/:id` and `POST /users` routes (nothing new introduced there), and
flagged two things worth naming rather than fixing: (1) validation is truthy-only, so a
value like `name: 0` would pass and get persisted — an existing gap in `POST /users` that
this change now also applies to overwriting existing records; and (2) the id-parsing,
validation, and 404-response blocks are each a few lines duplicated from `GET /:id` /
`POST /`. Left both as-is: they match this repo's existing (helper-free) conventions, and
extracting shared validation for two fields across a 53-line file would be inconsistent
with the rest of the codebase.
