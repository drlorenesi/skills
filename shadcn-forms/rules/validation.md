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

Missing either breaks either the visual or the accessible state.

---

## Server / root errors

Both modes surface non-field errors (DB failure, "invalid credentials") through the **action result**, never `setError`.

**Client-side (`useHookFormAction`)** — field-level *validation* errors returned by the action are **auto-mapped** onto `form.formState.errors` (and root-level `_errors` onto `errors.root`). Don't map them by hand. Display only the non-field `serverError`:

```tsx
const { form, action, handleSubmitWithAction } = useHookFormAction(myAction, zodResolver(schema))
const { formState: { errors } } = form

{action.result.serverError && (
  <p className="text-sm text-destructive">{action.result.serverError}</p>
)}
<Button type="submit" disabled={action.isPending}>Save</Button>
```

**Server-side (`useStateAction`)** — read errors straight off `result`:

```tsx
const { formAction, result, isPending } = useStateAction(myAction)

// per field, inside its <Field data-invalid={...}>:
{result.validationErrors?.email?._errors?.[0] && (
  <FieldDescription className="text-destructive">
    {result.validationErrors.email._errors[0]}
  </FieldDescription>
)}

// non-field:
{result.serverError && <p className="text-sm text-destructive">{result.serverError}</p>}
```

> The `_errors` shape above is next-safe-action's **default (formatted)** validation-error shape. Don't switch the client to the `flattened` shape — `useHookFormAction` expects the formatted one. For shape details see the **`safe-action-validation-errors`** skill.

---

## Validate late, recover early (client mode)

Pass `mode: "onTouched"` via `formProps` — errors don't appear until the user first leaves a field, then clear live as they type:

```tsx
const { form } = useHookFormAction(myAction, zodResolver(schema), {
  formProps: { mode: "onTouched" },
})
```

Use `mode: "onChange"` only when feedback is needed from the first keystroke (e.g. a password strength meter). Avoid `mode: "onBlur"` — no advantage over `"onTouched"`, and errors don't clear while typing.

---

## Reset after a successful submission (client mode)

Prefer **`resetFormAndAction()`** (returned by the adapter) — it clears the form *and* the action result. Call it from a `useEffect` on `action.hasSucceeded`:

```tsx
const { form, action, resetFormAndAction } = useHookFormAction(
  myAction,
  zodResolver(schema),
  { formProps: { defaultValues: { name: "", email: "", roles: [] } } }
)

useEffect(() => {
  if (action.hasSucceeded) resetFormAndAction()
}, [action.hasSucceeded, resetFormAndAction])
```

> **Don't reset from `actionProps.onSuccess`.** Referencing `resetFormAndAction` (or `form`) — the hook's own return values — inside the options object of that **same** `useHookFormAction(...)` call is a circular type: under strict TS it fails with *"implicitly has type 'any' because it is referenced directly or indirectly in its own initializer"* (TS7022/7023). A `useEffect` runs after the hook returns, so the binding is fully typed. Side-effect-only callbacks (e.g. `onSuccess: () => toast.success(...)`) are fine — they don't reference the hook's return.

If you reset the form directly instead, **always pass explicit values to `form.reset()` — never call it with no arguments.**

### Why no-arg `reset()` is broken

When called with no arguments, RHF's internal `_reset` walks the registered fields, finds the parent `<form>` element, and calls the **native DOM `form.reset()`** on it — then wipes the `_fields` registration map before re-registering on the next render. The native reset + re-registration cycle fires reset/focus side-effects through controlled components (Radix Checkbox, Switch, Calendar trigger button), causing a validation flash where required fields immediately re-display their error state on the *just-cleared* form.

Passing any defined object to `form.reset(values)` falls into a different branch — `isUndefined(formValues)` is false, the native DOM reset is skipped, and you get a clean RHF-only state update with `errors: {}` and `isSubmitted: false`.

### Pre-filling with fresh values

Same call signature — pass whatever object you want as the new "post-save" state:

```tsx
form.reset(serverResponse)   // pre-fill with server-returned values
form.reset(defaultValues)    // clear back to defaults
```
