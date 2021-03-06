# Fork TS Checker Webpack Plugin
[![Npm version](https://img.shields.io/npm/v/fork-ts-checker-webpack-plugin.svg?style=flat-square)](https://www.npmjs.com/package/fork-ts-checker-webpack-plugin)
[![Build Status](https://travis-ci.org/Realytics/fork-ts-checker-webpack-plugin.svg?branch=master)](https://travis-ci.org/realytics/fork-ts-checker-webpack-plugin)

Webpack plugin that runs typescript type checker (with optional linter) on separate processes.
 
## Installation ##
This plugin requires minimum **webpack 2**, **typescript 2.1** and optionally **tslint 5.0**
```sh
npm install --save fork-ts-checker-webpack-plugin
```
Basic webpack config (with [ts-loader](https://github.com/TypeStrong/ts-loader))
```js
var ForkTsCheckerWebpackPlugin = require('fork-ts-checker-webpack-plugin');

var webpackConfig = {
  context: __dirname, // to automatically find tsconfig.json
  entry: './src/index.ts',
  output: {
    path: 'dist',
    filename: 'index.js'
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        include: './src',
        loader: 'ts-loader',
        options: {
          // disable type checker - we will use it in fork plugin
          transpileOnly: true 
        }
      }
    ]
  },
  plugins: [
    new ForkTsCheckerWebpackPlugin({
      watch: './src' // optional but improves performance (less stat calls)
    })
  ]
};
```

## Motivation ##
There is already similar solution - [awesome-typescript-loader](https://github.com/s-panferov/awesome-typescript-loader). You can
add `CheckerPlugin` and delegate checker to the separate process. The problem with `awesome-typescript-loader` is that it's a lot slower 
than [ts-loader](https://github.com/TypeStrong/ts-loader) on incremental build in our case (~20s vs ~3s).
Secondly, we use [tslint](https://palantir.github.io/tslint/) and we wanted to run this also on separate process.
This is why we've created this plugin. The performance is great because of reusing Abstract Syntax Trees between compilations and sharing 
these trees with tslint. We can also scale checker with multi-process mode - it will split work between processes to utilize maximum cpu 
power.

## Options ##
**tsconfig** `string` - Path to tsconfig.json file. If not set, plugin will use `path.resolve(compiler.options.context, './tsconfig.json')`.

**tslint** `string | false` - Path to tslint.json file. If not set, plugin will use `path.resolve(compiler.options.context, './tslint.json')`. 
                            If `false`, disables tslint.

**watch** `string | string[]` - Directories or files to watch by service. Not necessary but improves performance 
                                (reduces number of `fs.stat` calls).
                                  
**blockEmit** `boolean` - If `true`, plugin will block emit until check will be done. It's good setting for ci/production build because 
                          webpack will return code != 0 if there are type/lint errors. Default: `false`. 

**ignoreDiagnostics** `number[]` - List of typescript diagnostic codes to ignore.

**ignoreLints** `string[]` - List of tslint rule names to ignore.

**colors** `boolean` - If `false`, disables built-in colors in logger messages. Default: `true`.

**logger** `object` - Logger instance. It should be object that implements method: `error`, `warn`, `info`. Default: `console`.

**silent** `boolean` - If `true`, logger will not be used. Default: `false`.

**workers** `number` - You can split type checking to few workers to speed-up on increment build. 
                       **Be careful** - if you don't want to increase build time, you should keep 1 core for *build* and 1 core for 
                       *system* free *(for example system with 4 cpu threads should use max 2 workers)*. 
                       Second thing - node doesn't share memory between workers so keep in mind that memory usage will increase 
                       linearly. If you want to use workers, please experiment with workers number. In some scenarios increasing this number 
                       **can increase check time** (and of course memory consumption).
                       Default: `ForkTsCheckerWebpackPlugin.ONE_CPU`.

Pre-computed consts:      
  * `ForkTsCheckerWebpackPlugin.ONE_CPU` - always use one cpu (core)
  * `ForkTsCheckerWebpackPlugin.ONE_CPU_FREE` - leave only one cpu for build (probably will increase build time)
  * `ForkTsCheckerWebpackPlugin.TWO_CPUS_FREE` - leave two cpus free (one for build, one for system)

**memoryLimit** `number` - Memory limit for service process in MB. If service exits with allocation failed error, increase this number.
                           Default: `2048`.

## Plugin Hooks ##
This plugin provides some custom webpack hooks (all are sync):

| Event name | Description | Params |
|------------|-------------|--------|
|`fork-ts-checker-cancel`| Cancellation has been requested | `cancellationToken` |
|`fork-ts-checker-waiting`| Waiting for results | `hasTsLint` |
|`fork-ts-checker-service-start`| Service will be started | `tsconfigPath`, `tslintPath`, `watchPaths`, `workersNumber`, `memoryLimit` |
|`fork-ts-checker-service-out-of-memory`| Service is out of memory | - |
|`fork-ts-checker-receive`| Plugin receives diagnostics and lints from service | `diagnostics`, `lints` | 
|`fork-ts-checker-emit`| Service will add errors and warnings to webpack compilation (`blockEmit: true`) | `diagnostics`, `lints`, `elapsed` |
|`fork-ts-checker-done`| Service finished type checking and webpack finished compilation (`blockEmit: false`) | `diagnostics`, `lints`, `elapsed` |

## License ##
MIT
