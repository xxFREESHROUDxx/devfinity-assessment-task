# Mountains of Nepal — Devfinity Assessment Task

A responsive, pixel-accurate build of the supplied Figma design using plain HTML and CSS — no
frameworks, no build step. Open [index.html](index.html) in a browser to view it.

## Time spent

| Phase                                                                                     | Time       |
| ----------------------------------------------------------------------------------------- | ---------- |
| Studying the Figma file, planning content structure and thinking through the architecture | ~1 hour    |
| Building the markup, styling and responsive behavior                                      | ~4-5 hours |

Most of the first hour went into deciding _how_ the design should be assembled — which pieces are
content, which are decoration, and how the diagonal shapes could be built so the desktop layout and
the mobile layout could share the same markup.

## Naming convention — BEM

All class names follow BEM (`block__element--modifier`), which keeps the relationship between
elements obvious from the name alone and avoids deep descendant selectors:

```
.scenery              .card
.scenery__band        .card__body
.scenery__streak      .card__title
.scenery__chevrons    .card__text
.scenery__streak--a   .card--everest
```

Specificity stays flat, and every rule can be traced back to a single block.

## Markup — semantic first, `div` only for shapes

Everything that carries meaning uses the correct element:

- `<main class="container">` — the page
- `<header class="intro">` with an `<h1>` — the intro block
- `<section class="mountains">` with a labelled region, and one `<article class="card">` per mountain
- `<h2>` / `<p>` / `<a>` inside each card
- `<figure class="photo">` for the two photos

`<div>` is used **only** where an element is purely presentational — the absolutely positioned
decorations and cut-out shapes that exist just to paint pixels: `.scenery__topo`,
`.scenery__band`, `.scenery__streak`, `.scenery__chevrons`. The whole `.scenery` block is marked
`aria-hidden="true"` because none of it conveys information to a screen reader.

## Responsive approach — mobile first

The stylesheet is written **mobile first**: the base rules describe the stacked single-column
layout, and one `@media (min-width: 1280px)` block layers the absolutely positioned desktop canvas
on top. Nothing is undone with `max-width` queries.

Instead of pushing every size change into breakpoints, sizing is done with **modern CSS** so most
of the scaling happens fluidly and the media query is only needed for the change in _layout_:

- **`clamp()`** for all fluid type, spacing and photo heights — e.g.
  `font-size: clamp(28px, 7vw, 44px)`, `--gutter: clamp(20px, 5vw, 40px)`.
- **`calc()`** to derive values from each other rather than repeating them — the card body's
  `padding-left` is `calc(var(--card-skew) + var(--gutter))`, so the text always clears the slanted
  edge no matter how the skew scales.
- **`clip-path`** for the diagonal cut-outs on the photos and the cards, instead of shipping extra
  images or fake overlay elements.
- **Custom properties** for per-card values (see below).
- **`container-type` / `cqw`** for the desktop canvas (see below).
- **`:focus-visible`** for a keyboard-only focus ring on the button.

Part of the goal here was also to use these newer properties deliberately as a learning exercise
rather than falling back on breakpoint-heavy CSS.

### Mobile spacing decision

The mobile layout intentionally has **no outer horizontal padding on the sections that are meant to
bleed**. The design is full-width edge to edge with no side gutters, so adding page padding would
have broken that look — the photos and the card shapes run to both edges, and only the text inside
them is inset via `--gutter`. This was a deliberate call to keep the mobile version faithful to the
full-width feel of the design.

The responsive behavior below 1280px is my own interpretation, since the Figma file only provided
the desktop frame.

## The `--design-px` unit — one design pixel

```css
.container {
  --design-px: min(1px, 0.0520833cqw);
  width: calc(1920 * var(--design-px));
}
```

The desktop design is a fixed 1920px composition where every element has an exact `top` / `left` /
`width`. Rather than converting each of those numbers to a percentage by hand (and losing the
ability to check them against Figma), `--design-px` represents **one design pixel**:

