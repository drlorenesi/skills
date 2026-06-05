---
name: shadcn-forms
description: Build type-safe forms with shadcn/ui, react-hook-form, zod, and next-safe-action. Covers the shadcn/ui + next-safe-action preconditions, client-side (react-hook-form adapter) vs server-side (native form action) modes, schema/action/component placement, field wiring, validation display, and submission. Use whenever building or modifying a form in this project.
user-invocable: true
---

# shadcn Forms — react-hook-form + zod + next-safe-action

Stack: **shadcn/ui** (Field/FieldGroup layout) + **react-hook-form** (client state) + **zod** (schema) + **next-safe-action** (server actions — **required**).

Every form in this project submits through a **next-safe-action** action. There is no "post directly to an external API" mode — if you need an external call (auth provider, third-party API), wrap it inside a safe action.

---

## Step 0 — Preconditions (do not skip)

**1. shadcn/ui must be initialized.** Check for `components.json` at the project root.

- **Missing?** shadcn/ui is the foundation of this skill — it cannot proceed without it. Tell the user to initialize it **themselves** first:
  ```bash
  npx shadcn@latest init
  ```
  (or invoke the `shadcn` skill). Then **stop** — do not write any form code until `components.json` exists.
- **Present?** Continue.

**2. next-safe-action must have a client.** Check `lib/safe-action.ts`. This project already exports two clients — **reuse them, do not recreate the file**:

| Client | Use for |
|---|---|
| `actionClient` | user-facing forms — `handleServerError` returns a generic message |
| `diagnosticActionClient` | admin/debug tools — passes the real error message through |

If `lib/safe-action.ts` does not exist (a different project), create it once — defer to the **`safe-action-client`** skill.

---

## Step 1 — Ask: client-side or server-side?

|  | Client-side | Server-side |
|---|---|---|
| Form library | react-hook-form + `useHookFormAction` adapter | none — native `<form action>` |
| Component | `"use client"` | dispatcher hook is `"use client"`; markup can be minimal |
| Validation feedback | per-field, on change/blur, before submit | after submit (server round-trip) |
| Action method | `.action()` | `.stateAction()` |
| Best for | rich, interactive forms (most app forms) | simple forms, progressive enhancement |

Both validate with the **same zod schema** on the server. Client-side *also* validates in the browser via `zodResolver`.

---

## Step 2 — Install missing packages

Detect what's already in `package.json` and add **only the gap**. Run it yourself if you have permission; otherwise print the command and ask the user to run it.

```bash
# Client-side (react-hook-form adapter)
npm install react-hook-form @hookform/resolvers zod next-safe-action @next-safe-action/adapter-react-hook-form

# Server-side (native form)
npm install zod next-safe-action
```

> In this project, only `@next-safe-action/adapter-react-hook-form` is missing — `react-hook-form`, `@hookform/resolvers`, `zod`, and `next-safe-action` are already installed.

---

## Step 3 — Install the shadcn components the form uses

Derive the list from the fields you're about to build, then add them in one command (omit any already in `components/ui/`):

```bash
npx shadcn@latest add field button label input select checkbox switch textarea radio-group calendar popover
```

`field`, `button`, and `label` are always needed; add one control per field type. Permission-aware, same as Step 2.

---

## Step 4 — Build

Use the colocated layout, then follow the section for your chosen mode.

### File layout (colocated under the route)

```
app/<ruta>/
  page.tsx            renders the form
  <nombre>-form.tsx   the form component
  actions.ts          "use server" — the safe action(s)
  schema.ts           neutral (NO directive) — zod schema, exported
lib/safe-action.ts    shared client — reuse, don't recreate
```

> The zod schema lives in its own `schema.ts`, **never** in `actions.ts`. Next.js treats every export of a `"use server"` file as a server-action proxy; a schema exported from there breaks `zodResolver` on the client. `schema.ts` has no directive, so both `actions.ts` (server) and `<nombre>-form.tsx` (client) can import it. See [schema.md](./rules/schema.md).

---

## Mode A — Client-side form (react-hook-form + adapter)

