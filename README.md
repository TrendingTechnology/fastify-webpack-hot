<a name="user-content-fastify-webpack-hot"></a>
<a name="fastify-webpack-hot"></a>
# fastify-webpack-hot 🔥

[![NPM version](http://img.shields.io/npm/v/fastify-webpack-hot.svg?style=flat-square)](https://www.npmjs.org/package/fastify-webpack-hot)
[![Canonical Code Style](https://img.shields.io/badge/code%20style-canonical-blue.svg?style=flat-square)](https://github.com/gajus/canonical)
[![Twitter Follow](https://img.shields.io/twitter/follow/kuizinas.svg?style=social&label=Follow)](https://twitter.com/kuizinas)

A [Fastify](https://github.com/fastify/fastify) plugin for serving files emitted by [Webpack](https://github.com/webpack/webpack) with Hot Module Replacement (HMR).

<a name="user-content-fastify-webpack-hot-basic-hmr-setup"></a>
<a name="fastify-webpack-hot-basic-hmr-setup"></a>
## Basic HMR Setup

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

<a name="user-content-fastify-webpack-hot-examples"></a>
<a name="fastify-webpack-hot-examples"></a>
## Examples

* [TypeScript, Fastify and Webpack HRM example](./examples/webpack) (uses [Webpack Hot Module Replacement API](https://webpack.js.org/api/hot-module-replacement/))
* [TypeScript, Fastify, Webpack and React HRM example](./examples/react) (uses [`ReactRefreshWebpackPlugin`](https://github.com/pmmmwh/react-refresh-webpack-plugin))

<a name="user-content-fastify-webpack-hot-recipes"></a>
<a name="fastify-webpack-hot-recipes"></a>
## Recipes

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

<a name="user-content-fastify-webpack-hot-response-compression"></a>
<a name="fastify-webpack-hot-response-compression"></a>
## Response Compression

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

Note: You may also try using `fastify-compress`, however, beware of the outstanding issue that may cause the server to crash ([fastify-compress#215](https://github.com/fastify/fastify-compress/issues/215)).

<a name="user-content-fastify-webpack-hot-difference-from-webpack-dev-server"></a>
<a name="fastify-webpack-hot-difference-from-webpack-dev-server"></a>
## Difference from webpack-dev-server

* Supports [Hot Module Replacement](https://webpack.js.org/concepts/hot-module-replacement).
* Does not allow to override default HTTP methods (GET, HEAD).
* Does not allow to provide custom headers.
* Does not allow to create an index.
* Does not support [`serverSideRender`](https://github.com/webpack/webpack-dev-middleware#serversiderender)
* Does not support [`writeToDisk`](https://github.com/webpack/webpack-dev-middleware#writetodisk)
* Does not support [`MultiCompiler`](https://webpack.js.org/api/node/#multicompiler)
* Does not support [`Accept-Ranges`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Ranges)

All of the above are relatively straightforward to implement, however, I didn't have a use-case for them. If you have a use-case, please raise a PR.

<a name="user-content-fastify-webpack-hot-debugging"></a>
<a name="fastify-webpack-hot-debugging"></a>
## Debugging

This project uses [`roarr`](https://www.npmjs.com/package/roarr) logger to output the program's state.

Export `ROARR_LOG=true` environment variable to enable log printing to `stdout`.

Use [`roarr-cli`](https://github.com/gajus/roarr-cli) program to pretty-print the logs.