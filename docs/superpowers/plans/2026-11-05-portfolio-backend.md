# Portfolio Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Laravel 13 Blade admin panel (with React + Ant Design islands) and a read-only `/api/v1` JSON API whose responses match the Next.js frontend's `data/*.ts` shapes, backed by MySQL 8 migrations and seeders that reproduce the current site content.

**Architecture:** One Laravel app in `backend/`. Server-rendered Blade pages handle auth, layout, lists and forms; small React islands (antd 6) mount into `[data-island]` elements for the media picker/library, drag-to-reorder lists, the article block editor and tag inputs, talking to session-authenticated JSON endpoints under `/admin/api`. The public API presents Eloquent models as the frontend's TypeScript shapes; images are resolved through a central `media` table with WebP variants.

**Tech Stack:** PHP 8.4 (Herd), Laravel 13, Pest 5, MySQL 8 (tests: in-memory SQLite), Intervention Image 3 (GD), Tailwind CSS 4, Vite 8, React 19, Ant Design 6, dnd-kit.

**Spec:** `docs/superpowers/specs/2026-11-05-portfolio-backend-design.md`

## Global Constraints

- Working directory for every command: `C:\projects\my-portfolio\backend` (Windows; use Git Bash or PowerShell).
- PHP via Herd (`php`, `composer` on PATH). Node 24.
- Composer additions: `intervention/image:^3.11` only.
- npm additions (exact ranges): `react@^19.3.1`, `react-dom@^19.3.1`, `antd@^6.7.0`, `@ant-design/icons@^6.3.4`, `@dnd-kit/core@^6.3.1`, `@dnd-kit/sortable@^10.0.0`, `@dnd-kit/utilities@^3.2.2`; dev: `@vitejs/plugin-react@^6.2.1`.
- Models use the skeleton's attribute style: `#[Fillable([...])]`, `#[Table('...')]` where needed, `casts()` method.
- Tests: Pest feature tests with `RefreshDatabase`, in-memory SQLite from `phpunit.xml`. `tests/TestCase.php` calls `$this->withoutVite()`.
- Public API: prefix `/api/v1`, every response wrapped as `{ "data": ... }`, keys and shapes identical to `frontend/types/*.ts`; optional fields (`href`, `date`, `subtitle`, `featured`) are omitted, never `null`.
- Missing image → exactly `{ "src": null, "alt": <fallback>, "label": "Upload an image in the admin" }`.
- API 404 body exactly `{ "message": "Not found." }`.
- API responses carry `Cache-Control` with `public` and `max-age=60`, plus an `ETag`.
- Media: allowed extensions `jpg, jpeg, png, webp, gif, avif, pdf`; max `20480` KB; SVG not allowed; WebP variants `thumb` 400, `md` 1024, `lg` 1920 px wide, quality 82, never upscaled; JPEG/WebP originals re-encoded at quality 90 to drop EXIF.
- Article `type`: `travel` | `note`. `status`: `draft` | `published` | `hidden`.
- Reading time for published articles: `max(1, ceil(words / 200)) . " min read"` unless `reading_time` override is set; drafts return `"Draft"`.
- Admin login throttled to 5 attempts per minute per lowercase email + IP.
- Commit after every task (the repo is created in Task 0).

---

## File Structure

```
backend/
  app/
    Admin/AdminResource.php                 base class for simple CRUD resources
    Admin/AdminResources.php                registry of resource classes
    Admin/Resources/*.php                   Projects, Services, TimelineEntries, SkillGroups,
                                            CreativeItems, TravelPhotos, NavLinks, SocialLinks,
                                            TravelCategories
    Console/Commands/CreateAdminCommand.php admin:create
    Enums/ArticleStatus.php, ArticleType.php
    Exceptions/MediaInUseException.php      409 with usages
    Http/Controllers/Admin/                 Auth/LoginController, DashboardController,
                                            ResourceController, ArticleController,
                                            SiteSettingsController
    Http/Controllers/Admin/Api/             MediaController, ReorderController
    Http/Controllers/Api/V1/                SiteController, ProjectController, ArticleController
    Http/Requests/Admin/ArticleRequest.php
    Http/Resources/Admin/MediaResource.php
    Http/Resources/Api/ArticleResource.php, ProjectResource.php
    Models/                                 Media, Project, CreativeItem, SkillGroup, Skill,
                                            Service, TimelineEntry, Article, TravelCategory,
                                            TravelPhoto, NavLink, SocialLink, SiteSetting
    Models/Concerns/HasSortOrder.php
    Services/Media/MediaStore.php, MediaUsage.php
    Support/ArticleBlocks.php, ImageRef.php, MediaLookup.php, Slug.php
  config/portfolio.php, config/cors.php
  database/migrations/2026_11_05_0000{01..04}_*.php
  database/factories/MediaFactory.php
  database/seeders/DatabaseSeeder.php, PortfolioSeeder.php, data/frontend.json
  scripts/export-frontend-data.mjs
  resources/css/app.css
  resources/js/islands.jsx, resources/js/api.js, resources/js/islands/*.jsx
  resources/views/components/island.blade.php, components/admin/layout.blade.php
  resources/views/admin/...                 auth/login, dashboard, media, resources/{index,form},
                                            articles/form, settings, partials/field
  routes/web.php, routes/api.php
  tests/Feature/*.php
```

---

### Task 0: Repository, tooling and configuration

**Files:**
- Modify: `tests/Pest.php`, `tests/TestCase.php`, `.env.example`, `package.json` (via npm), `composer.json` (via composer)
- Create: `config/portfolio.php`, `config/cors.php`

**Interfaces:**
- Produces: config keys `portfolio.frontend_url`, `portfolio.media.disk`, `portfolio.media.max_upload_kb`, `portfolio.media.extensions`, `portfolio.media.variants`, `portfolio.media.webp_quality`; `RefreshDatabase` + `withoutVite()` for every feature test.

- [ ] **Step 1: Create the git repository and baseline commit**

```bash
git init
git add -A
git commit -m "chore: Laravel 13 Blade skeleton"
```

- [ ] **Step 2: Install Laravel Boost guidelines (required by `backend/CLAUDE.md`)**

Run: `php artisan boost:install`
Choose Claude Code as the agent when prompted and accept the default guidelines. Then re-read `CLAUDE.md` / `AGENTS.md`; follow any conventions they add in the remaining tasks.

- [ ] **Step 3: Install dependencies**

```bash
composer require intervention/image:^3.11
npm install react@^19.3.1 react-dom@^19.3.1 antd@^6.7.0 @ant-design/icons@^6.3.4 @dnd-kit/core@^6.3.1 @dnd-kit/sortable@^10.0.0 @dnd-kit/utilities@^3.2.2
npm install -D @vitejs/plugin-react@^6.2.1
```

- [ ] **Step 4: Enable database refresh and disable Vite in tests**

In `tests/Pest.php` replace

```php
pest()->extend(TestCase::class)
 // ->use(RefreshDatabase::class)
    ->in('Feature');
```

with

```php
pest()->extend(TestCase::class)
    ->use(RefreshDatabase::class)
    ->in('Feature');
```

Replace `tests/TestCase.php` with:

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function setUp(): void
    {
        parent::setUp();

        // Blade layouts call @vite; tests never need the compiled manifest.
        $this->withoutVite();
    }
}
```

- [ ] **Step 5: Add portfolio and CORS configuration**

Create `config/portfolio.php`:

```php
<?php

return [

    // Public Next.js site, used for "Preview on site" links.
    'frontend_url' => rtrim((string) env('FRONTEND_URL', 'http://localhost:3000'), '/'),

    'media' => [
        'disk' => env('MEDIA_DISK', 'public'),
        'max_upload_kb' => (int) env('MEDIA_MAX_UPLOAD_KB', 20480),
        'extensions' => ['jpg', 'jpeg', 'png', 'webp', 'gif', 'avif', 'pdf'],
        // Ascending widths; a variant is skipped once the original is no wider than the previous one.
        'variants' => ['thumb' => 400, 'md' => 1024, 'lg' => 1920],
        'webp_quality' => 82,
    ],

];
```

Create `config/cors.php`:

```php
<?php

return [

    'paths' => ['api/*'],

    'allowed_methods' => ['GET'],

    'allowed_origins' => array_values(array_filter(array_map(
        'trim',
        explode(',', (string) env('CORS_ALLOWED_ORIGINS', 'http://localhost:3000')),
    ))),

    'allowed_origins_patterns' => [],

    'allowed_headers' => ['Accept', 'Content-Type', 'If-None-Match'],

    'exposed_headers' => ['ETag'],

    'max_age' => 3600,

    'supports_credentials' => false,

];
```

- [ ] **Step 6: Point `.env.example` at MySQL 8 and add portfolio variables**

In `.env.example` replace

```
DB_CONNECTION=sqlite
# DB_HOST=127.0.0.1
# DB_PORT=3306
# DB_DATABASE=laravel
# DB_USERNAME=root
# DB_PASSWORD=
```

with

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=portfolio
DB_USERNAME=portfolio
DB_PASSWORD=
```

and append:

```
# Portfolio
FRONTEND_URL=http://localhost:3000
CORS_ALLOWED_ORIGINS=http://localhost:3000
MEDIA_DISK=public
MEDIA_MAX_UPLOAD_KB=20480
```

Also set `APP_NAME="Portfolio Admin"` in `.env.example`.

- [ ] **Step 7: Run the existing suite**