`useHookFormAction` composes `useAction` + `useForm` and **auto-maps server validation errors** onto the form fields. Call it with the **action first, the resolver second** — it *returns* the form; you never pass one in.

```tsx
"use client"

import { zodResolver } from "@hookform/resolvers/zod"
import { useHookFormAction } from "@next-safe-action/adapter-react-hook-form/hooks"

import { Button } from "@/components/ui/button"
import { Field, FieldDescription, FieldGroup, FieldLabel } from "@/components/ui/field"
import { Input } from "@/components/ui/input"
import { loginAction } from "./actions"
import { loginSchema } from "./schema"

export function LoginForm() {
  const { form, action, handleSubmitWithAction } = useHookFormAction(
    loginAction,
    zodResolver(loginSchema),
    {
      formProps: { mode: "onTouched", defaultValues: { email: "", password: "" } },
    }
  )

  const { register, formState: { errors } } = form

  return (
    <form onSubmit={handleSubmitWithAction} noValidate>
      <FieldGroup>
        <Field data-invalid={!!errors.email}>
          <FieldLabel htmlFor="email">Email</FieldLabel>
          <Input id="email" type="email" autoComplete="email" aria-invalid={!!errors.email} {...register("email")} />
          {errors.email && (
            <FieldDescription className="text-destructive">{errors.email.message}</FieldDescription>
          )}
        </Field>

        <Field data-invalid={!!errors.password}>
          <FieldLabel htmlFor="password">Password</FieldLabel>
          <Input id="password" type="password" autoComplete="current-password" aria-invalid={!!errors.password} {...register("password")} />
          {errors.password && (
            <FieldDescription className="text-destructive">{errors.password.message}</FieldDescription>
          )}
        </Field>
      </FieldGroup>

      {action.result.serverError && (
        <p className="mt-2 text-sm text-destructive">{action.result.serverError}</p>
      )}

      <Button type="submit" disabled={action.isPending} className="mt-6 w-full">
        {action.isPending ? "Signing in…" : "Sign in"}
      </Button>
    </form>
  )
}
```

