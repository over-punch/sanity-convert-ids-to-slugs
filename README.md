# Sanity Convert IDs to Slugs

A Sanity Studio utility component that **migrates a typeface's font document `_id`s from auto-generated IDs to slug-based IDs** — and, in the same pass, **rewrites every reference and deletes the originals** so nothing is left dangling. The payoff is cleaner, human-readable document IDs (and the tidier URLs and content management that follow); the mechanism is a destructive, irreversible migration, gated behind an explicit Danger Mode.

[![npm](https://img.shields.io/npm/v/@overpunch/sanity-convert-ids-to-slugs.svg)](https://www.npmjs.com/package/@overpunch/sanity-convert-ids-to-slugs)
![Sanity](https://img.shields.io/badge/Sanity-Studio_v3_to_v6-f03e2f.svg)
![React](https://img.shields.io/badge/React-18_and_19-61dafb.svg)
![license](https://img.shields.io/badge/license-MIT-blue.svg)

> **Heads up — this tool rewrites and deletes documents.** For each font it
> create-replaces the document under a **new `_id`** (the slug), repoints every
> referencing document, then **deletes the original**. This is **irreversible**
> and there is **no dry-run** in the published build. Read
> [Safety model](#safety-model) and **back up your dataset** before running it
> on production data.

---

## Blast radius

This is the most destructive tool in the Liiift Sanity suite. It **creates,
rewrites and deletes documents** in a single unattended pass. Read this in full
before running it anywhere but a throwaway dataset copy.

| Question | Answer |
|---|---|
| **What does it mutate?** | Three separate kinds of write per font: a **new document created** at `_id = slug.current`; **every document that references the old ID patched** to point at the new one; and the **original document deleted**. |
| **Does it delete?** | **Yes** — `client.delete(oldId)` on every successfully migrated font. |
| **Is it reversible?** | **No.** There is no undo, no dry-run, and no confirmation beyond the Danger Mode toggle. Recovery means restoring from a `sanity dataset export` you took first. |
| **Drafts or published?** | **Both, inconsistently — this is the sharpest edge.** See below. |
| **Blind spot** | `createOrReplace` **silently overwrites** any document that already occupies the target slug ID. See below. |
| **Scope of a mistake** | The whole reference graph. Every document anywhere in the dataset that references a migrated font gets patched, not just the selected typeface. |
| **Progress reporting** | **Console only.** The panel itself shows no progress, no success state and no error state — keep the browser console open or you are running blind. |

### Drafts are handled inconsistently

- The **typeface picker** correctly lists published documents only
  (`!(_id in path('drafts.**'))`).
- The **conversion itself re-queries** with
  `*[_type == "typeface" && title match "${title}*"][0]` — **no draft filter, and
  no ordering**. It can therefore resolve the *draft* of your typeface instead of
  the published one, and with a prefix match it can resolve a *different*
  typeface that merely shares the prefix.
- `*[references(oldId)]` **includes drafts**, so draft documents are patched too.
- `client.delete(oldId)` removes **only that exact `_id`**. Any
  **`drafts.<oldId>` is left behind**, still carrying the old ID. Publishing that
  orphaned draft afterwards resurrects the document you just migrated away from.

**Publish everything before running**, as the panel itself advises — with pending
drafts in play the outcome is genuinely hard to predict.

### Slug collisions overwrite without warning

The new document is written with
`client.createOrReplace({...oldDoc, _id: slug})`. If a document **already exists
at that `_id`** — another font that legitimately owns the slug, or a leftover
from an earlier run — it is **replaced outright and its contents are lost**. The
tool does not check for a collision and does not report one.

### A failed run leaves the dataset half-migrated

The migration loop has no error handling. If any step throws — a permissions
error, a network blip, a strong reference refusing a delete — the loop aborts
where it stands. Fonts already processed stay migrated; the rest do not. A font
can also be left with the **new document created but the old one not yet
deleted**, leaving both IDs live at once. Re-read the console output before
deciding whether to re-run.

---

## How it works

You give it a Sanity client and toggle on Danger Mode. It loads the `typeface`
documents in your dataset; you pick one. On **Convert**, it resolves that
typeface's font references (`styles.fonts[]._ref`) and, for each font that has a
`slug.current`, it:

1. **Create-replaces** the document under a new `_id` equal to `slug.current`
   (`client.createOrReplace`).
2. **Finds every document that references the old ID** (`references(oldId)`) and
   rewrites each `_ref` from the old ID to the slug (`patch().set().commit()`).
3. **Deletes the original document** (`client.delete(oldId)`).

Fonts without a `slug.current` are skipped and logged. Progress is reported to
the browser console.

<p align="center">
  <img
    src="https://raw.githubusercontent.com/over-punch/sanity-convert-ids-to-slugs/main/assets/data-flow.svg?v=1"
    alt="Data flow: pick a typeface in the Danger Mode panel; a GROQ query resolves the typeface's font references against the Sanity dataset; for each font _id, if it has a slug.current the tool create-replaces the document under the slug as its new _id, rewrites every referencing document from the old ID to the slug, and deletes the original (irreversible) — fonts without a slug are skipped."
    width="560"
  />
</p>

Regenerate the diagram with `npm run capture` (source: `scripts/data-flow.mmd`).

---

## Features

- 🔑 **ID → slug migration** — replaces auto-generated document `_id`s with their
  own `slug.current`, giving you readable, stable IDs.
- 🔗 **Automatic reference updating** — every document that referenced the old ID
  is repointed to the new slug-based ID, so no reference is left dangling.
- 🧹 **Old document cleanup** — the original auto-ID document is deleted once its
  references have moved.
- 🛡️ **Danger-Mode gated** — the convert UI only appears behind an explicit,
  modal-confirmed Danger Mode toggle (suppressible for 48 hours).
- 🧰 **Drop-in component** — a plain React component for Sanity Studio; mount it
  in a tool, a structure view, or a dashboard widget.

---

## Installation

```bash
npm install @overpunch/sanity-convert-ids-to-slugs
```

> The package is **scoped** — use the full `@overpunch/…` name. There is no
> unscoped `sanity-convert-ids-to-slugs` package.

---

## Requirements

This package supports **Sanity Studio v3, v4, v5 and v6** from a single build.
It declares the following peer dependencies (you provide them):

| Peer | Declared range | What that means |
|------|----------------|-----------------|
| `sanity` | `>=3 <7` | Studio **v3 through v6** |
| `@sanity/ui` | `>=2 <5` | v2, v3, v4 — see the note below, `<5` is **correct** for Studio v6 |
| `@sanity/icons` | `>=2 <6` | v2 through v5 |
| `react` | `^18.0.0 \|\| ^19.0.0` | React 18 or 19 |

> The `@sanity/ui` ceiling of `<5` looks like a mistake at a glance and is not.
> **Studio v6 ships `@sanity/ui` v4, not v5** — so `>=2 <5` covers every Studio
> major listed above.

The `client` you pass must have **write and delete** access for the conversion
to commit.

### How one build spans four Studio majors

The two libraries made breaking changes that are invisible to the type-checker:

- **`@sanity/ui` v4** moved `Tooltip`, `Menu`, `MenuButton`, `MenuItem`, `Code`,
  `Popover`, `Autocomplete`, `Toast` and `useToast` out of the package root and
  into subpath entries.
- **`@sanity/icons` v5** removed every named `*Icon` export.

The trap is that **both packages still *declare* the removed names in their
`.d.ts`, typed as `never`.** A named import therefore type-checks cleanly,
compiles, ships — and then throws at runtime in the Studio.

So this package **imports no `@sanity/ui` or `@sanity/icons` symbol directly.**
Every primitive and icon is routed through
[`@overpunch/sanity-ui-compat`](https://www.npmjs.com/package/@overpunch/sanity-ui-compat),
which resolves the *installed* namespace at runtime and falls back to a plain DOM
element if a given primitive is absent. That indirection, not a version matrix in
CI, is what makes one artifact work across v3–v6.

> **How far this is actually verified.** v6 support rests on the declared peer
> ranges, a green build, and use in three in-house Studios. It has **not** been
> exercised broadly in a running Sanity 6 Studio — treat v6 as supported and
> lightly travelled, and please file an issue if you hit a gap.

---

## Quick start

The package's **default export** is the `ConvertIdsToSlug` component. Render it
inside a Sanity Studio tool, dashboard widget, or any custom view, passing it a
client and a small amount of state to track Danger Mode.

```tsx
import {useState} from 'react'
import {useClient} from 'sanity'
import {TransferIcon} from '@sanity/icons'
import ConvertIdsToSlug from '@overpunch/sanity-convert-ids-to-slugs'

export default function IdSlugMigrator() {
	const client = useClient({apiVersion: '2024-01-01'})
	const [dangerMode, setDangerMode] = useState(false)

	return (
		<ConvertIdsToSlug
			client={client}
			displayName="Convert IDs to Slug"
			icon={TransferIcon}
			utilityId="convert-ids-to-slug"
			dangerMode={dangerMode}
			onDangerModeChange={(_utilityId, enabled) => setDangerMode(enabled)}
		/>
	)
}
```

The component asks for danger-mode *intent* (via `onDangerModeChange`), but
**you own the `dangerMode` boolean** — keep it in state and scope it however your
Studio needs. The typeface picker and **Convert** button only render while
`dangerMode` is `true`.

### Mounting it in Studio

`ConvertIdsToSlug` is a plain component, so wire it in wherever you put custom UI.
A minimal tool registration:

```tsx
// sanity.config.ts
import {defineConfig} from 'sanity'
import {TransferIcon} from '@sanity/icons'
import IdSlugMigrator from './IdSlugMigrator' // the component from the quick start above

export default defineConfig({
	// ...project, dataset, plugins, schema...
	tools: (prev) => [
		...prev,
		{
			name: 'convert-ids-to-slug',
			title: 'Convert IDs to Slug',
			icon: TransferIcon,
			component: IdSlugMigrator,
		},
	],
})
```

`utilityId` is a stable string you assign per instance. It is echoed back as the
first argument to `onDangerModeChange`, so if you render several utilities you
can tell which one toggled Danger Mode and track each one's state independently.

---

## Props

`ConvertIdsToSlug` (default export):

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `client` | `SanityClient` | ✅ | Authenticated Sanity client (typically from `useClient`). Needs **write and delete** access for the migration to commit. |
| `displayName` | `string` | ✅ | Heading shown above the utility. |
| `utilityId` | `string` | ✅ | Stable identifier for this instance, passed back in `onDangerModeChange`. |
| `dangerMode` | `boolean` | ✅ | Whether the destructive convert UI is shown. You control this value. |
| `onDangerModeChange` | `(utilityId: string, enabled: boolean) => void` | ✅ | Called when the user toggles Danger Mode (after confirming the warning modal). |
| `icon` | `React.ComponentType<{style?: React.CSSProperties}>` | – | Optional icon rendered in the heading. |

---

## Data-model assumptions

The published component is built for this monorepo's **typeface → fonts** model
rather than an arbitrary document type. Concretely, it:

- queries `*[_type == "typeface"]` to populate the picker;
- reads the selected typeface's font references from `styles.fonts[]._ref`;
- migrates each referenced font document whose `slug.current` is set, using that
  slug as the new `_id`.

If your schema differs, adapt the source (`src/ConvertIdsToSlug.jsx`) — the
field paths and `_type` are not yet configurable via props.

---

## Safety model

This utility performs **destructive, irreversible** writes. Treat every run as a
one-way migration:

- **Create-replace + delete.** Each font document is rewritten under a new `_id`
  (the slug) and the **original is deleted**. There is no automatic undo.
- **Reference rewriting.** Every document referencing the old ID is patched to
  point at the new slug-based ID. A failure partway through can leave a mix of
  old and new IDs, so review the console output.
- **No dry-run** in the published build. Nothing is previewed before it commits.
- **Danger Mode gate.** The convert UI is hidden until you enable Danger Mode,
  which raises a warning modal. The warning can be suppressed for 48 hours
  (`localStorage`), so it will **not** re-prompt every time.

Before running on real data: **publish your pending changes**, **back up the
dataset** (e.g. `sanity dataset export`), and verify on a copy first.

---

## Regenerating the diagram

The data-flow diagram is generated from a committed source so anyone can refresh
it after a behavior change:

```bash
npm run capture   # renders scripts/data-flow.mmd -> assets/data-flow.svg
```

It uses [`@mermaid-js/mermaid-cli`](https://github.com/mermaid-js/mermaid-cli)
via `npx`. The `assets/` directory is intentionally kept **out of the npm
tarball** (only `dist` and `src` are published); registries pull README images
from the repo over absolute raw URLs.

---

## License

MIT © Quinn Keaveney
