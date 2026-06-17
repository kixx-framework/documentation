# GPT 5.5

Use a versioned cache key, but make the content version explicit.

Your current key:

build_id + pathname

only invalidates source-code changes. It has no signal for content changes, so you need one of these two mechanisms:

build_id + pathname + page_content_revision

or an explicit purge/delete when content changes.

The cleaner CMS design is usually:

page_html:{build_id}:{pathname}:{content_revision}

Where content_revision changes whenever anything that affects that page changes:

- metadata JSON changes
- any text blob used by the page changes
- page-to-blob relationships change
- published/draft state changes
- slug/path changes

That revision can be a monotonically increasing number, timestamp, UUID, or digest. I’d prefer a monotonic render_revision or content_revision stored with the page record, because it is simple to
reason about and cheap to update.

The tradeoff is important: on each request you still need a tiny lookup to discover the current revision for the pathname. But that lookup can be much cheaper than rebuilding the page. For example:

pathname -> page_id, content_revision

Then:

1. Request /about
2. Read page index for /about
3. Build cache key: html:{BUILD_ID}:/about:{content_revision}
4. Return cached HTML if present
5. Otherwise load metadata, blobs, templates, render HTML, cache it under that key

This avoids stale reads without needing to delete old cached HTML immediately. Old entries become unreachable and can expire naturally.

The alternative is purge-based invalidation:

html:{BUILD_ID}:{pathname}

Then when metadata/blob content changes, your CMS must know every affected pathname and delete/purge those cache entries. That can work, but it requires reliable dependency tracking:

blob_id -> affected page paths
metadata_id -> affected page paths

For Cloudflare-style distributed caching, purge semantics also matter. A local cache delete may not clear every edge location depending on which cache layer you use, so versioned keys are usually
more robust than relying only on deletion.

My recommendation:

cache key = html:{build_id}:{pathname}:{page_render_revision}

And maintain a small page routing/index record:

{
"pathname": "/about",
"page_id": "page_123",
"render_revision": 42,
"status": "published"
}

When a page’s metadata or blobs are updated, increment render_revision for every affected page. That gives you deterministic invalidation while still avoiding expensive metadata/blob/template/render
work on normal requests.

For planning purposes, the key question is whether content updates are always page-scoped, or whether one blob/partial content item can be shared by many pages. If content can be shared, you’ll also
  need dependency tracking from content asset to affected pages.

---

# Opus

This is the classic "two version axes" problem, and naming it that way is the key to solving it cleanly.

Your buildID is a deploy-time version. It changes when source changes (templates, partials, helpers). But metadata and text blobs change on a completely different axis — content-time — out of band
from any deploy. A single version key can't track both axes, so you need a second component in your key that moves when content moves.

The unavoidable tradeoff

You can't detect a content change for free. There are only two shapes of solution, and the choice between them is really the whole decision:

- (A) Pull / version-in-key — on every request, do one cheap read to learn the current content version, then look up the rendered HTML by buildID + pathname + contentVersion. Always correct,
invalidation is automatic, but you pay one small read per request.
- (B) Push / purge-on-write — the read path stays at buildID + pathname with zero lookups, but every content write must actively delete the affected cache entries. Fastest reads, but you inherit a
reverse-index problem and distributed-purge complexity.

★ Insight ─────────────────────────────────────
The expensive work you're trying to avoid is three things: blob reads, template compile, and render. The version check is a fourth, much cheaper thing. The trap is conflating them — you can keep
paying the cheap check while caching away the three expensive steps. "Avoid lookups on every request" usually really means "avoid the expensive lookups."
─────────────────────────────────────────────────

My recommendation: version-stamped key, metadata doc as the version authority

Make the metadata document the single source of truth for the page's content version, and fold the blob versions into it:

1. The metadata doc carries a rev (a counter or updatedAt) and a manifest of its associated blob keys → ETags/versions.
2. Whenever metadata is edited or an associated blob is uploaded, the writer bumps the metadata doc's rev (recording the new blob ETag). Now one read of the metadata doc tells you the complete
content version of the page.
3. Read path:
- Read the metadata doc by pathname — small JSON, and you likely need it anyway for status/draft/redirect/permission checks.
- Build the key buildID : pathname : rev.
- Cache hit → serve the HTML with no blob reads, no compile, no render. Miss → read blobs, render, store under that key.