Run: `php artisan test`
Expected: PASS (the skeleton's two example tests).

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "chore: tooling, dependencies and portfolio config"
```

---

### Task 1: Schema and models

**Files:**
- Create: `database/migrations/2026_11_05_000001_create_media_table.php`, `..._000002_create_portfolio_tables.php`, `..._000003_create_article_tables.php`, `..._000004_create_site_tables.php`
- Create: `app/Enums/ArticleType.php`, `app/Enums/ArticleStatus.php`, `app/Models/Concerns/HasSortOrder.php`, all models listed in File Structure, `database/factories/MediaFactory.php`
- Test: `tests/Feature/SchemaTest.php`

**Interfaces:**
- Produces:
  - `ArticleType::Travel|Note` (`'travel'|'note'`), `->segment(): string` (`'travel'|'notes'`), `->label(): string` (`'Travel stories'|'Notes'`).
  - `ArticleStatus::Draft|Published|Hidden`.
  - Scope `ordered()` on every sortable model (orders by `sort_order`, `id`); new rows get `max(sort_order) + 1`.
  - `Media::url(?string $variant = null): string`, `Media::isImage(): bool`; `variants` cast `array` shaped `['thumb' => ['path'=>..., 'width'=>int, 'height'=>int], ...]`.
  - `Project::cover()`, `Article::cover()`, `Article::travelCategories()`, `TravelCategory::articles()`, `SkillGroup::skills()`, `CreativeItem::media()`, `TravelPhoto::media()`, `NavLink` scope `visible()`.
  - `SiteSetting::read(string $key, mixed $default = null): mixed`, `SiteSetting::write(string $key, mixed $value): void`, `SiteSetting::allValues(): array<string, mixed>`.

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/SchemaTest.php`:

```php
<?php

use App\Enums\ArticleStatus;
use App\Enums\ArticleType;
use App\Models\Article;
use App\Models\Media;
use App\Models\Project;
use App\Models\SiteSetting;
use App\Models\SkillGroup;
use App\Models\TravelCategory;
use Illuminate\Database\QueryException;

it('stores portfolio records with casts, relations and sort order', function () {
    $media = Media::factory()->create();

    $first = Project::create([
        'slug' => 'ez-bus', 'title' => 'EZ-Bus', 'category' => 'Transport', 'description' => 'Booking',
        'highlights' => ['Seat selection'], 'tech' => ['Laravel'], 'cover_media_id' => $media->id,
    ]);
    $second = Project::create(['slug' => 'sathipro', 'title' => 'SathiPro', 'category' => 'Platform', 'description' => 'Services']);

    expect($first->highlights)->toBe(['Seat selection'])
        ->and($first->cover->is($media))->toBeTrue()
        ->and($first->is_visible)->toBeTrue()
        ->and([$first->sort_order, $second->sort_order])->toBe([1, 2])
        ->and(Project::ordered()->pluck('slug')->all())->toBe(['ez-bus', 'sathipro']);

    $group = SkillGroup::create(['title' => 'Development']);
    $group->skills()->create(['name' => 'PHP']);
    expect($group->skills->pluck('name')->all())->toBe(['PHP']);

    $category = TravelCategory::create(['name' => 'Trekking', 'slug' => 'trekking']);
    $article = Article::create([
        'type' => ArticleType::Travel, 'slug' => 'langtang', 'title' => 'Langtang', 'excerpt' => 'Trek',
        'category' => 'Trekking', 'destination' => 'Langtang Valley', 'status' => ArticleStatus::Draft,
        'blocks' => [['type' => 'paragraph', 'text' => 'Hello']],
    ]);
    $article->travelCategories()->attach($category);

    $fresh = $article->fresh();
    expect($fresh->blocks)->toBe([['type' => 'paragraph', 'text' => 'Hello']])
        ->and($fresh->status)->toBe(ArticleStatus::Draft)
        ->and($fresh->type)->toBe(ArticleType::Travel)
        ->and($fresh->travelCategories->pluck('name')->all())->toBe(['Trekking'])
        ->and(ArticleType::Note->segment())->toBe('notes');

    SiteSetting::write('marquee_items', ['REACT', 'LARAVEL']);
    expect(SiteSetting::read('marquee_items'))->toBe(['REACT', 'LARAVEL'])
        ->and(SiteSetting::read('missing', 'fallback'))->toBe('fallback')
        ->and(SiteSetting::allValues())->toBe(['marquee_items' => ['REACT', 'LARAVEL']]);
});

it('keeps article slugs unique per type only', function () {
    $base = ['title' => 'Same', 'excerpt' => 'x', 'category' => 'c', 'status' => ArticleStatus::Draft, 'blocks' => []];

    Article::create([...$base, 'type' => ArticleType::Travel, 'slug' => 'same']);
    Article::create([...$base, 'type' => ArticleType::Note, 'slug' => 'same']);

    expect(fn () => Article::create([...$base, 'type' => ArticleType::Travel, 'slug' => 'same']))
        ->toThrow(QueryException::class);
});

it('builds media URLs from variants with a fallback to the largest one', function () {
    $media = Media::factory()->create([
        'path' => 'media/2026/11/abc.jpg',
        'variants' => [
            'thumb' => ['path' => 'media/2026/11/abc-thumb.webp', 'width' => 400, 'height' => 300],
            'md' => ['path' => 'media/2026/11/abc-md.webp', 'width' => 800, 'height' => 600],
        ],
    ]);

    expect($media->url())->toEndWith('/storage/media/2026/11/abc.jpg')
        ->and($media->url('thumb'))->toEndWith('abc-thumb.webp')
        ->and($media->url('lg'))->toEndWith('abc-md.webp');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter=SchemaTest`
Expected: FAIL with `Class "App\Models\Media" not found`.

- [ ] **Step 3: Write the migrations**

`database/migrations/2026_11_05_000001_create_media_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('media', function (Blueprint $table) {
            $table->id();
            $table->uuid('uuid')->unique();
            $table->string('disk', 50);
            $table->string('path', 500);
            $table->string('original_name');
            $table->string('mime_type', 150)->index();
            $table->string('extension', 20);
            $table->unsignedBigInteger('size_bytes');
            $table->unsignedInteger('width')->nullable();
            $table->unsignedInteger('height')->nullable();
            $table->string('alt')->nullable();
            $table->json('variants')->nullable();
            $table->timestamps();
            $table->softDeletes();

            $table->index('original_name');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('media');
    }
};
```

`database/migrations/2026_11_05_000002_create_portfolio_tables.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('projects', function (Blueprint $table) {
            $table->id();
            $table->string('slug')->unique();
            $table->string('title');
            $table->string('category');
            $table->text('description');
            $table->json('highlights')->nullable();
            $table->json('tech')->nullable();
            $table->string('href', 500)->nullable();
            $table->foreignId('cover_media_id')->nullable()->constrained('media')->nullOnDelete();
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->boolean('is_visible')->default(true);
            $table->timestamps();
        });

        Schema::create('creative_items', function (Blueprint $table) {
            $table->id();
            $table->string('key', 100)->unique();
            $table->string('title');
            $table->string('discipline');
            $table->string('shape', 10);
            $table->foreignId('media_id')->nullable()->constrained('media')->nullOnDelete();
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->timestamps();
        });

        Schema::create('skill_groups', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->timestamps();
        });

        Schema::create('skills', function (Blueprint $table) {
            $table->id();
            $table->foreignId('skill_group_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->timestamps();

            $table->unique(['skill_group_id', 'name']);
        });

        Schema::create('services', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('body');
            $table->string('tech')->nullable();
            $table->string('cta_label')->nullable();
            $table->string('cta_href', 500)->nullable();
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->timestamps();
        });

        Schema::create('timeline_entries', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('detail');
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('timeline_entries');
        Schema::dropIfExists('services');
        Schema::dropIfExists('skills');
        Schema::dropIfExists('skill_groups');
        Schema::dropIfExists('creative_items');
        Schema::dropIfExists('projects');
    }
};
```

`database/migrations/2026_11_05_000003_create_article_tables.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('travel_categories', function (Blueprint $table) {
            $table->id();
            $table->string('name')->unique();
            $table->string('slug')->unique();
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->timestamps();
        });

        Schema::create('articles', function (Blueprint $table) {
            $table->id();
            $table->string('type', 10);
            $table->string('slug');
            $table->string('title');
            $table->string('subtitle')->nullable();
            $table->text('excerpt');
            $table->string('category', 100);
            $table->string('topic', 100)->nullable();
            $table->string('destination')->nullable();
            $table->boolean('featured')->default(false);
            $table->foreignId('cover_media_id')->nullable()->constrained('media')->nullOnDelete();
            $table->string('status', 12)->default('draft');
            $table->timestamp('published_at')->nullable();
            $table->string('reading_time', 50)->nullable();
            $table->json('blocks')->nullable();
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->timestamps();

            $table->unique(['type', 'slug']);
            $table->index(['type', 'status']);
        });

        Schema::create('article_travel_category', function (Blueprint $table) {
            $table->foreignId('article_id')->constrained()->cascadeOnDelete();
            $table->foreignId('travel_category_id')->constrained()->cascadeOnDelete();

            $table->primary(['article_id', 'travel_category_id']);
        });

        Schema::create('travel_photos', function (Blueprint $table) {
            $table->id();
            $table->string('key', 100)->unique();
            $table->string('theme');
            $table->string('aspect', 10);
            $table->foreignId('media_id')->nullable()->constrained('media')->nullOnDelete();
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('travel_photos');
        Schema::dropIfExists('article_travel_category');
        Schema::dropIfExists('articles');
        Schema::dropIfExists('travel_categories');
    }
};
```

`database/migrations/2026_11_05_000004_create_site_tables.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('nav_links', function (Blueprint $table) {
            $table->id();
            $table->string('label');
            $table->string('href', 500);
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->boolean('is_visible')->default(true);
            $table->timestamps();
        });

        Schema::create('social_links', function (Blueprint $table) {
            $table->id();
            $table->string('label');
            $table->string('url', 500)->nullable();
            $table->unsignedInteger('sort_order')->default(0)->index();
            $table->timestamps();
        });

        Schema::create('site_settings', function (Blueprint $table) {
            $table->id();
            $table->string('key', 150)->unique();
            $table->json('value')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('site_settings');
        Schema::dropIfExists('social_links');
        Schema::dropIfExists('nav_links');
    }
};
```

- [ ] **Step 4: Write the enums and the sort-order concern**

`app/Enums/ArticleType.php`:

```php
<?php

namespace App\Enums;

enum ArticleType: string
{
    case Travel = 'travel';
    case Note = 'note';

    /** URL segment used by the admin and the frontend. */
    public function segment(): string
    {
        return match ($this) {
            self::Travel => 'travel',
            self::Note => 'notes',
        };
    }

    public function label(): string
    {
        return match ($this) {
            self::Travel => 'Travel stories',
            self::Note => 'Notes',
        };
    }
}
```

`app/Enums/ArticleStatus.php`:

```php
<?php

namespace App\Enums;

enum ArticleStatus: string
{
    case Draft = 'draft';
    case Published = 'published';
    case Hidden = 'hidden';
}
```

`app/Models/Concerns/HasSortOrder.php`:

```php
<?php

namespace App\Models\Concerns;

use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

trait HasSortOrder
{
    public static function bootHasSortOrder(): void
    {
        static::creating(function (Model $model) {
            if ($model->getAttribute('sort_order') === null) {
                $model->setAttribute('sort_order', (int) static::query()->max('sort_order') + 1);
            }
        });
    }

    #[Scope]
    protected function ordered(Builder $query): void
    {
        $query->orderBy($this->qualifyColumn('sort_order'))->orderBy($this->qualifyColumn('id'));
    }
}
```

- [ ] **Step 5: Write the models**

`app/Models/Media.php`:

```php
<?php

namespace App\Models;

use Database\Factories\MediaFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

#[Table('media')]
#[Fillable(['uuid', 'disk', 'path', 'original_name', 'mime_type', 'extension', 'size_bytes', 'width', 'height', 'alt', 'variants'])]
class Media extends Model
{
    /** @use HasFactory<MediaFactory> */
    use HasFactory, SoftDeletes;

    protected static function booted(): void
    {
        static::creating(function (Media $media) {
            $media->uuid ??= (string) Str::uuid();
        });

        static::forceDeleted(function (Media $media) {
            Storage::disk($media->disk)->delete([$media->path, ...array_column($media->variants ?? [], 'path')]);
        });
    }

    protected function casts(): array
    {
        return [
            'variants' => 'array',
            'size_bytes' => 'integer',
            'width' => 'integer',
            'height' => 'integer',
        ];
    }

    public function isImage(): bool
    {
        return str_starts_with($this->mime_type, 'image/');
    }

    /**
     * Public URL of the original, or of a WebP variant. A missing variant (small
     * originals skip the larger sizes) falls back to the largest one available.
     */
    public function url(?string $variant = null): string
    {
        $path = $this->path;
        $variants = $this->variants ?? [];

        if ($variant !== null && $variants !== []) {
            $path = $variants[$variant]['path'] ?? end($variants)['path'];
        }

        return Storage::disk($this->disk)->url($path);
    }
}
```

`app/Models/Project.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Fillable(['slug', 'title', 'category', 'description', 'highlights', 'tech', 'href', 'cover_media_id', 'sort_order', 'is_visible'])]
class Project extends Model
{
    use HasSortOrder;

    protected $attributes = ['is_visible' => true];

    protected function casts(): array
    {
        return ['highlights' => 'array', 'tech' => 'array', 'is_visible' => 'boolean', 'sort_order' => 'integer'];
    }

    /** @return BelongsTo<Media, $this> */
    public function cover(): BelongsTo
    {
        return $this->belongsTo(Media::class, 'cover_media_id');
    }
}
```

`app/Models/CreativeItem.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Fillable(['key', 'title', 'discipline', 'shape', 'media_id', 'sort_order'])]
class CreativeItem extends Model
{
    use HasSortOrder;

    public const SHAPES = ['tall' => 'Tall', 'wide' => 'Wide', 'square' => 'Square'];

    protected function casts(): array
    {
        return ['sort_order' => 'integer'];
    }

    /** @return BelongsTo<Media, $this> */
    public function media(): BelongsTo
    {
        return $this->belongsTo(Media::class);
    }
}
```

`app/Models/SkillGroup.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

#[Fillable(['title', 'sort_order'])]
class SkillGroup extends Model
{
    use HasSortOrder;

    /** @return HasMany<Skill, $this> */
    public function skills(): HasMany
    {
        return $this->hasMany(Skill::class)->orderBy('sort_order')->orderBy('id');
    }
}
```

`app/Models/Skill.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Fillable(['skill_group_id', 'name', 'sort_order'])]
class Skill extends Model
{
    use HasSortOrder;

    /** @return BelongsTo<SkillGroup, $this> */
    public function group(): BelongsTo
    {
        return $this->belongsTo(SkillGroup::class, 'skill_group_id');
    }
}
```

`app/Models/Service.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;

#[Fillable(['title', 'body', 'tech', 'cta_label', 'cta_href', 'sort_order'])]
class Service extends Model
{
    use HasSortOrder;
}
```

`app/Models/TimelineEntry.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;

#[Fillable(['title', 'detail', 'sort_order'])]
class TimelineEntry extends Model
{
    use HasSortOrder;
}
```

`app/Models/Article.php`:

```php
<?php

namespace App\Models;

use App\Enums\ArticleStatus;
use App\Enums\ArticleType;
use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

#[Fillable([
    'type', 'slug', 'title', 'subtitle', 'excerpt', 'category', 'topic', 'destination', 'featured',
    'cover_media_id', 'status', 'published_at', 'reading_time', 'blocks', 'sort_order',
])]
class Article extends Model
{
    use HasSortOrder;

    protected $attributes = ['status' => 'draft', 'featured' => false];

    protected function casts(): array
    {
        return [
            'type' => ArticleType::class,
            'status' => ArticleStatus::class,
            'featured' => 'boolean',
            'published_at' => 'datetime',
            'blocks' => 'array',
            'sort_order' => 'integer',
        ];
    }

    /** @return BelongsTo<Media, $this> */
    public function cover(): BelongsTo
    {
        return $this->belongsTo(Media::class, 'cover_media_id');
    }

    /** @return BelongsToMany<TravelCategory, $this> */
    public function travelCategories(): BelongsToMany
    {
        return $this->belongsToMany(TravelCategory::class)->orderBy('travel_categories.sort_order');
    }
}
```

`app/Models/TravelCategory.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

#[Fillable(['name', 'slug', 'sort_order'])]
class TravelCategory extends Model
{
    use HasSortOrder;

    /** @return BelongsToMany<Article, $this> */
    public function articles(): BelongsToMany
    {
        return $this->belongsToMany(Article::class);
    }
}
```

`app/Models/TravelPhoto.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Fillable(['key', 'theme', 'aspect', 'media_id', 'sort_order'])]
class TravelPhoto extends Model
{
    use HasSortOrder;

    public const ASPECTS = ['portrait' => 'Portrait', 'landscape' => 'Landscape'];

    /** @return BelongsTo<Media, $this> */
    public function media(): BelongsTo
    {
        return $this->belongsTo(Media::class);
    }
}
```

`app/Models/NavLink.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

#[Fillable(['label', 'href', 'sort_order', 'is_visible'])]
class NavLink extends Model
{
    use HasSortOrder;

    protected $attributes = ['is_visible' => true];

    protected function casts(): array
    {
        return ['is_visible' => 'boolean'];
    }

    #[Scope]
    protected function visible(Builder $query): void
    {
        $query->where('is_visible', true);
    }
}
```

`app/Models/SocialLink.php`:

```php
<?php

namespace App\Models;

use App\Models\Concerns\HasSortOrder;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;

#[Fillable(['label', 'url', 'sort_order'])]
class SocialLink extends Model
{
    use HasSortOrder;
}
```

`app/Models/SiteSetting.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;

#[Fillable(['key', 'value'])]
class SiteSetting extends Model
{
    protected function casts(): array
    {
        return ['value' => 'json'];
    }

    public static function read(string $key, mixed $default = null): mixed
    {
        return static::query()->where('key', $key)->first()?->value ?? $default;
    }

    public static function write(string $key, mixed $value): void
    {
        static::query()->updateOrCreate(['key' => $key], ['value' => $value]);
    }

    /** @return array<string, mixed> */
    public static function allValues(): array
    {
        return static::query()->get()->mapWithKeys(fn (self $setting) => [$setting->key => $setting->value])->all();
    }
}
```

`database/factories/MediaFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\Media;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

/**
 * @extends Factory<Media>
 */
class MediaFactory extends Factory
{
    public function definition(): array
    {
        $uuid = (string) Str::uuid();

        return [
            'uuid' => $uuid,
            'disk' => 'public',
            'path' => "media/2026/11/{$uuid}.jpg",
            'original_name' => fake()->word().'.jpg',
            'mime_type' => 'image/jpeg',
            'extension' => 'jpg',
            'size_bytes' => 120_000,
            'width' => 1600,
            'height' => 900,
            'alt' => fake()->sentence(3),
            'variants' => null,
        ];
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `php artisan test --filter=SchemaTest`
Expected: PASS (3 tests).

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: portfolio schema, models and enums"
```

---

### Task 2: Article block rules

**Files:**
- Create: `app/Support/ArticleBlocks.php`, `app/Support/Slug.php`
- Test: `tests/Feature/ArticleBlocksTest.php`

**Interfaces:**
- Consumes: `Media` (Task 1) for the `exists` rule.
- Produces:
  - `ArticleBlocks::TYPES` = `['heading','paragraph','image','quote','route','tips']`.
  - `ArticleBlocks::validate(mixed $blocks): array<string, list<string>>` — errors keyed `blocks.{i}.{field}`; empty array when valid.
  - `ArticleBlocks::normalize(array $blocks): array` — whitelisted keys only, heading `id` auto-slugged from `text`, image `media_id` int|null, empty captions removed.
  - `ArticleBlocks::mediaIds(?array $blocks): list<int>`.
  - `ArticleBlocks::readingTime(?array $blocks): string` (e.g. `"3 min read"`).
  - `Slug::unique(Builder $query, string $value, ?int $ignoreId = null): string`.

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/ArticleBlocksTest.php`:

```php
<?php

use App\Models\Media;
use App\Models\Project;
use App\Support\ArticleBlocks;
use App\Support\Slug;

it('accepts every valid block type', function () {
    $media = Media::factory()->create();

    $blocks = [
        ['type' => 'heading', 'text' => 'The route'],
        ['type' => 'paragraph', 'text' => 'We walked.'],
        ['type' => 'image', 'media_id' => $media->id, 'caption' => 'Trail'],
        ['type' => 'image', 'media_id' => null],
        ['type' => 'quote', 'text' => 'Go slow.'],
        ['type' => 'route', 'title' => 'Route outline', 'stages' => [['label' => 'A → B', 'detail' => 'Bus']]],
        ['type' => 'tips', 'title' => 'Tips', 'items' => ['Bring water']],
    ];

    expect(ArticleBlocks::validate($blocks))->toBe([]);
});

it('reports per-block errors with indexed keys', function () {
    $errors = ArticleBlocks::validate([
        ['type' => 'heading', 'text' => ''],
        ['type' => 'video', 'url' => 'x'],
        ['type' => 'route', 'title' => 'R', 'stages' => []],
        ['type' => 'tips', 'title' => 'T', 'items' => ['']],
        ['type' => 'image', 'media_id' => 999],
    ]);

    expect($errors)->toHaveKeys(['blocks.0.text', 'blocks.1.type', 'blocks.2.stages', 'blocks.3.items.0', 'blocks.4.media_id'])
        ->and(ArticleBlocks::validate('not a list'))->toHaveKey('blocks');
});

it('normalizes blocks to whitelisted keys', function () {
    $normalized = ArticleBlocks::normalize([
        ['type' => 'heading', 'text' => 'Trip Details & Tips', 'extra' => 'drop me'],
        ['type' => 'image', 'media_id' => '7', 'caption' => ''],
        ['type' => 'tips', 'title' => 'T', 'items' => ['a', '', 'b']],
    ]);

    expect($normalized)->toBe([
        ['type' => 'heading', 'id' => 'trip-details-tips', 'text' => 'Trip Details & Tips'],
        ['type' => 'image', 'media_id' => 7],
        ['type' => 'tips', 'title' => 'T', 'items' => ['a', 'b']],
    ]);
});

it('collects media ids and estimates reading time', function () {
    $blocks = [
        ['type' => 'image', 'media_id' => 4],
        ['type' => 'paragraph', 'text' => str_repeat('word ', 450)],
        ['type' => 'image', 'media_id' => null],
    ];

    expect(ArticleBlocks::mediaIds($blocks))->toBe([4])
        ->and(ArticleBlocks::readingTime($blocks))->toBe('3 min read')
        ->and(ArticleBlocks::readingTime([]))->toBe('1 min read');
});

it('generates unique slugs within a query scope', function () {
    Project::create(['slug' => 'ez-bus', 'title' => 'EZ-Bus', 'category' => 'c', 'description' => 'd']);

    expect(Slug::unique(Project::query(), 'EZ Bus'))->toBe('ez-bus-2')
        ->and(Slug::unique(Project::query(), 'EZ Bus', Project::first()->id))->toBe('ez-bus');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter=ArticleBlocksTest`
Expected: FAIL with `Class "App\Support\ArticleBlocks" not found`.

- [ ] **Step 3: Implement `ArticleBlocks` and `Slug`**

`app/Support/ArticleBlocks.php`:

```php
<?php

namespace App\Support;

use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Str;
use Illuminate\Validation\Rule;

/**
 * Rules for the typed article blocks the frontend renders
 * (frontend/types/article.ts: heading, paragraph, image, quote, route, tips).
 */
final class ArticleBlocks
{
    public const TYPES = ['heading', 'paragraph', 'image', 'quote', 'route', 'tips'];

    /**
     * @return array<string, list<string>> errors keyed "blocks.{index}.{field}"
     */
    public static function validate(mixed $blocks): array
    {
        if (! is_array($blocks) || ! array_is_list($blocks)) {
            return ['blocks' => ['The content blocks must be a list.']];
        }

        $errors = [];

        foreach ($blocks as $index => $block) {
            $type = is_array($block) ? ($block['type'] ?? null) : null;

            if (! in_array($type, self::TYPES, true)) {
                $errors["blocks.{$index}.type"] = ['Choose a valid block type.'];

                continue;
            }

            $validator = Validator::make($block, self::rules($type));

            foreach ($validator->errors()->messages() as $field => $messages) {
                $errors["blocks.{$index}.{$field}"] = $messages;
            }
        }

        return $errors;
    }

    /**
     * @param  list<array<string, mixed>>  $blocks
     * @return list<array<string, mixed>>
     */
    public static function normalize(array $blocks): array
    {
        return array_values(array_map(function (array $block): array {
            return match ($block['type']) {
                'heading' => [
                    'type' => 'heading',
                    'id' => Str::slug((string) (($block['id'] ?? '') ?: $block['text'])),
                    'text' => trim((string) $block['text']),
                ],
                'paragraph', 'quote' => ['type' => $block['type'], 'text' => trim((string) $block['text'])],
                // media_id is always present (null = placeholder until an image is chosen); empty captions are dropped.
                'image' => array_filter([
                    'type' => 'image',
                    'media_id' => isset($block['media_id']) && $block['media_id'] !== '' ? (int) $block['media_id'] : null,
                    'caption' => trim((string) ($block['caption'] ?? '')) ?: null,
                ], fn ($value, $key) => $key === 'media_id' || $value !== null, ARRAY_FILTER_USE_BOTH),
                'route' => [
                    'type' => 'route',
                    'title' => trim((string) $block['title']),
                    'stages' => array_values(array_map(fn (array $stage) => [
                        'label' => trim((string) $stage['label']),
                        'detail' => trim((string) $stage['detail']),
                    ], $block['stages'])),
                ],
                'tips' => [
                    'type' => 'tips',
                    'title' => trim((string) $block['title']),
                    'items' => array_values(array_filter(array_map(fn ($item) => trim((string) $item), $block['items']), fn ($item) => $item !== '')),
                ],
            };
        }, $blocks));
    }

    /**
     * @return list<int>
     */
    public static function mediaIds(?array $blocks): array
    {
        return array_values(array_unique(array_map('intval', array_filter(
            array_map(fn ($block) => is_array($block) && ($block['type'] ?? null) === 'image' ? ($block['media_id'] ?? null) : null, $blocks ?? []),
        ))));
    }

    public static function readingTime(?array $blocks): string
    {
        $words = 0;
        $blocks ??= [];

        array_walk_recursive($blocks, function ($value, $key) use (&$words) {
            if (is_string($value) && in_array($key, ['text', 'caption', 'title', 'label', 'detail'], true)) {
                $words += str_word_count($value);
            } elseif (is_string($value) && is_int($key)) {
                $words += str_word_count($value); // tips items
            }
        });

        return max(1, (int) ceil($words / 200)).' min read';
    }

    /**
     * @return array<string, mixed>
     */
    private static function rules(string $type): array
    {
        return match ($type) {
            'heading' => ['id' => ['nullable', 'string', 'max:100'], 'text' => ['required', 'string', 'max:255']],
            'paragraph' => ['text' => ['required', 'string', 'max:10000']],
            'quote' => ['text' => ['required', 'string', 'max:2000']],
            'image' => [
                'media_id' => ['nullable', 'integer', Rule::exists('media', 'id')->whereNull('deleted_at')],
                'caption' => ['nullable', 'string', 'max:500'],
            ],
            'route' => [
                'title' => ['required', 'string', 'max:255'],
                'stages' => ['required', 'array', 'min:1', 'max:50'],
                'stages.*.label' => ['required', 'string', 'max:255'],
                'stages.*.detail' => ['required', 'string', 'max:2000'],
            ],
            'tips' => [
                'title' => ['required', 'string', 'max:255'],
                'items' => ['required', 'array', 'min:1', 'max:50'],
                'items.*' => ['required', 'string', 'max:500'],
            ],
        };
    }
}
```

`app/Support/Slug.php`:

```php
<?php

namespace App\Support;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Support\Str;

final class Slug
{
    /**
     * A slug unique within the given query (e.g. articles of one type).
     */
    public static function unique(Builder $query, string $value, ?int $ignoreId = null): string
    {
        $base = Str::limit(Str::slug($value), 180, '') ?: Str::lower(Str::random(8));
        $slug = $base;
        $suffix = 2;

        while ((clone $query)->where('slug', $slug)->when($ignoreId, fn (Builder $q) => $q->whereKeyNot($ignoreId))->exists()) {
            $slug = "{$base}-{$suffix}";
            $suffix++;
        }

        return $slug;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `php artisan test --filter=ArticleBlocksTest`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: article block validation, normalization and reading time"
```

---

### Task 3: Media pipeline

**Files:**
- Create: `app/Services/Media/MediaStore.php`, `app/Services/Media/MediaUsage.php`, `app/Exceptions/MediaInUseException.php`, `app/Support/ImageRef.php`
- Modify: `app/Providers/AppServiceProvider.php`
- Test: `tests/Feature/MediaPipelineTest.php`

**Interfaces:**
- Consumes: `Media`, `ArticleBlocks::mediaIds()`, `SiteSetting`, config `portfolio.media.*`.
- Produces:
  - `MediaStore::store(UploadedFile $file, ?string $alt = null): Media`
  - `MediaStore::delete(Media $media): void` — throws `MediaInUseException` (renders `409 { message, usages }`).
  - `MediaUsage::find(Media $media): list<array{type: string, id: int, label: string}>`
  - `ImageRef::make(?Media $media, string $fallbackAlt, string $variant = 'lg'): array`; constant `ImageRef::PLACEHOLDER_LABEL = 'Upload an image in the admin'`.

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/MediaPipelineTest.php`:

```php
<?php

use App\Enums\ArticleStatus;
use App\Enums\ArticleType;
use App\Exceptions\MediaInUseException;
use App\Models\Article;
use App\Models\Media;
use App\Models\Project;
use App\Models\SiteSetting;
use App\Services\Media\MediaStore;
use App\Services\Media\MediaUsage;
use App\Support\ImageRef;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

beforeEach(function () {
    Storage::fake('public');
    config(['portfolio.media.disk' => 'public']);
});

it('stores an image with dimensions and WebP variants', function () {
    $media = app(MediaStore::class)->store(UploadedFile::fake()->image('Langtang.jpg', 1600, 900), 'Valley at dawn');

    expect($media->width)->toBe(1600)
        ->and($media->height)->toBe(900)
        ->and($media->alt)->toBe('Valley at dawn')
        ->and($media->original_name)->toBe('Langtang.jpg')
        ->and(array_keys($media->variants))->toBe(['thumb', 'md', 'lg'])
        ->and($media->variants['thumb']['width'])->toBe(400)
        ->and($media->variants['lg']['width'])->toBe(1600);

    Storage::disk('public')->assertExists([$media->path, $media->variants['md']['path']]);
});

it('never upscales small images', function () {
    $media = app(MediaStore::class)->store(UploadedFile::fake()->image('icon.png', 300, 300));

    expect(array_keys($media->variants))->toBe(['thumb'])
        ->and($media->variants['thumb']['width'])->toBe(300);
});

it('re-encodes JPEG originals so embedded metadata is dropped', function () {
    $upload = UploadedFile::fake()->image('photo.jpg', 800, 600);
    $original = file_get_contents($upload->getRealPath());

    $media = app(MediaStore::class)->store($upload);

    expect(Storage::disk('public')->get($media->path))->not->toBe($original);
});

it('stores PDFs without variants', function () {
    $media = app(MediaStore::class)->store(UploadedFile::fake()->create('cv.pdf', 120, 'application/pdf'));

    expect($media->variants)->toBeNull()->and($media->isImage())->toBeFalse();
});

it('builds frontend image refs with placeholders', function () {
    $media = Media::factory()->create(['alt' => null, 'variants' => ['md' => ['path' => 'media/x-md.webp', 'width' => 1024, 'height' => 576]]]);

    expect(ImageRef::make(null, 'EZ-Bus'))->toBe(['src' => null, 'alt' => 'EZ-Bus', 'label' => 'Upload an image in the admin'])
        ->and(ImageRef::make($media, 'EZ-Bus', 'md'))->toBe(['src' => $media->url('md'), 'alt' => 'EZ-Bus']);
});

it('finds every usage and refuses to delete media in use', function () {
    $cover = Media::factory()->create();
    $inline = Media::factory()->create();
    $setting = Media::factory()->create();
    $unused = Media::factory()->create();

    Project::create(['slug' => 'p', 'title' => 'Project', 'category' => 'c', 'description' => 'd', 'cover_media_id' => $cover->id]);
    Article::create([
        'type' => ArticleType::Travel, 'slug' => 'a', 'title' => 'Story', 'excerpt' => 'e', 'category' => 'c',
        'status' => ArticleStatus::Draft, 'blocks' => [['type' => 'image', 'media_id' => $inline->id]],
    ]);
    SiteSetting::write('site.cv_media_id', $setting->id);

    $usage = app(MediaUsage::class);
    expect($usage->find($cover))->toBe([['type' => 'project', 'id' => 1, 'label' => 'Project']])
        ->and($usage->find($inline)[0]['type'])->toBe('article')
        ->and($usage->find($setting)[0])->toMatchArray(['type' => 'setting', 'label' => 'site.cv_media_id'])
        ->and($usage->find($unused))->toBe([]);

    expect(fn () => app(MediaStore::class)->delete($cover))->toThrow(MediaInUseException::class);

    app(MediaStore::class)->delete($unused);
    expect($unused->fresh()->trashed())->toBeTrue();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter=MediaPipelineTest`
Expected: FAIL with `Class "App\Services\Media\MediaStore" not found`.

- [ ] **Step 3: Bind the image manager**

Replace the `register()` method in `app/Providers/AppServiceProvider.php` with:

```php
    public function register(): void
    {
        $this->app->singleton(
            \Intervention\Image\ImageManager::class,
            fn () => new \Intervention\Image\ImageManager(new \Intervention\Image\Drivers\Gd\Driver),
        );
        $this->app->scoped(\App\Support\MediaLookup::class);
    }
```

(`MediaLookup` is created in Task 4; the `scoped()` binding is harmless until then because Laravel resolves the class lazily.)

- [ ] **Step 4: Implement the media classes**

`app/Exceptions/MediaInUseException.php`:

```php
<?php

namespace App\Exceptions;

use Illuminate\Http\JsonResponse;
use RuntimeException;

class MediaInUseException extends RuntimeException
{
    /**
     * @param  list<array{type: string, id: int, label: string}>  $usages
     */
    public function __construct(public readonly array $usages)
    {
        parent::__construct('This file is still in use. Remove it from the listed content first.');
    }

    public function render(): JsonResponse
    {
        return response()->json(['message' => $this->getMessage(), 'usages' => $this->usages], 409);
    }
}
```

`app/Services/Media/MediaUsage.php`:

```php
<?php

namespace App\Services\Media;

use App\Models\Article;
use App\Models\CreativeItem;
use App\Models\Media;
use App\Models\Project;
use App\Models\SiteSetting;
use App\Models\TravelPhoto;
use App\Support\ArticleBlocks;

/**
 * Every place a media item is referenced, so shared files are never deleted while in use.
 */
class MediaUsage
{
    /**
     * @return list<array{type: string, id: int, label: string}>
     */
    public function find(Media $media): array
    {
        $id = $media->id;
        $usages = [];
        $add = function (string $type, int $ownerId, string $label) use (&$usages) {
            $usages[] = ['type' => $type, 'id' => $ownerId, 'label' => $label];
        };

        Project::query()->where('cover_media_id', $id)->get()->each(fn (Project $p) => $add('project', $p->id, $p->title));
        CreativeItem::query()->where('media_id', $id)->get()->each(fn (CreativeItem $c) => $add('creative_item', $c->id, $c->title));
        TravelPhoto::query()->where('media_id', $id)->get()->each(fn (TravelPhoto $t) => $add('travel_photo', $t->id, $t->theme));
        Article::query()->where('cover_media_id', $id)->get()->each(fn (Article $a) => $add('article', $a->id, $a->title));

        // Image blocks live inside JSON; scan in PHP so it works the same on MySQL and SQLite.
        Article::query()->whereNotNull('blocks')->get(['id', 'title', 'blocks'])
            ->filter(fn (Article $a) => in_array($id, ArticleBlocks::mediaIds($a->blocks), true))
            ->each(fn (Article $a) => $add('article', $a->id, $a->title));

        SiteSetting::query()->where('key', 'like', '%media_id')->get()
            ->filter(fn (SiteSetting $s) => is_numeric($s->value) && (int) $s->value === $id)
            ->each(fn (SiteSetting $s) => $add('setting', $s->id, $s->key));

        return $usages;
    }
}
```

`app/Services/Media/MediaStore.php`:

```php
<?php

namespace App\Services\Media;

use App\Exceptions\MediaInUseException;
use App\Models\Media;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;
use Intervention\Image\ImageManager;

class MediaStore
{
    private const RASTER = ['image/jpeg', 'image/png', 'image/webp', 'image/gif', 'image/avif'];

    public function __construct(
        private readonly ImageManager $images,
        private readonly MediaUsage $usage,
    ) {}

    public function store(UploadedFile $file, ?string $alt = null): Media
    {
        $disk = (string) config('portfolio.media.disk');
        $uuid = (string) Str::uuid();
        $extension = strtolower($file->getClientOriginalExtension() ?: ($file->guessExtension() ?? 'bin'));
        $mime = $file->getMimeType() ?: 'application/octet-stream';
        $path = 'media/'.now()->format('Y/m')."/{$uuid}.{$extension}";
        $binary = (string) file_get_contents($file->getRealPath());
        $width = $height = null;

        if (in_array($mime, self::RASTER, true)) {
            $image = $this->images->read($binary); // auto-orients from EXIF
            $width = $image->width();
            $height = $image->height();

            // Re-encode JPEG/WebP so EXIF (including GPS) is not published.
            if ($mime === 'image/jpeg') {
                $binary = (string) $image->toJpeg(quality: 90);
            } elseif ($mime === 'image/webp') {
                $binary = (string) $image->toWebp(quality: 90);
            }
        }

        Storage::disk($disk)->put($path, $binary, ['visibility' => 'public']);

        $media = Media::create([
            'uuid' => $uuid,
            'disk' => $disk,
            'path' => $path,
            'original_name' => Str::limit($file->getClientOriginalName(), 250, ''),
            'mime_type' => $mime,
            'extension' => $extension,
            'size_bytes' => strlen($binary),
            'width' => $width,
            'height' => $height,
            'alt' => $alt,
        ]);

        if (in_array($mime, self::RASTER, true)) {
            $media->update(['variants' => $this->variants($media, $binary)]);
        }

        return $media;
    }

    public function delete(Media $media): void
    {
        $usages = $this->usage->find($media);

        if ($usages !== []) {
            throw new MediaInUseException($usages);
        }

        $media->delete();
    }

    /**
     * @return array<string, array{path: string, width: int, height: int}>
     */
    private function variants(Media $media, string $binary): array
    {
        $source = $this->images->read($binary);
        $quality = (int) config('portfolio.media.webp_quality');
        $base = preg_replace('/\.[^.\/]+$/', '', $media->path);
        $variants = [];
        $previousWidth = null;

        foreach (config('portfolio.media.variants') as $name => $width) {
            if ($previousWidth !== null && $source->width() <= $previousWidth) {
                break;
            }

            $image = clone $source;
            $image->scaleDown(width: $width);
            $path = "{$base}-{$name}.webp";

            Storage::disk($media->disk)->put($path, (string) $image->toWebp(quality: $quality), ['visibility' => 'public']);
            $variants[$name] = ['path' => $path, 'width' => $image->width(), 'height' => $image->height()];
            $previousWidth = $width;
        }

        return $variants;
    }
}
```

`app/Support/ImageRef.php`:

```php
<?php

namespace App\Support;

use App\Models\Media;

/**
 * The frontend's ImageRef shape ({src, alt, label?}, frontend/types/common.ts).
 */
final class ImageRef
{
    public const PLACEHOLDER_LABEL = 'Upload an image in the admin';

    /**
     * @return array{src: string|null, alt: string, label?: string}
     */
    public static function make(?Media $media, string $fallbackAlt, string $variant = 'lg'): array
    {
        if ($media === null || $media->trashed()) {
            return ['src' => null, 'alt' => $fallbackAlt, 'label' => self::PLACEHOLDER_LABEL];
        }

        return ['src' => $media->url($variant), 'alt' => $media->alt ?: $fallbackAlt];
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `php artisan test --filter=MediaPipelineTest`
Expected: PASS (6 tests).

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: media storage with WebP variants, usage tracking and image refs"
```

---

### Task 4: Public API

**Files:**
- Create: `routes/api.php`, `app/Support/MediaLookup.php`, `app/Http/Resources/Api/ProjectResource.php`, `app/Http/Resources/Api/ArticleResource.php`, `app/Http/Controllers/Api/V1/SiteController.php`, `app/Http/Controllers/Api/V1/ProjectController.php`, `app/Http/Controllers/Api/V1/ArticleController.php`
- Modify: `bootstrap/app.php`
- Test: `tests/Feature/PublicApiTest.php`

**Interfaces:**
- Consumes: all models, `ImageRef::make`, `ArticleBlocks::mediaIds/readingTime`.
- Produces: `GET /api/v1/site|projects|travel|travel/{slug}|notes|notes/{slug}`; `MediaLookup::preload(iterable $ids): void`, `MediaLookup::get(?int $id): ?Media`. Site setting keys read: `site.name`, `site.short_name`, `site.title`, `site.description`, `site.url`, `site.email`, `site.location`, `site.cv_media_id`, `site.portrait_media_id`, `site.portrait2_media_id`, `education`, `marquee_items`, `collab_areas`.

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/PublicApiTest.php`:

```php
<?php

use App\Enums\ArticleStatus;
use App\Enums\ArticleType;
use App\Models\Article;
use App\Models\CreativeItem;
use App\Models\Media;
use App\Models\NavLink;
use App\Models\Project;
use App\Models\Service;
use App\Models\SiteSetting;
use App\Models\SkillGroup;
use App\Models\SocialLink;
use App\Models\TimelineEntry;
use App\Models\TravelCategory;
use App\Models\TravelPhoto;

function article(array $overrides = []): Article
{
    return Article::create([
        'type' => ArticleType::Travel, 'slug' => 'langtang', 'title' => 'Langtang Valley', 'excerpt' => 'A trek',
        'category' => 'Trekking', 'destination' => 'Langtang Valley, Nepal', 'status' => ArticleStatus::Draft,
        'blocks' => [['type' => 'heading', 'id' => 'overview', 'text' => 'Overview'], ['type' => 'paragraph', 'text' => 'We walked.']],
        ...$overrides,
    ]);
}

it('returns site-wide data in the frontend shapes', function () {
    SiteSetting::write('site.name', 'Sarthak Neupane');
    SiteSetting::write('site.short_name', 'SN.');
    SiteSetting::write('marquee_items', ['REACT', 'LARAVEL']);
    SiteSetting::write('education', 'IT STUDENT');
    NavLink::create(['label' => 'Home', 'href' => '/']);
    NavLink::create(['label' => 'Hidden', 'href' => '/x', 'is_visible' => false]);
    SocialLink::create(['label' => 'GitHub', 'url' => null]);
    SkillGroup::create(['title' => 'Development'])->skills()->create(['name' => 'PHP']);
    Service::create(['title' => 'Development', 'body' => 'Apps', 'tech' => 'React', 'cta_label' => 'See', 'cta_href' => '/work']);
    TimelineEntry::create(['title' => 'EZ-Bus', 'detail' => 'Laravel']);
    CreativeItem::create(['key' => 'graphic', 'title' => 'Graphic Design', 'discipline' => 'Print', 'shape' => 'tall']);
    TravelCategory::create(['name' => 'Trekking', 'slug' => 'trekking']);
    TravelPhoto::create(['key' => 'road', 'theme' => 'Roads', 'aspect' => 'landscape']);

    $this->getJson('/api/v1/site')
        ->assertOk()
        ->assertJsonPath('data.site.name', 'Sarthak Neupane')
        ->assertJsonPath('data.site.shortName', 'SN.')
        ->assertJsonPath('data.site.cvPath', null)
        ->assertJsonPath('data.site.portrait', ['src' => null, 'alt' => 'Portrait of Sarthak Neupane', 'label' => 'Upload an image in the admin'])
        ->assertJsonPath('data.site.socials', [['label' => 'GitHub', 'url' => null]])
        ->assertJsonPath('data.navLinks', [['label' => 'Home', 'href' => '/']])
        ->assertJsonPath('data.marqueeItems', ['REACT', 'LARAVEL'])
        ->assertJsonPath('data.education', 'IT STUDENT')
        ->assertJsonPath('data.timeline', [['title' => 'EZ-Bus', 'detail' => 'Laravel']])
        ->assertJsonPath('data.skillGroups', [['title' => 'Development', 'items' => ['PHP']]])
        ->assertJsonPath('data.services.0', ['index' => '01', 'title' => 'Development', 'body' => 'Apps', 'tech' => 'React', 'cta' => 'See', 'href' => '/work'])
        ->assertJsonPath('data.creativeItems.0.id', 'graphic')
        ->assertJsonPath('data.creativeItems.0.shape', 'tall')
        ->assertJsonPath('data.travelFilters', ['All Stories', 'Trekking'])
        ->assertJsonPath('data.travelPhotos.0', ['id' => 'road', 'theme' => 'Roads', 'aspect' => 'landscape', 'image' => ['src' => null, 'alt' => 'Roads', 'label' => 'Upload an image in the admin']]);
});

it('lists visible projects with positional index and omits empty href', function () {
    $cover = Media::factory()->create(['alt' => 'Seat map', 'variants' => ['lg' => ['path' => 'media/p-lg.webp', 'width' => 1920, 'height' => 1080]]]);
    Project::create(['slug' => 'sathipro', 'title' => 'SathiPro', 'category' => 'Platform', 'description' => 'd', 'highlights' => ['A'], 'tech' => ['NestJS'], 'href' => 'https://sathipro.example']);
    Project::create(['slug' => 'ez-bus', 'title' => 'EZ-Bus', 'category' => 'Transport', 'description' => 'd', 'cover_media_id' => $cover->id]);
    Project::create(['slug' => 'hidden', 'title' => 'Hidden', 'category' => 'x', 'description' => 'd', 'is_visible' => false]);

    $response = $this->getJson('/api/v1/projects')->assertOk()->assertJsonCount(2, 'data');

    $response->assertJsonPath('data.0', [
        'slug' => 'sathipro', 'index' => '01', 'title' => 'SathiPro', 'category' => 'Platform', 'description' => 'd',
        'highlights' => ['A'], 'tech' => ['NestJS'],
        'image' => ['src' => null, 'alt' => 'SathiPro', 'label' => 'Upload an image in the admin'],
        'href' => 'https://sathipro.example',
    ]);
    $response->assertJsonPath('data.1.index', '02')
        ->assertJsonPath('data.1.image', ['src' => $cover->url('lg'), 'alt' => 'Seat map'])
        ->assertJsonPath('data.1.highlights', [])
        ->assertJsonMissingPath('data.1.href');
});

it('shows drafts with the draft flag and hides hidden articles', function () {
    $category = TravelCategory::create(['name' => 'Trekking', 'slug' => 'trekking']);
    article()->travelCategories()->attach($category);
    article(['slug' => 'secret', 'status' => ArticleStatus::Hidden]);

    $this->getJson('/api/v1/travel')
        ->assertOk()
        ->assertJsonCount(1, 'data')
        ->assertJsonPath('data.0.slug', 'langtang')
        ->assertJsonPath('data.0.draft', true)
        ->assertJsonPath('data.0.readingTime', 'Draft')
        ->assertJsonPath('data.0.categories', ['Trekking'])
        ->assertJsonPath('data.0.destination', 'Langtang Valley, Nepal')
        ->assertJsonMissingPath('data.0.date')
        ->assertJsonMissingPath('data.0.featured')
        ->assertJsonMissingPath('data.0.subtitle');

    $this->getJson('/api/v1/travel/secret')->assertNotFound()->assertExactJson(['message' => 'Not found.']);
    $this->getJson('/api/v1/travel/nope')->assertNotFound()->assertExactJson(['message' => 'Not found.']);
});

it('renders a published story with date, reading time and image blocks', function () {
    $photo = Media::factory()->create(['alt' => 'Trail']);
    article([
        'status' => ArticleStatus::Published, 'published_at' => '2026-10-01 09:00:00', 'featured' => true, 'subtitle' => 'Into the mountains',
        'blocks' => [['type' => 'image', 'media_id' => $photo->id, 'caption' => 'Riverside'], ['type' => 'image', 'media_id' => null]],
    ]);

    $this->getJson('/api/v1/travel/langtang')
        ->assertOk()
        ->assertJsonPath('data.draft', false)
        ->assertJsonPath('data.date', '2026-10-01')
        ->assertJsonPath('data.readingTime', '1 min read')
        ->assertJsonPath('data.featured', true)
        ->assertJsonPath('data.subtitle', 'Into the mountains')
        ->assertJsonPath('data.blocks.0', ['type' => 'image', 'image' => ['src' => $photo->url('lg'), 'alt' => 'Trail'], 'caption' => 'Riverside'])
        ->assertJsonPath('data.blocks.1', ['type' => 'image', 'image' => ['src' => null, 'alt' => 'Langtang Valley', 'label' => 'Upload an image in the admin']]);
});

it('serves notes with their topic', function () {
    article(['type' => ArticleType::Note, 'slug' => 'booking-systems', 'title' => 'Booking systems', 'topic' => 'Engineering', 'destination' => null]);

    $this->getJson('/api/v1/notes')->assertOk()
        ->assertJsonPath('data.0.topic', 'Engineering')
        ->assertJsonMissingPath('data.0.destination')
        ->assertJsonMissingPath('data.0.categories');
    $this->getJson('/api/v1/notes/booking-systems')->assertOk()->assertJsonPath('data.slug', 'booking-systems');
    $this->getJson('/api/v1/travel/booking-systems')->assertNotFound();
});

it('sends cache and CORS headers', function () {
    $this->withHeaders(['Origin' => 'http://localhost:3000'])
        ->getJson('/api/v1/projects')
        ->assertOk()
        ->assertHeader('ETag')
        ->assertHeader('Access-Control-Allow-Origin', 'http://localhost:3000')
        ->assertHeaderContains('Cache-Control', 'max-age=60')
        ->assertHeaderContains('Cache-Control', 'public');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter=PublicApiTest`
Expected: FAIL with 404s (no `/api` routes registered).

- [ ] **Step 3: Register API routing and the 404 format**

Replace `bootstrap/app.php` with:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Http\Request;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        //
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        $exceptions->shouldRenderJsonWhen(
            fn (Request $request) => $request->is('api/*') || $request->expectsJson(),
        );

        // Never leak model names: every API 404 has the same body.
        $exceptions->render(function (NotFoundHttpException $e, Request $request) {
            if ($request->is('api/*')) {
                return response()->json(['message' => 'Not found.'], 404);
            }
        });
    })->create();
```

Create `routes/api.php`:

```php
<?php

use App\Http\Controllers\Api\V1\ArticleController;
use App\Http\Controllers\Api\V1\ProjectController;
use App\Http\Controllers\Api\V1\SiteController;
use Illuminate\Support\Facades\Route;

Route::prefix('v1')->middleware('cache.headers:public;max_age=60;etag')->group(function () {
    Route::get('site', SiteController::class);
    Route::get('projects', ProjectController::class);

    Route::get('travel', [ArticleController::class, 'index'])->defaults('type', 'travel');
    Route::get('travel/{slug}', [ArticleController::class, 'show'])->defaults('type', 'travel');
    Route::get('notes', [ArticleController::class, 'index'])->defaults('type', 'note');
    Route::get('notes/{slug}', [ArticleController::class, 'show'])->defaults('type', 'note');
});
```

- [ ] **Step 4: Implement the lookup, resources and controllers**

`app/Support/MediaLookup.php`:

```php
<?php

namespace App\Support;

use App\Models\Media;

/**
 * Request-scoped media cache so lists resolve covers and image blocks in one query.
 */
final class MediaLookup
{
    /** @var array<int, Media|null> */
    private array $items = [];

    /**
     * @param  iterable<int|string|null>  $ids
     */
    public function preload(iterable $ids): void
    {
        $missing = collect($ids)->filter()->map(fn ($id) => (int) $id)->unique()
            ->reject(fn (int $id) => array_key_exists($id, $this->items))->values();

        if ($missing->isEmpty()) {
            return;
        }

        $found = Media::query()->whereIn('id', $missing)->get()->keyBy('id');

        foreach ($missing as $id) {
            $this->items[$id] = $found->get($id);
        }
    }

    public function get(int|string|null $id): ?Media
    {
        if ($id === null || $id === '') {
            return null;
        }

        $this->preload([$id]);

        return $this->items[(int) $id];
    }
}
```

`app/Http/Resources/Api/ProjectResource.php`:

```php
<?php

namespace App\Http\Resources\Api;

use App\Models\Project;
use App\Support\ImageRef;
use App\Support\MediaLookup;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

/**
 * frontend/types/project.ts → Project. `position` is set by the controller.
 *
 * @mixin Project
 */
class ProjectResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'slug' => $this->slug,
            'index' => str_pad((string) $this->resource->getAttribute('position'), 2, '0', STR_PAD_LEFT),
            'title' => $this->title,
            'category' => $this->category,
            'description' => $this->description,
            'highlights' => $this->highlights ?? [],
            'tech' => $this->tech ?? [],
            'image' => ImageRef::make(app(MediaLookup::class)->get($this->cover_media_id), $this->title),
            ...($this->href ? ['href' => $this->href] : []),
        ];
    }
}
```

`app/Http/Resources/Api/ArticleResource.php`:

```php
<?php

namespace App\Http\Resources\Api;

use App\Enums\ArticleStatus;
use App\Enums\ArticleType;
use App\Models\Article;
use App\Support\ArticleBlocks;
use App\Support\ImageRef;
use App\Support\MediaLookup;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

/**
 * frontend/types/travel.ts → TravelStory, frontend/types/note.ts → Note.
 *
 * @mixin Article
 */
class ArticleResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        $lookup = app(MediaLookup::class);
        $draft = $this->status === ArticleStatus::Draft;

        $data = [
            'slug' => $this->slug,
            'title' => $this->title,
            ...($this->subtitle ? ['subtitle' => $this->subtitle] : []),
            'excerpt' => $this->excerpt,
            'category' => $this->category,
            ...(! $draft && $this->published_at ? ['date' => $this->published_at->toDateString()] : []),
            'readingTime' => $draft ? 'Draft' : ($this->reading_time ?: ArticleBlocks::readingTime($this->blocks)),
            'cover' => ImageRef::make($lookup->get($this->cover_media_id), $this->title),
            'draft' => $draft,
            'blocks' => $this->presentBlocks($lookup),
        ];

        if ($this->type === ArticleType::Travel) {
            $data['destination'] = (string) $this->destination;
            $data['categories'] = $this->travelCategories->pluck('name')->all();
            if ($this->featured) {
                $data['featured'] = true;
            }
        } else {
            $data['topic'] = (string) $this->topic;
        }

        return $data;
    }

    /**
     * @return list<array<string, mixed>>
     */
    private function presentBlocks(MediaLookup $lookup): array
    {
        return array_map(function (array $block) use ($lookup): array {
            if ($block['type'] !== 'image') {
                return $block;
            }

            return [
                'type' => 'image',
                'image' => ImageRef::make($lookup->get($block['media_id'] ?? null), $block['caption'] ?? $this->title),
                ...(! empty($block['caption']) ? ['caption' => $block['caption']] : []),
            ];
        }, $this->blocks ?? []);
    }
}
```

Note: the test expects a missing-image block's `alt` to be the article title when there is no caption, and the caption-bearing block's alt to come from the media item. `ImageRef::make` already prefers `$media->alt`, so `$block['caption'] ?? $this->title` is only the fallback.

`app/Http/Controllers/Api/V1/ProjectController.php`:

```php
<?php

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Http\Resources\Api\ProjectResource;
use App\Models\Project;
use App\Support\MediaLookup;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class ProjectController extends Controller
{
    public function __invoke(MediaLookup $lookup): AnonymousResourceCollection
    {
        $projects = Project::query()->where('is_visible', true)->ordered()->get()
            ->values()
            ->each(fn (Project $project, int $i) => $project->setAttribute('position', $i + 1));

        $lookup->preload($projects->pluck('cover_media_id'));

        return ProjectResource::collection($projects);
    }
}
```

`app/Http/Controllers/Api/V1/ArticleController.php`:

```php
<?php

namespace App\Http\Controllers\Api\V1;

use App\Enums\ArticleStatus;
use App\Http\Controllers\Controller;
use App\Http\Resources\Api\ArticleResource;
use App\Models\Article;
use App\Support\ArticleBlocks;
use App\Support\MediaLookup;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class ArticleController extends Controller
{
    public function index(Request $request, MediaLookup $lookup): AnonymousResourceCollection
    {
        $articles = $this->visible($request)->ordered()->get();
        $this->preload($articles, $lookup);

        return ArticleResource::collection($articles);
    }

    public function show(Request $request, MediaLookup $lookup): ArticleResource
    {
        $article = $this->visible($request)->where('slug', $request->route('slug'))->firstOrFail();
        $this->preload(new Collection([$article]), $lookup);

        return new ArticleResource($article);
    }

    /**
     * @return Builder<Article>
     */
    private function visible(Request $request): Builder
    {
        return Article::query()
            ->where('type', $request->route('type'))
            ->where('status', '!=', ArticleStatus::Hidden->value)
            ->with('travelCategories');
    }

    /**
     * @param  Collection<int, Article>  $articles
     */
    private function preload(Collection $articles, MediaLookup $lookup): void
    {
        $lookup->preload($articles->pluck('cover_media_id')->merge(
            $articles->flatMap(fn (Article $article) => ArticleBlocks::mediaIds($article->blocks)),
        ));
    }
}
```

`app/Http/Controllers/Api/V1/SiteController.php`:

```php
<?php

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Models\CreativeItem;
use App\Models\NavLink;
use App\Models\Service;
use App\Models\SiteSetting;
use App\Models\SkillGroup;
use App\Models\SocialLink;
use App\Models\TimelineEntry;
use App\Models\TravelCategory;
use App\Models\TravelPhoto;
use App\Support\ImageRef;
use App\Support\MediaLookup;
use Illuminate\Http\JsonResponse;

/**
 * Everything the frontend imports from data/site.ts, skills.ts, experience.ts and the
 * non-article parts of projects.ts / travel.ts, in one request.
 */
class SiteController extends Controller
{
    public function __invoke(MediaLookup $lookup): JsonResponse
    {
        $settings = SiteSetting::allValues();
        $name = (string) ($settings['site.name'] ?? '');
        $creative = CreativeItem::query()->ordered()->get();
        $photos = TravelPhoto::query()->ordered()->get();

        $lookup->preload([
            $settings['site.cv_media_id'] ?? null,
            $settings['site.portrait_media_id'] ?? null,
            $settings['site.portrait2_media_id'] ?? null,
            ...$creative->pluck('media_id'),
            ...$photos->pluck('media_id'),
        ]);

        $cv = $lookup->get($settings['site.cv_media_id'] ?? null);
        $pad = fn (int $i) => str_pad((string) ($i + 1), 2, '0', STR_PAD_LEFT);

        return response()->json(['data' => [
            'site' => [
                'name' => $name,
                'shortName' => (string) ($settings['site.short_name'] ?? ''),
                'title' => (string) ($settings['site.title'] ?? ''),
                'description' => (string) ($settings['site.description'] ?? ''),
                'url' => (string) ($settings['site.url'] ?? ''),
                'email' => (string) ($settings['site.email'] ?? ''),
                'location' => (string) ($settings['site.location'] ?? ''),
                'cvPath' => $cv?->url(),
                'portrait' => ImageRef::make($lookup->get($settings['site.portrait_media_id'] ?? null), "Portrait of {$name}"),
                'portrait2' => ImageRef::make($lookup->get($settings['site.portrait2_media_id'] ?? null), "Portrait of {$name}"),
                'socials' => SocialLink::query()->ordered()->get()->map(fn (SocialLink $s) => ['label' => $s->label, 'url' => $s->url])->all(),
            ],
            'navLinks' => NavLink::query()->visible()->ordered()->get()->map(fn (NavLink $l) => ['label' => $l->label, 'href' => $l->href])->all(),
            'marqueeItems' => $settings['marquee_items'] ?? [],
            'collabAreas' => $settings['collab_areas'] ?? [],
            'education' => (string) ($settings['education'] ?? ''),
            'timeline' => TimelineEntry::query()->ordered()->get()->map(fn (TimelineEntry $t) => ['title' => $t->title, 'detail' => $t->detail])->all(),
            'skillGroups' => SkillGroup::query()->with('skills')->ordered()->get()
                ->map(fn (SkillGroup $g) => ['title' => $g->title, 'items' => $g->skills->pluck('name')->all()])->all(),
            'services' => Service::query()->ordered()->get()->values()->map(fn (Service $s, int $i) => [
                'index' => $pad($i),
                'title' => $s->title,
                'body' => $s->body,
                'tech' => (string) $s->tech,
                'cta' => (string) $s->cta_label,
                'href' => (string) $s->cta_href,
            ])->all(),
            'creativeItems' => $creative->map(fn (CreativeItem $c) => [
                'id' => $c->key,
                'title' => $c->title,
                'discipline' => $c->discipline,
                'shape' => $c->shape,
                'image' => ImageRef::make($lookup->get($c->media_id), $c->title, 'md'),
            ])->all(),
            'travelFilters' => ['All Stories', ...TravelCategory::query()->ordered()->pluck('name')->all()],
            'travelPhotos' => $photos->map(fn (TravelPhoto $p) => [
                'id' => $p->key,
                'theme' => $p->theme,
                'aspect' => $p->aspect,
                'image' => ImageRef::make($lookup->get($p->media_id), $p->theme, 'md'),
            ])->all(),
        ]]);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `php artisan test --filter=PublicApiTest`
Expected: PASS (6 tests).

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: read-only /api/v1 matching the frontend data shapes"
```

---

### Task 5: Seed the current site content

**Files:**
- Create: `scripts/export-frontend-data.mjs`, `database/seeders/data/frontend.json` (generated), `database/seeders/PortfolioSeeder.php`
- Modify: `database/seeders/DatabaseSeeder.php`
- Test: `tests/Feature/SeederTest.php`

**Interfaces:**
- Consumes: all models, `SiteSetting::write`, API from Task 4 (for the round-trip assertion).
- Produces: `PortfolioSeeder` (idempotent, `firstOrCreate` on natural keys); `frontend.json` with keys `site, navLinks, marqueeItems, collabAreas, projects, creativeItems, skillGroups, services, education, timeline, travelStories, travelFilters, travelPhotos, notes`.

- [ ] **Step 1: Write the export script**

Create `scripts/export-frontend-data.mjs`:

```js
// One-time export of frontend/data/*.ts to database/seeders/data/frontend.json.
// Node 24 strips TypeScript types natively; the data files only use `import type`.
import { copyFileSync, mkdirSync, mkdtempSync, rmSync, writeFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { dirname, join } from 'node:path';
import { fileURLToPath, pathToFileURL } from 'node:url';

const here = dirname(fileURLToPath(import.meta.url));
const dataDir = join(here, '..', '..', 'frontend', 'data');
const outFile = join(here, '..', 'database', 'seeders', 'data', 'frontend.json');
const work = mkdtempSync(join(tmpdir(), 'portfolio-export-'));

// Copy to .mts so Node always treats them as ES modules.
const load = (name) => {
  const target = join(work, `${name}.mts`);
  copyFileSync(join(dataDir, `${name}.ts`), target);
  return import(pathToFileURL(target).href);
};

try {
  const [site, projects, skills, experience, travel, notes] = await Promise.all(
    ['site', 'projects', 'skills', 'experience', 'travel', 'notes'].map(load),
  );

  const out = {
    site: site.site,
    navLinks: site.navLinks,
    marqueeItems: site.marqueeItems,
    collabAreas: site.collabAreas,
    projects: projects.projects,
    creativeItems: projects.creativeItems,
    skillGroups: skills.skillGroups,
    services: skills.services,
    education: experience.education,
    timeline: experience.timeline,
    travelStories: travel.travelStories,
    travelFilters: travel.travelFilters,
    travelPhotos: travel.travelPhotos,
    notes: notes.notes,
  };

  mkdirSync(dirname(outFile), { recursive: true });
  writeFileSync(outFile, `${JSON.stringify(out, null, 2)}\n`);
  console.log(`wrote ${outFile}: ${out.projects.length} projects, ${out.travelStories.length} travel stories, ${out.notes.length} notes`);
} finally {
  rmSync(work, { recursive: true, force: true });
}
```

- [ ] **Step 2: Generate the JSON**

Run: `node scripts/export-frontend-data.mjs`
Expected: `wrote ...frontend.json: 4 projects, <N> travel stories, 6 notes`. Open the file and confirm `site.name` is `"Sarthak Neupane"` and `travelFilters[0]` is `"All Stories"`.

- [ ] **Step 3: Write the failing test**

Create `tests/Feature/SeederTest.php`:

```php
<?php

use App\Enums\ArticleStatus;
use App\Models\Article;
use App\Models\CreativeItem;
use App\Models\NavLink;
use App\Models\Project;
use App\Models\Service;
use App\Models\SiteSetting;
use App\Models\Skill;
use App\Models\SkillGroup;
use App\Models\SocialLink;
use App\Models\TimelineEntry;
use App\Models\TravelCategory;
use App\Models\TravelPhoto;
use Database\Seeders\DatabaseSeeder;

function frontendData(): array
{
    return json_decode(file_get_contents(database_path('seeders/data/frontend.json')), true, flags: JSON_THROW_ON_ERROR);
}

it('reproduces the frontend content and is idempotent', function () {
    $data = frontendData();

    $this->seed(DatabaseSeeder::class);
    $this->seed(DatabaseSeeder::class);

    expect(Project::count())->toBe(count($data['projects']))
        ->and(CreativeItem::count())->toBe(count($data['creativeItems']))
        ->and(SkillGroup::count())->toBe(count($data['skillGroups']))
        ->and(Skill::count())->toBe(array_sum(array_map(fn ($g) => count($g['items']), $data['skillGroups'])))
        ->and(Service::count())->toBe(count($data['services']))
        ->and(TimelineEntry::count())->toBe(count($data['timeline']))
        ->and(TravelCategory::count())->toBe(count($data['travelFilters']) - 1)
        ->and(TravelPhoto::count())->toBe(count($data['travelPhotos']))
        ->and(NavLink::count())->toBe(count($data['navLinks']))
        ->and(SocialLink::count())->toBe(count($data['site']['socials']))
        ->and(Article::where('type', 'travel')->count())->toBe(count($data['travelStories']))
        ->and(Article::where('type', 'note')->count())->toBe(count($data['notes']))
        ->and(Article::where('status', '!=', ArticleStatus::Draft->value)->count())->toBe(0)
        ->and(SiteSetting::read('site.name'))->toBe($data['site']['name'])
        ->and(SiteSetting::read('marquee_items'))->toBe($data['marqueeItems']);
});

it('serves seeded projects in the frontend order and shape', function () {
    $data = frontendData();
    $this->seed(DatabaseSeeder::class);

    $projects = $this->getJson('/api/v1/projects')->assertOk()->json('data');

    expect(array_column($projects, 'slug'))->toBe(array_column($data['projects'], 'slug'))
        ->and(array_column($projects, 'index'))->toBe(array_column($data['projects'], 'index'))
        ->and($projects[0]['highlights'])->toBe($data['projects'][0]['highlights'])
        ->and($projects[0]['tech'])->toBe($data['projects'][0]['tech']);

    $firstStory = $data['travelStories'][0];
    $this->getJson("/api/v1/travel/{$firstStory['slug']}")->assertOk()
        ->assertJsonPath('data.categories', $firstStory['categories'])
        ->assertJsonPath('data.blocks.0', $firstStory['blocks'][0]);
});
```

- [ ] **Step 4: Run test to verify it fails**

Run: `php artisan test --filter=SeederTest`
Expected: FAIL (`DatabaseSeeder` still creates a test user and no portfolio data; counts are 0).

- [ ] **Step 5: Implement the seeders**

Replace `database/seeders/DatabaseSeeder.php`:

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed the portfolio content. Admin accounts are created with `php artisan admin:create`.
     */
    public function run(): void
    {
        $this->call(PortfolioSeeder::class);
    }
}
```

Create `database/seeders/PortfolioSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Enums\ArticleStatus;
use App\Enums\ArticleType;
use App\Models\Article;
use App\Models\CreativeItem;
use App\Models\NavLink;
use App\Models\Project;
use App\Models\Service;
use App\Models\SiteSetting;
use App\Models\SkillGroup;
use App\Models\SocialLink;
use App\Models\TimelineEntry;
use App\Models\TravelCategory;
use App\Models\TravelPhoto;
use App\Support\ArticleBlocks;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

/**
 * Recreates the content of frontend/data/*.ts (exported to data/frontend.json).
 * Idempotent: existing rows (matched by natural key) are left untouched.
 */
class PortfolioSeeder extends Seeder
{
    public function run(): void
    {
        $data = json_decode(file_get_contents(database_path('seeders/data/frontend.json')), true, flags: JSON_THROW_ON_ERROR);

        DB::transaction(function () use ($data) {
            $this->settings($data);
            $this->navigation($data);
            $this->portfolio($data);
            $this->travel($data);
            $this->notes($data);
        });
    }

    private function settings(array $data): void
    {
        $site = $data['site'];
        $values = [
            'site.name' => $site['name'],
            'site.short_name' => $site['shortName'],
            'site.title' => $site['title'],
            'site.description' => $site['description'],
            'site.url' => $site['url'],
            'site.email' => $site['email'],
            'site.location' => $site['location'],
            'site.cv_media_id' => null,
            'site.portrait_media_id' => null,
            'site.portrait2_media_id' => null,
            'education' => $data['education'],
            'marquee_items' => $data['marqueeItems'],
            'collab_areas' => $data['collabAreas'],
        ];

        foreach ($values as $key => $value) {
            SiteSetting::query()->firstOrCreate(['key' => $key], ['value' => $value]);
        }
    }

    private function navigation(array $data): void
    {
        foreach ($data['navLinks'] as $i => $link) {
            NavLink::query()->firstOrCreate(['label' => $link['label']], ['href' => $link['href'], 'sort_order' => $i + 1]);
        }

        foreach ($data['site']['socials'] as $i => $social) {
            SocialLink::query()->firstOrCreate(['label' => $social['label']], ['url' => $social['url'], 'sort_order' => $i + 1]);
        }
    }

    private function portfolio(array $data): void
    {
        foreach ($data['projects'] as $i => $p) {
            Project::query()->firstOrCreate(['slug' => $p['slug']], [
                'title' => $p['title'],
                'category' => $p['category'],
                'description' => $p['description'],
                'highlights' => $p['highlights'],
                'tech' => $p['tech'],
                'href' => $p['href'] ?? null,
                'sort_order' => $i + 1,
            ]);
        }

        foreach ($data['creativeItems'] as $i => $c) {
            CreativeItem::query()->firstOrCreate(['key' => $c['id']], [
                'title' => $c['title'], 'discipline' => $c['discipline'], 'shape' => $c['shape'], 'sort_order' => $i + 1,
            ]);
        }

        foreach ($data['skillGroups'] as $i => $g) {
            $group = SkillGroup::query()->firstOrCreate(['title' => $g['title']], ['sort_order' => $i + 1]);
            foreach ($g['items'] as $j => $name) {
                $group->skills()->firstOrCreate(['name' => $name], ['sort_order' => $j + 1]);
            }
        }

        foreach ($data['services'] as $i => $s) {
            Service::query()->firstOrCreate(['title' => $s['title']], [
                'body' => $s['body'], 'tech' => $s['tech'], 'cta_label' => $s['cta'], 'cta_href' => $s['href'], 'sort_order' => $i + 1,
            ]);
        }

        foreach ($data['timeline'] as $i => $t) {
            TimelineEntry::query()->firstOrCreate(['title' => $t['title']], ['detail' => $t['detail'], 'sort_order' => $i + 1]);
        }
    }

    private function travel(array $data): void
    {
        foreach (array_values(array_diff($data['travelFilters'], ['All Stories'])) as $i => $name) {
            TravelCategory::query()->firstOrCreate(['name' => $name], ['slug' => Str::slug($name), 'sort_order' => $i + 1]);
        }

        foreach ($data['travelPhotos'] as $i => $photo) {
            TravelPhoto::query()->firstOrCreate(['key' => $photo['id']], [
                'theme' => $photo['theme'], 'aspect' => $photo['aspect'], 'sort_order' => $i + 1,
            ]);
        }

        foreach ($data['travelStories'] as $i => $story) {
            $article = $this->article(ArticleType::Travel, $story, $i, ['destination' => $story['destination'], 'featured' => $story['featured'] ?? false]);

            if ($article->wasRecentlyCreated) {
                $article->travelCategories()->sync(TravelCategory::query()->whereIn('name', $story['categories'])->pluck('id'));
            }
        }
    }

    private function notes(array $data): void
    {
        foreach ($data['notes'] as $i => $note) {
            $this->article(ArticleType::Note, $note, $i, ['topic' => $note['topic']]);
        }
    }

    /**
     * @param  array<string, mixed>  $extra
     */
    private function article(ArticleType $type, array $source, int $index, array $extra): Article
    {
        return Article::query()->firstOrCreate(['type' => $type->value, 'slug' => $source['slug']], [
            'title' => $source['title'],
            'subtitle' => $source['subtitle'] ?? null,
            'excerpt' => $source['excerpt'],
            'category' => $source['category'],
            'status' => ($source['draft'] ?? true) ? ArticleStatus::Draft : ArticleStatus::Published,
            'published_at' => $source['date'] ?? null,
            // normalize() turns frontend image blocks ({image: ImageRef, caption}) into {media_id: null, caption}.
            'blocks' => ArticleBlocks::normalize($source['blocks']),
            'sort_order' => $index + 1,
            ...$extra,
        ]);
    }
}
```

Note: `ArticleBlocks::normalize` re-derives heading ids with `Str::slug`; the frontend ids (`overview`, `route`, `diary`, ...) are already slugs, so they are preserved. The second test asserts `blocks.0` equals the frontend's first block, which is a heading.

- [ ] **Step 6: Run test to verify it passes**

Run: `php artisan test --filter=SeederTest`
Expected: PASS (2 tests).

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: seed the current frontend content"
```

---

### Task 6: Admin authentication, layout and dashboard

**Files:**
- Create: `app/Http/Controllers/Admin/Auth/LoginController.php`, `app/Http/Controllers/Admin/DashboardController.php`, `app/Console/Commands/CreateAdminCommand.php`, `resources/views/components/admin/layout.blade.php`, `resources/views/admin/auth/login.blade.php`, `resources/views/admin/dashboard.blade.php`, `resources/views/components/island.blade.php`, `resources/js/islands.jsx` (empty registry for now)
- Modify: `routes/web.php`, `bootstrap/app.php`, `vite.config.js`, `resources/css/app.css`, `tests/Feature/ExampleTest.php`
- Delete: `resources/views/welcome.blade.php`, `resources/js/app.js`
- Test: `tests/Feature/AdminAuthTest.php`

**Interfaces:**
- Produces: route names `admin.login`, `admin.login.store`, `admin.logout`, `admin.dashboard`; `<x-admin.layout title="...">` component; `<x-island name="..." :props="[...]" />` rendering `<div data-island="name" data-props="{json}">`; CSS classes `btn-primary`, `btn-secondary`, `btn-danger`, `input`; command `admin:create {--name=} {--email=} {--password=} {--force}`.

- [ ] **Step 1: Write the failing test**

Replace `tests/Feature/ExampleTest.php` with:

```php
<?php

test('the root url sends visitors to the admin', function () {
    $this->get('/')->assertRedirect('/admin');
});
```

Create `tests/Feature/AdminAuthTest.php`:

```php
<?php

use App\Models\User;
use Illuminate\Support\Facades\Hash;

it('redirects guests to the login page', function () {
    $this->get('/admin')->assertRedirect('/admin/login');
    $this->get('/admin/login')->assertOk()->assertSee('Sign in');
});

it('signs in with valid credentials and shows the dashboard', function () {
    $user = User::factory()->create(['email' => 'admin@example.test', 'password' => 'secret-password']);

    $this->post('/admin/login', ['email' => 'ADMIN@example.test', 'password' => 'secret-password'])
        ->assertRedirect('/admin');

    $this->assertAuthenticatedAs($user);
    $this->get('/admin')->assertOk()->assertSee('Dashboard')->assertSee('Projects');
});

it('rejects wrong passwords and throttles after five attempts', function () {
    User::factory()->create(['email' => 'admin@example.test', 'password' => 'secret-password']);

    foreach (range(1, 5) as $attempt) {
        $this->post('/admin/login', ['email' => 'admin@example.test', 'password' => 'wrong'])
            ->assertSessionHasErrors(['email' => 'These credentials do not match our records.']);
    }

    $this->post('/admin/login', ['email' => 'admin@example.test', 'password' => 'secret-password'])
        ->assertSessionHasErrors('email');
    $this->assertGuest();
});

it('signs out', function () {
    $this->actingAs(User::factory()->create())
        ->post('/admin/logout')
        ->assertRedirect('/admin/login');

    $this->assertGuest();
});

it('creates an admin account from the console', function () {
    $this->artisan('admin:create', ['--name' => 'Sarthak', '--email' => 'me@example.test', '--password' => 'long-password-1'])
        ->assertSuccessful();

    $user = User::where('email', 'me@example.test')->sole();
    expect(Hash::check('long-password-1', $user->password))->toBeTrue();

    $this->artisan('admin:create', ['--name' => 'S', '--email' => 'me@example.test', '--password' => 'short'])
        ->assertFailed();
});

it('renders island mount points with JSON props', function () {
    $this->blade('<x-island name="tags-input" :props="[\'name\' => \'tech\', \'value\' => [\'PHP\']]" />')
        ->assertSee('data-island="tags-input"', false)
        ->assertSee('&quot;value&quot;:[&quot;PHP&quot;]', false);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter="AdminAuthTest|ExampleTest"`
Expected: FAIL (`/admin` is 404; `admin:create` is not defined).

- [ ] **Step 3: Configure redirects, routes and Vite**

In `bootstrap/app.php` replace the `withMiddleware` callback body `//` with:

```php
        $middleware->redirectGuestsTo('/admin/login');
        $middleware->redirectUsersTo('/admin');
```

Replace `routes/web.php`:

```php
<?php

use App\Http\Controllers\Admin\Auth\LoginController;
use App\Http\Controllers\Admin\DashboardController;
use Illuminate\Support\Facades\Route;

Route::redirect('/', '/admin');

Route::prefix('admin')->name('admin.')->group(function () {
    Route::middleware('guest')->group(function () {
        Route::get('login', [LoginController::class, 'show'])->name('login');
        Route::post('login', [LoginController::class, 'store'])->name('login.store');
    });

    Route::middleware('auth')->group(function () {
        Route::post('logout', [LoginController::class, 'destroy'])->name('logout');
        Route::get('/', DashboardController::class)->name('dashboard');
    });
});
```

Replace `vite.config.js`:

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import { bunny } from 'laravel-vite-plugin/fonts';
import tailwindcss from '@tailwindcss/vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/islands.jsx'],
            refresh: true,
            fonts: [
                bunny('Instrument Sans', {
                    weights: [400, 500, 600],
                }),
            ],
        }),
        react(),
        tailwindcss(),
    ],
    server: {
        watch: {
            ignored: ['**/storage/framework/views/**'],
        },
    },
});
```

Append to `resources/css/app.css`:

```css
@layer components {
    .btn-primary {
        @apply inline-flex items-center gap-2 rounded-lg bg-blue-600 px-4 py-2 text-sm font-medium text-white shadow-sm hover:bg-blue-700;
    }
    .btn-secondary {
        @apply inline-flex items-center gap-2 rounded-lg border border-slate-300 bg-white px-4 py-2 text-sm font-medium text-slate-700 shadow-sm hover:bg-slate-50;
    }
    .btn-danger {
        @apply inline-flex items-center gap-2 rounded-lg bg-red-600 px-4 py-2 text-sm font-medium text-white shadow-sm hover:bg-red-700;
    }
    .input {
        @apply block w-full rounded-lg border border-slate-300 bg-white px-3 py-2 text-sm shadow-sm focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20 focus:outline-none;
    }
}
```

Delete `resources/views/welcome.blade.php` and `resources/js/app.js`.

Create `resources/js/islands.jsx` (components are registered in Task 8):

```jsx
import { App as AntApp, ConfigProvider } from 'antd';
import { createRoot } from 'react-dom/client';

