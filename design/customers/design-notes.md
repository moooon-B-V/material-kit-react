Design system check. Files read: `package.json` (root; this is not a monorepo), `pnpm-lock.yaml`, `src/styles/global.css` (the only global CSS, imported by `src/app/layout.tsx`).
Verdict: **not on Motir Design.** `package.json` has no `@motir/design-system` dependency, and `src/styles/global.css` imports only `@fontsource/*` fonts, not `@motir/design-system/theme.css`. The repository has its own theme (MUI 7, `src/styles/theme/`) and shared components (`src/components/`), so branch **(b)** applies: the design is built from the repository's own components and theme.

# Customers — design notes

Area: `/dashboard/customers` (`src/app/dashboard/customers/page.tsx`).

## Empty states (MOTIR-7066)

Mock: `design/customers/customers-empty.mock.html`. It has three panels, each drawn inside the full dashboard shell.

### What ships today (rendered before drawing)

I ran `next dev` from this branch and took screenshots of `/dashboard/customers` at 1440 px and 390 px, first with the shipped array and then with the array emptied locally. That edit was never committed. With no rows, the table card shows only the column header (`Name · Email · Location · Phone · Signed Up`, with a checkbox), no body rows, and the pagination footer reading `Rows per page: 5 · 0–0 of 0`. The search card sits above it and works normally. Nothing tells the user why the table is blank. Below `lg` the SideNav is hidden and the MainNav shows its menu button, which opens `MobileNav`. The table header scrolls sideways because the table has `minWidth: 800px`.

### Built from (all existing files)

- Theme: `src/styles/theme/create-theme.ts`, `color-schemes.ts`, `colors.ts`, `typography.ts`, `shadows.ts`. The mock inlines the `--mui-*` CSS variables MUI generates from these files, copied verbatim from the running app (light scheme). Every colour, font, spacing step and shadow in the mock is one of those variables.
- Component overrides: `src/styles/theme/components/card.tsx` (20px radius and the card shadow), `button.tsx` (12px radius, padding, no text-transform), `table-head.tsx` (`background-level1` header, `text-secondary`), `table-cell.tsx` (`TableCell-border`, checkbox padding).
- Shell: `src/app/dashboard/layout.tsx` (`--SideNav-width`, `--MobileNav-width`, Container `py: 64px`), `src/components/dashboard/layout/side-nav.tsx`, `main-nav.tsx`, `mobile-nav.tsx`, `config.ts` (nav items and their order).
- Page: `src/app/dashboard/customers/page.tsx` (header row, Import / Export / Add), `src/components/dashboard/customer/customers-filters.tsx`, `customers-table.tsx`.
- Icon disc: the 56px `Avatar` with a palette background from `src/components/dashboard/overview/budget.tsx` and `total-customers.tsx`.
- Icons: `@phosphor-icons/react`, already a dependency. The mock uses its own path data for Users, MagnifyingGlass, Plus, X, Upload, Download, List, Bell and the nav icons.
- Global: `src/styles/global.css` (`--icon-fontSize-*`).

The mock adds no dependency, and it uses no `@motir/design-system` import and no Motir token.

### Access path

You reach this surface through **Customers** in the SideNav (desktop, panels 1–2) and through **Customers** in the MobileNav drawer, which the MainNav menu button opens (panel 3b). Both are drawn with a dashed outline. Neither is new.

### Settled decisions

1. **Filters card when there are no customers at all: hidden.** There is nothing to search, and an empty search box next to "No customers yet" suggests the search caused the emptiness. It comes back as soon as one customer exists.
2. **Pagination footer when the list is empty: hidden, in both states.** "0–0 of 0" with disabled arrows adds nothing. The `Divider` above it goes too. The footer returns whenever `count > 0`.
3. **Column header:**
   - *No customers yet:* hidden. The card holds only the empty block.
   - *No search match:* kept, so the card still reads as a filtered table. This is part of how the two states are told apart.