Invalidation becomes a non-event: bump rev → the key changes → the old entry is simply orphaned (cleaned up by TTL). You never delete anything, which sidesteps the hardest part of (B).

★ Insight ─────────────────────────────────────
Two properties fall out of version-in-key for free. (1) No reverse index: a blob shared across N pages doesn't need a "blob → pages" lookup, because each page's render already records the blob
version it built from — staleness just means the key never matches. (2) Free rollback: revert content to a prior rev and the previously-rendered HTML is still cached under that key, so you get an
instant hit instead of a rebuild.
─────────────────────────────────────────────────

This is also the portable choice across your target runtimes (Node/Deno/Cloudflare/Lambda) — it needs nothing but a key/value cache. The push model (B) leans on platform-specific features: cache
tags / surrogate keys (Fastly Surrogate-Key, Cloudflare Enterprise Cache Tags) are the "industry standard" way to do association-based purging, but they tie you to a vendor.

Two things to design in regardless of which model you pick:
- TTL as garbage collection, not as the correctness mechanism — it reaps orphaned rev entries.
- Stampede protection on miss — when rev bumps, the first wave of requests will all miss and rebuild simultaneously. A per-key build lock or stale-while-revalidate avoids the thundering herd.

Before I'd turn this into a plan, a few things shape the answer

1. Do you read the metadata doc on every request anyway for status/draft/redirect/auth? If yes, model (A) is nearly free and is clearly the right call. If you genuinely serve cached pages without
any read today, the calculus shifts toward (B).
2. Are text blobs ever shared across multiple pages, or is it 1 page → its own blobs? Sharing is what makes purge-on-write painful and makes version-in-key shine.
3. What's the cache substrate and primary deploy target? In-memory per-instance, Cloudflare KV/Cache API, R2, D1? Distributed/edge caches make active purge much harder and push hard toward
  version-in-key.

---

# Gemini

You’ve hit on one of the classic "two hard problems in computer science." Your current strategy using `BUILD_ID + URL pathname` is excellent for deployment-led invalidation, but handling dynamic content updates requires a shift in how you either structure your keys or manage your cache lifecycle.

Here are the three most effective architectural patterns to handle content-driven invalidation for your specific setup, ranked from lowest to highest complexity.

### 1. Event-Driven Purging (The Active Approach)

If your CMS has a clear "Save" or "Publish" action when an editor updates the metadata or text blobs, you can use this event to actively evict the stale cache.

* **How it works:** When a write operation happens in the JSON database or object store, your application triggers a cache deletion for the specific key associated with that page.
* **The Key Structure:** Stays exactly as you designed it: `BUILD_ID + URL_PATHNAME`.
* **The Catch:** Your CMS backend must know the relationship between the content being edited and the `URL_PATHNAME`. If it’s a 1:1 relationship (e.g., editing the "About Us" page metadata directly maps to `/about`), this is incredibly simple. If a single text blob is shared across multiple pages, you'll need a dependency graph or mapping table to know which URLs to purge.

### 2. Content Versioning in the Cache Key (The Passive Approach)

If you want to avoid active purging or if tracking URL dependencies is too complex, you can bake a **Content Fingerprint** into your compound cache key.

* **The Key Structure:** `BUILD_ID + URL_PATHNAME + CONTENT_TIMESTAMP`
* **How it works:** Your JSON document database likely stores a lightweight `updated_at` or `version` field for the page metadata. When a request comes in, you perform a highly optimized, indexed lookup *only* for that timestamp/version number.
* **Why it works:** Checking an index for a single timestamp is incredibly cheap compared to fetching full text blobs and compiling templates. If the timestamp matches the cached key, serve the HTML. If the content was updated, the timestamp changes, the old cache key is naturally abandoned, and a new one is generated.

### 3. Cache Tagging / Surrogate Keys (The Scalable Approach)

If you are using an advanced caching layer like Redis, Fastly, or Cloudflare, you can leverage **Cache Tags**. This is ideal if your pages are modular (e.g., a text blob is reused on multiple pages).

* **How it works:** When you cache the rendered HTML, you attach metadata tags to that cache entry representing the underlying data dependencies.
* *Cache Key:* `BUILD_ID + /about`
* *Tags:* `meta:page-123`, `blob:hero-text-456`


* **Invalidation:** When `blob:hero-text-456` is updated in your object store, you issue a command to your cache layer: *“Invalidate all entries tagged with `blob:hero-text-456`.”* The caching layer automatically handles wiping out every page that relies on that specific blob.