// name used in <x-island name="..."> → React component
const registry = {};

const theme = {
    token: {
        colorPrimary: '#2563eb',
        borderRadius: 8,
        fontFamily: "'Instrument Sans', ui-sans-serif, system-ui, sans-serif",
    },
};

export function registerIslands(components) {
    Object.assign(registry, components);
    mount();
}

function mount() {
    document.querySelectorAll('[data-island]').forEach((element) => {
        const Component = registry[element.dataset.island];
        if (!Component || element.dataset.mounted) return;
        element.dataset.mounted = '1';
        const props = JSON.parse(element.dataset.props || '{}');
        createRoot(element).render(
            <ConfigProvider theme={theme}>
                <AntApp>
                    <Component {...props} />
                </AntApp>
            </ConfigProvider>,
        );
    });
}

if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', mount);
} else {
    mount();
}
```

- [ ] **Step 4: Implement controllers and the command**

`app/Http/Controllers/Admin/Auth/LoginController.php`:

```php
<?php

namespace App\Http\Controllers\Admin\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;
use Illuminate\View\View;

class LoginController extends Controller
{
    public function show(): View
    {
        return view('admin.auth.login');
    }

    public function store(Request $request): RedirectResponse
    {
        $credentials = $request->validate([
            'email' => ['required', 'string', 'email'],
            'password' => ['required', 'string'],
        ]);
        $credentials['email'] = Str::lower($credentials['email']);
        $key = $credentials['email'].'|'.$request->ip();

        if (RateLimiter::tooManyAttempts($key, 5)) {
            throw ValidationException::withMessages([
                'email' => 'Too many login attempts. Try again in '.RateLimiter::availableIn($key).' seconds.',
            ]);
        }

        if (! Auth::attempt($credentials, $request->boolean('remember'))) {
            RateLimiter::hit($key, 60);

            throw ValidationException::withMessages(['email' => 'These credentials do not match our records.']);
        }

        RateLimiter::clear($key);
        $request->session()->regenerate();

        return redirect()->intended(route('admin.dashboard'));
    }

