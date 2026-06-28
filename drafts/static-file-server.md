# Serving Static Files

Use the `StaticFileRequestHandler` handler to serve static files from your request routing configuration. In `virtual-hosts.js`:

```js
import { StaticFileRequestHandler } from './kixx/static-file-server/static-file-server-request-handlers.js';

export default [
    {
        name: 'kixx-app',
        hostname: 'localhost',
        routes: [
            {
                pattern: '/favicon.ico',
                name: 'favicon-ico',
                targets: [
                    {
                        name: 'serve',
                        methods: [ 'GET', 'HEAD' ],
                        requestHandlers: [
                            StaticFileRequestHandler(),
                        ],
                    },
                ],
            },
        ],
    },
];
```

The `StaticFileRequestHandler` factory accepts an options object as the single parameter:

- `options.contentType` - Explicitly set the Content-Type header for the response. By default the Content-Type header is derived from the file extension.
- `options.cacheControl` - Explicitly set the Cache-Control header for the response. By default the Cache-Conrol header is set to "public, max-age=0, must-revalidate" which tells the browser that the asset can be cached, but that the browser should revalidate the freshness of the content every time before using it.
- `options.computeEtag` - The Etag header is computed as a hash of the file contents. If set to false, the Etag header will not be computed for this file. The `computeEtag` flag is `true` by default so browsers can use this in subsequent requests with an If-None-Match header to check for freshness, without needing to re-download the entire file in the case of a match.
- `options.pathname` - Override the URL pathname for every request to the handler. This can be useful to rewrite certain URL pathnames before attempting to read the file from the store.
- `options.throwNotFound` - When this flag is `true` the handler will throw a `NotFoundError` when the file does not exist, which will result in a 404 response being sent from the error handler. This is `true` by default. Set it to `false` when you want subsequent request handlers to handle the request when a file is not found.
- `options.skipWhenFound` - When this flag is `true` the handler will skip remaining request handlers on this route if the static file is found. This is false by default.

Here is another example of serving a file with customized options. In `virtual-hosts.js`:

```js
import { StaticFileRequestHandler } from './kixx/static-file-server/static-file-server-request-handlers.js';

export default [
    {
        name: 'kixx-app',
        hostname: 'localhost',
        routes: [
            {
                pattern: '/css/images/*pathname',
                name: 'css-images',
                targets: [
                    {
                        name: 'serve',
                        methods: [ 'GET', 'HEAD' ],
                        requestHandlers: [
                            StaticFileRequestHandler({
                                cacheControl: 'public, max-age=86400'
                            }),
                        ],
                    },
                ],
            },
        ],
    },
];
```

The `StaticFileRequestHandler` works well when used in the catch-all route with `throwNotFound` set to `false` and `skipWhenFound` set to `true`. In `virtual-hosts.js`:

```js
import { StaticFileRequestHandler } from './kixx/static-file-server/static-file-server-request-handlers.js';

export default [
    {
        name: 'kixx-app',
        hostname: 'localhost',
        routes: [
            {
                pattern: '*',
                name: 'hyperview-static-catch-all',
                targets: [
                    {
                        // Catch-all renderer for static Hyperview static pages, including the
                        // site root, with optional JSON page data responses.
                        name: 'render-static-page',
                        methods: [ 'GET', 'HEAD' ],
                        requestHandlers: [
                            StaticFileRequestHandler({
                                throwNotFound: false,
                                skipWhenFound: true,
                            }),
                            HyperviewStaticPageHandler(),
                        ],
                    },
                ],
            },
        ],
    },
];
```

## How Static File Serving Works

The `StaticFileRequestHandler` delegates to an internal Kixx component called the `StaticFileStore` which stores files by key. The `StaticFileRequestHandler` uses the request pathname, excluding query parameters and hashes, as the file key along with the current Build ID as the namespace to support Atomic Deployments. The `StaticFileStore` then looks up the file by key and namespace and returns a result.

There is an implementation of the `StaticFileStore` interface for each supported runtime.

### Under the Hood: Node.js StaticFileStore

When using the Node.js platform the StaticFileStore simply serves files from the project `/public` directory. If your project is using Atomic Deployments with a Build ID, then static files will be namespaced by the Build ID under the `/public` directory on your remote server when the Kixx deployment tooling uploads them.

### Under the Hood: Cloudflare Workers StaticFileStore

For Cloudflare Workers the StaticFileStore is backed by the KV Store. Files are cached in the KV Store indefinitely. When a new build is deployed the application will begin using the newly deployed static files from the KV Store under the latest Build ID value as a namespace.

The Kixx deployment tooling uploads static files from your project `public/` directory to your remote application using the Kixx Publishing API.

When deploying to Cloudflare, Kixx will always use Atomic Deployments with a Build ID.