- `0.0520833cqw` is `1 / 1920` of the container width, so at 1920px `--design-px` equals exactly
  `1px` and the layout is pixel perfect.
- Below 1920px, `--design-px` shrinks proportionally, so the entire canvas scales down as one piece
  and all the diagonals stay aligned.
- `min(1px, …)` caps it at `1px` so the design never scales _up_ past its intended size on very
  wide monitors.

The result is that every offset in the media query is the **raw Figma number** —
`top: calc(932 * var(--design-px))`, `width: calc(815.4 * var(--design-px))` — which makes the CSS
directly verifiable against the design file.

### Why `cqw` and not `vw`

`100vw` includes the width of the vertical scrollbar, so a `vw`-based unit computes a canvas
slightly _wider_ than the space actually available and the page ends up with a horizontal
scrollbar. `cqw` resolves against the container's content box instead, which excludes the
scrollbar. `container-type: inline-size` is set on `<body>` to make it a query container so `cqw`
has something to resolve against.

## Per-card custom properties

Each card differs from the next only in its vertical position, its width and the two gradient
stops. Those four numbers are declared as variables on the modifier class and consumed once by the
shared rules:

```css
.card--everest {
  --card-top: 365;
  --card-width: 1358.62;
  --card-gradient-start: 14.58%;
  --card-gradient-end: 51.56%;
}
```

```css
.card {
  top: calc(var(--card-top) * var(--design-px));
  width: calc(var(--card-width) * var(--design-px));
}
.card::before {
  background: linear-gradient(
    65deg,
    var(--flame) var(--card-gradient-start),
    var(--slate) var(--card-gradient-end)
  );
}
```

This keeps all five cards on one code path — the geometry lives in data, not in five duplicated
blocks — and makes it easy to compare the values against Figma side by side. The color palette is
handled the same way, with named custom properties on `:root`.

## Working around the Figma file

The Figma file was not fully consistent, and many measurements came through as long fractions
(`4.5463`, `1358.62`, `115.75`…) rather than clean values. Where a Figma value was clearly the
result of a rounding artifact — or where honoring it exactly made the layout look _worse_ than the
design reads visually — I rounded to my own value and matched the intent instead. A couple of
examples:

- The chevron stack: Figma spaces the first two icons `99.38px` apart and the remaining ones
  `115.75px`. Since that inconsistency isn't intentional, the stack uses a single **consistent
  `115.75px` gap** via a flex column, with the container positioned at `top: 79px` / `left: 192.5px`
  (rounded from Figma's fractional values). This also removed five hard-coded `top` offsets — the
  height now comes from the content.
- Skew angles and gradient angles were rounded to one or two decimals where the extra precision
  made no visible difference.

## Image optimization

All large raster assets were converted from PNG to **lossless WebP**, which cut the total page
weight by roughly 65% with no visible quality loss:

| Asset               | PNG          | WebP (lossless) | Saved    |
| ------------------- | ------------ | --------------- | -------- |
| `everest-base-camp` | 9681 KB      | 2537 KB         | ~74%     |
| `klausdie-mountain` | 4604 KB      | 928 KB          | ~80%     |
| `vector-background` | 5466 KB      | 3423 KB         | ~37%     |
| **Total**           | **~19.3 MB** | **~6.7 MB**     | **~65%** |

The chevron icon stayed as PNG — at 712 bytes there is nothing to gain.

Images also carry explicit `width` / `height` attributes to reserve space and avoid layout shift,
and the Google Fonts request uses `preconnect` plus `display=swap`.

## Extras

- A **favicon** ([favicon.png](favicon.png)) is wired up in the `<head>`.
- Decorative imagery uses empty `alt=""` so screen readers skip it.
- The desktop-only line breaks in the card text are handled with `.card__text br { display: none }`
  on mobile, so the copy reflows naturally on small screens without duplicating the markup.

## Structure

```
index.html      markup
style.css       all styling (mobile first, one desktop breakpoint)
favicon.png
assets/         optimized WebP photos + icon
```