### Which one should you choose?

* Go with **Event-Driven Purging** if your content changes infrequently and your admin panel explicitly knows which URL is being updated. It keeps your read paths completely free of database lookups.
* Go with **Content Versioning** if your data changes frequently or via background processes where tracking URL paths is difficult, and your database can handle cheap index lookups.

To help narrow this down, how intertwined are your text blobs and metadata—are text blobs uniquely tied to a single page, or can a single blob be shared across multiple URLs?

---


# Deploy Your Site on a Node.js Server

Kixx projects can be deployed to any server which is running Node.js.

First, you need to decide how you want to track your source code and content.

A common pattern many developers use is to track the source code *and* content in git:

- **Source Code** - in src/ is tracked in git
- **Templates** - in templates/ are tracked in git
- **Content** - in pages/ is tracked in git

Source code, templates, and content can all be deployed to your Node.js server using git, rsync, a zip archive, or any one of your favorite deployment tools. The exact mechanisms for deployment are left to you, the developer, and outside the scope of this document. Kixx doesn't try to force you down any particular deployment path.

If you have template caching turned on for your remote environment then pushing new source code and templates to it will have no immediate impact until you restart the server.

After deploying new content, restarting the server will cause it to purge the caches and begin consuming your new content and templates as they are requested. The server does not pre-warm the cache.

__Note__ - If your old templates are expecting data in a different shape from your latest content push, then impacted pages could break in unexpected ways until you restart the server. To avoid the possibility of breaking your site, you can use the Kixx content publishing system.

