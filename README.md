# WRAPPER FOR Fork ts checker webpack plugin
This wrapper removes the blocker for loading the tsconfig.json file only once, and loads it every iteration. Ugly fix for people with tsconfig problems with the origional package. All creed to the origional package described below!


# Fork TS Checker Webpack Plugin
[![Npm version](https://img.shields.io/npm/v/fork-ts-checker-webpack-plugin.svg?style=flat-square)](https://www.npmjs.com/package/fork-ts-checker-webpack-plugin)
[![Build Status](https://travis-ci.org/Realytics/fork-ts-checker-webpack-plugin.svg?branch=master)](https://travis-ci.org/realytics/fork-ts-checker-webpack-plugin)

Webpack plugin that runs typescript type checker on a separate process.
 
## Installation
This plugin requires minimum **webpack 2.3**, **typescript 2.1** and optionally **tslint 5.0**
```sh
npm install --save-dev fork-ts-checker-webpack-plugin
```
Basic webpack config (with [ts-loader](https://github.com/TypeStrong/ts-loader))
```js
var ForkTsCheckerWebpackPlugin = require('fork-ts-checker-webpack-plugin');

var webpackConfig = {
  context: __dirname, // to automatically find tsconfig.json
  entry: './src/index.ts',
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        loader: 'ts-loader',
        options: {
          // disable type checker - we will use it in fork plugin
          transpileOnly: true 
        }
      }
    ]
  },
  plugins: [
    new ForkTsCheckerWebpackPlugin()
  ]
};
```

## Motivation
There is already similar solution - [awesome-typescript-loader](https://github.com/s-panferov/awesome-typescript-loader). You can
add `CheckerPlugin` and delegate checker to the separate process. The problem with `awesome-typescript-loader` was that, in our case,
it was a lot slower than [ts-loader](https://github.com/TypeStrong/ts-loader) on an incremental build (~20s vs ~3s).
Secondly, we use [tslint](https://palantir.github.io/tslint) and we wanted to run this, along with type checker, in a separate process.
This is why we've created this plugin. To provide better performance, plugin reuses Abstract Syntax Trees between compilations and shares 
these trees with tslint. It can be scaled with a multi-process mode to utilize maximum CPU power.

## Modules resolution
It's very important to be aware that **this plugin uses [typescript](https://github.com/Microsoft/TypeScript)'s, not 
[webpack](https://github.com/webpack/webpack)'s modules resolution**. It means that you have to setup `tsconfig.json` correctly. For example 
if you set `files: ['./src/someFile.ts']` in `tsconfig.json`, this plugin will check only `someFile.ts` for semantic errors. It's because 
of performance. The goal of this plugin is to be *as fast as possible*. With typescript's module resolution we don't have to wait for webpack 
to compile files (which traverses dependency graph during compilation) - we have a full list of files from the begin.

To debug typescript's modules resolution, you can use `tsc --traceResolution` command.

## TSLint
If you have installed [tslint](https://palantir.github.io/tslint), you can enable it by setting `tslint: true` or 
`tslint: './path/to/tslint.json'`. We recommend changing `defaultSeverity` to a `"warning"` in `tslint.json` file. 
It helps to distinguish lints from typescript's diagnostics.

## Options
* **tsconfig** `string`:
Path to *tsconfig.json* file. Default: `path.resolve(compiler.options.context, './tsconfig.json')`.

* **tslint** `string | true`: 
Path to *tslint.json* file or `true`. If `true`, uses `path.resolve(compiler.options.context, './tslint.json')`. Default: `undefined`.

* **watch** `string | string[]`: 
Directories or files to watch by service. Not necessary but improves performance (reduces number of `fs.stat` calls).

* **async** `boolean`:
True by default - `async: false` can block webpack's emit to wait for type checker/linter and to add errors to the webpack's compilation.
We recommend to use it in projects where type checking is faster than webpack's build - it's better for integration with other plugins.

* **ignoreDiagnostics** `number[]`:
List of typescript diagnostic codes to ignore.

* **ignoreLints** `string[]`: 
List of tslint rule names to ignore.

* **colors** `boolean`:
If `false`, disables built-in colors in logger messages. Default: `true`.

* **logger** `object`:
Logger instance. It should be object that implements method: `error`, `warn`, `info`. Default: `console`.

* **formatter** `'default' | 'codeframe' | ((message: NormalizedMessage, useColors: boolean) => string)`:
Formatter for diagnostics and lints. By default uses `default` formatter. You can also pass your own formatter as a function
(see `src/NormalizedMessage.js` and `src/formatter/` for api reference).

* **formatterOptions** `object`:
Options passed to formatters (currently only `codeframe` - see [available options](https://www.npmjs.com/package/babel-code-frame#options))

* **silent** `boolean`:
If `true`, logger will not be used. Default: `false`.

* **checkSyntacticErrors** `boolean`: 
This option is useful if you're using ts-loader in `happyPackMode` with [HappyPack](https://github.com/amireh/happypack) or [thread-loader](https://github.com/webpack-contrib/thread-loader) to parallelise your builds.  It will ensure that the plugin checks for both syntactic errors (eg `const array = [{} {}];`) and semantic errors (eg `const x: number = '1';`).  By default the plugin only checks for semantic errors.  This is because when ts-loader is used in `transpileOnly` mode, ts-loader will still report syntactic errors. When used in `happyPackMode` it does not. Default: `false`.

* **memoryLimit** `number`: 
Memory limit for service process in MB. If service exits with allocation failed error, increase this number. Default: `2048`.

* **workers** `number`:
You can split type checking to a few workers to speed-up increment build. **Be careful** - if you don't want to increase build time, you 
should keep free 1 core for *build* and 1 core for a *system* *(for example system with 4 CPUs should use max 2 workers)*. Second thing -
node doesn't share memory between workers - keep in mind that memory usage will increase. Be aware that in some scenarios increasing workers
number **can increase checking time**. Default: `ForkTsCheckerWebpackPlugin.ONE_CPU`.

### Pre-computed consts:      
  * `ForkTsCheckerWebpackPlugin.ONE_CPU` - always use one CPU
  * `ForkTsCheckerWebpackPlugin.ALL_CPUS` - always use all CPUs (will increase build time)
  * `ForkTsCheckerWebpackPlugin.ONE_CPU_FREE` - leave only one CPU for build (probably will increase build time)
  * `ForkTsCheckerWebpackPlugin.TWO_CPUS_FREE` - **recommended** - leave two CPUs free (one for build, one for system)

## Different behaviour in watch mode

If you turn on [webpacks watch mode](https://webpack.js.org/configuration/watch/#watch) the `fork-ts-checker-notifier-webpack-plugin` will take care of logging type errors, _not_ webpack itself. That means if you set `silent: true` you won't see type errors in your console in watch mode.

You can either set `silent: false` to show the logging from `fork-ts-checker-notifier-webpack-plugin` _or_ set `async: false`. Now webpack itself will log type errors again, but note that this can slow down your builds depending on the size of your project.

## Notifier

You may already be using the excellent [webpack-notifier](https://github.com/Turbo87/webpack-notifier) plugin to make build failures more obvious in the form of system notifications. There's an equivalent notifier plugin designed to work with the `fork-ts-checker-webpack-plugin`.  It is the `fork-ts-checker-notifier-webpack-plugin` and can be found [here](https://github.com/johnnyreilly/fork-ts-checker-notifier-webpack-plugin). This notifier deliberately has a similar API as the `webpack-notifier` plugin to make migration easier.

## Known Issue Watching Non-Emitting Files

At present there is an issue with the plugin regarding the triggering of type-checking when a change is made in a source file that will not emit js. If you have a file which contains only `interface`s and / or `type`s then changes to it will **not** trigger the type checker whilst in watch mode. Sorry about that.

We hope this will be resolved in future; the issue can be tracked [here](https://github.com/Realytics/fork-ts-checker-webpack-plugin/issues/36).

## Plugin Hooks
This plugin provides some custom webpack hooks (all are sync):

| Event name | Description | Params |
|------------|-------------|--------|
|`fork-ts-checker-cancel`| Cancellation has been requested | `cancellationToken` |
|`fork-ts-checker-waiting`| Waiting for results | `hasTsLint` |
|`fork-ts-checker-service-before-start`| Async plugin that can be used for delaying `fork-ts-checker-service-start` | - |
|`fork-ts-checker-service-start`| Service will be started | `tsconfigPath`, `tslintPath`, `watchPaths`, `workersNumber`, `memoryLimit` |
|`fork-ts-checker-service-start-error` | Cannot start service | `error` |
|`fork-ts-checker-service-out-of-memory`| Service is out of memory | - |
|`fork-ts-checker-receive`| Plugin receives diagnostics and lints from service | `diagnostics`, `lints` | 
|`fork-ts-checker-emit`| Service will add errors and warnings to webpack compilation ('build' mode) | `diagnostics`, `lints`, `elapsed` |
|`fork-ts-checker-done`| Service finished type checking and webpack finished compilation ('watch' mode) | `diagnostics`, `lints`, `elapsed` |

## License
MIT
