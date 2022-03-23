<a name="user-content-fastify-webpack-hot"></a>
<a name="fastify-webpack-hot"></a>
# fastify-webpack-hot 🔥

[![NPM version](http://img.shields.io/npm/v/fastify-webpack-hot.svg?style=flat-square)](https://www.npmjs.org/package/fastify-webpack-hot)
[![Canonical Code Style](https://img.shields.io/badge/code%20style-canonical-blue.svg?style=flat-square)](https://github.com/gajus/canonical)
[![Twitter Follow](https://img.shields.io/twitter/follow/kuizinas.svg?style=social&label=Follow)](https://twitter.com/kuizinas)

A [Fastify](https://github.com/fastify/fastify) plugin for serving files emitted by [Webpack](https://github.com/webpack/webpack) with Hot Module Replacement (HMR).

![fastify-webpack-hot in action](./.README/fastify-webpack-hot.gif)

(`fastify-webpack-hot` immediately propagates hot updates to the server and browser.)

* [fastify-webpack-hot 🔥](#user-content-fastify-webpack-hot)
    * [Recipes](#user-content-fastify-webpack-hot-recipes)
        * [Basic HMR Setup](#user-content-fastify-webpack-hot-recipes-basic-hmr-setup)
        * [Accessing Webpack Stats](#user-content-fastify-webpack-hot-recipes-accessing-webpack-stats)
        * [Accessing Output File System](#user-content-fastify-webpack-hot-recipes-accessing-output-file-system)
        * [Compressing Response](#user-content-fastify-webpack-hot-recipes-compressing-response)
    * [Project Setup Examples](#user-content-fastify-webpack-hot-project-setup-examples)
    * [Difference from `webpack-dev-server` and `webpack-hot-middleware`](#user-content-fastify-webpack-hot-difference-from-webpack-dev-server-and-webpack-hot-middleware)
    * [Troubleshooting](#user-content-fastify-webpack-hot-troubleshooting)
        * [Browser Logging](#user-content-fastify-webpack-hot-troubleshooting-browser-logging)
        * [Node.js Logging](#user-content-fastify-webpack-hot-troubleshooting-node-js-logging)


<a name="user-content-fastify-webpack-hot-recipes"></a>
<a name="fastify-webpack-hot-recipes"></a>
## Recipes

<a name="user-content-fastify-webpack-hot-recipes-basic-hmr-setup"></a>
<a name="fastify-webpack-hot-recipes-basic-hmr-setup"></a>
### Basic HMR Setup

All you need to enable [Webpack Hot Module Replacement](https://webpack.js.org/api/hot-module-replacement/) is:

* Register `fastify-webpack-hot` Fastify plugin
* Enable `HotModuleReplacementPlugin` Webpack plugin
* Prepend `fastify-webpack-hot/client` entry script

Example:

```ts
import webpack from 'webpack';
import {
  fastifyWebpackHot,
} from 'fastify-webpack-hot';

const compiler = webpack({
  entry: [
    'fastify-webpack-hot/client',
    path.resolve(__dirname, '../app/main.js'),
  ],
  mode: 'development',
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
  ],
});

void app.register(fastifyWebpackHot, {
  compiler,
});

```

For more thorough instructions, refer to the [Project Setup Examples](#user-content-project-setup-examples).

<a name="user-content-fastify-webpack-hot-recipes-accessing-webpack-stats"></a>
<a name="fastify-webpack-hot-recipes-accessing-webpack-stats"></a>
### Accessing Webpack Stats

[Stats](https://webpack.js.org/configuration/stats/) instance is accessible under `request.webpack.stats`:

```ts
app.get('*', async (request, reply) => {
  const stats = request.webpack.stats.toJson({
    all: false,
    entrypoints: true,
  });

  // ...
);
```

The most common use case for accessing stats is for identifying and constructing the entrypoint assets, e.g.

```ts
for (const asset of stats.entrypoints?.main.assets ?? []) {
  if (asset.name.endsWith('.js')) {
    htmlBody +=
      '<script defer="defer" src="/' + asset.name + '"></script>\n';
  }
}
```

<a name="user-content-fastify-webpack-hot-recipes-accessing-output-file-system"></a>
<a name="fastify-webpack-hot-recipes-accessing-output-file-system"></a>
### Accessing Output File System

You can access Output File System by referencing `compiler.outputFileSystem`. However, this will have the type of `OutputFileSystem`, which is incompatible with [`memfs`](https://npmjs.com/package/memfs), which is used by this package. Therefore, a better way to access `outputFileSystem` is by referencing `request.webpack.outputFileSystem`:

```ts
app.get('*', async (request, reply) => {
  const stats = JSON.parse(
    await request.webpack.outputFileSystem.promises.readFile(
      path.join(__dirname, '../dist/stats.json'),
      'utf8'
    )
  );

  // ...
);
```

This example shows how you would access `stats.json` generated by [`webpack-stats-plugin`](https://www.npmjs.com/package/webpack-stats-plugin).

Note: You likely won't need to use this because `fastify-webpack-hot` automatically detects which assets have been generated and serves them at `output.publicPath`.

<a name="user-content-fastify-webpack-hot-recipes-compressing-response"></a>
<a name="fastify-webpack-hot-recipes-compressing-response"></a>
### Compressing Response

This plugin is compatible with [`compression-webpack-plugin`](https://www.npmjs.com/package/compression-webpack-plugin), i.e. This plugin will serve compressed files if the following conditions are true:

* Your outputs include compressed file versions (either `.br` or `.gz`)
* Request includes a matching `accept-encoding` header

Example `compression-webpack-plugin` configuration:

```ts
new CompressionPlugin({
  algorithm: 'brotliCompress',
  deleteOriginalAssets: false,
  filename: '[path][base].br',
  compressionOptions: {
    level: zlib.constants.BROTLI_MIN_QUALITY,
  },
  minRatio: 0.8,
  test: /\.(js|css|html|svg)$/,
  threshold: 10_240,
})
```

<a name="user-content-fastify-webpack-hot-project-setup-examples"></a>
<a name="fastify-webpack-hot-project-setup-examples"></a>
## Project Setup Examples

These are complete project setup examples that you can run locally to evaluate `fastify-webpack-hot` plugin:

* [TypeScript, Fastify and Webpack HRM example](./examples/webpack) (uses [Webpack Hot Module Replacement API](https://webpack.js.org/api/hot-module-replacement/))
* [TypeScript, Fastify, Webpack and React HRM example](./examples/react) (uses [`ReactRefreshWebpackPlugin`](https://github.com/pmmmwh/react-refresh-webpack-plugin))

<a name="user-content-fastify-webpack-hot-difference-from-webpack-dev-server-and-webpack-hot-middleware"></a>
<a name="fastify-webpack-hot-difference-from-webpack-dev-server-and-webpack-hot-middleware"></a>
## Difference from <code>webpack-dev-server</code> and <code>webpack-hot-middleware</code>

`webpack-dev-server` and `webpack-hot-middleware` were built for [express](https://npmjs.com/package/express) framework and as such they require compatibility plugins to work with Fastify. Additionally, both libraries are being maintained with intent to support legacy webpack versions (all the way to webpack v1). As a result, they contain a lot of bloat that makes them slower and harder to maintain.

`fastify-webpack-hot` is built from the ground up leveraging the latest APIs of Fastify and webpack, and it encompasses functionality of both libraries. It is faster and easier to maintain.

<a name="user-content-fastify-webpack-hot-troubleshooting"></a>
<a name="fastify-webpack-hot-troubleshooting"></a>
## Troubleshooting

<a name="user-content-fastify-webpack-hot-troubleshooting-browser-logging"></a>
<a name="fastify-webpack-hot-troubleshooting-browser-logging"></a>
### Browser Logging

This project uses [`roarr`](https://www.npmjs.com/package/roarr) logger to output the application's state.

In order to output logs in browser, you need to provide output interface. The easiest way of doing it is by including [`@roarr/browser-log-writer`](https://github.com/gajus/roarr-browser-log-writer) in your project. 

```ts
import '@roarr/browser-log-writer/init';
```

Afterwards, to output all logs set `ROARR_LOG=true` in `localStorage`:

```ts
localStorage.setItem('ROARR_LOG', 'true');
```

Note: Ensure that you have enabled verbose logs in DevTools to see all `fastify-webpack-hot` logs.

<a name="user-content-fastify-webpack-hot-troubleshooting-node-js-logging"></a>
<a name="fastify-webpack-hot-troubleshooting-node-js-logging"></a>
### Node.js Logging

This project uses [`roarr`](https://www.npmjs.com/package/roarr) logger to output the program's state.

Export `ROARR_LOG=true` environment variable to enable log printing to `stdout`.

Use [`roarr-cli`](https://github.com/gajus/roarr-cli) program to pretty-print the logs.