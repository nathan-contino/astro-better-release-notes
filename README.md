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

- `ReleaseNotes` - renders the filter chips and the full version list
- `ReleaseNotesItem` - a single categorized release note row (used inside MDX content)
- `ReleaseNotesSelector` - a sticky version jump dropdown

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

```astro
---
import { getCollection, render } from 'astro:content';
import {
  ReleaseNotes,
  ReleaseNotesSelector,
  DEFAULT_CATEGORIES,
} from 'astro-better-release-notes';

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

<ReleaseNotesSelector updates={rendered} />
<ReleaseNotes updates={rendered} categories={DEFAULT_CATEGORIES} />
```

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

Or provide these values in your own wrapper by computing them from your category list.

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

## Dark mode

The stylesheet supports both `@media (prefers-color-scheme: dark)` and a `.dark` ancestor class.
