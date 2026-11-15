# UI and Styling Standards

All UI surfaces must use shadcn/ui primitives and Tailwind CSS. No ad-hoc HTML or CSS.

## Rules

### shadcn/ui + Tailwind only

- **Every button is a `<Button>` from shadcn** (or custom `Button` if it exists in the
  project).
- **Every input is a shadcn primitive** (`Input`, `Textarea`, `Select`, etc.).
- **Every dropdown is a `DropdownMenu`** or `Select` from shadcn/Radix.
- **No inline `<style>` tags or CSS-in-JS** outside of Tailwind.
- **No component libraries other than shadcn**.

### Match the existing app

- **Use the app's existing colors, design, text styles, and UI patterns.** Don't invent
  a new visual language unless explicitly asked.
- **Never use a raw `<button>`** — always the shared `Button` component, reusing an
  existing variant rather than restyling it.
- **Reuse an existing component before creating one.** If nothing fits, build it as a
  reusable component rather than a one-off.
- **Don't introduce a new UI or styling library** when the existing stack covers it.

### Tailwind CSS

- **No manual CSS.** Tailwind only — no stylesheets, `<style>` tags, or CSS-in-JS. The
  sole exception is email templates in `src/emails/`, where inline styles are required
  for mail-client compatibility.
- **Use only semantic Tailwind tokens**: `text-primary`, `bg-secondary`, `border-muted`,
  etc.
- **Never hardcode colors**: use the design tokens defined in `tailwind.config.ts`.
- **Spacing is on the 4px grid**: `p-2`, `p-4`, `p-6`, `gap-2`, `gap-4`, etc.
- **Margin and padding use the grid**: `m-2`, `my-4`, `px-3`, etc.
- **Do not override Tailwind config** unless absolutely necessary and documented.

### Typography

- **Use semantic HTML**: `<h1>`, `<h2>`, `<h3>`, `<p>`, `<strong>`, `<em>`, etc.
- **Font sizes via Tailwind**: `text-xs`, `text-sm`, `text-base`, `text-lg`, `text-xl`,
  `text-2xl`.
- **Line height via Tailwind**: `leading-tight`, `leading-normal`, `leading-relaxed`.
- **Letter spacing via Tailwind**: `tracking-tight`, `tracking-normal`, `tracking-wide`.

### Responsiveness

**Everything must be responsive.** Every screen and component works on mobile, tablet,
and desktop — this is part of the work, not a follow-up task.

- **Design mobile-first**: unprefixed classes target mobile, then layer `md:` / `lg:`.
- **Verify all three breakpoints before calling work done** — not just the width you
  happened to develop at.
- **Watch the usual break points**: fixed pixel widths, wide tables, multi-column grids
  that don't collapse, dialogs taller than a phone viewport, and long unbroken strings.

```tsx
// ✅ Good: collapses to one column on mobile, three on desktop
<div className="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3">

// ❌ Bad: fixed width overflows on mobile
<div className="w-[900px]">
```

### Accessibility (a11y)

- **Use semantic HTML**: `<button>`, `<a>`, `<label>`, `<fieldset>`, etc.
- **Link inputs to labels**: every `<input>` has an associated `<label>` with `htmlFor`.
- **ARIA labels for icon-only buttons**: `aria-label="Close"`.
- **Color is not the only indicator**: use text, icons, or patterns in addition to color
  for status (error, success, warning).
- **Test with a screen reader**: at least for critical flows.

## Example: Form

```tsx
// ✅ Good: semantic HTML, shadcn primitives, Tailwind
'use client';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { useForm } from 'react-hook-form';

export function ProjectForm() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4 max-w-md">
      <div className="space-y-2">
        <Label htmlFor="address">Address</Label>
        <Input
          id="address"
          type="text"
          placeholder="Enter address"
          {...register('address', { required: 'Address is required' })}
          aria-invalid={!!errors.address}
        />
        {errors.address && (
          <p className="text-sm text-red-600">{errors.address.message}</p>
        )}
      </div>

      <Button type="submit">Create Project</Button>
    </form>
  );
}
```

```tsx
// ❌ Bad: custom styling, no labels, no error handling
export function ProjectForm() {
  return (
    <form style={{ padding: '16px', display: 'flex', flexDirection: 'column' }}>
      <input type="text" placeholder="Address" />
      <button style={{ backgroundColor: 'blue', color: 'white' }}>
        Create Project
      </button>
    </form>
  );
}
```

## Dark Mode Support

- **shadcn/ui and Tailwind handle dark mode automatically**.
- **Use `dark:` prefix for dark-mode-specific classes** only when the default isn't
  sufficient.
- **Test in both light and dark modes** before calling work done.

```tsx
// ✅ Good: automatic dark mode support
<div className="bg-white text-black dark:bg-slate-950 dark:text-white">
  Content
</div>
```