    public function destroy(Request $request): RedirectResponse
    {
        Auth::logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect()->route('admin.login');
    }
}
```

`app/Http/Controllers/Admin/DashboardController.php`:

```php
<?php

namespace App\Http\Controllers\Admin;

use App\Enums\ArticleStatus;
use App\Http\Controllers\Controller;
use App\Models\Article;
use App\Models\Media;
use App\Models\Project;
use Illuminate\View\View;

class DashboardController extends Controller
{
    public function __invoke(): View
    {
        return view('admin.dashboard', [
            'counts' => [
                'Projects' => Project::count(),
                'Travel stories' => Article::where('type', 'travel')->count(),
                'Notes' => Article::where('type', 'note')->count(),
                'Media files' => Media::count(),
            ],
            'drafts' => Article::query()->where('status', ArticleStatus::Draft->value)->latest('updated_at')->limit(8)->get(),
        ]);
    }
}
```

`app/Console/Commands/CreateAdminCommand.php`:

```php
<?php

namespace App\Console\Commands;

use App\Models\User;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Validator;
use Illuminate\Support\Str;

use function Laravel\Prompts\password;
use function Laravel\Prompts\text;

class CreateAdminCommand extends Command
{
    protected $signature = 'admin:create {--name= : Display name} {--email= : Login email} {--password= : Password (omit to be prompted)} {--force : Reset an existing account without asking}';

    protected $description = 'Create (or reset) the portfolio admin account';

    public function handle(): int
    {
        $name = $this->stringOption('name') ?? text('Name', default: 'Sarthak Neupane', required: true);
        $email = Str::lower($this->stringOption('email') ?? text('Email', required: true));
        $password = $this->stringOption('password') ?? password('Password', required: true);

        $validator = Validator::make(compact('name', 'email', 'password'), [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'max:255'],
            'password' => ['required', 'string', 'min:8'],
        ]);

        if ($validator->fails()) {
            foreach ($validator->errors()->all() as $error) {
                $this->components->error($error);
            }

            return self::FAILURE;
        }

        $exists = User::query()->where('email', $email)->exists();

        if ($exists && ! $this->option('force') && $this->input->isInteractive()
            && ! $this->confirm("{$email} already exists. Reset its password?")) {
            return self::FAILURE;
        }

        User::query()->updateOrCreate(['email' => $email], ['name' => $name, 'password' => $password]);
        $this->components->info(($exists ? 'Updated' : 'Created')." admin {$email}.");

        return self::SUCCESS;
    }

    private function stringOption(string $name): ?string
    {
        $value = $this->option($name);

        return is_string($value) && $value !== '' ? $value : null;
    }
}
```

- [ ] **Step 5: Write the Blade views**

`resources/views/components/island.blade.php`:

```blade
@props(['name', 'props' => []])
<div data-island="{{ $name }}" data-props="{{ json_encode($props, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE) }}" {{ $attributes }}></div>
```

`resources/views/components/admin/layout.blade.php`:

```blade
@props(['title' => 'Admin'])
@php
    $nav = [
        'Overview' => [['Dashboard', '/admin', 'admin']],
        'Content' => [
            ['Projects', '/admin/projects', 'admin/projects*'],
            ['Travel stories', '/admin/travel', 'admin/travel*'],
            ['Notes', '/admin/notes', 'admin/notes*'],
            ['Creative work', '/admin/creative-items', 'admin/creative-items*'],
            ['Travel photos', '/admin/travel-photos', 'admin/travel-photos*'],
            ['Travel categories', '/admin/travel-categories', 'admin/travel-categories*'],
        ],
        'About' => [
            ['Skills', '/admin/skill-groups', 'admin/skill-groups*'],
            ['Services', '/admin/services', 'admin/services*'],
            ['Timeline', '/admin/timeline', 'admin/timeline*'],
        ],
        'Site' => [
            ['Media library', '/admin/media', 'admin/media*'],
            ['Navigation', '/admin/nav-links', 'admin/nav-links*'],
            ['Social links', '/admin/social-links', 'admin/social-links*'],
            ['Site settings', '/admin/settings', 'admin/settings*'],
        ],
    ];
@endphp
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <meta name="robots" content="noindex, nofollow">
    <title>{{ $title }} · Portfolio admin</title>
    @fonts
    @vite(['resources/css/app.css', 'resources/js/islands.jsx'])
</head>
<body class="min-h-screen bg-slate-50 font-sans text-slate-900 antialiased">
    <aside id="admin-sidebar" class="fixed inset-y-0 left-0 z-40 hidden w-64 overflow-y-auto bg-slate-900 text-slate-300 lg:block">
        <div class="flex h-16 items-center gap-3 border-b border-white/10 px-5">
            <span class="flex h-8 w-8 items-center justify-center rounded-lg bg-white text-sm font-bold text-slate-900">SN</span>
            <span class="text-sm font-semibold text-white">Portfolio admin</span>
        </div>
        <nav class="space-y-6 px-3 py-5">
            @foreach ($nav as $group => $items)
                <div>
                    <p class="mb-2 px-3 text-[11px] font-semibold tracking-wider text-slate-500 uppercase">{{ $group }}</p>
                    @foreach ($items as [$label, $href, $pattern])
                        <a href="{{ $href }}" @class([
                            'block rounded-lg px-3 py-2 text-sm',
                            'bg-white/10 font-medium text-white' => request()->is($pattern),
                            'hover:bg-white/5 hover:text-white' => ! request()->is($pattern),
                        ])>{{ $label }}</a>
                    @endforeach
                </div>
            @endforeach
        </nav>
    </aside>

    <div class="lg:pl-64">
        <header class="sticky top-0 z-30 flex h-16 items-center gap-3 border-b border-slate-200 bg-white/90 px-4 backdrop-blur sm:px-6">
            <button type="button" class="rounded-lg p-2 text-slate-600 hover:bg-slate-100 lg:hidden" aria-label="Open menu"
                onclick="document.getElementById('admin-sidebar').classList.toggle('hidden')">☰</button>
            <span class="text-sm text-slate-500">{{ $title }}</span>
            <form method="POST" action="{{ route('admin.logout') }}" class="ml-auto">
                @csrf
                <button type="submit" class="text-sm text-slate-600 hover:text-slate-900">Sign out ({{ auth()->user()->name }})</button>
            </form>
        </header>

        <main class="mx-auto max-w-6xl px-4 py-8 sm:px-6">
            @if (session('status'))
                <div class="mb-6 rounded-lg border border-emerald-200 bg-emerald-50 px-4 py-3 text-sm text-emerald-800" role="status">{{ session('status') }}</div>
            @endif
            @if ($errors->any())
                <div class="mb-6 rounded-lg border border-red-200 bg-red-50 px-4 py-3 text-sm text-red-700" role="alert">Please fix the highlighted fields.</div>
            @endif
            {{ $slot }}
        </main>
    </div>