[Kixx Content Publishing System](#kixx-content-publishing)

Also, restarting your server could cause a moment of downtime while the server restarts, but should be less than several seconds. To avoid downtime you can adopt a *zero downtime* deployment strategy.

## Zero Downtime Deployment

TODO: This should apply to both Node.js and Deno runtimes.

# Kixx Content Publishing System

## Deployments with Content Publishing for Node.js and Deno Environments

**Create a new content version** - Use the Kixx CLI tool or the admin console in your remote app to create a new content version. You'll need the content version in the next step. If you want to gaurd against the possibility of publishing content with the incorrect templates, you should leave it in draft mode without publishing yet.

**Create a new app deployment** - Use the Kixx CLI tool or the admin console in your remote app to create a new app deployment. Either way, you'll need to set a deployment ID. You can use a git commit hash, let the tool create a deployment ID for you, or just enter an arbitrary value. Then you'll need to associate your new deployment with a content version. The content version can be a draft, or already published. If the content version is in draft mode, the next step, promoting your app, will automatically publish it.

**Release the new app deployment** - Once your deployment is ready, release the new changes by simply restarting the server.

---

## Key Insights

1. As soon as we use more than one server node, then there is no way to invalidate the cache without restarting the process.
2. If the deployment mechanism restarts nodes on every deployment, then everything can be cached
3. Using templates in page data (html, markdown) is too expensive. We need to find a way to pre-render them.
4. Many Kixx instances will not be tracking page data in git. Templates need to go in templates/ where they can be tracked.
5. Static pages can be pre-rendered so that they can still use templates (html, markdown).

- `templates/` - All dynamic templates
- `pages/` - All static page data. Precompiled (on deployment or on first read).

---

Right now, both static and dynamic page metadata is in pages. But, if we pre-render static pages then we need a way to differentiate dynamic pages.

---

## Request Handler

Get the latest cache key/namespace.

### Static Content

1. Check for the pre-rendered page (using namespace). If it exists, then serve it.
2. Read the page metadata (using namespace).
3. Load included content (using namespace), assume each one is a template, and render it, holding it in memory for the next step.
4. Merge included content into page metadata to become template data.
5. Load the page template (using namespace), and render it as the template data body.
6. Load the base template (using namespace), and render it using template data.
7. Save the pre-rendered page (using namespace) and then serve it.

```js
async function serveStaticPage() {
    const cacheKey = await getCacheKey(context.runtime.build.id);
    const cachedPage = await pageCache.get(cacheKey, pathname);
    if (cachedPage) {
        return cachedPage;
    }

    const metadata = await pageDataStore.getMetadata(cacheKey, pathname);

    const includedFiles = await Promise.all(metadata.includes.map((item) => {
        return pageDataStore.getTextFile(cacheKey, item);
    }));

    const includes = includedFiles.map((file) => {
        return buildAndRenderTemplate(file, metadata);
    });

    const templateContext = merge(metadata, includes);

    const template = await this.getPageTemplate(cacheKey, pathname);

    if (template) {
        templateContext.body = template(templateContext);
    }

    const baseTemplate = await this.getBaseTemplate(cacheKey, pathname);

    const page = baseTemplate(templateContext);
    await pageCache.put(cacheKey, page);
    return page;
}
```

### Dynamic Content

1. Read the page metadata (using namespace).
2. Check for pre-rendered included content (using namespace). If none, then:
3. Load included content (using namespace), assume each one is a template, render it, and cache it (using namespace).
4. Merge response props and included content into page metadata to become template data.
5. Load the page template (using namespace), and render it as the template data body.
6. Load the base template (using namespace), and render it using template data.

```js
async function serveDynamicPage() {
    const cacheKey = await getCacheKey(context.runtime.build.id);

    const metadata = await pageDataStore.getMetadata(cacheKey, pathname);

    const includedFiles = await Promise.all(metadata.includes.map((item) => {
        return pageDataStore.getTextFile(cacheKey, item);
    }));

    merge(metadata, response.props);

    const includes = includedFiles.map((file) => {
        return buildAndRenderTemplate(file, metadata);
    });

    const templateContext = merge(metadata, includes);

    const template = await this.getPageTemplate(cacheKey, pathname);

    if (template) {
        templateContext.body = template(templateContext);
    }

    const baseTemplate = await this.getBaseTemplate(cacheKey, pathname);

    return baseTemplate(templateContext);
}
```

## Static Deployment

Prebuild static content and deploy it. This excludes dynamic content.

The static build process is TBD.

## Git/Rsync Deployment

1. Push all source, including content.
2. Restart the servers

- Content and templates are just kept in flat files on the server
- Without cache key namespacing it is possible for the site to break when new page metadata is pushed

## AWS Lamba Deployment

1. Deploy source -> yields BUILD_ID
2. Publish content with BUILD_ID.
3. Promote app version (BUILD_ID is changed automatically).

- CLI Tool creates and uploads a zip archive of source and generates a BUILD_ID
- CLI Tool publishes page data and templates using the BUILD_ID on the Kixx remote API
- CLI Tool sets the BUILD_ID on the remote and promotes the build

## Cloudflare Deployment

1. Deploy source -> yields BUILD_ID
2. Publish content with BUILD_ID.
3. Promote app version (BUILD_ID is changed automatically).

- CLI Tool generates a BUILD_ID
- CLI Tool publishes page data and templates using the BUILD_ID on the Kixx remote API
- CLI Tool creates and uploads a bundle of source code and sets the BUILD_ID on the remote worker

---

We do not cache page data, since it is cheap to read.

Templates are expensive to build, so we want to cache templates as much as we can.

But if a site has a lot of page templates, it may be too much to hold in memory.

So, maybe by default, page templates are NOT cached, but can be marked as cacheable by their page data.

Included page files are NOT cached, because they can be large and are cheap to fetch.

The namespaces in the template file store and page data store make deployments idempotent.

The caches are naturally purged and rebuilt, using the new deployment namespace, when the processes are restarted.

*I think we need to separate dynamic endpoints into templates/ while static endpoints remain in pages/ *

We need to support the case where a developer wants to keep templates for dynamic content in source control while leaving the pages/ content as driven by the CMS.

But I still want the ability to edit the page data for the dynamic endpoints.

I don't think that ^ makes sense, because the page data should be bound to the source/template commit.

## Use Case: Use rsync or git to push the new code to the server.

How do we update the cache key?

- We could make a call to the KV store to update it: But, when would we do that? Before restarting could break things. After restarting could mean that things are broken until the key is updated.
- The cache key could be part of the config or an env variable. But then how would the app know what the source of truth is? The KV store or the config?

The namespace for the cache key is the build ID, and the initial value for the cache key is the build id.

Ok, we need to start over:

*I think we need to separate dynamic endpoints into templates/ while static endpoints remain in pages/ *

How do we work with content?

Save locally, everyone has a different copy, push drafts, and version them?


---

So here is how this will work (use cases below):

The pages/ are data only - no templates - just Content. They can reference a template, and can include HTML content, but will not be respected as templates.

There is a Content Version which represents a checkpoint in the content (pages/).

Multiple content versions can be saved as drafts. You can update an existing draft, or create a new one.

A Content Version can be published.

There is Build ID which represents the latest source code (including templates/).

There is an App Version which is a composite of the Build ID and Content Version.

To create a deployment you do the build and get a Build ID, then associate a Content Version with that build. The Content Version can be in draft mode.

The Build ID is set as an environment variable which is read once at startup and becomes the namespace for the PageDataStore and TemplateFileStore.

When the newly deployed code starts, it reads the Build ID environment variable.

- It uses the Build ID as the namespace for the TemplateFileStore
- It uses the Build ID to get the App Version which references the Content Version then uses the ContentVersion as the namespace for the PageDataStore

---

## Use Case: 1

- No deployment tooling is used (no Build ID, no Published Version).
- Git is tracking templates/ (not pages/)
- Publish content through the admin console or CLI tool

I'm not sure anyone would do it this way, but I guess you could. It's just really hard to manage and you need to expect some downtime.

1) Push everything to the server with git or rsync (except for pages/):

