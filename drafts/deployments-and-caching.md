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

*I think we need to separate dynamic end points into templates/ while static endpoints remain in pages/ *

We need to support the case where a developer wants to keep templates for dynamic content in source control while leaving the pages/ content as driven by the CMS.

But I still want the ability to edit the page data for the dynamic endpoints.

## Push a Backend + Template Update

Use rsync or git to push the new code to the server.

How do we update the cache key?

- We could make a call to the KV store to update it: But, when would we do that? Before restarting could break things. After restarting could mean that things are broken until the key is updated.
- The cache key could be part of the config or an env variable. But then how would the app know what the source of truth is? The KV store or the config?

The namespace for the cache key is the build ID, and the initial value for the cache key is the build id.

---

There are several objectives Kixx is trying to achieve with the deployment and caching system:

1. Idempotent deployments of server code and templates with rollback capability.
2. Idempotent publishing of content with rollback capability.
3. Caching of compiled templates for better performance.
