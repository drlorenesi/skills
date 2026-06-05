# register() vs Controller

> **Client-side mode only.** This applies to forms built with react-hook-form (`useHookFormAction`). Server-side native forms (`useStateAction` + `<form action>`) use neither `register` nor `Controller` — they use uncontrolled shadcn controls with `name` + `defaultValue`.

## Use `register()` for native inputs

`Input`, `Textarea`, and `NativeSelect` accept a `ref` and forward DOM events — spread `register()` directly:

```tsx
<Input id="name" {...register("name")} />
<Textarea id="bio" {...register("bio")} />
```

`InputGroupInput` and `InputGroupTextarea` wrap `Input`/`Textarea` — `register()` works on them too:

```tsx
<InputGroup>
  <InputGroupAddon><InputGroupText>+1</InputGroupText></InputGroupAddon>
  <InputGroupInput id="phone" type="tel" autoComplete="tel" {...register("phone")} />
</InputGroup>
```

---

## Always pass `field.name` to Controller-wrapped components

`register()` automatically spreads `name` onto the input. `Controller` does not — you must forward `field.name` manually. Without it, browsers cannot autofill the field and devtools will warn "form field missing id or name".

Pass `name={field.name}` to every controlled component. All shadcn components forward it via `{...props}`:

```tsx
// Select
<Select name={field.name} value={field.value ?? ""} onValueChange={field.onChange}>

// Checkbox
<Checkbox id="terms" name={field.name} checked={field.value} onCheckedChange={field.onChange} />

// Switch
<Switch id="newsletter" name={field.name} checked={field.value} onCheckedChange={field.onChange} />

// RadioGroup
<RadioGroup name={field.name} value={field.value ?? ""} onValueChange={field.onChange}>

// Slider — root is a <span>, not labelable; use FieldTitle + aria-labelledby
<FieldTitle id="rating-label">Rating: {field.value} / 5</FieldTitle>
<Slider name={field.name} aria-labelledby="rating-label" value={[field.value]} onValueChange={(v) => field.onChange(v[0])} />

// RadioGroup — root is a <div role="radiogroup">, not labelable; same pattern
<FieldTitle id="contact-label">Preferred contact method</FieldTitle>
<RadioGroup name={field.name} aria-labelledby="contact-label" value={field.value ?? ""} onValueChange={field.onChange}>
```

---

## Use `Controller` for controlled shadcn components

Components that manage their own state and expose `value`/`onChange` props need `Controller`:

| Component | Why |
|---|---|
| `Select` | value is a string, not a DOM event |
| `Checkbox` | `checked` boolean, not `event.target.value` |
| `Switch` | same as Checkbox |
| `RadioGroup` | value is a string |
| `Combobox` | async/searchable, custom value type |
| `ToggleGroup` | value can be string or string[] |
| `InputOTP` | custom completion callback |
| `Slider` | value is `number[]` |
| `DatePicker` | value is `Date \| undefined` |

```tsx
import { Controller } from "react-hook-form"
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select"

<Controller
  name="role"
  control={control}
  render={({ field }) => (
    <Field data-invalid={!!errors.role}>
      <FieldLabel htmlFor="role">Role</FieldLabel>
      <Select name={field.name} value={field.value ?? ""} onValueChange={field.onChange}>
        <SelectTrigger id="role" aria-invalid={!!errors.role}>
          <SelectValue placeholder="Select a role" />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="admin">Admin</SelectItem>
          <SelectItem value="editor">Editor</SelectItem>
          <SelectItem value="viewer">Viewer</SelectItem>
        </SelectContent>
      </Select>
      {errors.role && (
        <FieldDescription className="text-destructive">{errors.role.message}</FieldDescription>
      )}
    </Field>
  )}
/>
```

### Checkbox with Controller

```tsx
<Controller
  name="terms"
  control={control}
  render={({ field }) => (
    <Field data-invalid={!!errors.terms} orientation="horizontal">
      <Checkbox
        id="terms"
        name={field.name}
        checked={field.value}
        onCheckedChange={field.onChange}
        aria-invalid={!!errors.terms}
      />
      <FieldLabel htmlFor="terms" className="font-normal">
        I agree to the terms
      </FieldLabel>
    </Field>
  )}
/>
```