</body>
</html>
```

`resources/views/admin/auth/login.blade.php`:

```blade
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="robots" content="noindex, nofollow">
    <title>Sign in · Portfolio admin</title>
    @fonts
    @vite(['resources/css/app.css'])
</head>
<body class="flex min-h-screen items-center justify-center bg-slate-100 px-4 font-sans antialiased">
    <div class="w-full max-w-sm">
        <div class="mb-8 text-center">
            <div class="mx-auto mb-4 flex h-12 w-12 items-center justify-center rounded-xl bg-slate-900 text-lg font-bold text-white">SN</div>
            <h1 class="text-xl font-semibold text-slate-900">Portfolio admin</h1>
            <p class="mt-1 text-sm text-slate-500">Sign in to manage your content</p>
        </div>
        <form method="POST" action="{{ route('admin.login.store') }}" class="space-y-4 rounded-2xl border border-slate-200 bg-white p-6 shadow-sm">
            @csrf
            @error('email')
                <div class="rounded-lg bg-red-50 px-3 py-2 text-sm text-red-700">{{ $message }}</div>
            @enderror
            <label class="block space-y-1.5">
                <span class="text-sm font-medium text-slate-700">Email</span>
                <input type="email" name="email" value="{{ old('email') }}" required autofocus autocomplete="username" class="input">
            </label>
            <label class="block space-y-1.5">
                <span class="text-sm font-medium text-slate-700">Password</span>
                <input type="password" name="password" required autocomplete="current-password" class="input">
            </label>
            <label class="flex items-center gap-2 text-sm text-slate-600">
                <input type="checkbox" name="remember" value="1" class="h-4 w-4 rounded border-slate-300"> Remember me
            </label>
            <button type="submit" class="btn-primary w-full justify-center">Sign in</button>
        </form>
    </div>
</body>
</html>
```

`resources/views/admin/dashboard.blade.php`:

```blade
<x-admin.layout title="Dashboard">
    <h1 class="mb-6 text-2xl font-semibold tracking-tight">Dashboard</h1>

    <div class="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
        @foreach ($counts as $label => $count)
            <div class="rounded-xl border border-slate-200 bg-white p-5 shadow-sm">
                <p class="text-sm text-slate-500">{{ $label }}</p>
                <p class="mt-2 text-3xl font-semibold">{{ $count }}</p>
            </div>
        @endforeach
    </div>

    <section class="mt-8 rounded-xl border border-slate-200 bg-white shadow-sm">
        <h2 class="border-b border-slate-100 px-5 py-3 text-sm font-semibold">Drafts to finish</h2>
        @forelse ($drafts as $draft)
            <a href="/admin/{{ $draft->type->segment() }}/{{ $draft->id }}/edit" class="flex items-center justify-between border-b border-slate-100 px-5 py-3 text-sm last:border-0 hover:bg-slate-50">
                <span class="font-medium">{{ $draft->title }}</span>
                <span class="text-xs text-slate-500">{{ $draft->type->label() }} · {{ $draft->updated_at->diffForHumans() }}</span>
            </a>
        @empty
            <p class="px-5 py-6 text-sm text-slate-500">No drafts. Everything is published.</p>
        @endforelse
    </section>
</x-admin.layout>
```

- [ ] **Step 6: Run tests**

Run: `php artisan test --filter="AdminAuthTest|ExampleTest"`
Expected: PASS (7 tests).

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: admin login, layout, dashboard and admin:create command"
```

---

### Task 7: Admin JSON endpoints (media and reorder)

**Files:**
- Create: `app/Http/Resources/Admin/MediaResource.php`, `app/Http/Controllers/Admin/Api/MediaController.php`, `app/Http/Controllers/Admin/Api/ReorderController.php`
- Modify: `routes/web.php`
- Test: `tests/Feature/AdminApiTest.php`

**Interfaces:**
- Consumes: `MediaStore`, `MediaUsage`, models.
- Produces:
  - `MediaResource` array: `{ id, url, thumb, alt, original_name, mime_type, size_bytes, width, height, is_image }` (`thumb` = `url('thumb')` for images, else `null`).
  - `GET /admin/api/media?search=&type=image&page=` → paginated `{data, links, meta}` (24 per page); `POST /admin/api/media` (`files[]`, optional `alt`) → 201 `{data: [...]}`; `PATCH /admin/api/media/{media}` (`alt`) → `{data}`; `DELETE /admin/api/media/{media}` → 204 or 409 `{message, usages}`; `GET /admin/api/media/{media}/usages` → `{data: usages}`.
  - `POST /admin/api/reorder` `{resource, ids}` with `resource` ∈ `projects, services, timeline_entries, skill_groups, creative_items, travel_photos, nav_links, social_links, travel_categories, articles` → `{message: "Order saved."}`.

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/AdminApiTest.php`:

```php
<?php

use App\Models\Media;
use App\Models\Project;
use App\Models\User;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

beforeEach(function () {
    Storage::fake('public');
    config(['portfolio.media.disk' => 'public']);
});

it('rejects guests with 401', function () {
    $this->getJson('/admin/api/media')->assertUnauthorized();
    $this->postJson('/admin/api/reorder', [])->assertUnauthorized();
});

it('uploads, lists, searches and updates media', function () {
    $this->actingAs(User::factory()->create());

    $this->post('/admin/api/media', ['files' => [UploadedFile::fake()->image('everest.jpg', 1200, 800), UploadedFile::fake()->create('cv.pdf', 50, 'application/pdf')]], ['Accept' => 'application/json'])
        ->assertCreated()
        ->assertJsonCount(2, 'data')
        ->assertJsonPath('data.0.is_image', true)
        ->assertJsonPath('data.1.thumb', null);

    $this->getJson('/admin/api/media?type=image')->assertOk()->assertJsonCount(1, 'data')->assertJsonPath('meta.total', 1);
    $this->getJson('/admin/api/media?search=cv')->assertOk()->assertJsonPath('data.0.original_name', 'cv.pdf');

    $id = Media::where('original_name', 'everest.jpg')->value('id');
    $this->patchJson("/admin/api/media/{$id}", ['alt' => 'Everest at dusk'])->assertOk()->assertJsonPath('data.alt', 'Everest at dusk');
});

it('refuses disallowed uploads', function () {
    $this->actingAs(User::factory()->create());

    $this->post('/admin/api/media', ['files' => [UploadedFile::fake()->create('logo.svg', 2, 'image/svg+xml')]], ['Accept' => 'application/json'])
        ->assertUnprocessable()
        ->assertJsonValidationErrors('files.0');
});

it('protects media in use and deletes unused media', function () {
    $this->actingAs(User::factory()->create());
    $used = Media::factory()->create();
    $unused = Media::factory()->create();
    Project::create(['slug' => 'p', 'title' => 'Project', 'category' => 'c', 'description' => 'd', 'cover_media_id' => $used->id]);

    $this->getJson("/admin/api/media/{$used->id}/usages")->assertOk()->assertJsonPath('data.0.label', 'Project');
    $this->deleteJson("/admin/api/media/{$used->id}")->assertStatus(409)->assertJsonPath('usages.0.type', 'project');
    $this->deleteJson("/admin/api/media/{$unused->id}")->assertNoContent();

    expect($unused->fresh()->trashed())->toBeTrue();
});

it('saves drag-and-drop order for allowed resources only', function () {
    $this->actingAs(User::factory()->create());
    $a = Project::create(['slug' => 'a', 'title' => 'A', 'category' => 'c', 'description' => 'd']);
    $b = Project::create(['slug' => 'b', 'title' => 'B', 'category' => 'c', 'description' => 'd']);

    $this->postJson('/admin/api/reorder', ['resource' => 'projects', 'ids' => [$b->id, $a->id]])->assertOk();
    expect(Project::ordered()->pluck('slug')->all())->toBe(['b', 'a']);

    $this->postJson('/admin/api/reorder', ['resource' => 'users', 'ids' => [1]])->assertUnprocessable();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter=AdminApiTest`
Expected: FAIL (404 for `/admin/api/...`).

- [ ] **Step 3: Add the routes**

In `routes/web.php`, add these `use` lines:

```php
use App\Http\Controllers\Admin\Api\MediaController as MediaApiController;
use App\Http\Controllers\Admin\Api\ReorderController;
```

and inside the `Route::middleware('auth')->group(...)` closure, after the dashboard route, add:

```php
        Route::prefix('api')->name('api.')->group(function () {
            Route::get('media', [MediaApiController::class, 'index'])->name('media.index');
            Route::post('media', [MediaApiController::class, 'store'])->name('media.store');
            Route::patch('media/{media}', [MediaApiController::class, 'update'])->name('media.update');
            Route::delete('media/{media}', [MediaApiController::class, 'destroy'])->name('media.destroy');
            Route::get('media/{media}/usages', [MediaApiController::class, 'usages'])->name('media.usages');
            Route::post('reorder', ReorderController::class)->name('reorder');
        });
```

- [ ] **Step 4: Implement the resource and controllers**

`app/Http/Resources/Admin/MediaResource.php`:

```php
<?php

namespace App\Http\Resources\Admin;

use App\Models\Media;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

/**
 * @mixin Media
 */
class MediaResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'url' => $this->url(),
            'thumb' => $this->isImage() ? $this->url('thumb') : null,
            'alt' => $this->alt,
            'original_name' => $this->original_name,
            'mime_type' => $this->mime_type,
            'size_bytes' => $this->size_bytes,
            'width' => $this->width,
            'height' => $this->height,
            'is_image' => $this->isImage(),
        ];
    }
}
```

`app/Http/Controllers/Admin/Api/MediaController.php`:

```php
<?php

namespace App\Http\Controllers\Admin\Api;

use App\Http\Controllers\Controller;
use App\Http\Resources\Admin\MediaResource;
use App\Models\Media;
use App\Services\Media\MediaStore;
use App\Services\Media\MediaUsage;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Illuminate\Http\Response;
use Illuminate\Validation\Rules\File;

class MediaController extends Controller
{
    public function index(Request $request): AnonymousResourceCollection
    {
        $media = Media::query()
            ->when($request->string('search')->trim()->value(), fn ($q, $term) => $q->where(fn ($w) => $w
                ->where('original_name', 'like', "%{$term}%")->orWhere('alt', 'like', "%{$term}%")))
            ->when($request->input('type') === 'image', fn ($q) => $q->where('mime_type', 'like', 'image/%'))
            ->latest()
            ->orderByDesc('id')
            ->paginate(24);

        return MediaResource::collection($media);
    }

    public function store(Request $request, MediaStore $store): JsonResponse
    {
        $request->validate([
            'files' => ['required', 'array', 'min:1', 'max:20'],
            'files.*' => ['required', File::types(config('portfolio.media.extensions'))->max(config('portfolio.media.max_upload_kb'))],
            'alt' => ['nullable', 'string', 'max:255'],
        ]);

        $created = collect($request->file('files'))->map(fn ($file) => $store->store($file, $request->input('alt')));

        return MediaResource::collection($created)->response()->setStatusCode(201);
    }

    public function update(Request $request, Media $media): MediaResource
    {
        $media->update($request->validate(['alt' => ['nullable', 'string', 'max:255']]));

        return new MediaResource($media);
    }

    public function destroy(Media $media, MediaStore $store): Response
    {
        $store->delete($media); // MediaInUseException renders 409 with usages

        return response()->noContent();
    }

    public function usages(Media $media, MediaUsage $usage): JsonResponse
    {
        return response()->json(['data' => $usage->find($media)]);
    }
}
```

`app/Http/Controllers/Admin/Api/ReorderController.php`:

```php
<?php

namespace App\Http\Controllers\Admin\Api;

use App\Http\Controllers\Controller;
use App\Models\Article;
use App\Models\CreativeItem;
use App\Models\NavLink;
use App\Models\Project;
use App\Models\Service;
use App\Models\SkillGroup;
use App\Models\SocialLink;
use App\Models\TimelineEntry;
use App\Models\TravelCategory;
use App\Models\TravelPhoto;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Validation\Rule;

class ReorderController extends Controller
{
    /** Table name (sent by the SortableList island) → model. */
    public const MODELS = [
        'projects' => Project::class,
        'services' => Service::class,
        'timeline_entries' => TimelineEntry::class,
        'skill_groups' => SkillGroup::class,
        'creative_items' => CreativeItem::class,
        'travel_photos' => TravelPhoto::class,
        'nav_links' => NavLink::class,
        'social_links' => SocialLink::class,
        'travel_categories' => TravelCategory::class,
        'articles' => Article::class,
    ];

    public function __invoke(Request $request): JsonResponse
    {
        $data = $request->validate([
            'resource' => ['required', 'string', Rule::in(array_keys(self::MODELS))],
            'ids' => ['required', 'array', 'min:1', 'max:500'],
            'ids.*' => ['integer', 'distinct'],
        ]);

        $model = self::MODELS[$data['resource']];

        DB::transaction(function () use ($model, $data) {
            foreach ($data['ids'] as $index => $id) {
                $model::query()->whereKey($id)->update(['sort_order' => $index + 1]);
            }
        });

        return response()->json(['message' => 'Order saved.']);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `php artisan test --filter=AdminApiTest`
Expected: PASS (5 tests).

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: admin JSON endpoints for media and ordering"
```

---

### Task 8: React islands and the media library page

**Files:**
- Create: `resources/js/api.js`, `resources/js/islands/MediaBrowser.jsx`, `resources/js/islands/MediaPicker.jsx`, `resources/js/islands/MediaLibrary.jsx`, `resources/js/islands/SortableList.jsx`, `resources/js/islands/TagsInput.jsx`, `resources/js/islands/BlockEditor.jsx`, `resources/views/admin/media.blade.php`
- Modify: `resources/js/islands.jsx`, `routes/web.php`
- Test: `tests/Feature/MediaPageTest.php`, plus `npm run build`

**Interfaces:**
- Consumes: `/admin/api/*` from Task 7; `<x-island>` from Task 6.
- Produces island names and props:
  - `media-picker` `{ name?: string, value: number|null, preview: MediaResource|null, accept?: 'image'|'any', onChange?: (id, media) => void }` — renders `<input type="hidden" name={name}>` when `name` is given.
  - `media-library` `{}`.
  - `sortable-list` `{ resource: string (table name), items: [{ id, label, sublabel?, editUrl, deleteUrl }], emptyText?: string }`.
  - `tags-input` `{ name: string, value: string[], placeholder?: string }` — renders `name[]` hidden inputs, or one empty `name` input when the list is empty.
  - `block-editor` `{ name: 'blocks', value: Block[], previews: { [mediaId]: MediaResource }, errors: { ["blocks.i.field"]: string[] } }` — renders one hidden `<input name="blocks">` holding the JSON array.

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/MediaPageTest.php`:

```php
<?php

use App\Models\User;

it('hosts the media library island', function () {
    $this->actingAs(User::factory()->create())
        ->get('/admin/media')
        ->assertOk()
        ->assertSee('Media library')
        ->assertSee('data-island="media-library"', false);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter=MediaPageTest`
Expected: FAIL (404).

- [ ] **Step 3: Add the page route and view**

In `routes/web.php` inside the `auth` group add:

```php
        Route::view('media', 'admin.media')->name('media');
```

`resources/views/admin/media.blade.php`:

```blade
<x-admin.layout title="Media library">
    <h1 class="mb-1 text-2xl font-semibold tracking-tight">Media library</h1>
    <p class="mb-6 text-sm text-slate-500">Upload once, reuse everywhere. Files still in use cannot be deleted.</p>
    <div class="rounded-xl border border-slate-200 bg-white p-5 shadow-sm">
        <x-island name="media-library" />
    </div>
</x-admin.layout>
```

- [ ] **Step 4: Run test to verify it passes**

Run: `php artisan test --filter=MediaPageTest`
Expected: PASS.

- [ ] **Step 5: Write the JavaScript**

`resources/js/api.js`:

```js
const csrfToken = () => document.querySelector('meta[name="csrf-token"]')?.content ?? '';

/**
 * fetch() wrapper for the session-authenticated /admin/api endpoints.
 * Throws an Error with `status` and `data` (the JSON body) on non-2xx responses.
 */
export async function request(url, { method = 'GET', body } = {}) {
    const headers = { Accept: 'application/json', 'X-CSRF-TOKEN': csrfToken(), 'X-Requested-With': 'XMLHttpRequest' };
    let payload = body;

    if (body !== undefined && !(body instanceof FormData)) {
        headers['Content-Type'] = 'application/json';
        payload = JSON.stringify(body);
    }

    const response = await fetch(url, { method, headers, body: payload, credentials: 'same-origin' });
    if (response.status === 204) return null;

    const data = await response.json().catch(() => ({}));
    if (!response.ok) {
        const firstError = data.errors ? Object.values(data.errors)[0]?.[0] : null;
        const error = new Error(firstError || data.message || `Request failed (${response.status})`);
        error.status = response.status;
        error.data = data;
        throw error;
    }

    return data;
}
```

`resources/js/islands/MediaBrowser.jsx`:

```jsx
import { FileOutlined, UploadOutlined } from '@ant-design/icons';
import { App, Button, Empty, Flex, Input, Pagination, Spin, Upload } from 'antd';
import { useCallback, useEffect, useState } from 'react';
import { request } from '../api';

/** Searchable grid of the media library with an upload button. */
export default function MediaBrowser({ accept = 'image', onSelect, selectedId = null }) {
    const { message } = App.useApp();
    const [items, setItems] = useState([]);
    const [meta, setMeta] = useState({ current_page: 1, total: 0, per_page: 24 });
    const [page, setPage] = useState(1);
    const [search, setSearch] = useState('');
    const [loading, setLoading] = useState(false);

    const load = useCallback(async () => {
        setLoading(true);
        try {
            const query = new URLSearchParams({ page, search, ...(accept === 'image' ? { type: 'image' } : {}) });
            const response = await request(`/admin/api/media?${query}`);
            setItems(response.data);
            setMeta(response.meta);
        } catch (error) {
            message.error(error.message);
        } finally {
            setLoading(false);
        }
    }, [page, search, accept, message]);

    useEffect(() => {
        load();
    }, [load]);

    const upload = async ({ file, onSuccess, onError }) => {
        const form = new FormData();
        form.append('files[]', file);
        try {
            const response = await request('/admin/api/media', { method: 'POST', body: form });
            onSuccess(response);
            message.success(`${file.name} uploaded`);
            page === 1 ? load() : setPage(1);
        } catch (error) {
            onError(error);
            message.error(error.message);
        }
    };

    return (
        <Flex vertical gap="middle">
            <Flex gap="small" wrap>
                <Input.Search allowClear placeholder="Search by name or alt text" style={{ maxWidth: 320 }}
                    onSearch={(value) => { setPage(1); setSearch(value); }} />
                <Upload customRequest={upload} showUploadList={false} multiple accept={accept === 'image' ? 'image/*' : undefined}>
                    <Button icon={<UploadOutlined />}>Upload</Button>
                </Upload>
            </Flex>
            <Spin spinning={loading}>
                {items.length === 0 ? (
                    <Empty description="No media yet" />
                ) : (
                    <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(128px, 1fr))', gap: 12 }}>
                        {items.map((item) => (
                            <button key={item.id} type="button" onClick={() => onSelect(item)}
                                className={`overflow-hidden rounded-lg border bg-white text-left transition hover:shadow ${selectedId === item.id ? 'border-blue-500 ring-2 ring-blue-500' : 'border-slate-200'}`}>
                                {item.thumb ? (
                                    <img src={item.thumb} alt={item.alt || ''} loading="lazy" className="aspect-square w-full object-cover" />
                                ) : (
                                    <div className="flex aspect-square w-full items-center justify-center bg-slate-100 text-2xl text-slate-400"><FileOutlined /></div>
                                )}
                                <div className="truncate px-2 py-1 text-xs text-slate-600">{item.original_name}</div>
                            </button>
                        ))}
                    </div>
                )}
            </Spin>
            <Pagination size="small" align="end" current={meta.current_page} total={meta.total} pageSize={meta.per_page} onChange={setPage} hideOnSinglePage />
        </Flex>
    );
}
```

`resources/js/islands/MediaPicker.jsx`:

```jsx
import { DeleteOutlined, FileOutlined, PictureOutlined } from '@ant-design/icons';
import { Button, Flex, Modal } from 'antd';
import { useState } from 'react';
import MediaBrowser from './MediaBrowser';

/**
 * Pick one item from the media library. As a form island it writes the id into a
 * hidden input named `name`; inside other islands pass `onChange` instead.
 */
export default function MediaPicker({ name, value = null, preview = null, accept = 'image', onChange }) {
    const [id, setId] = useState(value);
    const [media, setMedia] = useState(preview);
    const [open, setOpen] = useState(false);

    const choose = (item) => {
        setId(item.id);
        setMedia(item);
        setOpen(false);
        onChange?.(item.id, item);
    };

    const clear = () => {
        setId(null);
        setMedia(null);
        onChange?.(null, null);
    };

    return (
        <div>
            {name && <input type="hidden" name={name} value={id ?? ''} />}
            <Flex gap="middle" align="center" wrap>
                {id ? (
                    <>
                        {media?.thumb ? (
                            <img src={media.thumb} alt={media.alt || ''} className="h-20 w-20 rounded-lg border border-slate-200 object-cover" />
                        ) : (
                            <div className="flex h-20 w-20 items-center justify-center rounded-lg bg-slate-100 text-xl text-slate-400"><FileOutlined /></div>
                        )}
                        <span className="max-w-48 truncate text-sm text-slate-600">{media?.original_name ?? `Media #${id}`}</span>
                        <Button onClick={() => setOpen(true)}>Change</Button>
                        <Button danger icon={<DeleteOutlined />} onClick={clear}>Remove</Button>
                    </>
                ) : (
                    <Button icon={<PictureOutlined />} onClick={() => setOpen(true)}>Choose from library</Button>
                )}
            </Flex>
            <Modal title="Media library" open={open} onCancel={() => setOpen(false)} footer={null} width={960} destroyOnHidden>
                <MediaBrowser accept={accept} onSelect={choose} selectedId={id} />
            </Modal>
        </div>
    );
}
```

`resources/js/islands/MediaLibrary.jsx`:

```jsx
import { App, Button, Descriptions, Drawer, Flex, Input, List } from 'antd';
import { useState } from 'react';
import { request } from '../api';
import MediaBrowser from './MediaBrowser';

const formatBytes = (bytes) => (bytes > 1048576 ? `${(bytes / 1048576).toFixed(1)} MB` : `${Math.ceil(bytes / 1024)} KB`);

/** Full media manager: browse, upload, edit alt text, see usages, delete when unused. */
export default function MediaLibrary() {
    const { message, modal } = App.useApp();
    const [selected, setSelected] = useState(null);
    const [alt, setAlt] = useState('');
    const [usages, setUsages] = useState([]);
    const [saving, setSaving] = useState(false);
    const [reloadKey, setReloadKey] = useState(0);

    const open = async (item) => {
        setSelected(item);
        setAlt(item.alt ?? '');
        setUsages([]);
        try {
            setUsages((await request(`/admin/api/media/${item.id}/usages`)).data);
        } catch (error) {
            message.error(error.message);
        }
    };

    const save = async () => {
        setSaving(true);
        try {
            const response = await request(`/admin/api/media/${selected.id}`, { method: 'PATCH', body: { alt } });
            setSelected(response.data);
            setReloadKey((k) => k + 1);
            message.success('Alt text saved');
        } catch (error) {
            message.error(error.message);
        } finally {
            setSaving(false);
        }
    };

    const remove = () =>
        modal.confirm({
            title: `Delete ${selected.original_name}?`,
            okText: 'Delete',
            okButtonProps: { danger: true },
            onOk: async () => {
                try {
                    await request(`/admin/api/media/${selected.id}`, { method: 'DELETE' });
                    message.success('Deleted');
                    setSelected(null);
                    setReloadKey((k) => k + 1);
                } catch (error) {
                    if (error.status === 409) {
                        modal.error({
                            title: 'This file is still in use',
                            content: <ul>{error.data.usages.map((u) => <li key={`${u.type}-${u.id}`}>{u.type.replace('_', ' ')}: {u.label}</li>)}</ul>,
                        });
                    } else {
                        message.error(error.message);
                    }
                }
            },
        });

    return (
        <>
            <MediaBrowser key={reloadKey} accept="any" onSelect={open} selectedId={selected?.id} />
            <Drawer title={selected?.original_name} open={Boolean(selected)} onClose={() => setSelected(null)} size={440}>
                {selected && (
                    <Flex vertical gap="large">
                        {selected.is_image && <img src={selected.thumb} alt={selected.alt || ''} className="w-full rounded-lg border border-slate-200" />}
                        <Descriptions column={1} size="small" items={[
                            { key: 'type', label: 'Type', children: selected.mime_type },
                            { key: 'size', label: 'Size', children: formatBytes(selected.size_bytes) },
                            ...(selected.width ? [{ key: 'dim', label: 'Dimensions', children: `${selected.width} × ${selected.height}px` }] : []),
                            { key: 'url', label: 'URL', children: <a href={selected.url} target="_blank" rel="noreferrer">Open original</a> },
                        ]} />
                        <Flex vertical gap="small">
                            <label className="text-sm font-medium">Alt text</label>
                            <Input.TextArea rows={2} value={alt} onChange={(e) => setAlt(e.target.value)} placeholder="Describe the image" />
                            <Button type="primary" loading={saving} onClick={save}>Save alt text</Button>
                        </Flex>
                        <List size="small" header={<strong>Used in</strong>} dataSource={usages}
                            locale={{ emptyText: 'Not used anywhere' }}
                            renderItem={(u) => <List.Item>{u.type.replace('_', ' ')}: {u.label}</List.Item>} />
                        <Button danger onClick={remove}>Delete file</Button>
                    </Flex>
                )}
            </Drawer>
        </>
    );
}
```

`resources/js/islands/SortableList.jsx`:

```jsx
import { DeleteOutlined, EditOutlined, HolderOutlined } from '@ant-design/icons';
import { closestCenter, DndContext, KeyboardSensor, PointerSensor, useSensor, useSensors } from '@dnd-kit/core';
import { arrayMove, SortableContext, sortableKeyboardCoordinates, useSortable, verticalListSortingStrategy } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';
import { App, Button, Empty } from 'antd';
import { useState } from 'react';
import { request } from '../api';

function Row({ item, onDelete }) {
    const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id: item.id });

    return (
        <div ref={setNodeRef} style={{ transform: CSS.Transform.toString(transform), transition, opacity: isDragging ? 0.6 : 1 }}
            className="flex items-center gap-3 border-b border-slate-100 bg-white px-4 py-3 last:border-0">
            <button type="button" aria-label="Drag to reorder" className="cursor-grab touch-none text-slate-400 hover:text-slate-600" {...attributes} {...listeners}>
                <HolderOutlined />
            </button>
            <div className="min-w-0 flex-1">
                <a href={item.editUrl} className="font-medium text-slate-900 hover:text-blue-600">{item.label}</a>
                {item.sublabel && <div className="truncate text-xs text-slate-500">{item.sublabel}</div>}
            </div>
            <Button size="small" icon={<EditOutlined />} href={item.editUrl}>Edit</Button>
            <Button size="small" danger icon={<DeleteOutlined />} aria-label={`Delete ${item.label}`} onClick={() => onDelete(item)} />
        </div>
    );
}

