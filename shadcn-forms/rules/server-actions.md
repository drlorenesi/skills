# Server Actions with next-safe-action

Every form in this project submits through a next-safe-action action. Validation runs on the **server** (always) and, in client-side mode, **also** in the browser via react-hook-form + `zodResolver`. This file covers the wiring for both modes; for next-safe-action depth (error shapes, `useStateAction` vs `useAction`, bind args, file uploads) **defer to the `safe-action-forms`, `safe-action-hooks`, and `safe-action-validation-errors` skills** rather than re-deriving it here.

---

## Reuse the existing client — don't recreate it

This project already defines its clients in `lib/safe-action.ts`:

```ts
// lib/safe-action.ts — already exists; import from here
export const actionClient = createSafeActionClient({ /* returns a generic error message */ })
export const diagnosticActionClient = createSafeActionClient({ /* passes the real error through */ })
```

Use **`actionClient`** for user-facing forms. Use `diagnosticActionClient` only for admin/debug tools where surfacing the raw error is intentional. If `lib/safe-action.ts` does not exist (a different project), create it once — defer to the **`safe-action-client`** skill.

---

## File layout (colocated under the route)

```
app/<ruta>/
  page.tsx
  <nombre>-form.tsx   form component
  actions.ts          "use server" — the action
  schema.ts           neutral — exported zod schema
```

> **Critical:** the schema lives in `schema.ts`, never in `actions.ts`. Next.js treats every export of a `"use server"` file as a server-action proxy, so a schema exported there breaks `zodResolver` on the client. See [schema.md](./schema.md).

---

## Schema (`app/<ruta>/schema.ts`)

```ts
import { z } from "zod"

export const loginSchema = z.object({
  email: z.email("Invalid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
})
```

---

## Action (`app/<ruta>/actions.ts`)

Same client, same schema — only the **terminal method** differs by mode: `.action()` for client-side (react-hook-form) forms, `.stateAction()` for server-side (native `<form action>`) forms.

```ts
"use server"

import { actionClient } from "@/lib/safe-action"
import { loginSchema } from "./schema"

export const loginAction = actionClient
  .inputSchema(loginSchema)
  .action(async ({ parsedInput }) => {
    const user = await findUser(parsedInput.email)
    if (!user || !verifyPassword(parsedInput.password, user.passwordHash)) {
      throw new Error("Invalid email or password.")
    }
    return { userId: user.id }
  })
```

For a **server-side** form, keep everything the same but end with `.stateAction()` instead of `.action()`:

```ts
export const loginAction = actionClient
  .inputSchema(loginSchema)
  .stateAction(async ({ parsedInput }) => { /* … */ return { ok: true } })
```

- `parsedInput` is fully typed and already validated before the body runs.
- Throw a plain `Error` for non-validation failures; `handleServerError` in `lib/safe-action.ts` decides what reaches the client (`actionClient` → generic; `diagnosticActionClient` → the real message).
- For **field-specific** server errors (e.g. "email already taken"), use `returnValidationErrors` — see the **`safe-action-validation-errors`** skill.

---

## Wiring the form

**Client-side** — `useHookFormAction(action, zodResolver(schema), props?)` returns `{ form, action, handleSubmitWithAction, resetFormAndAction }`:

```tsx
"use client"
import { zodResolver } from "@hookform/resolvers/zod"
import { useHookFormAction } from "@next-safe-action/adapter-react-hook-form/hooks"
import { loginAction } from "./actions"
import { loginSchema } from "./schema"

const { form, action, handleSubmitWithAction } = useHookFormAction(
  loginAction,
  zodResolver(loginSchema),
  { formProps: { mode: "onTouched", defaultValues: { email: "", password: "" } } },
)
// <form onSubmit={handleSubmitWithAction}> · {...form.register("email")} · errors auto-mapped · action.result.serverError · disable on action.isPending
```

> The adapter expects the **default (formatted)** validation-error shape — don't set `defaultValidationErrorShape: "flattened"` on a client used with `useHookFormAction`.

**Server-side** — `loginAction` defined with `.stateAction()`; `useStateAction` gives a `formAction` for `<form action={formAction}>`:

```tsx
"use client"
import { useStateAction } from "next-safe-action/hooks"
import { loginAction } from "./actions"

const { formAction, result, isPending } = useStateAction(loginAction)
// <form action={formAction}> · uncontrolled inputs with name= · result.validationErrors?.field?._errors · disable on isPending
```

Full field markup for both modes is in [SKILL.md](../SKILL.md) (Mode A / Mode B).

---

## What NOT to do

**Don't** post with a raw `fetch` inside `onSubmit` — use a safe action; the schema then validates on both sides and the adapter maps errors for you:

```tsx
// ❌
async function onSubmit(data) {
  await fetch("/api/login", { method: "POST", body: JSON.stringify(data) })
}
```

**Don't** call `useHookFormAction(form, action)` — the adapter **creates** the form. The signature is `useHookFormAction(action, zodResolver(schema), props?)`.

**Don't** manually map field validation errors in client mode — the adapter already does:

```tsx
// ❌ unnecessary — these are already on form.formState.errors
if (action.result.validationErrors?.email) {
  setError("email", { message: action.result.validationErrors.email._errors[0] })
}
```

**Don't** export the zod schema from `actions.ts` — keep it in `schema.ts`.
