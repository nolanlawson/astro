# @astrojs/cloudflare

An SSR adapter for use with Cloudflare Pages Functions targets. Write your code in Astro/Javascript and deploy to Cloudflare Pages.

## Install

Add the Cloudflare adapter to enable SSR in your Astro project with the following `astro add` command. This will install the adapter and make the appropriate changes to your `astro.config.mjs` file in one step.

```sh
# Using NPM
npx astro add cloudflare
# Using Yarn
yarn astro add cloudflare
# Using PNPM
pnpm astro add cloudflare
```

If you prefer to install the adapter manually instead, complete the following two steps:

1. Add the Cloudflare adapter to your project's dependencies using your preferred package manager. If you’re using npm or aren’t sure, run this in the terminal:

```bash
npm install @astrojs/cloudflare
```

2. Add the following to your `astro.config.mjs` file:

```js ins={3, 6-7}
// astro.config.mjs
import { defineConfig } from 'astro/config';
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
  output: 'server',
  adapter: cloudflare(),
});
```

## Options

### `mode: "advanced" | "directory"`

default `"advanced"`

Cloudflare Pages has 2 different modes for deploying functions, `advanced` mode which picks up the `_worker.js` in the `dist` folder, or a directory mode where pages will compile the worker out of a `functions` folder in the project root.

#### `mode:directory`