/** Drag-to-reorder list; order is saved to /admin/api/reorder immediately. */
export default function SortableList({ resource, items: initialItems, emptyText = 'Nothing here yet.' }) {
    const { message, modal } = App.useApp();
    const [items, setItems] = useState(initialItems);
    const sensors = useSensors(
        useSensor(PointerSensor, { activationConstraint: { distance: 4 } }),
        useSensor(KeyboardSensor, { coordinateGetter: sortableKeyboardCoordinates }),
    );

    const onDragEnd = async ({ active, over }) => {
        if (!over || active.id === over.id) return;
        const previous = items;
        const next = arrayMove(items, items.findIndex((i) => i.id === active.id), items.findIndex((i) => i.id === over.id));
        setItems(next);
        try {
            await request('/admin/api/reorder', { method: 'POST', body: { resource, ids: next.map((i) => i.id) } });
            message.success('Order saved');
        } catch (error) {
            setItems(previous);
            message.error(error.message);
        }
    };

    const onDelete = (item) =>
        modal.confirm({
            title: `Delete “${item.label}”?`,
            content: 'This cannot be undone.',
            okText: 'Delete',
            okButtonProps: { danger: true },
            onOk: async () => {
                try {
                    await request(item.deleteUrl, { method: 'DELETE' });
                    setItems((current) => current.filter((i) => i.id !== item.id));
                    message.success('Deleted');
                } catch (error) {
                    message.error(error.message);
                }
            },
        });

    if (items.length === 0) return <Empty description={emptyText} />;

    return (
        <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={onDragEnd}>
            <SortableContext items={items.map((i) => i.id)} strategy={verticalListSortingStrategy}>
                <div className="overflow-hidden rounded-xl border border-slate-200">
                    {items.map((item) => <Row key={item.id} item={item} onDelete={onDelete} />)}
                </div>
            </SortableContext>
        </DndContext>
    );
}
```

`resources/js/islands/TagsInput.jsx`:

```jsx
import { Select } from 'antd';
import { useState } from 'react';

/** List-of-strings field (highlights, tech, skills, marquee items) posted as name[]. */
export default function TagsInput({ name, value = [], placeholder = 'Type and press Enter' }) {
    const [tags, setTags] = useState(value);

    return (
        <>
            <Select mode="tags" value={tags} onChange={setTags} tokenSeparators={[',']} placeholder={placeholder}
                open={false} suffixIcon={null} style={{ width: '100%' }} />
            {tags.length > 0
                ? tags.map((tag, index) => <input key={`${index}-${tag}`} type="hidden" name={`${name}[]`} value={tag} />)
                : <input type="hidden" name={name} value="" />}
        </>
    );
}
```

`resources/js/islands/BlockEditor.jsx`:

```jsx
import { DeleteOutlined, HolderOutlined, PlusOutlined } from '@ant-design/icons';
import { closestCenter, DndContext, KeyboardSensor, PointerSensor, useSensor, useSensors } from '@dnd-kit/core';
import { arrayMove, SortableContext, sortableKeyboardCoordinates, useSortable, verticalListSortingStrategy } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';
import { Button, Card, Dropdown, Empty, Flex, Input, Select, Typography } from 'antd';
import { useMemo, useState } from 'react';
import MediaPicker from './MediaPicker';

const TYPES = { heading: 'Heading', paragraph: 'Paragraph', image: 'Image', quote: 'Quote', route: 'Route', tips: 'Tips' };
const BLANK = {
    heading: { text: '' },
    paragraph: { text: '' },
    image: { media_id: null, caption: '' },
    quote: { text: '' },
    route: { title: '', stages: [{ label: '', detail: '' }] },
    tips: { title: '', items: [] },
};

let sequence = 0;
const key = () => `block-${++sequence}`;

function Err({ message }) {
    return message ? <Typography.Text type="danger" style={{ fontSize: 12 }}>{message}</Typography.Text> : null;
}

function Fields({ block, onChange, error, previews, onMedia }) {
    switch (block.type) {
        case 'heading':
            return (
                <Flex vertical gap={6}>
                    <Input value={block.text} placeholder="Heading text" status={error('text') && 'error'} onChange={(e) => onChange({ text: e.target.value })} />
                    <Err message={error('text')} />
                    <Input value={block.id ?? ''} placeholder="Anchor id (optional, generated from the text)" onChange={(e) => onChange({ id: e.target.value })} />
                </Flex>
            );
        case 'paragraph':
        case 'quote':
            return (
                <Flex vertical gap={6}>
                    <Input.TextArea autoSize={{ minRows: 3 }} value={block.text} status={error('text') && 'error'}
                        placeholder={block.type === 'quote' ? 'Quote' : 'Paragraph'} onChange={(e) => onChange({ text: e.target.value })} />
                    <Err message={error('text')} />
                </Flex>
            );
        case 'image':
            return (
                <Flex vertical gap={6}>
                    <MediaPicker value={block.media_id} preview={previews[block.media_id] ?? null} onChange={onMedia} />
                    <Err message={error('media_id')} />
                    <Input value={block.caption ?? ''} placeholder="Caption (optional)" onChange={(e) => onChange({ caption: e.target.value })} />
                </Flex>
            );
        case 'route':
            return (
                <Flex vertical gap={8}>
                    <Input value={block.title} placeholder="Route title" status={error('title') && 'error'} onChange={(e) => onChange({ title: e.target.value })} />
                    <Err message={error('title') || error('stages')} />
                    {block.stages.map((stage, i) => (
                        <Flex key={i} gap={8} align="start">
                            <Input style={{ width: '35%' }} value={stage.label} placeholder="Stage (e.g. Kathmandu → Syabrubesi)"
                                onChange={(e) => onChange({ stages: block.stages.map((s, j) => (j === i ? { ...s, label: e.target.value } : s)) })} />
                            <Input.TextArea autoSize value={stage.detail} placeholder="Detail"
                                onChange={(e) => onChange({ stages: block.stages.map((s, j) => (j === i ? { ...s, detail: e.target.value } : s)) })} />
                            <Button icon={<DeleteOutlined />} aria-label="Remove stage" disabled={block.stages.length === 1}
                                onClick={() => onChange({ stages: block.stages.filter((_, j) => j !== i) })} />
                        </Flex>
                    ))}
                    <Button type="dashed" icon={<PlusOutlined />} onClick={() => onChange({ stages: [...block.stages, { label: '', detail: '' }] })}>Add stage</Button>
                </Flex>
            );
        case 'tips':
            return (
                <Flex vertical gap={6}>
                    <Input value={block.title} placeholder="Tips title" status={error('title') && 'error'} onChange={(e) => onChange({ title: e.target.value })} />
                    <Select mode="tags" value={block.items} open={false} suffixIcon={null} placeholder="Type a tip and press Enter"
                        onChange={(items) => onChange({ items })} />
                    <Err message={error('title') || error('items')} />
                </Flex>
            );
        default:
            return null;
    }
}

function SortableBlock({ block, index, errors, previews, onChange, onRemove, onMedia }) {
    const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id: block._key });
    const error = (field) => errors[`blocks.${index}.${field}`]?.[0];

    return (
        <div ref={setNodeRef} style={{ transform: CSS.Transform.toString(transform), transition, opacity: isDragging ? 0.6 : 1 }}>
            <Card size="small"
                title={<Flex gap={8} align="center"><span className="cursor-grab touch-none text-slate-400" {...attributes} {...listeners}><HolderOutlined /></span>{TYPES[block.type] ?? block.type}</Flex>}
                extra={<Button type="text" danger icon={<DeleteOutlined />} aria-label="Remove block" onClick={onRemove} />}>
                <Err message={error('type')} />
                <Fields block={block} onChange={onChange} error={error} previews={previews} onMedia={onMedia} />
            </Card>
        </div>
    );
}

/** Ordered article blocks, serialized into one hidden JSON input for the Blade form. */
export default function BlockEditor({ name = 'blocks', value = [], previews: initialPreviews = {}, errors = {} }) {
    const [blocks, setBlocks] = useState(() => value.map((block) => ({ ...block, _key: key() })));
    const [previews, setPreviews] = useState(initialPreviews);
    const sensors = useSensors(
        useSensor(PointerSensor, { activationConstraint: { distance: 4 } }),
        useSensor(KeyboardSensor, { coordinateGetter: sortableKeyboardCoordinates }),
    );

    const serialized = useMemo(() => JSON.stringify(blocks.map(({ _key, ...block }) => block)), [blocks]);
    const update = (blockKey, patch) => setBlocks((current) => current.map((b) => (b._key === blockKey ? { ...b, ...patch } : b)));

    const onDragEnd = ({ active, over }) => {
        if (!over || active.id === over.id) return;
        setBlocks((current) => arrayMove(current, current.findIndex((b) => b._key === active.id), current.findIndex((b) => b._key === over.id)));
    };

    return (
        <div>
            <input type="hidden" name={name} value={serialized} />
            {blocks.length === 0 && <Empty description="No content blocks yet" />}
            <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={onDragEnd}>
                <SortableContext items={blocks.map((b) => b._key)} strategy={verticalListSortingStrategy}>
                    <Flex vertical gap="middle">
                        {blocks.map((block, index) => (
                            <SortableBlock key={block._key} block={block} index={index} errors={errors} previews={previews}
                                onChange={(patch) => update(block._key, patch)}
                                onRemove={() => setBlocks((current) => current.filter((b) => b._key !== block._key))}
                                onMedia={(id, media) => {
                                    if (media) setPreviews((p) => ({ ...p, [id]: media }));
                                    update(block._key, { media_id: id });
                                }} />
                        ))}
                    </Flex>
                </SortableContext>
            </DndContext>
            <Dropdown menu={{ items: Object.entries(TYPES).map(([type, label]) => ({ key: type, label })),
                onClick: ({ key: type }) => setBlocks((current) => [...current, { type, ...structuredClone(BLANK[type]), _key: key() }]) }}>
                <Button icon={<PlusOutlined />} style={{ marginTop: 16 }}>Add block</Button>
            </Dropdown>
        </div>
    );
}
```

Replace the registry line in `resources/js/islands.jsx` so the file starts:

```jsx
import { App as AntApp, ConfigProvider } from 'antd';
import { createRoot } from 'react-dom/client';
import BlockEditor from './islands/BlockEditor';
import MediaLibrary from './islands/MediaLibrary';
import MediaPicker from './islands/MediaPicker';
import SortableList from './islands/SortableList';
import TagsInput from './islands/TagsInput';

// name used in <x-island name="..."> → React component
const registry = {
    'block-editor': BlockEditor,
    'media-library': MediaLibrary,
    'media-picker': MediaPicker,
    'sortable-list': SortableList,
    'tags-input': TagsInput,
};
```

(keep the rest of the file from Task 6 unchanged; `registerIslands` remains as an extension point).

- [ ] **Step 6: Build the assets**

Run: `npm run build`
Expected: `✓ built` with `public/build/manifest.json` listing `resources/css/app.css` and `resources/js/islands.jsx`. A chunk-size warning for antd is acceptable.

- [ ] **Step 7: Manual check**

Run `composer run dev`, sign in at `http://127.0.0.1:8000/admin/login` (create an account first with `php artisan admin:create`), open `/admin/media`, upload a JPEG, edit its alt text, and confirm delete works for an unused file. Check the browser console has no errors.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: antd islands for media, ordering, tags and blocks"
```

---

### Task 9: Admin CRUD for portfolio collections

**Files:**
- Create: `app/Admin/AdminResource.php`, `app/Admin/AdminResources.php`, `app/Admin/Resources/{Projects,Services,TimelineEntries,SkillGroups,CreativeItems,TravelPhotos,NavLinks,SocialLinks,TravelCategories}.php`, `app/Http/Controllers/Admin/ResourceController.php`, `resources/views/admin/resources/index.blade.php`, `resources/views/admin/resources/form.blade.php`, `resources/views/admin/partials/field.blade.php`
- Modify: `routes/web.php`
- Test: `tests/Feature/AdminResourcesTest.php`

**Interfaces:**
- Consumes: `Slug::unique`, `MediaResource`, `<x-island>` (`sortable-list`, `media-picker`, `tags-input`).
- Produces:
  - `AdminResource` static API: `key()`, `model()`, `title()`, `singular()`, `fields()`, `rules(?Model)`, `table()`, `label(Model)`, `sublabel(Model)`, `save(Model, array)`, `values(Model)`, `cleanList(mixed)`.
  - Field definition: `['name' => string, 'label' => string, 'type' => 'text'|'url'|'textarea'|'select'|'checkbox'|'media'|'tags', 'options'?: array<string,string>, 'hint'?: string, 'accept'?: 'image'|'any']`.
  - Routes `admin.resources.{index,create,store,edit,update,destroy}` at `/admin/{resource}` for keys `projects, services, timeline, skill-groups, creative-items, travel-photos, nav-links, social-links, travel-categories`.
  - Partial `admin.partials.field` with variables `$field`, `$value`, `$previews` (reused by Tasks 10–11).

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/AdminResourcesTest.php`:

```php
<?php

use App\Models\Project;
use App\Models\SkillGroup;
use App\Models\SocialLink;
use App\Models\TravelCategory;
use App\Models\User;

beforeEach(function () {
    $this->actingAs(User::factory()->create());
});

it('creates, lists, edits and updates a project', function () {
    $this->post('/admin/projects', [
        'title' => 'EZ-Bus', 'slug' => '', 'category' => 'Transport', 'description' => 'Bus booking',
        'highlights' => ['Seat selection', 'Routes'], 'tech' => ['Laravel'], 'href' => '', 'cover_media_id' => '', 'is_visible' => '1',
    ])->assertRedirect('/admin/projects');

    $project = Project::sole();
    expect($project->slug)->toBe('ez-bus')
        ->and($project->highlights)->toBe(['Seat selection', 'Routes'])
        ->and($project->href)->toBeNull();

    $this->get('/admin/projects')->assertOk()->assertSee('EZ-Bus')->assertSee('data-island="sortable-list"', false);
    $this->get("/admin/projects/{$project->id}/edit")->assertOk()->assertSee('Bus booking');

    $this->put("/admin/projects/{$project->id}", [
        'title' => 'EZ-Bus', 'slug' => 'ez-bus', 'category' => 'Transport', 'description' => 'Updated',
        'highlights' => '', 'tech' => ['Laravel', 'MySQL'], 'href' => 'https://ezbus.example', 'is_visible' => '0',
    ])->assertRedirect('/admin/projects');

    $project->refresh();
    expect($project->description)->toBe('Updated')
        ->and($project->highlights)->toBe([])
        ->and($project->tech)->toBe(['Laravel', 'MySQL'])
        ->and($project->is_visible)->toBeFalse();
});

it('validates required fields and urls', function () {
    $this->post('/admin/projects', ['title' => '', 'href' => 'not a url'])
        ->assertSessionHasErrors(['title', 'category', 'description', 'href']);
});

it('syncs skills from the tags field in order', function () {
    $this->post('/admin/skill-groups', ['title' => 'Development', 'items' => ['PHP', 'Laravel']])->assertRedirect();
    $group = SkillGroup::sole();

    $this->put("/admin/skill-groups/{$group->id}", ['title' => 'Development', 'items' => ['Laravel', 'React']])->assertRedirect();

    expect($group->fresh()->skills->pluck('name')->all())->toBe(['Laravel', 'React']);
});

it('allows social links without a url and slugs categories', function () {
    $this->post('/admin/social-links', ['label' => 'GitHub', 'url' => ''])->assertRedirect();
    $this->post('/admin/travel-categories', ['name' => 'Road Trips'])->assertRedirect();

    expect(SocialLink::sole()->url)->toBeNull()
        ->and(TravelCategory::sole()->slug)->toBe('road-trips');
});

it('deletes via JSON for the sortable list and 404s unknown resources', function () {
    $project = Project::create(['slug' => 'p', 'title' => 'P', 'category' => 'c', 'description' => 'd']);

    $this->deleteJson("/admin/projects/{$project->id}")->assertNoContent();
    expect(Project::count())->toBe(0);

    $this->get('/admin/unknown')->assertNotFound();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter=AdminResourcesTest`
Expected: FAIL (404s).

- [ ] **Step 3: Write the resource framework**

`app/Admin/AdminResource.php`:

```php
<?php

namespace App\Admin;

use Illuminate\Database\Eloquent\Model;

/**
 * A simple, ordered admin collection rendered by ResourceController.
 */
abstract class AdminResource
{
    /** URL segment, e.g. "projects". */
    abstract public static function key(): string;

    /** @return class-string<Model> */
    abstract public static function model(): string;

    abstract public static function title(): string;

    abstract public static function singular(): string;

    /**
     * @return list<array{name: string, label: string, type: string, options?: array<string, string>, hint?: string, accept?: string}>
     */
    abstract public static function fields(): array;

    /** @return array<string, mixed> */
    abstract public static function rules(?Model $record): array;

    public static function table(): string
    {
        $model = static::model();

        return (new $model)->getTable();
    }

    public static function label(Model $record): string
    {
        return (string) $record->getAttribute(static::fields()[0]['name']);
    }

    public static function sublabel(Model $record): ?string
    {
        return null;
    }

    /**
     * @param  array<string, mixed>  $data
     */
    public static function save(Model $record, array $data): void
    {
        $record->fill($data)->save();
    }

    /** @return array<string, mixed> */
    public static function values(Model $record): array
    {
        return collect(static::fields())->mapWithKeys(fn (array $field) => [$field['name'] => $record->getAttribute($field['name'])])->all();
    }

    /** @return list<string> */
    public static function cleanList(mixed $value): array
    {
        return array_values(array_filter(array_map(fn ($item) => trim((string) $item), (array) ($value ?? [])), fn (string $item) => $item !== ''));
    }
}
```

`app/Admin/AdminResources.php`:

```php
<?php

namespace App\Admin;

use App\Admin\Resources\CreativeItems;
use App\Admin\Resources\NavLinks;
use App\Admin\Resources\Projects;
use App\Admin\Resources\Services;
use App\Admin\Resources\SkillGroups;
use App\Admin\Resources\SocialLinks;
use App\Admin\Resources\TimelineEntries;
use App\Admin\Resources\TravelCategories;
use App\Admin\Resources\TravelPhotos;

final class AdminResources
{
    /** @var list<class-string<AdminResource>> */
    public const ALL = [
        Projects::class, Services::class, TimelineEntries::class, SkillGroups::class, CreativeItems::class,
        TravelPhotos::class, NavLinks::class, SocialLinks::class, TravelCategories::class,
    ];

    /** @return list<string> */
    public static function keys(): array
    {
        return array_map(fn (string $class) => $class::key(), self::ALL);
    }

    /** @return class-string<AdminResource> */
    public static function find(string $key): string
    {
        foreach (self::ALL as $class) {
            if ($class::key() === $key) {
                return $class;
            }
        }

        abort(404);
    }
}
```

- [ ] **Step 4: Write the nine resource definitions**

`app/Admin/Resources/Projects.php`:

```php
<?php

namespace App\Admin\Resources;

use App\Admin\AdminResource;
use App\Models\Project;
use App\Support\Slug;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Validation\Rule;

class Projects extends AdminResource
{
    public static function key(): string { return 'projects'; }
    public static function model(): string { return Project::class; }
    public static function title(): string { return 'Projects'; }
    public static function singular(): string { return 'Project'; }

    public static function fields(): array
    {
        return [
            ['name' => 'title', 'label' => 'Title', 'type' => 'text'],
            ['name' => 'slug', 'label' => 'Slug', 'type' => 'text', 'hint' => 'Leave empty to generate from the title.'],
            ['name' => 'category', 'label' => 'Category', 'type' => 'text', 'hint' => 'Shown above the title, e.g. TRANSPORTATION & BOOKING.'],
            ['name' => 'description', 'label' => 'Description', 'type' => 'textarea'],
            ['name' => 'highlights', 'label' => 'Highlights', 'type' => 'tags'],
            ['name' => 'tech', 'label' => 'Technologies', 'type' => 'tags'],
            ['name' => 'href', 'label' => 'Project URL', 'type' => 'url', 'hint' => 'Only when a real URL exists.'],
            ['name' => 'cover_media_id', 'label' => 'Cover image', 'type' => 'media'],
            ['name' => 'is_visible', 'label' => 'Show on the site', 'type' => 'checkbox'],
        ];
    }

    public static function rules(?Model $record): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'slug' => ['nullable', 'string', 'max:255', 'alpha_dash', Rule::unique('projects', 'slug')->ignore($record?->getKey())],
            'category' => ['required', 'string', 'max:255'],
            'description' => ['required', 'string', 'max:5000'],
            'highlights' => ['nullable', 'array', 'max:30'],
            'highlights.*' => ['string', 'max:255'],
            'tech' => ['nullable', 'array', 'max:30'],
            'tech.*' => ['string', 'max:100'],
            'href' => ['nullable', 'url:http,https', 'max:500'],
            'cover_media_id' => ['nullable', 'integer', Rule::exists('media', 'id')->whereNull('deleted_at')],
            'is_visible' => ['boolean'],
        ];
    }

    public static function sublabel(Model $record): ?string
    {
        return $record->category.($record->is_visible ? '' : ' · hidden');
    }

    public static function save(Model $record, array $data): void
    {
        $data['slug'] = Slug::unique(Project::query(), ($data['slug'] ?? null) ?: $data['title'], $record->getKey());
        $data['highlights'] = static::cleanList($data['highlights'] ?? null);
        $data['tech'] = static::cleanList($data['tech'] ?? null);
        $record->fill($data)->save();
    }
}
```

`app/Admin/Resources/Services.php`:

