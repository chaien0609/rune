---
name: "animation-patterns"
pack: "@rune/ui"
description: "Motion design patterns — micro-interactions, page transitions, scroll animations, loading states. Applies CSS transitions, Framer Motion, or GSAP based on project stack. Always respects prefers-reduced-motion."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# animation-patterns

Motion design patterns — micro-interactions, page transitions, scroll animations, loading states. Applies CSS transitions, Framer Motion, or GSAP based on project stack. Always respects `prefers-reduced-motion`.

#### Workflow

**Step 1 — Detect interaction points**
Use Grep to find hover handlers (`onMouseEnter`, `:hover`), route changes (Next.js `useRouter`, SvelteKit `goto`), and loading states (`isLoading`, `isPending`). Read component files to understand where motion can add feedback or polish.

**Step 2 — Apply micro-interactions**
For each interaction point, select the appropriate pattern: hover → scale + shadow lift; button click → press-down (scale 0.97); data load → skeleton pulse then fade-in; route change → slide or fade transition. Emit the updated component with motion classes or Framer Motion variants.

**Step 3 — Audit reduced-motion compliance**
Use Grep to find every animation/transition declaration. Verify each is wrapped in a `prefers-reduced-motion: no-preference` media query or uses Framer Motion's `useReducedMotion()` hook. Flag any that are not.

**Step 4 — Page transition patterns**
Apply View Transitions API for same-document navigations (SvelteKit, Astro, vanilla JS). For React/Next.js, use Framer Motion `AnimatePresence` + `layoutId` for shared layout animations. Emit transition wrapper component with both strategies.

#### Example

```tsx
// Tailwind micro-interaction with reduced-motion respect
<button
  className="
    transform transition-all duration-200 ease-out
    hover:scale-105 hover:shadow-md
    active:scale-95
    motion-reduce:transform-none motion-reduce:transition-none
  "
>
  Confirm
</button>

// Framer Motion with reduced-motion hook
const prefersReduced = useReducedMotion()

<motion.div
  initial={{ opacity: 0, y: prefersReduced ? 0 : 16 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: prefersReduced ? 0 : 0.25 }}
/>
```

```tsx
// Shared layout animation — card expands to modal (Framer Motion)
// Works because both use the same layoutId="card-{id}"
function CardGrid({ items }: { items: Item[] }) {
  const [selected, setSelected] = useState<string | null>(null)
  return (
    <>
      {items.map((item) => (
        <motion.div
          key={item.id}
          layoutId={`card-${item.id}`}
          onClick={() => setSelected(item.id)}
          className="rounded-xl bg-[var(--bg-card)] border border-[var(--border)] cursor-pointer"
        >
          <motion.h3 layoutId={`title-${item.id}`}>{item.title}</motion.h3>
        </motion.div>
      ))}

      <AnimatePresence>
        {selected && (
          <motion.div
            layoutId={`card-${selected}`}
            className="fixed inset-8 z-50 rounded-2xl bg-[var(--bg-card)] p-8"
          >
            <motion.h3 layoutId={`title-${selected}`} className="text-h2 font-bold">
              {items.find(i => i.id === selected)?.title}
            </motion.h3>
            <button onClick={() => setSelected(null)} aria-label="Close">
              <X aria-hidden="true" />
            </button>
          </motion.div>
        )}
      </AnimatePresence>
    </>
  )
}
```

```css
/* View Transitions API — SvelteKit / Astro page transitions */
/* In app.css or global stylesheet */
@media (prefers-reduced-motion: no-preference) {
  ::view-transition-old(root) {
    animation: 200ms ease-out both fade-out;
  }
  ::view-transition-new(root) {
    animation: 250ms ease-in both fade-in;
  }
}

@keyframes fade-out { from { opacity: 1; } to { opacity: 0; } }
@keyframes fade-in  { from { opacity: 0; } to { opacity: 1; } }

/* SvelteKit: enable in svelte.config.js → experimental: { viewTransitions: true } */
```
