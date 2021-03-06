First, read [Getting Started](../getting-started.md).

## Building a TerriaMap against a modified TerriaJS

What if you need to make changes to [TerriaJS](https://github.com/TerriaJS/terriajs) while working on a site that depends on it?

In the process described in [Getting Started](../getting-started.md), the [TerriaJS package](https://www.npmjs.com/package/terriajs) is installed to the `node_modules` directory by `npm install`.  Please do not edit TerriaJS directly in the `node_modules` directory, because changes will be clobbered the next time you run `npm install`.

Instead, we want to clone TerriaJS from its [GitHub repo](https://github.com/TerriaJS/terriajs) and use that in our TerriaMap build.  Traditionally, `npm link` is the way to do this.  However, we do not recommend use of `npm link` because it frequently leads to multiple copies of some libraries being installed, which in turn leads to all sorts of frustrating build problems.  Instead, we recommend [npmgitdev](https://www.npmjs.com/package/npmgitdev).  `npmgitdev` lets us safely clone a git repo into our `node_modules` directory and use it pretty much as if npm had put it there itself.

First, install `npmgitdev` globally:

```
npm install -g npmgitdev
```

Then, in your TerriaMap directory, remove the existing `terriajs` package, and clone the terriajs repo there instead.

```
cd node_modules
rm -rf terriajs
git clone https://github.com/TerriaJS/terriajs.git
cd ..
```

This will give you the `master` branch of TerriaJS.  While we strive to keep `master` stable and usable at all times, you must be aware that `master` is less tested than actual releases, and it may not be commpatible with the `master` branch of TerriaMap.  So, you may want to check out the actual version of TerriaJS that you're using before you start making changes.  To do that:

```
grep terriajs package.json
# will print something like: "terriajs": "^4.5.0"
cd node_modules/terriajs
git checkout 4.5.0
cd ../..
```

Now, if you run `npm install`, `npm` will fail with an error like this:

```
npm ERR! git C:\github\TerriaMap\node_modules\terriajs: Appears to be a git repo or submodule.
npm ERR! git     C:\github\TerriaMap\node_modules\terriajs
npm ERR! git Refusing to remove it. Update manually,
npm ERR! git or move it out of the way first.
```

This is _good_!  It means `npm` recognizes that your `terriajs` directory is now a git repo, and it is refusing to touch it out of fear of making you lose your work.  This is where `npmgitdev` comes in:

```
npmgitdev install
```

`npmgitdev` is a wrapper around `npm` that, for each git repo in `node_modules`:

* Makes sure your working directory is clean (i.e. you have no uncommitted changes).
* Moves the `.git` directory out of the way so that `npm` can't see it or clobber it.
* Copies all the `devDependencies` in `package.json` into `dependencies`.  This way npm will install all your dev-time stuff too.
* Runs `npm` normally.
* Moves the `.git` directory back.

!!! note

	If you hit CTRL-C while `npmgitdev` is running, it will be unable to clean up after itself automatically.  Look for a `npmgitdev-XXXXX` directory.  The `mappings.json` file in that directory contains a record of the changes that `npmgitdev` made so that you can undo them manually.  Usually this just means moving the `.git` directory back.

Now, we can edit TerriaJS in `node_modules/terriajs` with the benefit of a full-feature git repo.

To switch TerriaMap back to using the npm version of TerriaJS (instead of the git repo), do:

```
# warning: make sure you don't need any of your changes to TerriaJS first!
rm -rf node_modules/terriajs
npmgitdev install
```

## Committing modifications

If you make changes to TerriaJS and TerriaMap together, here's the process for getting them to production.

First, commit your TerriaJS changes to a branch and open a pull request to merge that branch to master. Simultaneously, you may want to make a branch of TerriaMap that uses your modified version of TerriaJS.  To do that, modify TerriaMap's `package.json`.  Where it has a line like:

```
"terriajs": "^4.5.0",
```

Change it to:

```
"terriajs": "git://github.com/TerriaJS/terriajs.git#branchName",
```

Replace `branchName` with the name of the TerriaJS branch you want to use.  You may even use a repository other than `TerriaJS/terriajs` if your branch is in a fork of TerriaJS instead of in the official repository.

Once your TerriaJS pull request has been merged and a new version of the `terriajs` npm module has been published, please remember to update `package.json` to point to an official `terriajs` version instead of a branch in a GitHub repo.

The `package.json` in the `master` branch of TerriaMap should point to official releases of `terriajs` on npm, NOT GitHub branches.

## Documentation

Documentation is automatically generated from the source via JSDoc (reference) and MkDocs (user guide) by running:

```
npm run gulp docs
```

It will be placed in the `wwwroot/doc` folder.

You need a standalone install of MkDocs in order to build the user guide, see [http://www.mkdocs.org/#installation](http://www.mkdocs.org/#installation) for details.

## Tests / Specs

We use [Jasmine](https://jasmine.github.io/) for the TerriaJS tests, called specs in Jasmine parlance.  To run the specs, you first need to build them by running this in the TerriaJS  (not TerriaMap!) directory:

```
npm run gulp
```

And start the development web server by running (also from the TerriaJS and not TerriaMap! directory):

```
npm start
```

The test suite is run by opening a web browser on [http://localhost:3002/SpecRunner.html](http://localhost:3002/SpecRunner.html).  The source code for the specs is found in the `test/` directory.

## TerriaJS Gulp Tasks

Run any of these tasks with `npm run gulp <task name>` from within the TerriaJS directory:

* default - Invoked by running gulp without any arguments, this task invokes the `build` and `lint` tasks.
* `build` - Builds a non-minified version of the TerriaJS tests.  This task may take 10 seconds or more, which is the main reason for the next task.
* `watch` - Starts the same as `build` but then it stays running and watches for changes to any TerriaJS or Cesium source file that was pulled in to the build. When a change to any of these files is detected, a fast incremental build is automatically kicked off.  The incremental build is much faster than the full rebuild because dependencies between source files are cached.
* `release` - The same as `build` except that it also minifies the build tests.
* `lint` - Runs ESLint on the files in the `lib` folder and reports any problems.  The ESLint rules are defined in the `.eslintrc` file in the root directory of TerriaJS.  A stricter set of rules is also find in the `.eslintrc` file in `lib/ReactViews`.
* `docs` - Generates the user guide and reference documentation.  The user guide is served at `http://localhost:3002/doc/guide/` and the reference documentation is at `http://localhost:3002/doc/reference/`.
* `make-schema` - Generates [JSON Schema](http://json-schema.org/) for the TerriaJS [Initialization Files](../customizing/initialization-files.md) from the source code.  The schema is written to `wwwroot/schema`.
* `test` - Detects browsers available on the local system and launches the test suite in each.  The results are reported on the command line.
* `test-electron` - Runs the tests in Electron, a headless (no UI) Chrome-like browser.
* `test-saucelabs` - Runs the tests on a bunch of browsers on [Sauce Labs](https://saucelabs.com/).  You will need to [Set up Sauce Labs](setting-up-saucelabs.md).
* `test-browserstack` - Runs the tests on a bunch of browsers on [BrowserStack](https://www.browserstack.com/).  You will need to set up a BrowserStack account.

See `gulpfile.js` for more gulp tasks.

## TerriaMap Gulp Tasks

Run any of these tasks with `npm run gulp <task name>` from within the TerriaMap directory:

* default - Invoked by running gulp without any arguments, this task invokes the `build` and `lint` tasks.
* `build` - Builds a non-minified version of TerriaMap, TerriaJS, Cesium, and all other dependencies, together in one JS file (called `wwwroot/build/TerriaMap.js`). Only the parts of TerriaJS and Cesium that we use (directly or indirectly) are pulled in.  Web Workers, CSS, and other resources are also built by this task.  This task may take 10 seconds or more, which is the main reason for the next task.
* `watch` - Starts the same as `build` but then it stays running and watches for changes to any TerriaMap, TerriaJS, or Cesium resource. When a change to any of these files is detected, a fast incremental build is automatically kicked off.  The incremental build is much faster than the full rebuild because dependencies between source files are cached.
* `release` - The same as `build` except that it also minifies the built JavaScript files.  This task should be used when building for production.
* `lint` - Runs ESLint on `index.js` and the files in the `lib` folder and reports any problems.  The ESLint rules are defined in the `.eslintrc` file in the root directory of TerriaMap.
* `make-package` - Creates a `.tar.gz` package in `deploy/packages` from the current build.  This package can be copied to another machine to run the application there.  The arguments are:

| Argument | Description |
|----------|-------------|
| `--packageName <name>` | The name of the package.  If not specified, the name is `<npm_package_name>-<git_describe>`, where `<npm_package_name>` is the value of the `npm_package_name` environment variable and `<git_describe>` is the output of running `git describe`.  If you invoke this task using `npm run gulp make-package` instead of simply `gulp make-package`, the `npm_package_name` environment variable will be automatically set to the name of the project in `package.json`. |
| `--serverConfigOverride <file>` | The path to a file with overrides of the `devserverconfig.json` file.  If not specified, `devserverconfig.json` is used unmodified. |
| `--clientConfigOverride <file>` | The path to a file with overrides of the `wwwroot/config.json` file.  If not specified, `wwwroot/config.json` is used unmodified. |

* `clean` - Removes the `wwwroot/build` directory.
* `sync-terriajs-dependencies` - For all npm packages used by both TerriaMap and TerriaJS, updates TerriaMap's `package.json` to use the same version as TerriaJS.  This avoids build problems (errors, hangs) caused by package version conflicts.