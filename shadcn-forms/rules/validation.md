# Displaying Validation Errors

## Always use FieldDescription for errors

Never render errors as a raw `<p>` or `<span>` outside the `Field`:

**Incorrect:**

```tsx
<Input id="email" {...register("email")} />
<p className="text-red-500 text-sm">{errors.email?.message}</p>
```

**Correct:**

```tsx
<Field data-invalid={!!errors.email}>
  <FieldLabel htmlFor="email">Email</FieldLabel>
  <Input id="email" aria-invalid={!!errors.email} {...register("email")} />
  {errors.email && (
    <FieldDescription className="text-destructive">{errors.email.message}</FieldDescription>
  )}
</Field>
```

---

## Both attributes are required for validation state

| Attribute | Where | Effect |
|---|---|---|
| `data-invalid` | `Field` | Styles label + description |
| `aria-invalid` | input control | Styles the control + screen readers |

Missing either breaks either the visual or accessible state.

---

## Server / root errors

**Client-only mode** — use `setError("root")` for errors that don't belong to a specific field (e.g., wrong credentials):

```tsx
async function onSubmit(data: FormValues) {
  try {
    await signIn(data)
  } catch {
    setError("root", { message: "Invalid email or password." })
  }
}
```

Display above the submit button:

```tsx
{errors.root && (
  <p className="text-sm text-destructive">{errors.root.message}</p>
)}
<Button type="submit" disabled={isSubmitting}>Sign in</Button>
```

**Server action mode (next-safe-action)** — field-level server validation errors are **automatically mapped** to `formState.errors` by `useHookFormAction`. Do not call `setError` for field errors manually. For non-field runtime errors, display `result.serverError`:

```tsx
const { execute, result } = useHookFormAction(form, myAction)

{result?.serverError && (
  <p className="text-sm text-destructive">{result.serverError}</p>
)}
```

---

## Validate late, recover early

Set `mode: "onTouched"` — errors don't appear until the user leaves a field for the first time, then switch to live onChange validation so they clear as the user types:

```tsx
const form = useForm<FormValues>({
  resolver: zodResolver(schema),
  mode: "onTouched",
})
```

Use `mode: "onChange"` only when live feedback is required from the very first keystroke (e.g., a password strength meter). Never use `mode: "onBlur"` — it has no advantage over `"onTouched"` and produces worse UX (errors don't clear while typing).

---

## Reset after successful submission

**Always pass explicit values to `reset()` — never call it with no arguments.**

```tsx
const defaultValues = {
  name: "",
  email: "",
  roles: [] as FormValues["roles"],
  // …
}

const { reset } = useForm<FormValues>({ defaultValues, … })

async function onSubmit(data: FormValues) {
  await save(data)
  reset(defaultValues)
}
```

### Why no-arg `reset()` is broken

When called with no arguments, RHF's internal `_reset` walks the registered fields, finds the parent `<form>` element, and calls the **native DOM `form.reset()`** on it — then wipes the `_fields` registration map before re-registering on the next render. The native reset + re-registration cycle fires reset/focus side-effects through controlled components (Radix Checkbox, Switch, Calendar trigger button), causing a validation flash where required fields immediately re-display their error state on the *just-cleared* form.

Passing any defined object to `reset(values)` falls into a different branch — `isUndefined(formValues)` is false, the native DOM reset is skipped, and you get a clean RHF-only state update with `errors: {}` and `isSubmitted: false`.

### Pre-filling with fresh values

Same call signature — pass whatever object you want as the new "post-save" state:

```tsx
reset(serverResponse)   // pre-fill with server-returned values
reset(defaultValues)    // clear back to defaults
```
