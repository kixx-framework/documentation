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

- No deployment tooling is used (no Build ID, no Published Version).
- Git is tracks everything including templates/ and pages/

I can see a lot of developers doing it this way.

1) Push everything to the server with git or rsync:

- If caching is turned off, this could break the site since the new templates would be immediately consumed
- If caching is turned on, everything should continue working just fine (in the old version)

2) Restart the server:

- If caching is turned off, this will reset everything might fix the site if it was broken during deployment.
- If caching is turned on, the new version of the site should be immediately published

## Use Case: 3

- Using the Kixx deployment tool
- Git is tracking templates (not pages/)
- Publish content through the admin console or CLI tool

This could be common for agencies. This is what I'm doing with the platform + kixx-dot-dev

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
