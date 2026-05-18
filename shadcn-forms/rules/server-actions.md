# Server Actions with next-safe-action

Use this pattern when the form submits to a Next.js Server Action. Validation runs on **both** client (react-hook-form + zod) and server (next-safe-action + zod). Server validation errors are automatically mapped to `formState.errors` by the adapter — no manual `setError` calls for field errors.

---

## One-time project setup

Create `lib/safe-action.ts` once per project:

```ts
import { createSafeActionClient } from "next-safe-action"

export const actionClient = createSafeActionClient({
  handleServerError(e) {
    console.error("Action error:", e.message)
    return "Something went wrong. Please try again."
  },
})
```

This file is shared by all actions in the project.

---

## File structure

```
lib/
  safe-action.ts              ← one-time setup
schemas/
  login.ts                    ← zod schema (must NOT be in the action file)
actions/
  login.ts                    ← imports schema and defines the action
components/forms/
  LoginForm.tsx               ← imports schema + action
```

> **Critical:** The zod schema MUST live in a separate file (e.g. `schemas/login.ts`), NOT inside the `actions/` file. Next.js treats every export from a `"use server"` file as a server action proxy. Non-function exports like zod schemas get serialized and break `zodResolver` on the client with "Invalid input: not a Zod schema".

---

## Schema file (`schemas/login.ts`)

```ts
import { z } from "zod"

export const loginSchema = z.object({
  email: z.string().email("Invalid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
})
```

## Action file (`actions/login.ts`)

```ts
"use server"

import { actionClient } from "@/lib/safe-action"
import { loginSchema } from "@/schemas/login"

export const loginAction = actionClient
  .inputSchema(loginSchema)
  .action(async ({ parsedInput }) => {
    const user = await db.user.findUnique({
      where: { email: parsedInput.email },
    })

    if (!user || !verifyPassword(parsedInput.password, user.passwordHash)) {
      throw new Error("Invalid email or password.")
    }

    return { userId: user.id }
  })
```

- The `"use server"` directive is required at the top of action files.
- `parsedInput` is fully typed — the schema has already been validated before this runs.
- Throw a regular `Error` for non-validation failures (wrong credentials, DB error). The `handleServerError` in `safe-action.ts` controls what message reaches the client.

---

## Form component

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useHookFormAction } from "@next-safe-action/adapter-react-hook-form/hooks"
import { Controller } from "react-hook-form"

import { Button } from "@/components/ui/button"
import { Field, FieldDescription, FieldGroup, FieldLabel } from "@/components/ui/field"
import { Input } from "@/components/ui/input"
import { loginAction } from "@/actions/login"
import { loginSchema } from "@/schemas/login"

// Do NOT declare `type FormValues` here — useHookFormAction infers types
// from the action and resolver automatically. The explicit alias is unused
// and will trigger a TS warning. (Client-only forms still need it for useForm<FormValues>.)

export function LoginForm() {
  const { form, action, handleSubmitWithAction } = useHookFormAction(
    loginAction,
    zodResolver(loginSchema)
  )

  const { register, formState: { errors, isSubmitting } } = form

  return (
    <form onSubmit={handleSubmitWithAction} noValidate>
      <FieldGroup>
        <Field data-invalid={!!errors.email}>
          <FieldLabel htmlFor="email">Email</FieldLabel>
          <Input id="email" type="email" aria-invalid={!!errors.email} {...form.register("email")} />
          {errors.email && (
            <FieldDescription className="text-destructive">{errors.email.message}</FieldDescription>
          )}
        </Field>

        <Field data-invalid={!!errors.password}>
          <FieldLabel htmlFor="password">Password</FieldLabel>
          <Input id="password" type="password" aria-invalid={!!errors.password} {...form.register("password")} />
          {errors.password && (
            <FieldDescription className="text-destructive">{errors.password.message}</FieldDescription>
          )}
        </Field>
      </FieldGroup>

      {action.result.serverError && (
        <p className="mt-2 text-sm text-destructive">{action.result.serverError}</p>
      )}

      <Button type="submit" disabled={isSubmitting} className="mt-6 w-full">
        {isSubmitting ? "Signing in…" : "Sign in"}
      </Button>
    </form>
  )
}
```

Key differences from client-only mode:
- Import `useHookFormAction` from `@next-safe-action/adapter-react-hook-form/hooks` (note the `/hooks` subpath)
- Call `useHookFormAction(action, zodResolver(schema))` — it returns `{ form, action, handleSubmitWithAction, resetFormAndAction }`
- Use `handleSubmitWithAction` as the form's `onSubmit` handler
- Import the zod schema from `schemas/` (NOT from the action file — see File structure above)
- Display `action.result.serverError` for non-field runtime errors
- Field-level server validation errors are **automatically mapped** to `formState.errors` by the adapter

---

## What NOT to do

**Incorrect — raw fetch inside handleSubmit:**

```tsx
async function onSubmit(data: FormValues) {
  const res = await fetch("/api/login", { method: "POST", body: JSON.stringify(data) })
  // ...
}
```

**Correct** — use a safe action. The schema validates on both sides; the adapter handles error mapping.

**Incorrect — manual setError for field errors when using the adapter:**

```tsx
const { execute, result } = useHookFormAction(form, loginAction)

if (result?.validationErrors?.email) {
  setError("email", { message: result.validationErrors.email._errors[0] })
}
```

**Correct** — the adapter does this automatically. Only handle `result.serverError` manually.
