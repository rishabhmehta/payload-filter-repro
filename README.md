# Payload list filter: switching operator from `exists` to `equals` on a relationship crashes the list view

Minimal reproduction, built from `create-payload-app -t blank` (Payload **3.90.1**, `@payloadcms/db-postgres` 3.90.1, Next 16.3.3).

## What's different from the blank template

- `src/collections/Posts.ts`: a `posts` collection with `versions: { drafts: true }` and one field, `authors`, a `hasMany` relationship to `users`.
- `src/payload.config.ts`: registers `Posts`.

Nothing else was changed.

## Steps

1. `pnpm install`
2. Set `DATABASE_URL` to a local Postgres database in `.env` (see `.env.example`), then `pnpm dev`.
3. Open http://localhost:3000/admin and create the first user. Create a second user under **Users**.
4. Open **Posts** > **Filters** > **Add Filter**.
5. Set the field to **Authors**, the operator to **exists**, and the value to **True**. The URL becomes `...where[or][0][and][0][authors][exists]=true`. This works.
6. Change the operator from **exists** to **equals**. Do not touch the value.

## Expected

The value is cleared when the operator changes, as it is for other incompatible values, and the list view keeps working.

## Actual

The value stays as the string `'true'`, so the URL becomes `...[authors][equals]=true`. The list view renders blank and the server throws:

```
Error: Failed query: select distinct "_posts_v"."id", ... from "_posts_v" left join "_posts_v_rels" ...
params: version.authors,true,NaN,10
caused by: error: invalid input syntax for type integer: "NaN"
```

Selecting **Authors** > **equals** directly and then picking a user works, because the value starts empty.

## Where it comes from

`packages/ui/src/elements/WhereBuilder/Condition/validOperators.js` declares `equals: 'any'`. `handleOperatorChange` in `Condition/index.tsx` keeps the current value whenever the new operator accepts `'any'`:

```js
const isValidValue =
  validOperatorValue === 'any' ||
  typeof value === validOperatorValue ||
  (validOperatorValue === 'boolean' && (value === 'true' || value === 'false'))
```

`exists` stores `'true'` or `'false'`, which passes the `'any'` check for `equals`. On a relationship field, that string is then cast to a number for the integer `users.id` column, which gives `NaN`, and Postgres rejects it.

The same code is in 3.90.1, the latest release at the time of writing. Related, already-fixed reports: #10648 (same steps, closed; the linked fix, #11080, covered changing the field rather than the operator), #11136 (added the operator reset), #14204 / #15766 (`NaN` from filter reset).

## Notes

- Tested on Postgres only. I have not tried MongoDB or SQLite.
- The crash is on the versions table (`_posts_v`) because drafts are enabled. I have not tested without `versions.drafts`.