Switching to directory mode allows you to use [pages plugins](https://developers.cloudflare.com/pages/platform/functions/plugins/) such as [Sentry](https://developers.cloudflare.com/pages/platform/functions/plugins/sentry/) or write custom code to enable logging.

```js
// astro.config.mjs
export default defineConfig({
  adapter: cloudflare({ mode: 'directory' }),
});
```

In `directory` mode, the adapter will compile the client-side part of your app the same way as in `advanced` mode by default, but moves the worker script into a `functions` folder in the project root. In this case, the adapter will only ever place a `[[path]].js` in that folder, allowing you to add additional files manually.

To instead compile a separate bundle for each page, set the `functionPerPath` option in your Cloudflare adapter config. This option requires some manual maintenance of the `functions` folder. Files emitted by Astro will overwrite existing `functions` files with identical names, so you must choose unique file names for each file you manually add. Additionally, the adapter will never empty the `functions` folder of outdated files, so you must clean up the folder manually when you remove pages.

```js ins={7-8}
// astro.config.mjs
import {defineConfig} from "astro/config";
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
     adapter: cloudflare({
        mode: 'directory',
        functionPerRoute: true
    })
})
```

Astro will bundle the [Astro middleware](https://docs.astro.build/en/guides/middleware/) into each page bundle.

### `routes.strategy: "auto" | "include" | "exclude"`

default `"auto"`

Determines how `routes.json` will be generated if no custom [`_routes.json`](#custom-_routesjson) is provided.

There are three options available:

- **`"auto"` (default):** Will automatically select the strategy that generates the fewest entries. This should almost always be sufficient, so choose this option unless you have a specific reason not to.

- **`include`:** Pages and endpoints that are not pre-rendered are listed as `include` entries, telling Cloudflare to invoke these routes as functions. `exclude` entries are only used to resolve conflicts. Usually the best strategy when your website has mostly static pages and only a few dynamic pages or endpoints.

  Example: For `src/pages/index.astro` (static), `src/pages/company.astro` (static), `src/pages/users/faq.astro` (static) and `/src/pages/users/[id].astro` (SSR) this will produce the following `_routes.json`:

  ```json
  {
    "version": 1,
    "include": [
      "/_image", // Astro's image endpoint
      "/users/*" // Dynamic route
    ],
    "exclude": [
      // Static routes that needs to be exempted from the dynamic wildcard route above
      "/users/faq/",
      "/users/faq/index.html"
    ]
  }
  ```

- **`exclude`:** Pre-rendered pages are listed as `exclude` entries (telling Cloudflare to handle these routes as static assets). Usually the best strategy when your website has mostly dynamic pages or endpoints and only a few static pages.

  Example: For the same pages as in the previous example this will produce the following `_routes.json`:

  ```json
  {
    "version": 1,
    "include": [
      "/*" // Handle everything as function except the routes below
    ],
    "exclude": [
      // All static assets
      "/",
      "/company/",
      "/index.html",
      "/users/faq/",
      "/favicon.png",
      "/company/index.html",
      "/users/faq/index.html"
    ]
  }
  ```

### `routes.include: string[]`

default `[]`

If you want to use the automatic `_routes.json` generation, but want to include additional routes (e.g. when having custom functions in the `functions` folder), you can use the `routes.include` option to add additional routes to the `include` array.

### `routes.exclude: string[]`

default `[]`

If you want to use the automatic `_routes.json` generation, but want to exclude additional routes, you can use the `routes.exclude` option to add additional routes to the `exclude` array.

The following example automatically generates `_routes.json` while including and excluding additional routes. Note that that is only necessary if you have custom functions in the `functions` folder that are not handled by Astro.

```js ins={5-9}
// astro.config.mjs
export default defineConfig({
    adapter: cloudflare({
        mode: 'directory',
        routes: {
            strategy: 'include',
            include: ['/users/*'], // handled by custom function: functions/users/[id].js
            exclude: ['/users/faq'], // handled by static page: pages/users/faq.astro
        },
    }),
});
```

### `runtime: "off" | "local" | "remote"`

default `"off"`

This optional flag enables the Astro dev server to support all [Cloudflare bindings](https://developers.cloudflare.com/pages/platform/functions/bindings), [environment variables](https://developers.cloudflare.com/pages/platform/functions/bindings/#environment-variables), and the [cf object](https://developers.cloudflare.com/workers/runtime-apis/request/#incomingrequestcfproperties), avoiding the need for Wrangler. [Read more](#access-to-the-cloudflare-runtime)

- `local`: uses bindings mocking and locally static files
- `remote`: uses real remote bindings and a live fetched cf object
- `off`: legacy behaviour, use [`wrangler`](https://developers.cloudflare.com/workers/wrangler/) to preview with runtime

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
  output: 'server',
  adapter: cloudflare({
    runtime: 'off' | 'local' | 'remote',
  }),
});
```

### `wasmModuleImports: boolean`

default: `false`

Whether or not to import `.wasm` files [directly as ES modules](https://github.com/WebAssembly/esm-integration/tree/main/proposals/esm-integration), using the `.wasm?module` import syntax.

Add `wasmModuleImports: true` to `astro.config.mjs` to enable in both the Cloudflare build and the Astro dev server. [Read more](#use-wasm-modules)

```diff
// astro.config.mjs
import {defineConfig} from "astro/config";
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
    adapter: cloudflare({
+       wasmModuleImports: true
    }),
    output: 'server'
})
```

## Access to the Cloudflare runtime

You can access the runtime ([Cloudflare bindings](https://developers.cloudflare.com/pages/platform/functions/bindings), [environment variables](https://developers.cloudflare.com/pages/platform/functions/bindings/#environment-variables), and the [cf object](https://developers.cloudflare.com/workers/runtime-apis/request/#incomingrequestcfproperties)) from Astro components and API routes through `Astro.locals`.

Currently supported bindings:

- [D1](https://developers.cloudflare.com/d1/)

### Access

If you're inside an `.astro` file, you access the runtime using the `Astro.locals` global:

```astro
---
// src/pages/index.astro
const runtime = Astro.locals.runtime;
---
<div></div>
```

From an endpoint:

```js
// src/pages/api/someFile.js
export function GET(context) {
  const runtime = context.locals.runtime;

  return new Response('Some body');
}
```

### Typing

If you're using the `advanced` mode, you can type the `runtime` object as following:

```ts
// src/env.d.ts
/// <reference types="astro/client" />
type AdvancedRuntime = import('@astrojs/cloudflare').AdvancedRuntime;

type ENV = {
  SERVER_URL: string;
};

declare namespace App {
  interface Locals extends AdvancedRuntime<ENV> {
    user: {
      name: string;
      surname: string;
    };
  }
}
```

If you're using the `directory` mode, you can type the `runtime` object as following:

```ts
// src/env.d.ts
/// <reference types="astro/client" />
type DirectoryRuntime = import('@astrojs/cloudflare').DirectoryRuntime;

type ENV = {
  SERVER_URL: string;
};

declare namespace App {
  interface Locals extends DirectoryRuntime<ENV> {
    user: {
      name: string;
      surname: string;
    };
  }
}
```

## Headers, Redirects and function invocation routes

Cloudflare has support for adding custom [headers](https://developers.cloudflare.com/pages/platform/headers/), configuring static [redirects](https://developers.cloudflare.com/pages/platform/redirects/) and defining which routes should [invoke functions](https://developers.cloudflare.com/pages/platform/functions/routing/#function-invocation-routes). Cloudflare looks for `_headers`, `_redirects`, and `_routes.json` files in your build output directory to configure these features. This means they should be placed in your Astro project’s `public/` directory.

### Custom `_routes.json`

By default, `@astrojs/cloudflare` will generate a `_routes.json` file with `include` and `exclude` rules based on your applications's dynamic and static routes.
This will enable Cloudflare to serve files and process static redirects without a function invocation. Creating a custom `_routes.json` will override this automatic optimization. See [Cloudflare's documentation](https://developers.cloudflare.com/pages/platform/functions/routing/#create-a-_routesjson-file) for more details.

## Use Wasm modules

The following is an example of importing a Wasm module that then responds to requests by adding the request's number parameters together.

```javascript
// pages/add/[a]/[b].js
import mod from '../util/add.wasm?module';

// instantiate ahead of time to share module
const addModule: any = new WebAssembly.Instance(mod);

export async function GET(context) {
  const a = Number.parseInt(context.params.a);
  const b = Number.parseInt(context.params.b);
  return new Response(`${addModule.exports.add(a, b)}`);
}
```

While this example is trivial, Wasm can be used to accelerate computationally intensive operations which do not involve significant I/O such as embedding an image processing library.

## Node.js compatibility

Astro's Cloudflare adapter allows you to use any Node.js runtime API supported by Cloudflare:

- assert
- AsyncLocalStorage
- Buffer
- Diagnostics Channel
- EventEmitter
- path
- process
- Streams
- StringDecoder
- util

To use these APIs, your page or endpoint must be server-side rendered (not pre-rendered) and must use the the `import {} from 'node:*'` import syntax.

```js
// pages/api/endpoint.js
export const prerender = false;
import { Buffer } from 'node:buffer';
```

Additionally, you'll need to enable the Compatibility Flag in Cloudflare. The configuration for this flag may vary based on where you deploy your Astro site. For detailed guidance, please refer to the [Cloudflare documentation](https://developers.cloudflare.com/workers/runtime-apis/nodejs).

## Troubleshooting

For help, check out the `#support` channel on [Discord](https://astro.build/chat). Our friendly Support Squad members are here to help!

You can also check our [Astro Integration Documentation][astro-integration] for more on integrations.

## Contributing

This package is maintained by Astro's Core team. You're welcome to submit an issue or PR!

[astro-integration]: https://docs.astro.build/en/guides/integrations-guide/
