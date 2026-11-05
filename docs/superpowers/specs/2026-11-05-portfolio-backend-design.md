# Portfolio Backend — Design Spec

Date: 2026-11-05
Status: Approved in brainstorming, awaiting written-spec review

## 1. Goal

Build the backend for Sarthak Neupane's portfolio in the fresh Laravel 13 app at `backend/`:

- a **Blade admin panel** at `/admin` for editing all portfolio content, using **React + Ant Design islands**
  only where interactivity pays off;
- a **read-only public JSON API** at `/api/v1` that returns exactly the shapes the Next.js frontend
  (`frontend/`) already uses in `data/*.ts` and `types/*.ts`;
- **MySQL 8** schema via migrations, with seeders that reproduce the site's current content.

Decisions made during brainstorming:

| Topic | Decision |
| --- | --- |
| Ant Design usage | Blade pages + antd islands (not an SPA, not antd for every form) |
| Data model | Mirror the frontend's content model (no generic page builder) |
| Images | Central media library with WebP variants |
| Accounts | Single admin (session login, full access, no roles) |
| Database | MySQL 8; tests on in-memory SQLite |
| Architecture | One Laravel app: Blade admin + read-only API (approach A) |

Out of scope: switching the Next.js components to the API (separate follow-up task), a contact form
(the site uses `mailto:`), multiple users/roles, password-reset email.

## 2. Current state

- `backend/`: blank Laravel 13 skeleton — Blade, Tailwind 4, Vite 8, Pest; `.env` uses SQLite; not a git repo.
- `frontend/`: Next.js 16 site. All content lives in `data/*.ts`. `lib/api.ts` already calls
  `GET {API}/projects`, `/travel`, `/notes`, unwraps `json.data ?? json`, and falls back to local data when
  `NEXT_PUBLIC_API_BASE_URL` is empty or a request fails.

## 3. Data model

All tables have `id` and timestamps. Ordered collections use `sort_order` (unsigned int, indexed).

| Table | Columns | Frontend source |
| --- | --- | --- |
| `users` | skeleton default (`name`, `email`, `password`, `remember_token`) | — |
| `media` | `uuid` (unique), `disk`, `path`, `original_name`, `mime_type`, `extension`, `size_bytes`, `width` (null), `height` (null), `alt` (null), `variants` (JSON null), soft deletes | every `ImageRef` |
| `projects` | `slug` (unique), `title`, `category`, `description` (text), `highlights` (JSON list of strings), `tech` (JSON list of strings), `href` (null), `cover_media_id` (null FK → media, nullOnDelete), `sort_order`, `is_visible` (bool, default true) | `projects` in `data/projects.ts` |
| `creative_items` | `key` (unique), `title`, `discipline`, `shape` (`tall`/`wide`/`square`), `media_id` (null FK), `sort_order` | `creativeItems` |
| `skill_groups` | `title`, `sort_order` | `skillGroups` |
| `skills` | `skill_group_id` (FK, cascadeOnDelete), `name`, `sort_order` | `skillGroups[].items` |
| `services` | `title`, `body` (text), `tech`, `cta_label`, `cta_href`, `sort_order` | `services` in `data/skills.ts` |
| `timeline_entries` | `title`, `detail` (text), `sort_order` | `timeline` in `data/experience.ts` |
| `articles` | `type` (`travel`/`note`), `slug`, `title`, `subtitle` (null), `excerpt` (text), `category`, `topic` (null, notes), `destination` (null, travel), `featured` (bool), `cover_media_id` (null FK), `status` (`draft`/`published`/`hidden`), `published_at` (null), `reading_time` (null override string), `blocks` (JSON), `sort_order`; unique (`type`, `slug`); index (`type`, `status`) | `travelStories`, `notes` |
| `travel_categories` | `name` (unique), `slug` (unique), `sort_order` | `travelFilters` / `TravelCategory` |
| `article_travel_category` | `article_id`, `travel_category_id` (both FK cascadeOnDelete, composite PK) | `TravelStory.categories` |
| `travel_photos` | `key` (unique, frontend id), `theme`, `aspect` (`portrait`/`landscape`), `media_id` (null FK), `sort_order` | `travelPhotos` |
| `nav_links` | `label`, `href`, `sort_order`, `is_visible` | `navLinks` |
| `social_links` | `label`, `url` (null = "coming soon"), `sort_order` | `site.socials` |
| `site_settings` | `key` (unique), `value` (JSON null) | `site` scalars, `marqueeItems`, `collabAreas`, `education` |

### Site settings keys

