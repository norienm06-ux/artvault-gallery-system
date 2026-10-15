# ArtVault Gallery System

An AppSheet application for managing an art gallery's artists, paintings, exhibitions, venues, and categories — built on top of a normalized Google Sheets data source.

---

## 1. Overview

ArtVault Gallery System lets gallery staff:

- Maintain a catalog of **artists** and the **paintings** they've created
- Classify paintings by **category** (Portrait, Landscape, Abstract, etc.)
- Organize **exhibitions** at specific **venues**
- Track which paintings are displayed in which exhibition, and in what order/section
- See aggregate, computed insights (not just raw lists) through a dashboard
- View a single, flattened summary of exhibition → venue → painting → artist → category data without manually cross-referencing five tables

---

## 2. Entity-Relationship Design

### Entities

| Table | Primary Key | Purpose |
|---|---|---|
| Artist | ArtistID | Stores artist identity and contact info |
| Category | CategoryID | Classification lookup (Portrait, Abstract, etc.) |
| Venue | VenueID | Physical exhibition locations |
| Painting | PaintingID | Individual artwork records |
| Exhibition | ExhibitionID | A scheduled or completed show |
| ExhibitionPainting | ExhibitionPaintingID | Junction table: which paintings appear in which exhibition |

### Relationships

- **Artist → Painting** (1-to-many): one artist creates many paintings
- **Category → Painting** (1-to-many): one category classifies many paintings
- **Venue → Exhibition** (1-to-many): one venue hosts many exhibitions
- **Exhibition ↔ Painting** (many-to-many, resolved via **ExhibitionPainting**): an exhibition displays many paintings, and a painting can appear in multiple exhibitions over time

### Normalization rationale

Data is split across six tables instead of one flat sheet to avoid update anomalies — e.g., an artist's email is stored once in Artist, not repeated on every painting row they've created. Updating it in one place keeps every related record consistent. The tradeoff (data that's harder to read in one glance) is solved at the *view* layer, not the storage layer — see Section 5.

---

## 3. Core Features

### 3.1 CRUD on every table
Every table (Artist, Category, Venue, Painting, Exhibition, ExhibitionPainting) supports Add, Edit, and Delete directly from the app, with Ref-based dropdowns enforcing valid foreign key values at entry time (e.g., a Painting's ArtistID can only be set to an artist that actually exists).

### 3.2 Dashboard (aggregates, not lists)
The Home view combines five computed panels:

| Panel | Type | Question it answers |
|---|---|---|
| PaintingStatusChart | Aggregate pie chart | How much of the collection is on display vs. in storage? |
| PaintingsPerArtistChart | Histogram | Which artists make up the bulk of the collection? |
| PaintingsPerCategoryChart | Histogram | Which category has the most paintings? |
| PaintingsPerExhibitionChart | Histogram | How many paintings are displayed in each exhibition? |
| ExhibitedPaintingsList | Filtered table (slice-based) | Which specific paintings are currently Exhibited? (drill-down companion to the status chart) |

Four panels are true aggregates (count/group operations); the fifth is a deliberate drill-down paired with an aggregate, so a percentage or count is never left unactionable.

### 3.3 Denormalized view — Gallery Overview
A single Table view, built on the `ExhibitionPainting` table, that flattens data from **five tables** (Exhibition, Venue, Painting, Artist, Category) into one readable row per exhibition-painting pairing. Achieved via chained lookup (virtual) columns, e.g.:
```
ArtistNameLookup  =  [PaintingID].[ArtistNameLookup]
VenueNameLookup   =  [ExhibitionID].[VenueNameLookup]
```
The underlying tables remain normalized; only the *view* denormalizes, for readability — storage and presentation are deliberately decoupled.

**Built for scale:** the view is grouped by exhibition, sorted by exhibition start date, and filtered to a slice (`ExhibitionPainting_CurrentUpcoming`) that excludes already-ended exhibitions — so it stays usable as the dataset grows, rather than rendering one unfiltered flat list.

### 3.4 Hidden system fields
Every table's primary key (ArtistID, CategoryID, VenueID, PaintingID, ExhibitionID, ExhibitionPaintingID) is hidden from the UI (`Show? = false`) while remaining the active `Key` internally. Surrogate keys are system-internal identifiers, not business data — users identify records by name/title (the Label column), not by ID.

### 3.5 Quick-action buttons
- **Mark as Exhibited** (Painting) — flips Status from Available to Exhibited in one tap
- **Mark as Completed** (Exhibition) — flips Status from Planned to Completed in one tap
Both use conditional visibility (`Only if this condition is true`) so the button only appears when the action is actually valid for that record's current state.

### 3.6 Image uploads
The `PaintingImage` column (Image type, no formula) lets staff attach a real photo per painting directly from the app — camera capture on mobile, file picker on desktop — rather than requiring manual spreadsheet edits.

---

## 4. Branding
- Custom app name, logo, and launch image
- Dark theme with a custom primary accent color
- Status-based format rules (color-coded text) on Painting and Exhibition Status fields

---

## 5. Data Integrity Notes

- **Hard deletes, not soft deletes:** this app does not implement a soft-delete (`IsDeleted` flag) pattern. A delete action physically removes the row from the Google Sheet. The only recovery path is Google Sheets' built-in **Version History** (File → Version history), which restores a prior state of the entire sheet — not a single row. There is no AppSheet-level "Recycle Bin" for individual records.
- **Referential integrity is enforced at the application layer**, not the data layer. Ref columns and the delete guard only protect data when edits happen *through the app*. Editing the underlying Google Sheet directly bypasses all of this — a deleted Artist row edited directly in Sheets would leave every referencing Painting's ArtistID unresolved (Ref lookups would return blank), since Google Sheets itself has no concept of foreign keys.

---

## 6. Deployment

- App was run through AppSheet's **Deployment Check** (Manage → Deploy) and moved to Deployed state.
- Verified functional on a separate device (phone) not used during development — Add, Edit, and Delete tested live across multiple tables.
- Two deployment-check warnings (App description, Content caching) and one account-tier notice (free-plan sign-in restriction) are expected on a free AppSheet account and do not reflect app-level defects — confirmed with instructor as normal for this project scope.

---

## 7. Tech Summary

| Layer | Technology |
|---|---|
| App platform | Google AppSheet |
| Data source | Google Sheets (6 linked tables) |
| Relationships | AppSheet Ref columns |
| Computed fields | Virtual columns (lookup formulas) |
| Dashboard | AppSheet Dashboard view (chart + slice-based table panels) |
| Hosting/Access | AppSheet-hosted mobile/web app link |