### Checkbox group → array field

When a list of checkboxes maps to a single `string[]` field (`z.array(z.enum([...])).min(1)`), wrap the **whole group** in one `Controller` and toggle items in/out of `field.value`. Use `FieldSet` + `FieldLegend` for the group label, not `FieldLabel`.

```tsx
<Controller
  name="positions"           // positions: ("frontend" | "backend" | …)[]
  control={control}
  render={({ field }) => (
    <FieldSet data-invalid={!!errors.positions}>
      <FieldLegend>Positions of interest</FieldLegend>
      <FieldGroup data-slot="checkbox-group" className="gap-2">
        {positionOptions.map((opt) => {
          const checked = field.value?.includes(opt.value) ?? false
          return (
            <Field key={opt.value} orientation="horizontal">
              <Checkbox
                id={`position-${opt.value}`}
                name={field.name}
                checked={checked}
                onCheckedChange={(next) => {
                  const current = field.value ?? []
                  field.onChange(
                    next === true
                      ? [...current, opt.value]
                      : current.filter((v) => v !== opt.value)
                  )
                }}
                aria-invalid={!!errors.positions}
              />
              <FieldLabel htmlFor={`position-${opt.value}`} className="font-normal">
                {opt.label}
              </FieldLabel>
            </Field>
          )
        })}
      </FieldGroup>
      {errors.positions && (
        <FieldDescription className="text-destructive">{errors.positions.message}</FieldDescription>
      )}
    </FieldSet>
  )}
/>
```

Set `defaultValues: { positions: [] }` so `field.value` is never `undefined` on first render.

### DatePicker (Calendar + Popover) with Controller

shadcn ships `Calendar` and `Popover` separately — compose them inside one `Controller`. The schema field is `z.date()`; `field.value` is `Date | undefined`; `field.onChange` accepts the value `Calendar`'s `onSelect` returns directly (no adapter).

```tsx
<Controller
  name="availableFrom"       // availableFrom: Date
  control={control}
  render={({ field }) => (
    <Field data-invalid={!!errors.availableFrom}>
      <FieldLabel htmlFor="availableFrom">Available from</FieldLabel>
      <Popover>
        <PopoverTrigger asChild>
          <Button
            id="availableFrom"
            type="button"
            variant="outline"
            aria-invalid={!!errors.availableFrom}
            className={cn(
              "w-full justify-start text-left font-normal",
              !field.value && "text-muted-foreground"
            )}
          >
            <CalendarIcon className="mr-2 size-4" />
            {field.value ? format(field.value, "PPP") : "Pick a date"}
          </Button>
        </PopoverTrigger>
        <PopoverContent className="w-auto p-0" align="start">
          <Calendar mode="single" selected={field.value} onSelect={field.onChange} autoFocus />
        </PopoverContent>
      </Popover>
      {errors.availableFrom && (
        <FieldDescription className="text-destructive">{errors.availableFrom.message}</FieldDescription>
      )}
    </Field>
  )}
/>
```

Notes:
- `type="button"` on the trigger — without it, opening the popover submits the form.
- The trigger `Button` carries the `id` so `FieldLabel htmlFor` works; the calendar itself is not labelable.
- For a date *range*, use `mode="range"` and `z.object({ from: z.date(), to: z.date() })` as a single field — same Controller, same shape.

---

## Always use `field.value ?? ""` for Select and RadioGroup

If a field has no `defaultValues` entry, `field.value` starts as `undefined`. React treats `value={undefined}` as uncontrolled and warns when it later receives a string.

**Incorrect:**
```tsx
<Select value={field.value} onValueChange={field.onChange}>
```

**Correct:**
```tsx
<Select value={field.value ?? ""} onValueChange={field.onChange}>
```

The empty string keeps the Select controlled from mount (showing the placeholder) without needing a fake default in the schema.

---

## Destructure `field` — don't spread blindly on complex components

**Incorrect** — spreads `ref` and `name` onto components that don't accept them:

```tsx
render={({ field }) => <Select {...field} />}
```

**Correct** — pick only what the component accepts:

```tsx
render={({ field }) => (
  <Select value={field.value} onValueChange={field.onChange}>
    …
  </Select>
)}
```