`site.name`, `site.short_name`, `site.title`, `site.description`, `site.url`, `site.email`, `site.location`,
`site.cv_media_id`, `site.portrait_media_id`, `site.portrait2_media_id`, `education`, `marquee_items`
(list), `collab_areas` (list). Keys ending in `_media_id` hold a media ID.

### Article blocks

`articles.blocks` is an ordered JSON array. Each element has a `type` and is validated server-side:

| Type | Stored fields | API output |
| --- | --- | --- |
| `heading` | `id` (slug, auto from text if empty), `text` | same |
| `paragraph` | `text` | same |
| `image` | `media_id` (nullable until an image is chosen), `caption` (optional) | `{ type, image: {src, alt}, caption? }`; no media → placeholder ImageRef |
| `quote` | `text` | same |
| `route` | `title`, `stages: [{label, detail}]` (≥1) | same |
| `tips` | `title`, `items: string[]` (≥1) | same |

### Status semantics

- `draft` → listed in the API with `draft: true`, `readingTime: "Draft"`; the site shows its DraftTag.
- `published` → `draft: false`; `readingTime` = override or `ceil(words / 200)` min read, computed from block text;
  `date` = `published_at` (ISO date).
- `hidden` → never in lists; detail returns 404.

### Media references and safe deletion

A `MediaUsage` service finds every reference to a media item: `projects.cover_media_id`,
`creative_items.media_id`, `articles.cover_media_id`, `travel_photos.media_id`, image blocks inside
`articles.blocks`, and `site_settings` keys ending in `_media_id`. Deleting referenced media is refused and
lists the usages. Media uses soft deletes; force delete also removes files and variants.

## 4. Public API (`/api/v1`, read-only, no auth)

Responses are `{ "data": ... }`. Shapes equal the TypeScript types in `frontend/types`.

| Endpoint | Returns |
| --- | --- |
| `GET /site` | `{ site, navLinks, marqueeItems, collabAreas, education, timeline, skillGroups, services, creativeItems, travelFilters, travelPhotos }` |
| `GET /projects` | `Project[]` — `index` is the zero-padded position (`"01"`), `image` is the cover, `href` only when set; only `is_visible` |
| `GET /travel` | `TravelStory[]` incl. blocks, `categories`, `featured` |
| `GET /travel/{slug}` | `TravelStory` |
| `GET /notes` | `Note[]` |
| `GET /notes/{slug}` | `Note` |

Details:

- `site` mirrors `data/site.ts`: `name`, `shortName`, `title`, `description`, `url`, `email`, `location`,
  `cvPath` (CV media URL or null), `portrait`, `portrait2`, `socials: [{label, url|null}]`.
- `services[]`: `{ index, title, body, tech, cta, href }`; `skillGroups[]`: `{ title, items: string[] }`;
  `timeline[]`: `{ title, detail }`; `creativeItems[]`: `{ id, title, discipline, shape, image }`;
  `travelPhotos[]`: `{ id, theme, aspect, image }`; `travelFilters`: category names.
- **Images** are `{ src, alt }` with `src` an absolute URL: `lg` variant for covers and article images, `md`
  for cards/gallery, original if no variant. `alt` comes from the media item's `alt`, falling back to the
  owning record's title (theme for travel photos). When no media is set the API returns
  `{ src: null, alt: <record title>, label: "Upload an image in the admin" }` so the frontend `ImageSlot`
  placeholder still renders.
- **Optional fields** (`href`, `date`, `subtitle`, `featured`) are omitted when empty, not `null`.
- Hidden or unknown slugs → `404 { "message": "Not found." }`.
- `Cache-Control: public, max-age=60` + ETag on all API responses.
- CORS: `api/*` allows origins from `CORS_ALLOWED_ORIGINS` (default `http://localhost:3000`), GET only.

## 5. Admin panel (`/admin`)

### Auth

- Blade login form, session guard, "remember me", throttled (5/min per email+IP).
- No registration or password reset. Accounts are created with `php artisan admin:create`
  (prompts for name, email, hidden password; `--force` resets an existing one).
- `/` redirects to `/admin`; guests are redirected to `/admin/login`. CSRF on all forms.

### Layout

Blade + Tailwind layout with sidebar (Dashboard, Projects, Travel, Notes, Creative, Skills, Services,
Timeline, Travel photos, Media, Site & navigation), mobile drawer, flash status messages, inline validation errors.

### Screens

