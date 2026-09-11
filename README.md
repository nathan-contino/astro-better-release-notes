# astro-better-release-notes

Configurable release notes components for Astro with category chip filtering, sticky version navigation, and deep-linkable release note items.

## Installation

```
npm install astro-better-release-notes
```

Import the stylesheet once in your root layout:

```astro
---
import 'astro-better-release-notes/style.css';
---
```

## Components

- `ReleaseNotes` - renders a unified sticky header (version selector + category filter chips) and the full version list
- `ReleaseNotesItem` - a single categorized release note row (used inside MDX content)
- `ReleaseNotesSelector` - a standalone version jump dropdown (for custom layouts that need it separately)

## Usage

### Content collection setup

Your content collection entries must have at minimum:

```ts
// src/content.config.ts
const releases = defineCollection({
  schema: z.object({
    version: z.string(),
    date:    z.coerce.date(),
    name:    z.string().optional(),
  }),
});
```

Each `.mdx` file in the collection uses `ReleaseNotesItem`:

```mdx
---
version: "2.1.0"
date: 2024-05-01
---
import { ReleaseNotesItem } from 'astro-better-release-notes';

<ReleaseNotesItem category="new-feature">Added dark mode.</ReleaseNotesItem>
<ReleaseNotesItem category="fix" issue="42" issueBaseUrl="https://github.com/org/repo/issues/">Fixed login redirect loop.</ReleaseNotesItem>
```

### Page component

`ReleaseNotes` renders everything in one call: a sticky header containing the version selector and filter chips, then the full version list below.

```astro
---
import { getCollection, render } from 'astro:content';
import { ReleaseNotes, DEFAULT_CATEGORIES } from 'astro-better-release-notes';

const allUpdates = await getCollection('releases');
const updates = allUpdates
  .sort((a, b) => {
    const d = b.data.date.valueOf() - a.data.date.valueOf();
    return d !== 0 ? d : b.data.version.localeCompare(a.data.version, undefined, { numeric: true });
  });

const rendered = await Promise.all(
  updates.map(async item => {
    const { Content } = await render(item);
    return { data: item.data, Content };
  })
);
---

<ReleaseNotes updates={rendered} categories={DEFAULT_CATEGORIES} />
```

### ReleaseNotes props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `updates` | `ReleaseEntry[]` | required | Pre-rendered version entries |
| `categories` | `Category[]` | `DEFAULT_CATEGORIES` | Category definitions |
| `components` | `Record<string, any>` | `{}` | Component overrides (e.g. custom `ReleaseNotesItem`) |
| `showSelector` | `boolean` | `true` | Set `false` to hide the version jump dropdown |

### Sticky offset

The sticky header sticks below your site navigation. Set the `--release-notes-top` CSS custom property to match your nav height:

```css
/* in your root layout or global CSS */
:root {
  --release-notes-top: 56px; /* your mobile nav height */
}

@media (min-width: 1024px) {
  :root {
    --release-notes-top: 80px; /* your desktop nav height */
  }
}
```

The default is `3.5rem` (56px). Set the variable on `:root`, `html`, or any ancestor of `.release-sticky-header`.

### Mobile filter behavior

On narrow screens the category filter chips are hidden behind a "Filter" toggle button that sits next to the version selector. Tapping it expands the chips below the selector row. When any filter is active, the button changes color as a visual indicator.

On screens 768px and wider, the chips are always visible.

## Custom categories

Pass a `categories` array to `ReleaseNotes` to define your own set:

```js
const MY_CATEGORIES = [
  { id: 'new-feature', label: 'New Feature', color: '#16a34a' },
  { id: 'fix',         label: 'Fix',         color: '#d97706' },
  { id: 'security',    label: 'Security',    color: '#566f98' },
];
```

Category IDs must match the `category` prop values used in your MDX files.

## Custom ReleaseNotesItem

To inject project-specific behavior (e.g. a fixed issue base URL), pass a wrapper component via the `components` prop:

```astro
---
// src/components/MyReleaseNotesItem.astro
import { ReleaseNotesItem } from 'astro-better-release-notes';
const { issue, bonusIssue, ...rest } = Astro.props;
const base = 'https://github.com/org/repo/issues/';
---
<ReleaseNotesItem
  {...rest}
  issue={issue}
  bonusIssue={bonusIssue}
  issueBaseUrl={base}
  categoryColor={rest.categoryColor}
  categoryLabel={rest.categoryLabel}
><slot /></ReleaseNotesItem>
```

```astro
<ReleaseNotes
  updates={rendered}
  categories={MY_CATEGORIES}
  components={{ ReleaseNotesItem: MyReleaseNotesItem }}
/>
```

Note: for the `components` injection to work with categories, you also need to pass `categoryColor` and `categoryLabel` in your wrapper. Use the exported helpers:

```js
import { getCategoryColor, getCategoryLabel, DEFAULT_CATEGORIES } from 'astro-better-release-notes';
```

## ReleaseNotesItem props

| Prop | Type | Description |
|------|------|-------------|
| `category` | `string` | Category ID; must match a configured category |
| `issue` | `string` | Issue number shown as a link at row end |
| `bonusIssue` | `string` | Secondary issue number |
| `thanks` | `string` | GitHub username to credit |
| `bonusThanks` | `string` | Secondary GitHub username |
| `since` | `string` | Version this was introduced |
| `resolvedIn` | `string` | Version a known issue was resolved in |
| `viaIssue` | `string` | Issue that resolved a known issue |
| `issueBaseUrl` | `string` | Base URL for issue links (no default) |
| `categoryColor` | `string` | CSS color for the badge (default `#71717a`) |
| `categoryLabel` | `string` | Human label for the badge (default: generated from id) |

## Injecting category color and label automatically

`ReleaseNotes.astro` does not automatically inject `categoryColor`/`categoryLabel` into the MDX-rendered items -- those props come from the MDX markup. To automate this, create a wrapper that looks up the category in your list and injects the values, then pass it via `components`.

## Standalone ReleaseNotesSelector

`ReleaseNotesSelector` is available as a standalone component for custom layouts that need the version dropdown without the rest of the notes. It is not sticky by default; wrap it in your own sticky container if needed.

```astro
import { ReleaseNotesSelector } from 'astro-better-release-notes';
---
<div style="position: sticky; top: 56px;">
  <ReleaseNotesSelector {updates} />
</div>
```

## Dark mode

The stylesheet uses a `.dark` ancestor class (compatible with Tailwind's `darkMode: 'class'`). To also enable automatic OS-based dark mode, add to your project CSS:

```css
@media (prefers-color-scheme: dark) {
  html:not([data-theme="light"]) {
    /* copy the .dark rules you need, or add the .dark class via JS */
  }
}
```

Most Tailwind setups initialize the `.dark` class from OS preference on first load, so no extra CSS is needed.
