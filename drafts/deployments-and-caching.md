## Use Case 1
Deploy to a Node.js or Deno server with git or rsync.

### With Caching Off
You can leave page caching and template caching off. The tradeoff, of course, is that you will not get the performance benefits of caching. The benefit of turning caching off is that new templates or content is deployed they will be available without restarting the server. You'll only need to restart the server when making backend code changes. This could be useful for a staging server, and can even be performant enough for many production systems.

When backend code changes are made, there is a risk that the new backend code will use the old, incompatible, templates and page content before the server restart is complete. You can mitigate this risk with a zero downtime deployment mechanism which creates new instances of the server with the new code, template, and content, and then shift traffic to the new instances using a load balancer or reverse proxy before shutting down the old instance.

### With Template Caching On
If you turn on template caching you can get better page response performance at the cost of making your deployments more complicated.

With template caching on, when you deploy new templates, the existing templates are still cached in the runtime memory, so the new templates will not be used until you restart the server. If you have backend code changes or content updates which are not compatible with the old templates there is a risk that your site could be broken until the server is restarted. You can mitigate this risk with a zero downtime deployment mechanism which creates new instances of the server with the new code, template, and content, and then shift traffic to the new instances using a load balancer or reverse proxy before shutting down the old instance.

### With Page Caching On
If you turn on page caching you can get much better page response performance for your static pages (pages which do not depend on dynamic backend data).

Pages are cached in the Key Value Store and are not cached in the runtime memory, so restarting your servers will not invalidate the page cache. For your new pages to appear you'll need to wait for the page store TTL to expire. To avoid this problem you can use the Atomic Deployments strategy.

### Atomic Deployments
Alternatively you can optimize page and template caching AND avoid temporarily breaking your site by using the Atomic Deployment feature of Kixx. An atomic deployment assigns a build ID to your deployment and namespaces all your cached resources by that build ID. When the build ID changes, all caches are magically invalidated and new caches are established. For this to work you'll need to use the CLI tool to deploy templates and page content instead of git or rsync. The downside to this approach is that it complicates your deployment process because you'll need to deploy backend source code separately from templates and page content.

To use Atomic Deployments you'll also need to set a BUILD_ID environment variable which is read by the server when it starts. A common approach for getting a value for the BUILD_ID is to use a git commit hash, but you could use a date-time string, incremented integers, decimal string, or whatever works for your project.

When you're ready for a new deployment, first get a BUILD_ID for it. Once you have a BUILD_ID, use it with the CLI tool when uploading templates and page data. When your templates and page data have been deployed, and any backend code changes have been deployed, then update the BUILD_ID environment variable and restart the server. It will immediately begin using the latest page data and templates associated with the current BUILD_ID.

### Client-Side Stylesheets and JavaScript
To keep things simple, you can avoid a build or bundle step for your client side code. Without a build or bundling step you'll want to keep the HTTP caching parameters loose for your CSS and JavaScript assets so that clients will revalidate them quickly when you deploy a new version of your site.

To take full advantage of bundling and strong caching for better performance you can use the Atomic Deployments strategy and deploy with the CLI tool. Use your latest BUILD_ID with the CLI tool when you bundle your client side code. This will create bundle filenames suffixed with your BUILD_ID. When you deploy the latest version of your site, update the BUILD_ID environment variable, and restart it, then the latest client side bundles will be served.

If you don't want to use Atomic Deployments for your site, but still want to take advantage of bundled and strongly cached client assets, you can use the CLI tool to bundle your assets coupled with the BUNDLE_ID instead of the BUILD_ID.

## Use Case 2
Deploy to Cloudflare Workers with the CLI tool.

### Installing the Site
Use the CLI tool to install the site. This will establish the first BUILD_ID as well as deploy all the backend code, excluding templates and page content.

### Deploying the Site
Before using the CLI tool to deploy the site, you'll need to use the CLI tool to get a publishing API token.

Once you have an API token you can use the CLI to deploy the site.

Using the CLI tool to build and deploy the site will:

- Establish a BUILD_ID using git or a timestamp
- Bundle and deploy the client side assets, namespaced by the BUILD_ID
- Publish page content, namespaced by the BUILD_ID
- Deploy templates, namespaced by the BUILD_ID
- Bundle and deploy the backend code, simultaneously updating the BUILD_ID environment variable.

When the last step is complete, the site will immediately begin using the latest templates, client-side assets, and page content.

### Publishing Page Content
You can use the CLI tool to publish page content separately from backend source code or template changes. This assumes the new page content does not depend on backend source code or template updates.

When you publish page content, the CLI tool will read your local page content files and compare the local hash with the remote hash. If the hashes are different the CLI tool will upload the new page content, including the latest hash for each page content file.

---

# Running your site with Node.js or Deno in Hard Mode

If you're familiar with SSH and/or CI/CD workflows then you'll find this method of operating your website to be pretty easy and intuitive. If you're not familiar with these things, then this is a good place to learn if you have the patience.

## Installing your site on a Node.js server

First you'll need to install a bare Kixx instance.

- [ ] TODO: Instructions for installing with git
- [ ] TODO: Instructions for installing with rsync
- [ ] TODO: Instructions when using the Caddy HTTP server

If you are deploying your content (the `pages/` directory) using git or rsync, then you're done! Once you restart your app on the server you should be able to see your site, live at the configured hostname.

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