| Screen | Implementation |
| --- | --- |
| Dashboard (counts, drafts to finish ordered by most recently edited) | Blade |
| Projects, Services, Timeline, Skill groups/skills, Creative items, Travel photos, Nav links, Social links | Blade index + create/edit forms; index uses `<SortableList>` island for drag reorder |
| Travel stories, Notes | Blade metadata form + `<BlockEditor>` island writing a hidden `blocks` JSON input; "Preview on site" link to `FRONTEND_URL/travel/{slug}` or `/notes/{slug}` |
| Media library | Blade page hosting `<MediaLibrary>` island (antd Table/grid, Upload, Drawer: alt text, variants, usages, delete) |
| Image fields on any form | `<MediaPicker>` island writing a media ID into a hidden input |
| Site settings | Blade form; list values (marquee, collab areas) use an antd `Select mode="tags"` island |

### Islands

- `resources/js/islands.jsx` (Vite + `@vitejs/plugin-react`) scans for `[data-island]`, parses
  `data-props` JSON and mounts the matching component inside an antd 6 `ConfigProvider` themed to the Tailwind palette.
- Islands: `MediaPicker`, `MediaLibrary`, `SortableList`, `BlockEditor`, `TagsInput`.
- Islands use session-authenticated JSON endpoints under `/admin/api` with the CSRF token from a meta tag:
  `GET/POST /admin/api/media`, `PATCH/DELETE /admin/api/media/{id}`, `GET /admin/api/media/{id}/usages`,
  `POST /admin/api/reorder` (`{ resource, ids: [...] }`, allow-listed resources).

## 6. Media pipeline

- Disk: `MEDIA_DISK` (default `public`; `s3` in production), path `media/YYYY/MM/<uuid>.<ext>`.
- Allowed: jpg, jpeg, png, webp, gif, avif, pdf; max `MEDIA_MAX_UPLOAD_KB` (default 20480). SVG disabled.
- Images: re-encode JPEG/WebP originals to strip EXIF/GPS (orientation baked in); WebP variants
  `thumb` 400, `md` 1024, `lg` 1920 px wide, never upscaled (Intervention Image 3, GD driver).
- Width/height stored on upload. Variant generation runs synchronously.

## 7. Validation, errors, integrity

- Form Requests for every admin write; block JSON validated per type (section 3).
- Slugs auto-generated from title, unique per article type / per project.
- Article saves (metadata, travel categories, blocks) and reorders run in DB transactions.
- API 404s return `{ "message": "Not found." }`; no model names leak.

## 8. Seeding

- `database/seeders/data/*.json`: a one-time export of `frontend/data/*.ts` (generated by a small node
  script kept in `backend/scripts/export-frontend-data.mjs`, run once and its output committed).
- Seeders are idempotent and recreate the current site content: site settings, nav, socials, marquee,
  collab areas, education, timeline, skill groups, services, projects, creative items, travel categories,
  travel stories (draft), notes (draft), travel photos. No images are seeded; image fields stay empty so
  the frontend shows placeholders.

## 9. Testing (Pest, in-memory SQLite)

- Auth: login, throttling, logout, guest redirect from `/admin`.
- API: response keys/shapes for `/site`, `/projects`, `/travel`, `/travel/{slug}`, `/notes`; hidden → 404;
  drafts carry `draft: true` and `readingTime: "Draft"`; published reading time computed; optional fields omitted.
- Blocks: per-type validation accepts valid and rejects invalid input; image blocks render `{src, alt}`.
- Media: upload creates variants and strips EXIF; in-use media cannot be deleted; usages listed.
- Reorder endpoint persists order; unknown resource rejected.
- Seeders produce the same counts as the frontend data.
- `npm run build` must compile the Blade CSS and islands bundle.

## 10. Setup deliverables

- First step, per `backend/CLAUDE.md`: `php artisan boost:install` (Laravel Boost is already a dev
  dependency), then follow the generated guidelines.
- Composer: `intervention/image:^3.11`. npm: `react`, `react-dom`, `antd@^6.7`, `@ant-design/icons`, `@dnd-kit/core`, `@dnd-kit/sortable`,
  `@dnd-kit/utilities`, `@vitejs/plugin-react`.
- `.env.example`: MySQL 8 (`DB_CONNECTION=mysql`), `FRONTEND_URL`, `CORS_ALLOWED_ORIGINS`, `MEDIA_DISK`,
  `MEDIA_MAX_UPLOAD_KB`.
- `README.md`: setup (`composer install`, `npm install`, `migrate --seed`, `admin:create`, `storage:link`,
  `composer run dev`), env vars, API reference, frontend integration notes (`NEXT_PUBLIC_API_BASE_URL`,
  `images.remotePatterns`).

## 11. Follow-up (separate task)

Update `frontend/lib/api.ts` with `getSite()` and helpers for every collection, and switch components from
`@/data/*` imports to those helpers, keeping the local-data fallback.