```php
<?php

namespace App\Admin\Resources;

use App\Admin\AdminResource;
use App\Models\Service;
use Illuminate\Database\Eloquent\Model;

class Services extends AdminResource
{
    public static function key(): string { return 'services'; }
    public static function model(): string { return Service::class; }
    public static function title(): string { return 'Services'; }
    public static function singular(): string { return 'Service'; }

    public static function fields(): array
    {
        return [
            ['name' => 'title', 'label' => 'Title', 'type' => 'text'],
            ['name' => 'body', 'label' => 'Description', 'type' => 'textarea'],
            ['name' => 'tech', 'label' => 'Tools line', 'type' => 'text', 'hint' => 'e.g. React · Next.js · Laravel'],
            ['name' => 'cta_label', 'label' => 'Button label', 'type' => 'text'],
            ['name' => 'cta_href', 'label' => 'Button link', 'type' => 'text', 'hint' => 'A site path like /work or a full URL.'],
        ];
    }

    public static function rules(?Model $record): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'body' => ['required', 'string', 'max:2000'],
            'tech' => ['nullable', 'string', 'max:255'],
            'cta_label' => ['nullable', 'string', 'max:255'],
            'cta_href' => ['nullable', 'string', 'max:500'],
        ];
    }
}
```

`app/Admin/Resources/TimelineEntries.php`:

```php
<?php

namespace App\Admin\Resources;

use App\Admin\AdminResource;
use App\Models\TimelineEntry;
use Illuminate\Database\Eloquent\Model;

class TimelineEntries extends AdminResource
{
    public static function key(): string { return 'timeline'; }
    public static function model(): string { return TimelineEntry::class; }
    public static function title(): string { return 'Timeline'; }
    public static function singular(): string { return 'Timeline entry'; }

    public static function fields(): array
    {
        return [
            ['name' => 'title', 'label' => 'Title', 'type' => 'text'],
            ['name' => 'detail', 'label' => 'Detail', 'type' => 'textarea'],
        ];
    }

    public static function rules(?Model $record): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'detail' => ['required', 'string', 'max:2000'],
        ];
    }
}
```

`app/Admin/Resources/SkillGroups.php`:

```php
<?php

namespace App\Admin\Resources;

use App\Admin\AdminResource;
use App\Models\SkillGroup;
use Illuminate\Database\Eloquent\Model;

class SkillGroups extends AdminResource
{
    public static function key(): string { return 'skill-groups'; }
    public static function model(): string { return SkillGroup::class; }
    public static function title(): string { return 'Skills'; }
    public static function singular(): string { return 'Skill group'; }

    public static function fields(): array
    {
        return [
            ['name' => 'title', 'label' => 'Group title', 'type' => 'text'],
            ['name' => 'items', 'label' => 'Skills', 'type' => 'tags', 'hint' => 'In display order.'],
        ];
    }

    public static function rules(?Model $record): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'items' => ['nullable', 'array', 'max:100'],
            'items.*' => ['string', 'max:100'],
        ];
    }

    public static function sublabel(Model $record): ?string
    {
        return $record->skills->pluck('name')->implode(', ');
    }

    public static function values(Model $record): array
    {
        return ['title' => $record->title, 'items' => $record->exists ? $record->skills->pluck('name')->all() : []];
    }

    public static function save(Model $record, array $data): void
    {
        $record->fill(['title' => $data['title']])->save();
        $names = array_values(array_unique(static::cleanList($data['items'] ?? null)));

        $record->skills()->whereNotIn('name', $names)->delete();
        foreach ($names as $index => $name) {
            $record->skills()->updateOrCreate(['name' => $name], ['sort_order' => $index + 1]);
        }
    }
}
```

`app/Admin/Resources/CreativeItems.php`:

```php
<?php

namespace App\Admin\Resources;

use App\Admin\AdminResource;
use App\Models\CreativeItem;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Validation\Rule;

class CreativeItems extends AdminResource
{
    public static function key(): string { return 'creative-items'; }
    public static function model(): string { return CreativeItem::class; }
    public static function title(): string { return 'Creative work'; }
    public static function singular(): string { return 'Creative item'; }

    public static function fields(): array
    {
        return [
            ['name' => 'title', 'label' => 'Title', 'type' => 'text'],
            ['name' => 'key', 'label' => 'Key', 'type' => 'text', 'hint' => 'Stable id used by the frontend, e.g. graphic.'],
            ['name' => 'discipline', 'label' => 'Discipline', 'type' => 'text'],
            ['name' => 'shape', 'label' => 'Gallery shape', 'type' => 'select', 'options' => CreativeItem::SHAPES],
            ['name' => 'media_id', 'label' => 'Image', 'type' => 'media'],
        ];
    }

    public static function rules(?Model $record): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'key' => ['required', 'string', 'max:100', 'alpha_dash', Rule::unique('creative_items', 'key')->ignore($record?->getKey())],
            'discipline' => ['required', 'string', 'max:255'],
            'shape' => ['required', Rule::in(array_keys(CreativeItem::SHAPES))],
            'media_id' => ['nullable', 'integer', Rule::exists('media', 'id')->whereNull('deleted_at')],
        ];
    }

    public static function sublabel(Model $record): ?string
    {
        return "{$record->discipline} · {$record->shape}";
    }
}
```

`app/Admin/Resources/TravelPhotos.php`:

```php
<?php

namespace App\Admin\Resources;

use App\Admin\AdminResource;
use App\Models\TravelPhoto;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Validation\Rule;

class TravelPhotos extends AdminResource
{
    public static function key(): string { return 'travel-photos'; }
    public static function model(): string { return TravelPhoto::class; }
    public static function title(): string { return 'Travel photos'; }
    public static function singular(): string { return 'Travel photo'; }

    public static function fields(): array
    {
        return [
            ['name' => 'theme', 'label' => 'Theme', 'type' => 'text'],
            ['name' => 'key', 'label' => 'Key', 'type' => 'text', 'hint' => 'Stable id used by the frontend.'],
            ['name' => 'aspect', 'label' => 'Aspect', 'type' => 'select', 'options' => TravelPhoto::ASPECTS],
            ['name' => 'media_id', 'label' => 'Photo', 'type' => 'media'],
        ];
    }

    public static function rules(?Model $record): array
    {
        return [
            'theme' => ['required', 'string', 'max:255'],
            'key' => ['required', 'string', 'max:100', 'alpha_dash', Rule::unique('travel_photos', 'key')->ignore($record?->getKey())],
            'aspect' => ['required', Rule::in(array_keys(TravelPhoto::ASPECTS))],
            'media_id' => ['nullable', 'integer', Rule::exists('media', 'id')->whereNull('deleted_at')],
        ];
    }
}
```

`app/Admin/Resources/NavLinks.php`:

```php
<?php

namespace App\Admin\Resources;

use App\Admin\AdminResource;
use App\Models\NavLink;
use Illuminate\Database\Eloquent\Model;

class NavLinks extends AdminResource
{
    public static function key(): string { return 'nav-links'; }
    public static function model(): string { return NavLink::class; }
    public static function title(): string { return 'Navigation'; }
    public static function singular(): string { return 'Navigation link'; }

    public static function fields(): array
    {
        return [
            ['name' => 'label', 'label' => 'Label', 'type' => 'text'],
            ['name' => 'href', 'label' => 'Link', 'type' => 'text', 'hint' => 'A site path like /travel or a full URL.'],
            ['name' => 'is_visible', 'label' => 'Show in the menu', 'type' => 'checkbox'],
        ];
    }

    public static function rules(?Model $record): array
    {
        return [
            'label' => ['required', 'string', 'max:255'],
            'href' => ['required', 'string', 'max:500'],
            'is_visible' => ['boolean'],
        ];
    }

    public static function sublabel(Model $record): ?string
    {
        return $record->href.($record->is_visible ? '' : ' · hidden');
    }
}
```

`app/Admin/Resources/SocialLinks.php`:

```php
<?php

namespace App\Admin\Resources;

use App\Admin\AdminResource;
use App\Models\SocialLink;
use Illuminate\Database\Eloquent\Model;

class SocialLinks extends AdminResource
{
    public static function key(): string { return 'social-links'; }
    public static function model(): string { return SocialLink::class; }
    public static function title(): string { return 'Social links'; }
    public static function singular(): string { return 'Social link'; }

    public static function fields(): array
    {
        return [
            ['name' => 'label', 'label' => 'Label', 'type' => 'text'],
            ['name' => 'url', 'label' => 'URL', 'type' => 'url', 'hint' => 'Leave empty to show “coming soon”.'],
        ];
    }

    public static function rules(?Model $record): array
    {
        return [
            'label' => ['required', 'string', 'max:255'],
            'url' => ['nullable', 'url:http,https', 'max:500'],
        ];
    }

    public static function sublabel(Model $record): ?string
    {
        return $record->url ?? 'coming soon';
    }
}
```

`app/Admin/Resources/TravelCategories.php`:

```php
<?php

namespace App\Admin\Resources;

use App\Admin\AdminResource;
use App\Models\TravelCategory;
use App\Support\Slug;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Validation\Rule;

class TravelCategories extends AdminResource
{
    public static function key(): string { return 'travel-categories'; }
    public static function model(): string { return TravelCategory::class; }
    public static function title(): string { return 'Travel categories'; }
    public static function singular(): string { return 'Travel category'; }

    public static function fields(): array
    {
        return [['name' => 'name', 'label' => 'Name', 'type' => 'text', 'hint' => 'Also used as a filter on the travel page.']];
    }

    public static function rules(?Model $record): array
    {
        return ['name' => ['required', 'string', 'max:100', Rule::unique('travel_categories', 'name')->ignore($record?->getKey())]];
    }

    public static function save(Model $record, array $data): void
    {
        $record->fill([
            'name' => $data['name'],
            'slug' => Slug::unique(TravelCategory::query(), $data['name'], $record->getKey()),
        ])->save();
    }
}
```

- [ ] **Step 5: Write the controller and routes**

`app/Http/Controllers/Admin/ResourceController.php`:

```php
<?php

namespace App\Http\Controllers\Admin;

use App\Admin\AdminResource;
use App\Admin\AdminResources;
use App\Http\Controllers\Controller;
use App\Http\Resources\Admin\MediaResource;
use App\Models\Media;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Support\Facades\DB;
use Illuminate\View\View;

class ResourceController extends Controller
{
    public function index(string $resource): View
    {
        $definition = AdminResources::find($resource);
        $model = $definition::model();

        $items = $model::query()->ordered()->get()->map(fn (Model $record) => [
            'id' => $record->getKey(),
            'label' => $definition::label($record),
            'sublabel' => $definition::sublabel($record),
            'editUrl' => route('admin.resources.edit', [$resource, $record->getKey()]),
            'deleteUrl' => route('admin.resources.destroy', [$resource, $record->getKey()]),
        ])->all();

        return view('admin.resources.index', ['definition' => $definition, 'items' => $items]);
    }

    public function create(string $resource): View
    {
        $definition = AdminResources::find($resource);
        $model = $definition::model();

        return $this->form($definition, new $model);
    }

    public function store(Request $request, string $resource): RedirectResponse
    {
        $definition = AdminResources::find($resource);
        $model = $definition::model();
        $record = new $model;
        $data = $request->validate($definition::rules(null));

        DB::transaction(fn () => $definition::save($record, $data));

        return redirect()->route('admin.resources.index', $resource)->with('status', $definition::singular().' created.');
    }

    public function edit(string $resource, int $id): View
    {
        $definition = AdminResources::find($resource);

        return $this->form($definition, $definition::model()::query()->findOrFail($id));
    }

    public function update(Request $request, string $resource, int $id): RedirectResponse
    {
        $definition = AdminResources::find($resource);
        $record = $definition::model()::query()->findOrFail($id);
        $data = $request->validate($definition::rules($record));

        DB::transaction(fn () => $definition::save($record, $data));

        return redirect()->route('admin.resources.index', $resource)->with('status', $definition::singular().' saved.');
    }

    public function destroy(Request $request, string $resource, int $id): RedirectResponse|Response
    {
        $definition = AdminResources::find($resource);
        $definition::model()::query()->findOrFail($id)->delete();

        return $request->expectsJson()
            ? response()->noContent()
            : redirect()->route('admin.resources.index', $resource)->with('status', $definition::singular().' deleted.');
    }

    /**
     * @param  class-string<AdminResource>  $definition
     */
    private function form(string $definition, Model $record): View
    {
        $values = $definition::values($record);
        $mediaFields = collect($definition::fields())->where('type', 'media')->pluck('name');
        $ids = $mediaFields->map(fn (string $name) => (int) old($name, $values[$name] ?? 0))->filter();
        $media = Media::query()->whereIn('id', $ids)->get()->keyBy('id');

        return view('admin.resources.form', [
            'definition' => $definition,
            'record' => $record,
            'values' => $values,
            'previews' => $mediaFields->mapWithKeys(function (string $name) use ($media, $values) {
                $found = $media->get((int) old($name, $values[$name] ?? 0));

                return [$name => $found ? (new MediaResource($found))->resolve() : null];
            })->all(),
        ]);
    }
}
```

In `routes/web.php` add `use App\Admin\AdminResources;` and `use App\Http\Controllers\Admin\ResourceController;`, then inside the `auth` group add (keep it last in the group so explicit routes win):

```php
        $resourceKeys = AdminResources::keys();

        Route::controller(ResourceController::class)->prefix('{resource}')->name('resources.')
            ->group(function () use ($resourceKeys) {
                Route::get('/', 'index')->name('index')->whereIn('resource', $resourceKeys);
                Route::get('create', 'create')->name('create')->whereIn('resource', $resourceKeys);
                Route::post('/', 'store')->name('store')->whereIn('resource', $resourceKeys);
                Route::get('{id}/edit', 'edit')->name('edit')->whereIn('resource', $resourceKeys)->whereNumber('id');
                Route::put('{id}', 'update')->name('update')->whereIn('resource', $resourceKeys)->whereNumber('id');
                Route::delete('{id}', 'destroy')->name('destroy')->whereIn('resource', $resourceKeys)->whereNumber('id');
            });
```

- [ ] **Step 6: Write the views**

`resources/views/admin/partials/field.blade.php`:

```blade
@php
    $name = $field['name'];
    $type = $field['type'];
    $current = old($name, $value);
    $error = $errors->first($name) ?: $errors->first($name.'.*');
@endphp
<div class="space-y-1.5">
    @if ($type === 'checkbox')
        <input type="hidden" name="{{ $name }}" value="0">
        <label class="inline-flex items-center gap-2 text-sm font-medium text-slate-700">
            <input type="checkbox" name="{{ $name }}" value="1" class="h-4 w-4 rounded border-slate-300" @checked($current)>
            {{ $field['label'] }}
        </label>
    @else
        <label for="field-{{ $name }}" class="block text-sm font-medium text-slate-700">{{ $field['label'] }}</label>
        @switch($type)
            @case('textarea')
                <textarea id="field-{{ $name }}" name="{{ $name }}" rows="4" class="input">{{ $current }}</textarea>
                @break
            @case('select')
                <select id="field-{{ $name }}" name="{{ $name }}" class="input">
                    @foreach ($field['options'] as $optionValue => $optionLabel)
                        <option value="{{ $optionValue }}" @selected((string) $current === (string) $optionValue)>{{ $optionLabel }}</option>
                    @endforeach
                </select>
                @break
            @case('media')
                <x-island name="media-picker" :props="[
                    'name' => $name,
                    'value' => $current ? (int) $current : null,
                    'preview' => $previews[$name] ?? null,
                    'accept' => $field['accept'] ?? 'image',
                ]" />
                @break
            @case('tags')
                <x-island name="tags-input" :props="['name' => $name, 'value' => array_values(array_filter((array) ($current ?? [])))]" />
                @break
            @default
                <input id="field-{{ $name }}" type="{{ $type === 'url' ? 'url' : 'text' }}" name="{{ $name }}" value="{{ $current }}" class="input">
        @endswitch
    @endif

    @if ($error)
        <p class="text-xs text-red-600">{{ $error }}</p>
    @elseif (! empty($field['hint']))
        <p class="text-xs text-slate-500">{{ $field['hint'] }}</p>
    @endif
</div>
```

`resources/views/admin/resources/index.blade.php`:

```blade
<x-admin.layout :title="$definition::title()">
    <div class="mb-6 flex flex-wrap items-end justify-between gap-4">
        <div>
            <h1 class="text-2xl font-semibold tracking-tight">{{ $definition::title() }}</h1>
            <p class="mt-1 text-sm text-slate-500">Drag rows to change the order on the site. It saves automatically.</p>
        </div>
        <a href="{{ route('admin.resources.create', $definition::key()) }}" class="btn-primary">Add {{ strtolower($definition::singular()) }}</a>
    </div>

    <x-island name="sortable-list" :props="['resource' => $definition::table(), 'items' => $items]" />
</x-admin.layout>
```

`resources/views/admin/resources/form.blade.php`:

```blade
@php
    $heading = ($record->exists ? 'Edit ' : 'New ').strtolower($definition::singular());
    $action = $record->exists
        ? route('admin.resources.update', [$definition::key(), $record->getKey()])
        : route('admin.resources.store', $definition::key());
@endphp
<x-admin.layout :title="$heading">
    <a href="{{ route('admin.resources.index', $definition::key()) }}" class="text-sm text-slate-500 hover:text-slate-800">← {{ $definition::title() }}</a>
    <h1 class="mt-2 mb-6 text-2xl font-semibold tracking-tight">{{ $heading }}</h1>

    <form method="POST" action="{{ $action }}" class="max-w-3xl space-y-6 rounded-xl border border-slate-200 bg-white p-6 shadow-sm">
        @csrf
        @if ($record->exists)
            @method('PUT')
        @endif

        @foreach ($definition::fields() as $field)
            @include('admin.partials.field', ['field' => $field, 'value' => $values[$field['name']] ?? null, 'previews' => $previews])
        @endforeach

        <div class="flex gap-3 border-t border-slate-100 pt-6">
            <button type="submit" class="btn-primary">Save</button>
            <a href="{{ route('admin.resources.index', $definition::key()) }}" class="btn-secondary">Cancel</a>
        </div>
    </form>
</x-admin.layout>
```

- [ ] **Step 7: Run tests**

Run: `php artisan test --filter=AdminResourcesTest`
Expected: PASS (5 tests). If the `is_visible` update assertion fails, check the checkbox field renders the hidden `0` input before the checkbox.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: admin CRUD for portfolio collections"
```

---

### Task 10: Travel stories and notes editor

**Files:**
- Create: `app/Http/Requests/Admin/ArticleRequest.php`, `app/Http/Controllers/Admin/ArticleController.php`, `resources/views/admin/articles/index.blade.php`, `resources/views/admin/articles/form.blade.php`
- Modify: `routes/web.php`
- Test: `tests/Feature/AdminArticlesTest.php`

**Interfaces:**
- Consumes: `ArticleBlocks::validate/normalize/mediaIds`, `Slug::unique`, `MediaResource`, `admin.partials.field`, islands `block-editor`, `media-picker`, `sortable-list`.
- Produces: routes `admin.travel.{index,create,store,edit,update,destroy}` (`/admin/travel...`) and `admin.notes.*` (`/admin/notes...`), each with route default `type` = `travel` | `note`; `ArticleRequest::blocks(): array` (normalized).

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/AdminArticlesTest.php`:

```php
<?php

use App\Enums\ArticleStatus;
use App\Enums\ArticleType;
use App\Models\Article;
use App\Models\Media;
use App\Models\TravelCategory;
use App\Models\User;

beforeEach(function () {
    $this->actingAs(User::factory()->create());
});

function storyPayload(array $overrides = []): array
{
    return [
        'title' => 'Langtang Valley: Into the Mountains', 'slug' => '', 'subtitle' => '', 'excerpt' => 'A trek north.',
        'category' => 'Trekking', 'destination' => 'Langtang Valley, Nepal', 'status' => 'draft', 'featured' => '0',
        'published_at' => '', 'reading_time' => '', 'cover_media_id' => '',
        'blocks' => json_encode([
            ['type' => 'heading', 'text' => 'Overview'],
            ['type' => 'route', 'title' => 'Route', 'stages' => [['label' => 'Kathmandu → Syabrubesi', 'detail' => 'Bus']]],
        ]),
        ...$overrides,
    ];
}

it('creates a travel story with categories and normalized blocks', function () {
    $category = TravelCategory::create(['name' => 'Trekking', 'slug' => 'trekking']);

    $this->post('/admin/travel', storyPayload(['travel_category_ids' => [$category->id]]))
        ->assertRedirect('/admin/travel');

    $story = Article::sole();
    expect($story->type)->toBe(ArticleType::Travel)
        ->and($story->slug)->toBe('langtang-valley-into-the-mountains')
        ->and($story->blocks[0])->toBe(['type' => 'heading', 'id' => 'overview', 'text' => 'Overview'])
        ->and($story->travelCategories->pluck('name')->all())->toBe(['Trekking'])
        ->and($story->published_at)->toBeNull();
});

it('reports block errors against their position', function () {
    $this->post('/admin/travel', storyPayload(['blocks' => json_encode([['type' => 'paragraph', 'text' => '']])]))
        ->assertSessionHasErrors('blocks.0.text');

    $this->post('/admin/travel', storyPayload(['blocks' => 'not json']))->assertSessionHasErrors('blocks');
    expect(Article::count())->toBe(0);
});

it('stamps the publication date when publishing', function () {
    $this->post('/admin/travel', storyPayload(['status' => 'published']))->assertRedirect();

    expect(Article::sole()->status)->toBe(ArticleStatus::Published)
        ->and(Article::sole()->published_at)->not->toBeNull();
});

it('requires a topic for notes and allows the same slug across types', function () {
    $this->post('/admin/travel', storyPayload(['slug' => 'shared']))->assertRedirect();

    $note = storyPayload(['slug' => 'shared', 'topic' => 'Engineering']);
    unset($note['destination'], $note['featured']);

    $this->post('/admin/notes', [...$note, 'topic' => ''])->assertSessionHasErrors('topic');
    $this->post('/admin/notes', $note)->assertRedirect('/admin/notes');

    expect(Article::where('slug', 'shared')->count())->toBe(2);
});

it('edits a story and keeps media previews for image blocks', function () {
    $photo = Media::factory()->create(['original_name' => 'trail.jpg']);
    $story = Article::create([
        'type' => ArticleType::Travel, 'slug' => 's', 'title' => 'Story', 'excerpt' => 'e', 'category' => 'Trekking',
        'destination' => 'Nepal', 'status' => ArticleStatus::Draft, 'blocks' => [['type' => 'image', 'media_id' => $photo->id]],
    ]);

    $this->get("/admin/travel/{$story->id}/edit")->assertOk()
        ->assertSee('data-island="block-editor"', false)
        ->assertSee('trail.jpg');

    $this->get("/admin/notes/{$story->id}/edit")->assertNotFound();

    $this->put("/admin/travel/{$story->id}", storyPayload(['title' => 'Story', 'slug' => 's', 'status' => 'hidden']))
        ->assertRedirect('/admin/travel');
    expect($story->fresh()->status)->toBe(ArticleStatus::Hidden);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter=AdminArticlesTest`
Expected: FAIL (404).

- [ ] **Step 3: Implement the request**

`app/Http/Requests/Admin/ArticleRequest.php`:

```php
<?php

namespace App\Http\Requests\Admin;

use App\Enums\ArticleStatus;
use App\Enums\ArticleType;
use App\Models\Article;
use App\Support\ArticleBlocks;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Validator;

class ArticleRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function articleType(): ArticleType
    {
        return ArticleType::from((string) $this->route('type'));
    }

    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        $type = $this->articleType();
        $article = $this->route('article');

        return [
            'title' => ['required', 'string', 'max:255'],
            'slug' => ['nullable', 'string', 'max:255', 'alpha_dash',
                Rule::unique('articles', 'slug')->where('type', $type->value)->ignore($article instanceof Article ? $article->id : null)],
            'subtitle' => ['nullable', 'string', 'max:255'],
            'excerpt' => ['required', 'string', 'max:1000'],
            'category' => ['required', 'string', 'max:100'],
            'destination' => $type === ArticleType::Travel ? ['required', 'string', 'max:255'] : ['prohibited'],
            'topic' => $type === ArticleType::Note ? ['required', 'string', 'max:100'] : ['prohibited'],
            'featured' => ['boolean'],
            'status' => ['required', Rule::enum(ArticleStatus::class)],
            'published_at' => ['nullable', 'date'],
            'reading_time' => ['nullable', 'string', 'max:50'],
            'cover_media_id' => ['nullable', 'integer', Rule::exists('media', 'id')->whereNull('deleted_at')],
            'travel_category_ids' => ['nullable', 'array'],
            'travel_category_ids.*' => ['integer', 'exists:travel_categories,id'],
            'blocks' => ['required', 'string'],
        ];
    }

    /**
     * @return list<callable>
     */
    public function after(): array
    {
        return [
            function (Validator $validator) {
                $decoded = json_decode((string) $this->input('blocks'), true);

                if (! is_array($decoded)) {
                    $validator->errors()->add('blocks', 'The content blocks could not be read. Reload the page and try again.');

                    return;
                }

                foreach (ArticleBlocks::validate($decoded) as $key => $messages) {
                    foreach ($messages as $message) {
                        $validator->errors()->add($key, $message);
                    }
                }
            },
        ];
    }

    /**
     * @return list<array<string, mixed>>
     */
    public function blocks(): array
    {
        return ArticleBlocks::normalize(json_decode((string) $this->input('blocks'), true) ?? []);
    }
}
```