4. **Where the empty block sits:** inside the same `Card` as the table, **outside** the `Box sx={{ overflowX: 'auto' }}` / `<Table minWidth 800>`. It is not a `TableRow` with `colSpan`, because a cell inside an 800px scroller is off-centre, and can be clipped, below `md` (panel 3a).
5. **The header's Add button is unchanged.** The empty state's **Add customer** does the same thing. Today the header's Add has no handler, so MOTIR-7067 should give both buttons the same (currently no-op) action and not invent an add flow.
6. **"A search matched nothing" means** the stand-in array is non-empty and the query filters it to zero. Filtering the array by name or email is the only logic this needs. It is not a data source.

### How panels 1 and 2 are told apart at a glance

| | 1 · No customers yet | 2 · No search match |
|---|---|---|
| Search card | hidden | shown, with the query in it |
| Column header | hidden | shown |
| Icon disc | Users icon on `primary-main`, white glyph | MagnifyingGlass icon on `background-level2`, `text-secondary` glyph |
| Headline | "No customers yet" | "No customers match “‹query›”" (the query in `text-primary`) |
| Action | contained primary **Add customer** (Plus icon) | outlined neutral **Clear search** (X icon) |

Colour, the icon, the headline and the button weight all differ, so no single cue has to carry the difference.

### Copy

**No customers yet**
- Headline (h6): `No customers yet`
- Body (body2, `text-secondary`): `A customer is a person or company you sell to. Add one to keep their contact details, location and sign-up date in one place.`
- Action (contained, primary, Plus icon): `Add customer`

**No search match**
- Headline (h6): `No customers match “{query}”`. Use the trimmed query in typographic quotes. Truncate it with an ellipsis after about 40 characters.
- Body (body2, `text-secondary`): `Check the spelling, or search by a different name or email.`
- Action (outlined, inherit colour, X icon): `Clear search`. It empties the search input and shows the full list again. Focus returns to the input.

### Layout values (all theme variables)

- Empty block: a centred column, gap `spacing(2)`. Padding is `spacing(8) spacing(3)` from `md` up and `spacing(6) spacing(3)` below `md`.
- Text column: max-width 420px.
- Disc: 56 × 56, round, icon at `--icon-fontSize-lg`. This matches the overview-card Avatars.
- Headline uses the `h6` variant. Body uses `body2`.

### Narrow (`< md`) case — panel 3

- The page header, the MainNav and the MobileNav are exactly as they ship.
- The empty block fills the card's width, and the copy wraps to three lines at 390px.
- The button stays inline (auto width). It is not full-width.
- Panel 3b shows the MobileNav drawer that the menu button opens, with Customers active. Nothing in it changes.

### New component needed

- **`CustomersEmpty`**: product-local, at `src/components/dashboard/customer/customers-empty.tsx`. It is a small presentational block (disc, headline, body, one action) with two variants, `none` and `no-match` (the second takes `query` and `onClear`). The repository has no empty-state component, and this block is only needed on this page for now. It is made from MUI `Avatar`, `Stack`, `Typography` and `Button` plus Phosphor icons, all of which already ship. If a second list page later needs the same thing, it can be promoted to a shared `src/components/core/empty-state.tsx`. That is not proposed here.

No other new component is needed. `CustomersTable` needs only conditional rendering (header, footer, empty block), and `page.tsx` needs to hold the search query so both `CustomersFilters` and the table can read it. That is MOTIR-7067's build.

### Mock fidelity notes

- The mock uses initials ("SR") in place of `/assets/avatar.png` and leaves out the Pro-version illustration in the SideNav, to keep the file small. Both are unchanged in the product.
- The SideNav logo is `public/assets/logo.svg`, inlined as a data URI.
- The dark colour scheme isn't drawn. Every colour in the mock is a `--mui-palette-*` variable, so the dark scheme in `color-schemes.ts` applies unchanged.