- If caching is turned off, this could break the site since the new templates would be immediately consumed
- If caching is turned on, this still might break the site if the content (updated later) is expecting different templates.

2) Restart the server:

- If caching is turned on, this will pull in the new source, including the new templates.
- If caching is turned off, this will reset everything might fix the site if it was broken during deployment.

3) Push the pages/ content to the server using the CLI tool or the admin console:

- If the site was broken (because content expected different templates) this should fix it.

## Use Case: 2

- No deployment tooling is used (no Build Version).
- Git tracks everything including templates/ and pages/

I can see a lot of developers doing it this way.

1) Push everything to the server with git or rsync:

- If caching is turned off, this could break the site since the new templates would be immediately consumed
- If caching is turned on, everything should continue working just fine (in the old version)

2) Restart the server:

- If caching is turned off, this will reset everything and should fix the site if it was broken during deployment.
- If caching is turned on, the new version of the site should be immediately published

## Use Case: 3

- Using the Kixx deployment tool
- Git is tracking templates (not pages/)
- Publish content through the admin console or CLI tool

This could be common for agencies. This is close to what I'm doing with the platform + kixx-dot-dev

1) Push source code to the server (including templates) using git or rsync

- Assuming the cache is turned on, nothing will happen because the source code and templates will not be consumed yet

2) Create a new deployment using the CLI tool or the admin console

- Creates a new Build ID and sets the env variable
- Publish content with the new Build ID using the CLI tool or admin console
- This will have no effect on the running server since it is still pulling content using a previous Build ID or Published Version

3) Restart the server.

- The new version should be up and running, using the new Build ID

**For content-only updates**

Publish the content using the CLI tool or admin console.

- Uploads the content to the server
- Resets the Build ID

## Use Case: 4

- Deploy to Cloudflare
- Node.js or Deno for local development

1) The CLI tool or admin console does everything

- Bundles the source code and creates a new version (but does not deploy it)
- Pushes the templates/ and pages/ to the remote KV store at the new version namespace
- Sets the new build version
- Deploys the new version of the code

**For content-only updates**

- Creates a new Build ID
- pushes the content to the remote KV store using the Build ID as namespace
- updates the Build ID on the server

---

There are several objectives Kixx is trying to achieve with the deployment and caching system:

1. Idempotent deployments of server code and templates with rollback capability.
2. Idempotent publishing of content with rollback capability.
3. Caching of compiled templates for better performance.

---

We need to do some big refactoring for the HyperviewService

There are two important concepts you need to understand first:

1. Build ID - represents the latest source code build of this application, including templates in the TemplateFileStore.
2. Published Version - represents the latest published version of content, which includes everything in the PageDataStore.

The Build ID is set using an environment variable and is available on the ApplicationContext and RequestContext as `runtime.build.id`.

The Published Version is set by the HyperviewService whenever a new published version of the content is written.

All data in the TemplateFileStore and PageDataStore is namespaced.

- When the Build ID is updated, it is used as the namespace.
- When the Published Version is updated, it becomes the new namespace.

We determine the namespace by reading a special key from the KeyValueStore:

`content_namespace:[build_id]`

So, if the Build ID is "abc123" then the key will be `content_namespace:abc123`.

The value for the namespace key is initialized with the Build ID. So, using the example above:

`content_namespace:abc123 = "abc123"`

So, when HyperviewService starts for the first time, it should set the namespace key to the current Build ID.