- [ ] **Step 4: Implement the controller and routes**

`app/Http/Controllers/Admin/ArticleController.php`:

```php
<?php

namespace App\Http\Controllers\Admin;

use App\Enums\ArticleStatus;
use App\Enums\ArticleType;
use App\Http\Controllers\Controller;
use App\Http\Requests\Admin\ArticleRequest;
use App\Http\Resources\Admin\MediaResource;
use App\Models\Article;
use App\Models\Media;
use App\Models\TravelCategory;
use App\Support\ArticleBlocks;
use App\Support\Slug;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Support\Facades\DB;
use Illuminate\View\View;

class ArticleController extends Controller
{
    public function index(Request $request): View
    {
        $type = $this->type($request);

        $items = Article::query()->where('type', $type->value)->ordered()->get()->map(fn (Article $a) => [
            'id' => $a->id,
            'label' => $a->title,
            'sublabel' => ucfirst($a->status->value).' · '.$a->category,
            'editUrl' => route("admin.{$type->segment()}.edit", $a),
            'deleteUrl' => route("admin.{$type->segment()}.destroy", $a),
        ])->all();

        return view('admin.articles.index', ['type' => $type, 'items' => $items]);
    }

    public function create(Request $request): View
    {
        return $this->form($this->type($request), new Article(['status' => ArticleStatus::Draft, 'blocks' => []]));
    }

    public function store(ArticleRequest $request): RedirectResponse
    {
        $type = $request->articleType();
        $this->save($request, new Article(['type' => $type]));

        return redirect()->route("admin.{$type->segment()}.index")->with('status', 'Saved.');
    }

    public function edit(Request $request, Article $article): View
    {
        $type = $this->type($request);
        abort_unless($article->type === $type, 404);

        return $this->form($type, $article);
    }

    public function update(ArticleRequest $request, Article $article): RedirectResponse
    {
        $type = $request->articleType();
        abort_unless($article->type === $type, 404);
        $this->save($request, $article);

        return redirect()->route("admin.{$type->segment()}.index")->with('status', 'Saved.');
    }

    public function destroy(Request $request, Article $article): RedirectResponse|Response
    {
        $type = $this->type($request);
        abort_unless($article->type === $type, 404);
        $article->delete();

        return $request->expectsJson()
            ? response()->noContent()
            : redirect()->route("admin.{$type->segment()}.index")->with('status', 'Deleted.');
    }

    private function save(ArticleRequest $request, Article $article): void
    {
        $type = $request->articleType();

        DB::transaction(function () use ($request, $article, $type) {
            $article->fill($request->safe()->except(['slug', 'blocks', 'travel_category_ids']));
            $article->type = $type;
            $article->slug = Slug::unique(Article::query()->where('type', $type->value), $request->input('slug') ?: $request->input('title'), $article->id);
            $article->blocks = $request->blocks();

            if ($article->status === ArticleStatus::Published && $article->published_at === null) {
                $article->published_at = now();
            }

            $article->save();

            if ($type === ArticleType::Travel) {
                $article->travelCategories()->sync($request->input('travel_category_ids', []));
            }
        });
    }

    private function form(ArticleType $type, Article $article): View
    {
        $oldBlocks = old('blocks');
        $blocks = $oldBlocks !== null ? (json_decode($oldBlocks, true) ?: []) : ($article->blocks ?? []);
        $mediaIds = [...ArticleBlocks::mediaIds($blocks), (int) old('cover_media_id', $article->cover_media_id)];
        $media = Media::query()->whereIn('id', array_filter($mediaIds))->get()->keyBy('id');
        $errors = collect(session('errors')?->getBag('default')->getMessages() ?? [])
            ->filter(fn ($messages, string $key) => str_starts_with($key, 'blocks.'))->all();
        $cover = $media->get((int) old('cover_media_id', $article->cover_media_id));

        return view('admin.articles.form', [
            'type' => $type,
            'article' => $article,
            'categories' => $type === ArticleType::Travel ? TravelCategory::query()->ordered()->get() : collect(),
            'selectedCategories' => array_map('intval', old('travel_category_ids', $article->exists ? $article->travelCategories->pluck('id')->all() : [])),
            'coverPreview' => $cover ? (new MediaResource($cover))->resolve() : null,
            'editorProps' => [
                'name' => 'blocks',
                'value' => $blocks,
                'previews' => (object) $media->map(fn (Media $m) => (new MediaResource($m))->resolve())->all(),
                'errors' => (object) $errors,
            ],
        ]);
    }

    private function type(Request $request): ArticleType
    {
        return ArticleType::from((string) $request->route('type'));
    }
}
```

In `routes/web.php` add `use App\Enums\ArticleType;` and `use App\Http\Controllers\Admin\ArticleController;`, then inside the `auth` group, **before** the generic resource routes:

```php
        foreach (ArticleType::cases() as $articleType) {
            $segment = $articleType->segment();

            Route::controller(ArticleController::class)->prefix($segment)->name("{$segment}.")
                ->group(function () use ($articleType) {
                    Route::get('/', 'index')->name('index')->defaults('type', $articleType->value);
                    Route::get('create', 'create')->name('create')->defaults('type', $articleType->value);
                    Route::post('/', 'store')->name('store')->defaults('type', $articleType->value);
                    Route::get('{article}/edit', 'edit')->name('edit')->defaults('type', $articleType->value);
                    Route::put('{article}', 'update')->name('update')->defaults('type', $articleType->value);
                    Route::delete('{article}', 'destroy')->name('destroy')->defaults('type', $articleType->value);
                });
        }
```

- [ ] **Step 5: Write the views**

`resources/views/admin/articles/index.blade.php`:

```blade
<x-admin.layout :title="$type->label()">
    <div class="mb-6 flex flex-wrap items-end justify-between gap-4">
        <div>
            <h1 class="text-2xl font-semibold tracking-tight">{{ $type->label() }}</h1>
            <p class="mt-1 text-sm text-slate-500">Drafts appear on the site with a “Draft” tag; hidden entries are not shown at all.</p>
        </div>
        <a href="{{ route("admin.{$type->segment()}.create") }}" class="btn-primary">New {{ $type === \App\Enums\ArticleType::Travel ? 'story' : 'note' }}</a>
    </div>

    <x-island name="sortable-list" :props="['resource' => 'articles', 'items' => $items]" />
</x-admin.layout>
```

`resources/views/admin/articles/form.blade.php`:

```blade
@php
    use App\Enums\ArticleType;

    $isTravel = $type === ArticleType::Travel;
    $heading = $article->exists ? 'Edit '.($isTravel ? 'story' : 'note') : 'New '.($isTravel ? 'story' : 'note');
    $action = $article->exists ? route("admin.{$type->segment()}.update", $article) : route("admin.{$type->segment()}.store");
    $field = fn (string $name, string $label, string $fieldType = 'text', array $extra = []) => ['name' => $name, 'label' => $label, 'type' => $fieldType, ...$extra];
@endphp
<x-admin.layout :title="$heading">
    <a href="{{ route("admin.{$type->segment()}.index") }}" class="text-sm text-slate-500 hover:text-slate-800">← {{ $type->label() }}</a>
    <div class="mt-2 mb-6 flex flex-wrap items-center justify-between gap-4">
        <h1 class="text-2xl font-semibold tracking-tight">{{ $heading }}</h1>
        @if ($article->exists)
            <a href="{{ config('portfolio.frontend_url') }}/{{ $type->segment() }}/{{ $article->slug }}" target="_blank" rel="noreferrer" class="btn-secondary">Preview on site ↗</a>
        @endif
    </div>

    <form method="POST" action="{{ $action }}" class="grid gap-6 lg:grid-cols-[minmax(0,1fr)_20rem]">
        @csrf
        @if ($article->exists)
            @method('PUT')
        @endif

        <div class="min-w-0 space-y-6">
            <section class="space-y-5 rounded-xl border border-slate-200 bg-white p-6 shadow-sm">
                @include('admin.partials.field', ['field' => $field('title', 'Title'), 'value' => $article->title, 'previews' => []])
                @include('admin.partials.field', ['field' => $field('subtitle', 'Subtitle'), 'value' => $article->subtitle, 'previews' => []])
                @include('admin.partials.field', ['field' => $field('excerpt', 'Excerpt', 'textarea'), 'value' => $article->excerpt, 'previews' => []])
                <div class="grid gap-5 sm:grid-cols-2">
                    @include('admin.partials.field', ['field' => $field('category', 'Category', 'text', ['hint' => 'Label shown on cards, e.g. Trekking.']), 'value' => $article->category, 'previews' => []])
                    @if ($isTravel)
                        @include('admin.partials.field', ['field' => $field('destination', 'Destination'), 'value' => $article->destination, 'previews' => []])
                    @else
                        @include('admin.partials.field', ['field' => $field('topic', 'Topic', 'text', ['hint' => 'Short label in the notes list.']), 'value' => $article->topic, 'previews' => []])
                    @endif
                </div>
            </section>

            <section class="rounded-xl border border-slate-200 bg-white p-6 shadow-sm">
                <h2 class="mb-1 text-sm font-semibold">Content</h2>
                <p class="mb-4 text-xs text-slate-500">Headings, paragraphs, images, quotes, route outlines and tips. Drag cards to reorder.</p>
                @error('blocks')
                    <p class="mb-3 text-xs text-red-600">{{ $message }}</p>
                @enderror
                <x-island name="block-editor" :props="$editorProps" />
            </section>
        </div>

        <aside class="space-y-6">
            <section class="space-y-5 rounded-xl border border-slate-200 bg-white p-6 shadow-sm">
                @include('admin.partials.field', ['field' => $field('status', 'Status', 'select', ['options' => ['draft' => 'Draft (shown with a Draft tag)', 'published' => 'Published', 'hidden' => 'Hidden']]), 'value' => $article->status?->value, 'previews' => []])
                <div class="space-y-1.5">
                    <label for="field-published_at" class="block text-sm font-medium text-slate-700">Publication date</label>
                    <input id="field-published_at" type="datetime-local" name="published_at" class="input" value="{{ old('published_at', $article->published_at?->format('Y-m-d\TH:i')) }}">
                    <p class="text-xs text-slate-500">Set automatically when first published.</p>
                </div>
                @include('admin.partials.field', ['field' => $field('slug', 'Slug', 'text', ['hint' => 'Leave empty to generate from the title.']), 'value' => $article->slug, 'previews' => []])
                @include('admin.partials.field', ['field' => $field('reading_time', 'Reading time override', 'text', ['hint' => 'Leave empty to calculate, e.g. “6 min read”.']), 'value' => $article->reading_time, 'previews' => []])
                @if ($isTravel)
                    @include('admin.partials.field', ['field' => $field('featured', 'Featured on the home page', 'checkbox'), 'value' => $article->featured, 'previews' => []])
                @endif
            </section>

            <section class="space-y-3 rounded-xl border border-slate-200 bg-white p-6 shadow-sm">
                @include('admin.partials.field', ['field' => $field('cover_media_id', 'Cover image', 'media'), 'value' => $article->cover_media_id, 'previews' => ['cover_media_id' => $coverPreview]])
            </section>

            @if ($isTravel)
                <section class="space-y-2 rounded-xl border border-slate-200 bg-white p-6 shadow-sm">
                    <h2 class="text-sm font-semibold">Travel categories</h2>
                    @forelse ($categories as $category)
                        <label class="flex items-center gap-2 text-sm text-slate-700">
                            <input type="checkbox" name="travel_category_ids[]" value="{{ $category->id }}" class="h-4 w-4 rounded border-slate-300"
                                @checked(in_array($category->id, $selectedCategories, true))>
                            {{ $category->name }}
                        </label>
                    @empty
                        <p class="text-xs text-slate-500">Add categories under Travel categories.</p>
                    @endforelse
                </section>
            @endif

            <button type="submit" class="btn-primary w-full justify-center">Save</button>
        </aside>
    </form>
</x-admin.layout>
```

- [ ] **Step 6: Run tests**

Run: `php artisan test --filter=AdminArticlesTest`
Expected: PASS (5 tests).

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: travel story and note editor with block editor island"
```

---

### Task 11: Site settings

**Files:**
- Create: `app/Http/Controllers/Admin/SiteSettingsController.php`, `resources/views/admin/settings.blade.php`
- Modify: `routes/web.php`
- Test: `tests/Feature/AdminSettingsTest.php`

**Interfaces:**
- Consumes: `SiteSetting::read/write`, `admin.partials.field`, `MediaResource`.
- Produces: `GET /admin/settings` (`admin.settings.edit`), `PUT /admin/settings` (`admin.settings.update`); form fields map to the setting keys read by `SiteController` in Task 4.

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/AdminSettingsTest.php`:

```php
<?php

use App\Models\Media;
use App\Models\SiteSetting;
use App\Models\User;

beforeEach(function () {
    $this->actingAs(User::factory()->create());
});

it('saves site settings that the API reads', function () {
    $cv = Media::factory()->create(['mime_type' => 'application/pdf', 'extension' => 'pdf', 'path' => 'media/cv.pdf']);

    $this->put('/admin/settings', [
        'site_name' => 'Sarthak Neupane', 'site_short_name' => 'SN.', 'site_title' => 'Developer', 'site_description' => 'Portfolio',
        'site_url' => 'https://sarthak.example', 'site_email' => 'hello@sarthak.example', 'site_location' => 'Biratnagar, Nepal',
        'education' => 'IT STUDENT', 'marquee_items' => ['REACT', 'LARAVEL'], 'collab_areas' => ['Web apps'],
        'cv_media_id' => $cv->id, 'portrait_media_id' => '', 'portrait2_media_id' => '',
    ])->assertRedirect('/admin/settings');

    expect(SiteSetting::read('site.name'))->toBe('Sarthak Neupane')
        ->and(SiteSetting::read('marquee_items'))->toBe(['REACT', 'LARAVEL'])
        ->and(SiteSetting::read('site.cv_media_id'))->toBe($cv->id)
        ->and(SiteSetting::read('site.portrait_media_id'))->toBeNull();

    $this->getJson('/api/v1/site')->assertJsonPath('data.site.cvPath', $cv->url());
    $this->get('/admin/settings')->assertOk()->assertSee('Sarthak Neupane');
});

it('validates settings', function () {
    $this->put('/admin/settings', ['site_name' => '', 'site_email' => 'nope', 'cv_media_id' => 999])
        ->assertSessionHasErrors(['site_name', 'site_email', 'cv_media_id']);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `php artisan test --filter=AdminSettingsTest`
Expected: FAIL (404).

- [ ] **Step 3: Implement controller, route and view**

`app/Http/Controllers/Admin/SiteSettingsController.php`:

```php
<?php

namespace App\Http\Controllers\Admin;

use App\Http\Controllers\Controller;
use App\Http\Resources\Admin\MediaResource;
use App\Models\Media;
use App\Models\SiteSetting;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Validation\Rule;
use Illuminate\View\View;

class SiteSettingsController extends Controller
{
    /** Form field → setting key (dots are not usable in top-level form names). */
    public const FIELDS = [
        'site_name' => 'site.name',
        'site_short_name' => 'site.short_name',
        'site_title' => 'site.title',
        'site_description' => 'site.description',
        'site_url' => 'site.url',
        'site_email' => 'site.email',
        'site_location' => 'site.location',
        'education' => 'education',
        'marquee_items' => 'marquee_items',
        'collab_areas' => 'collab_areas',
        'cv_media_id' => 'site.cv_media_id',
        'portrait_media_id' => 'site.portrait_media_id',
        'portrait2_media_id' => 'site.portrait2_media_id',
    ];

    public function edit(): View
    {
        $values = collect(self::FIELDS)->map(fn (string $key) => SiteSetting::read($key))->all();
        $mediaFields = ['cv_media_id', 'portrait_media_id', 'portrait2_media_id'];
        $media = Media::query()->whereIn('id', array_filter(array_map(fn ($f) => (int) old($f, $values[$f]), $mediaFields)))->get()->keyBy('id');

        return view('admin.settings', [
            'values' => $values,
            'previews' => collect($mediaFields)->mapWithKeys(fn (string $f) => [
                $f => ($m = $media->get((int) old($f, $values[$f]))) ? (new MediaResource($m))->resolve() : null,
            ])->all(),
        ]);
    }

    public function update(Request $request): RedirectResponse
    {
        $mediaRule = ['nullable', 'integer', Rule::exists('media', 'id')->whereNull('deleted_at')];

        $data = $request->validate([
            'site_name' => ['required', 'string', 'max:120'],
            'site_short_name' => ['nullable', 'string', 'max:20'],
            'site_title' => ['nullable', 'string', 'max:255'],
            'site_description' => ['nullable', 'string', 'max:500'],
            'site_url' => ['nullable', 'url:http,https', 'max:255'],
            'site_email' => ['nullable', 'email', 'max:255'],
            'site_location' => ['nullable', 'string', 'max:120'],
            'education' => ['nullable', 'string', 'max:255'],
            'marquee_items' => ['nullable', 'array', 'max:40'],
            'marquee_items.*' => ['string', 'max:60'],
            'collab_areas' => ['nullable', 'array', 'max:20'],
            'collab_areas.*' => ['string', 'max:120'],
            'cv_media_id' => $mediaRule,
            'portrait_media_id' => $mediaRule,
            'portrait2_media_id' => $mediaRule,
        ]);

        DB::transaction(function () use ($data) {
            foreach (self::FIELDS as $field => $key) {
                $value = $data[$field] ?? null;

                if (in_array($field, ['marquee_items', 'collab_areas'], true)) {
                    $value = array_values(array_filter(array_map('trim', (array) ($value ?? []))));
                } elseif (str_ends_with($field, '_media_id')) {
                    $value = $value !== null ? (int) $value : null;
                }

                SiteSetting::write($key, $value);
            }
        });

        return redirect()->route('admin.settings.edit')->with('status', 'Settings saved.');
    }
}
```

In `routes/web.php` add `use App\Http\Controllers\Admin\SiteSettingsController;` and inside the `auth` group (before the resource routes):

```php
        Route::get('settings', [SiteSettingsController::class, 'edit'])->name('settings.edit');
        Route::put('settings', [SiteSettingsController::class, 'update'])->name('settings.update');
```

`resources/views/admin/settings.blade.php`:

```blade
@php
    $f = fn (string $name, string $label, string $type = 'text', array $extra = []) => ['name' => $name, 'label' => $label, 'type' => $type, ...$extra];
    $groups = [
        'Identity' => [
            $f('site_name', 'Name'),
            $f('site_short_name', 'Short name', 'text', ['hint' => 'Logo text, e.g. SN.']),
            $f('site_title', 'Site title', 'text', ['hint' => 'Browser tab and search results.']),
            $f('site_description', 'Description', 'textarea'),
            $f('site_url', 'Production URL', 'url'),
            $f('site_email', 'Contact email'),
            $f('site_location', 'Location'),
            $f('education', 'Education line'),
        ],
        'Home page' => [
            $f('marquee_items', 'Marquee words', 'tags'),
            $f('collab_areas', 'Collaboration areas', 'tags'),
        ],
        'Files' => [
            $f('portrait_media_id', 'Portrait (cut-out PNG)', 'media'),
            $f('portrait2_media_id', 'Second portrait', 'media'),
            $f('cv_media_id', 'CV (PDF)', 'media', ['accept' => 'any']),
        ],
    ];
@endphp
<x-admin.layout title="Site settings">
    <h1 class="mb-6 text-2xl font-semibold tracking-tight">Site settings</h1>

    <form method="POST" action="{{ route('admin.settings.update') }}" class="max-w-3xl space-y-6">
        @csrf
        @method('PUT')

        @foreach ($groups as $group => $fields)
            <section class="space-y-5 rounded-xl border border-slate-200 bg-white p-6 shadow-sm">
                <h2 class="text-sm font-semibold">{{ $group }}</h2>
                @foreach ($fields as $field)
                    @include('admin.partials.field', ['field' => $field, 'value' => $values[$field['name']] ?? null, 'previews' => $previews])
                @endforeach
            </section>
        @endforeach

        <button type="submit" class="btn-primary">Save settings</button>
    </form>
</x-admin.layout>
```

- [ ] **Step 4: Run tests**

Run: `php artisan test --filter=AdminSettingsTest`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: site settings screen"
```

---

### Task 12: Documentation and full verification

**Files:**
- Modify: `README.md`, `.env.example` (verify)
- Test: full suite, Pint, build, MySQL migration

**Interfaces:**
- Consumes: everything.

- [ ] **Step 1: Write the README**

Replace `README.md`:

````markdown
# Portfolio backend

Laravel 13 app that powers Sarthak Neupane's portfolio:

- **Admin panel** at `/admin` — Blade pages with React + Ant Design islands for the media library,
  drag-to-reorder lists, the article block editor and tag inputs.
- **Public API** at `/api/v1` — read-only JSON in exactly the shapes of `frontend/data/*.ts`.

Design: `docs/superpowers/specs/2026-11-05-portfolio-backend-design.md`.

## Setup

```bash
composer install
npm install
cp .env.example .env
php artisan key:generate
```

Create a MySQL 8 database and user, then fill `DB_*` in `.env`:

```sql
CREATE DATABASE portfolio CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'portfolio'@'localhost' IDENTIFIED BY 'choose-a-password';
GRANT ALL PRIVILEGES ON portfolio.* TO 'portfolio'@'localhost';
```

```bash
php artisan migrate --seed
php artisan admin:create
php artisan storage:link
composer run dev
```

Sign in at http://127.0.0.1:8000/admin.

The seeders import the current site content from `database/seeders/data/frontend.json`. Regenerate
that file from the frontend with `node scripts/export-frontend-data.mjs` (only needed before the
first seed; afterwards the database is the source of truth).

## Environment

| Variable | Purpose |
| --- | --- |
| `APP_URL` | Base URL; media URLs are built from it |
| `DB_*` | MySQL connection |
| `FRONTEND_URL` | Next.js site, used for "Preview on site" links |
| `CORS_ALLOWED_ORIGINS` | Comma-separated origins allowed to call `/api/*` |
| `MEDIA_DISK` | `public` locally, `s3` in production (`AWS_*` required) |
| `MEDIA_MAX_UPLOAD_KB` | Upload limit per file (default 20480) |

## API

All responses are `{ "data": ... }`, cached for 60 seconds with an ETag.

| Endpoint | Returns |
| --- | --- |
| `GET /api/v1/site` | `site`, `navLinks`, `marqueeItems`, `collabAreas`, `education`, `timeline`, `skillGroups`, `services`, `creativeItems`, `travelFilters`, `travelPhotos` |
| `GET /api/v1/projects` | `Project[]` |
| `GET /api/v1/travel`, `/travel/{slug}` | `TravelStory[]`, `TravelStory` |
| `GET /api/v1/notes`, `/notes/{slug}` | `Note[]`, `Note` |

Hidden articles return `404 {"message": "Not found."}`. Drafts are returned with `draft: true`.
Images are `{src, alt}`; a missing image is `{src: null, alt, label}` so the frontend's placeholder renders.

## Connecting the Next.js frontend

1. Set `NEXT_PUBLIC_API_BASE_URL=http://127.0.0.1:8000/api/v1` in `frontend/.env.local`.
2. Add the API host to `images.remotePatterns` in `frontend/next.config.ts`.
3. `lib/api.ts` already reads `/projects`, `/travel` and `/notes`; switching the remaining
   `@/data/*` imports to `/site` is a separate follow-up.

## Tests

```bash
php artisan test
vendor/bin/pint --test
npm run build
```
````

- [ ] **Step 2: Format and run everything**

```bash
vendor/bin/pint
php artisan test
npm run build
```

Expected: Pint fixes style only; all tests PASS; Vite build succeeds.

- [ ] **Step 3: Verify against MySQL 8**

Ask the user for MySQL credentials (or have them fill `DB_*` in `.env`), then run:

```bash
php artisan migrate:fresh --seed
```

Expected: all migrations and the seeder succeed. Then `php artisan admin:create` and smoke-test in the browser: sign in, reorder projects, upload an image, attach it as a project cover, edit a travel story's blocks (add an image block), and confirm `GET /api/v1/projects` and `/api/v1/travel/{slug}` show the image URLs.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "docs: setup, environment and API reference"
```
