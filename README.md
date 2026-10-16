# Design review request: mobile "persona switcher" for a portfolio site

You are reviewing one specific UI control. Give concrete, opinionated
suggestions. Assume you cannot see the site; everything you need is below.

## The product

`leethedev` — a single-page portfolio (Next.js 16, Pages Router, TypeScript,
no component library, plain CSS in a template literal). Dark, terminal/
monospace aesthetic (Fira Code). The site's signature feature: 6 personas
(Technical, Creative, Business, Academic, Collaborator, Maverick). Switching
persona restyles the ENTIRE page — colors, accent, headline (typewriter-
animated), bio, project emphasis. Each persona has: id, label, icon glyph
(`</>`, `✦`, `◈`, `§`, `⬡`, `⚡`), accent color, chipStyle. The switcher is
not settings — it IS the product. Everything else on the page is
conventional.

## Current implementation (just shipped today)

Header (sticky, z-100, 64px desktop):

- ≥947px: row of six icon+label pill chips ("pills"), active chip filled with
  persona accent, right-aligned next to `leethedev` logo.
- ≤946px: the SAME six chips become a horizontally scrollable rail
  (one control at every width — this replaced a native `<select>`):

```css
@media (max-width: 946px) {
  .persona-strip-shell {          /* the scroller */
    display: block; width: 100%; max-width: 100%; flex: 1 1 auto;
    overflow-x: auto; overscroll-behavior-x: contain;
    scrollbar-width: none; -webkit-overflow-scrolling: touch;
    mask-image: linear-gradient(90deg, #000 calc(100% - 40px), transparent);
  }
  .persona-strip {                 /* content: never box-locks to shell */
    justify-content: flex-start;   /* "lock": scroll origin = active chip */
    flex-shrink: 0; width: max-content;   /* "lock": no unscrollable
                                             start-side overflow */
    scroll-snap-type: x proximity; padding-inline: 12px 24px;
  }
  .persona-chip { min-height: 44px; padding: 10px 16px; }
}
```

JSX: `<div class="persona-strip-shell"><nav class="persona-strip"
aria-label="Portfolio view">` mapping `PERSONA_ORDER` to
`<button aria-pressed class="persona-chip">`. The active button gets a ref;
a 6-line `useEffect` on `activeId` does
`activeChipRef.scrollIntoView({inline:"center", block:"nearest",
behavior: reduceMotion ? "auto" : "smooth"})` only when
`scrollWidth > clientWidth`.

State: `activeId` via `useSyncExternalStore` + optional `personaOverride`
state (local URL param > localStorage > default). SSR renders the default
persona byte-identically to first client render (no hydration mismatch
allowed — localStorage is never read in render). A horizontal SWIPE on the
main content area also cycles personas (touch handlers on `<main>`, header
is a sibling so rail drags never trigger it).

## Measured facts (verified live on the deployed site)

- Six chips total 740px wide. Hidden share of the rail: 49% @390px viewport,
  22% @769, 6% @900, 0% @946. At 390px you see: active chip + ~1.5 more +
  a peek under the right fade mask.
- Chips are 44px tall (fixed from 31px). Old `<select>` was 41px.
- Sticky header @≤768px: logo row + full-bleed rail row ≈ 94px (~11% of a
  844px viewport). Desktop: 64px single row.
- There is a floating cluster fixed bottom-right (only ≤600px): a terminal
  FAB (48px `>_`) + a text watermark "Technical view" (134×21px) that is
  `aria-hidden="true"` — dead, untappable pixels every phone visitor sees.
- A full fake terminal exists (FAB opens it; it has a command parser,
  typewriter output, aria-live). A `view <persona>` command was proposed
  by three independent design reviewers but NOT built (users don't know
  what a terminal is; can't be load-bearing).

## History (matters for your judgment)

- Aug: original = scroll rail with JS-tracked edge fades (React state +
  ResizeObserver + rAF centering math) — worked, ~40 lines of JS.
- Last week: swapped for a native `<select>` at ≤946px (simplicity).
  Owner rejected it: "not ready to ship" — OS chrome over a dark terminal
  site, zero persona color/icons, 41px targets.
- Today: three AI design reviewers (given screenshots + live browser +
  this codebase) debated alternatives: (a) scroll rail, (b) bottom sheet via
  native `<dialog>` (six 56px rows with one-line flavor tags, triggered by
  reviving the dead watermark as a thumb-zone button), (c) terminal
  command. Verdict: rail won on "permanence = discoverability; the
  switcher is the product"; sheet won on thumb-reach + room for flavor
  text; all three converged on terminal-as-easter-egg. Shipped the rail
  with the locks + 44px + auto-center.

## Hard constraints for any suggestion

1. No hydration mismatch; no localStorage/window/matchMedia in render.
2. Works 320–1440px; tap targets ≥44px; keyboard + screen-reader operable
   (current: real buttons + `aria-pressed` in a nav; axe-clean).
3. Monospace/terminal aesthetic; every persona's accent must show.
4. Minimal JS (site ships almost none); no new dependencies.
5. Same markup at every width preferred (one mental model).
6. `prefers-reduced-motion` respected.

## Open questions — where you can beat us

1. At 390px the rail hides ~49% of personas behind a swipe. Is a visible
   peek + fade enough cue, or is discoverability of the other 4-5 personas
   still broken? Better passive cue?
2. The control lives top-of-screen (bad thumb reach). The bottom-right
   cluster (terminal FAB + dead watermark) is natural thumb-zone real
   estate. Revive the watermark as a secondary trigger? For what?
3. 601–946px band: pills fit inline at ≥769 but the dead watermark
   doesn't exist there (only ≤600px). Any coherent one-piece design for
   the whole ≤946 band?
4. Should any persona description/flavor be surfaced (the sheet debate
   argued flavor tags "sell" personas; rail shows only word+icon)? Where
   without a modal?
5. Anything fundamentally better than rail/sheet/hybrid we haven't
   considered? (Segmented control is out — Apple HIG caps ~5 segments,
   no mixed text+icons; native select is out; icon-only rail is out —
   unlabeled glyphs.)

Deliver: critique of current rail (specific), then 2-3 ranked concrete
alternatives or refinements with implementation notes against the code
above, and one wildcard.
