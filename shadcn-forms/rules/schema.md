# Zod Schema Patterns

## Collocate schema with the form file

**Client-only forms:** define the schema in the same file as the form component. Only extract to a shared file when two or more forms share the same shape.

**Server action forms (next-safe-action):** always define the schema in `schemas/[feature].ts` (never inside `actions/[feature].ts`). Import it into both the action file and the form component. Next.js treats every export from a `"use server"` file as a server proxy — a zod schema exported from an action file will be broken on the client.

---

## Common field schemas

```ts
import { z } from "zod"

const schema = z.object({
  // Required text
  name: z.string().min(1, "Name is required"),

  // Email
  email: z.string().email("Invalid email address"),

  // Optional field
  bio: z.string().optional(),

  // Number from a text input (always a string from the DOM)
  age: z.coerce.number().int().min(0, "Must be 0 or older"),

  // Boolean (Checkbox / Switch)
  terms: z.boolean().refine((v) => v === true, "You must accept the terms"),

  // Enum (Select / RadioGroup)
  role: z.enum(["admin", "editor", "viewer"], { message: "Select a role" }),

  // Array of strings (multi-select / CheckboxGroup)
  tags: z.array(z.string()).min(1, "Select at least one tag"),

  // URL
  website: z.string().url("Enter a valid URL").optional().or(z.literal("")),

  // Date from a date input (always a string from the DOM)
  birthdate: z.string().date("Invalid date"),
})
```

---

## Cross-field validation

Use `.superRefine` or `.refine` at the object level:

```ts
const schema = z
  .object({
    password: z.string().min(8),
    confirmPassword: z.string(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords do not match",
    path: ["confirmPassword"],
  })
```

Always set `path` to the field that should show the error.

---

## Infer the type — never duplicate it

```ts
// Correct
type FormValues = z.infer<typeof schema>

// Incorrect — duplicates the schema as a TS interface
interface FormValues {
  email: string
  password: string
}
```