- `defaultValues` and `mode` go in **`formProps`** (not as top-level `useForm` args).
- Disable the submit button with **`action.isPending`**.
- Field-level server validation errors are **auto-mapped** to `errors`; only `action.result.serverError` is shown manually.
- Reset after success with **`resetFormAndAction()`** — call it from a `useEffect` on `action.hasSucceeded`, **not** from `actionProps.onSuccess` (referencing the hook's own return inside its own options is a circular type under strict TS). See [validation.md](./rules/validation.md).
- Controlled inputs (`Select`, `Checkbox`, `Switch`, `DatePicker`, …) use `Controller` — see [controller.md](./rules/controller.md).

**Files:** `schema.ts` + `actions.ts` (with `.action()`) + `<nombre>-form.tsx`. See [server-actions.md](./rules/server-actions.md).

---

## Mode B — Server-side form (native `<form action>`)

No react-hook-form. The action is defined with **`.stateAction()`** and dispatched through `useStateAction`'s `formAction`, so the form works as a real `<form action={…}>`. Validation feedback appears after submit. For the full native-form decision table (`useStateAction` vs `useAction`, `prevResult`, bind args, file uploads) defer to the **`safe-action-forms`** skill — this section covers only the shadcn/ui markup.

```tsx
"use client"

import { useStateAction } from "next-safe-action/hooks"

import { Button } from "@/components/ui/button"
import { Field, FieldDescription, FieldGroup, FieldLabel } from "@/components/ui/field"
import { Input } from "@/components/ui/input"
import { Textarea } from "@/components/ui/textarea"
import { sendContact } from "./actions"

export function ContactForm() {
  const { formAction, result, isPending } = useStateAction(sendContact)
  const errors = result.validationErrors

  return (
    <form action={formAction} noValidate>
      <FieldGroup>
        <Field data-invalid={!!errors?.name}>
          <FieldLabel htmlFor="name">Name</FieldLabel>
          <Input id="name" name="name" autoComplete="name" aria-invalid={!!errors?.name} />
          {errors?.name?._errors?.[0] && (
            <FieldDescription className="text-destructive">{errors.name._errors[0]}</FieldDescription>
          )}
        </Field>

        <Field data-invalid={!!errors?.message}>
          <FieldLabel htmlFor="message">Message</FieldLabel>
          <Textarea id="message" name="message" aria-invalid={!!errors?.message} />
          {errors?.message?._errors?.[0] && (
            <FieldDescription className="text-destructive">{errors.message._errors[0]}</FieldDescription>
          )}
        </Field>
      </FieldGroup>

      {result.serverError && (
        <p className="mt-2 text-sm text-destructive">{result.serverError}</p>
      )}

      <Button type="submit" disabled={isPending} className="mt-6 w-full">
        {isPending ? "Sending…" : "Send"}
      </Button>
    </form>
  )
}
```

- Inputs are **uncontrolled** — pass `name` (matching the schema key) and `defaultValue` for edit forms. No `register`, no `Controller`.
- The action must use **`.stateAction()`** (not `.action()`).
- Errors arrive in `result.validationErrors?.<field>?._errors` (the default formatted shape); non-field errors in `result.serverError`.

**Files:** `schema.ts` + `actions.ts` (with `.stateAction()`) + `<nombre>-form.tsx`. See [server-actions.md](./rules/server-actions.md).

---

## Rules

- [schema.md](./rules/schema.md) — zod schema patterns + placement
- [controller.md](./rules/controller.md) — `register` vs `Controller` (client mode)
- [validation.md](./rules/validation.md) — displaying errors, validation timing, reset
- [server-actions.md](./rules/server-actions.md) — next-safe-action wiring for both modes

### Related skills (defer to these — don't re-derive)

- **`safe-action-forms`** — `useHookFormAction`, `useStateAction`, native forms, bind args, file uploads
- **`safe-action-hooks`** — `useAction` / `useStateAction` status, callbacks, `executeAsync`
- **`safe-action-validation-errors`** — error shapes, `returnValidationErrors`, `throwValidationErrors`
- **`safe-action-client`** — `createSafeActionClient` setup (only if `lib/safe-action.ts` is missing)
- **`shadcn`** — adding/searching components

### Quick reference

- **`useHookFormAction(action, zodResolver(schema), props?)`** — action first, resolver second. It **returns** `{ form, action, handleSubmitWithAction, resetFormAndAction }`; you do **not** pass a `form` in.
- **No `type FormValues` needed** in client mode — `useHookFormAction` infers form types from the action and resolver. Declaring one produces an unused-type warning.
- **Always use `zodResolver`** (client mode) — never hand-write validation.
- **`register()` for native inputs** (`Input`, `Textarea`, `NativeSelect`); **`Controller`** for controlled components (`Select`, `Checkbox`, `Switch`, `RadioGroup`, `Combobox`, `ToggleGroup`, `InputOTP`, `Slider`, `DatePicker`). Server mode uses neither — plain `name`/`defaultValue`.
- **Pair `data-invalid` (on `Field`) + `aria-invalid` (on the control).** Missing either breaks the visual or the accessible state.
- **Show errors via `FieldDescription`** — never a raw `<p>`/`<span>` inside the field.
- **`noValidate` on `<form>`** — let zod own validation.
- **Disable submit while pending** — `action.isPending` (client) / `isPending` (server). Prevents double-submits.
- **Defaults & timing (client):** put `defaultValues` and `mode: "onTouched"` in `formProps`.
- **Reset (client):** `resetFormAndAction()` from a `useEffect` on `action.hasSucceeded` (not `actionProps.onSuccess` — circular type), or `form.reset(values)`; never call `reset()` with no args (see [validation.md](./rules/validation.md)).
- **Server/root error:** `action.result.serverError` (client) / `result.serverError` (server).
- **`autoComplete`**: only where a standard token exists (`name`, `email`, `new-password`, `current-password`, `tel`, …). Omit on app-specific fields (role, category, rating). Never `autoComplete="off"` to silence a warning.
- **`htmlFor` on every `FieldLabel`** matching the control's `id`. Exception: non-labelable Radix roots (`Slider`, `ToggleGroup`, `RadioGroup`) — use `FieldTitle`/`FieldLegend` + `aria-labelledby`.
