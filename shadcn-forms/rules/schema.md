# Zod Schema Patterns

## Place the schema in `app/<ruta>/schema.ts`

Every form's schema lives in its own colocated `schema.ts` (no directive), exported, and imported by both the action and the form component:

```
app/<ruta>/
  schema.ts          export const <feature>Schema = z.object({ … })
  actions.ts         "use server" — import { <feature>Schema } from "./schema"
  <nombre>-form.tsx  import { <feature>Schema } from "./schema"  (client mode)
```

**Never define the schema inside `actions.ts`.** Next.js treats every export of a `"use server"` file as a server-action proxy; a zod schema exported from there is serialized and breaks `zodResolver` on the client with *"Invalid input: not a Zod schema"*. A neutral `schema.ts` (no `"use server"` / `"use client"`) imports safely from both sides.

> Server-only actions that are **not** forms (e.g. the diagnostic tools in `app/bds/*/actions.ts`) may keep their schema as a *local, unexported* `const` inside `actions.ts` — fine, because nothing on the client imports it. Forms are different: the schema is shared, so it must be exported from `schema.ts`.

---

## Common field schemas (zod v4)

```ts
import { z } from "zod"

export const schema = z.object({
  // Required text
  name: z.string().min(1, "Name is required"),

  // Email — top-level in zod v4 (not z.string().email())
  email: z.email("Invalid email address"),

  // Optional field
  bio: z.string().optional(),

  // Number from a text input (always a string from the DOM)
  age: z.coerce.number().int().min(0, "Must be 0 or older"),

  // Boolean (Checkbox / Switch)
  terms: z.boolean().refine((v) => v === true, "You must accept the terms"),

  // Enum (Select / RadioGroup) — custom message via { error } in zod v4
  role: z.enum(["admin", "editor", "viewer"], { error: "Select a role" }),

  // Array of strings (multi-select / CheckboxGroup)
  tags: z.array(z.string()).min(1, "Select at least one tag"),

  // URL — top-level in zod v4 (not z.string().url())
  website: z.url("Enter a valid URL").optional().or(z.literal("")),

  // Date string from a native <input type="date"> — z.iso.date() in zod v4
  birthdate: z.iso.date("Invalid date"),

  // Date object from a shadcn DatePicker (Calendar's onSelect returns a Date)
  availableFrom: z.date(),
})
```

---

## Cross-field validation

Use `.refine` or `.superRefine` at the object level; set `path` to the field that should show the error:

```ts
const schema = z
  .object({
    password: z.string().min(8),
    confirmPassword: z.string(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    error: "Passwords do not match",
    path: ["confirmPassword"],
  })
```

---

## Infer the type — never duplicate it

In **client mode**, `useHookFormAction` infers the form type for you — you usually don't need `FormValues` at all. If you do need the type elsewhere, infer it; never hand-write an interface:

```ts
// Correct
type FormValues = z.infer<typeof schema>

// Incorrect — duplicates the schema as a TS interface
interface FormValues {
  email: string
  password: string
}
```
