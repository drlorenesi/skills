---
name: shadcn-forms
description: Build type-safe forms with shadcn/ui components, react-hook-form, and zod. Covers schema definition, form setup, field wiring, validation display, and submission patterns. Use whenever building or modifying a form in this project.
user-invocable: true
---

# shadcn Forms with react-hook-form + zod

Stack: **shadcn/ui** (Field/FieldGroup layout) + **react-hook-form** (state/submission) + **zod** (schema validation).

---

## Setup — ask first

Before writing any form code, ask the user two questions:

**1. Is this form client-only or server-connected?**

- **Client-only** — submits to an external API or authentication provider (e.g. Auth.js, Supabase Auth). Validation runs only in the browser.
- **Server action** — submits to a Next.js Server Action. Validation runs on both client and server.

**2. Are the required packages installed?**

Install based on the answer above:

```bash
# Client-only
npm install react-hook-form zod @hookform/resolvers

# Server action (includes client-only packages)
npm install react-hook-form zod @hookform/resolvers next-safe-action @next-safe-action/adapter-react-hook-form
```

If server actions are used for the first time in the project, also create `lib/safe-action.ts` (one-time setup — see [server-actions.md](./rules/server-actions.md)).

---

## Mode A — Client-only

Use when submitting to an external API or auth provider.

**Pattern:**
1. Define a zod schema (colocated in the form file)
2. Infer the TypeScript type from it
3. Call `useForm` with `zodResolver`
4. Spread `register()` (or use `Controller`) onto every field
5. Display errors from `formState.errors`
6. Call `handleSubmit(onSubmit)` on `<form onSubmit>`

```tsx
"use client"

import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Field, FieldGroup, FieldLabel, FieldDescription } from "@/components/ui/field"

const schema = z.object({
  email: z.string().email("Invalid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
})

type FormValues = z.infer<typeof schema>

export function LoginForm() {
  const {
    register,
    handleSubmit,
    setError,
    formState: { errors, isSubmitting },
  } = useForm<FormValues>({ resolver: zodResolver(schema) })

  async function onSubmit(data: FormValues) {
    try {
      await signIn(data)
    } catch {
      setError("root", { message: "Invalid email or password." })
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <FieldGroup>
        <Field data-invalid={!!errors.email}>
          <FieldLabel htmlFor="email">Email</FieldLabel>
          <Input id="email" type="email" aria-invalid={!!errors.email} {...register("email")} />
          {errors.email && (
            <FieldDescription className="text-destructive">{errors.email.message}</FieldDescription>
          )}
        </Field>

        <Field data-invalid={!!errors.password}>
          <FieldLabel htmlFor="password">Password</FieldLabel>
          <Input id="password" type="password" aria-invalid={!!errors.password} {...register("password")} />
          {errors.password && (
            <FieldDescription className="text-destructive">{errors.password.message}</FieldDescription>
          )}
        </Field>
      </FieldGroup>

      {errors.root && (
        <p className="text-sm text-destructive mt-2">{errors.root.message}</p>
      )}

      <Button type="submit" disabled={isSubmitting} className="mt-6 w-full">
        {isSubmitting ? "Signing in…" : "Sign in"}
      </Button>
    </form>
  )
}
```

**Files:** 1 — form component with colocated schema.

---

## Mode B — Server action

Use when submitting to a Next.js Server Action. Validation runs on both client and server. See [server-actions.md](./rules/server-actions.md) for the full pattern.

**Files:**
| Scenario | Files |
|---|---|
| New safe action | `lib/safe-action.ts` (one-time) + `schemas/[feature].ts` + `actions/[feature].ts` + form component |
| safe-action.ts already exists | `schemas/[feature].ts` + `actions/[feature].ts` + form component |

> **Always** put the zod schema in `schemas/`, never in `actions/`. Non-function exports from `"use server"` files are broken on the client (Next.js proxies them).

---

## Rules

- [schema.md](./rules/schema.md) — zod schema patterns
- [controller.md](./rules/controller.md) — when to use `Controller` vs `register`
- [validation.md](./rules/validation.md) — displaying errors correctly
- [server-actions.md](./rules/server-actions.md) — next-safe-action pattern

### Quick reference

- **Always use `zodResolver`** — never write manual validation logic.
- **`register()` for native inputs** (`Input`, `Textarea`, `NativeSelect`). Use `Controller` for controlled components (`Select`, `Checkbox`, `Switch`, `RadioGroup`, `Combobox`, `ToggleGroup`, `InputOTP`).
- **Pair `data-invalid` + `aria-invalid`** — `data-invalid` on `Field`, `aria-invalid` on the control.
- **Show errors via `FieldDescription`** — never a raw `<p>` or `<span>` outside the field.
- **Add `noValidate` to `<form>`** — prevents browser native validation from conflicting with zod.
- **Use `isSubmitting` to disable the submit button** — prevents double-submits.
- **Root/server errors**: `setError("root", { message: "…" })` for client-only; with `useHookFormAction` field errors are auto-mapped, only `result.serverError` needs manual handling.
- **`type FormValues`**: declare it in client-only forms (`useForm<FormValues>` requires it). Omit it in server-action forms — `useHookFormAction` infers types automatically; declaring it produces an unused-type TS warning.
- **`autoComplete`**: add it only where the browser has a standard token — `"name"`, `"email"`, `"new-password"`, `"current-password"`, `"tel"`, etc. Omit it on app-specific fields (role, category, rating). Never use `autoComplete="off"` to silence the warning.
- **`htmlFor` on every `FieldLabel`** — always set `htmlFor` matching the control's `id`. Exception: Radix components whose root is not a labelable element (`Slider`, `ToggleGroup`) — use `FieldTitle` with an `id` + `aria-labelledby` on the control instead of `FieldLabel` + `htmlFor`.
