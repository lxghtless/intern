# Configuration

<!-- vim-markdown-toc GFM -->

- [Schema](#schema)
- [Environment-specific config](#environment-specific-config)
- [Properties](#properties)
  - [Suite glob expressions](#suite-glob-expressions)
  - [extends](#extends)
- [Displaying config information](#displaying-config-information)
  - [show-config](#show-config)
  - [describe](#describe)
- [Config structure](#config-structure)
- [Sources of configuration information](#sources-of-configuration-information)
  - [Config File](#config-file)
  - [Command line](#command-line)
  - [Environment variable](#environment-variable)
  - [Query args](#query-args)
  - [Programmatically](#programmatically)
- [Configuring loaders](#configuring-loaders)
- [Configuring plugins](#configuring-plugins)
- [Configuration resolution](#configuration-resolution)

<!-- vim-markdown-toc -->

Intern is configured with a standard JavaScript object. This object may contain
properties applicable to either environment that Intern can run in (Node or
browser). Config properties may be set via a file, the command line, browser
query args, or an environment variable. All of these methods use the same basic
syntax and provide the same capabilities. Assuming Intern is being run with the
default [Node runner](running.md#node) or [browser runner](running.md#browser)
and without a `config` argument, Intern will attempt to load configuration
informatioon from an `intern.json` file in the project root.

Wherever config property values come from, the executor will validate and
normalize them into a canonical format. It will perform addition "resolution"
steps just before testing starts, such as expanding suite globs. This allows the
executor’s constructor or `configure` method to be flexible in what data it
accepts. For example, the canonical form of the `environments` property is an
array of objects:

```ts
environments: [{ browserName: 'chrome' }];
```

However, Intern will accept a simple string for the `environments` property and
will expand it into an array of a single object where the `browserName` property
is the given string.

## Schema

Intern includes a
[JSON schema](https://json-schema.org/understanding-json-schema/index.html) for
its config format. This schema describes all the properties that might be
included in an `intern.json` config file, and includes help text for each
property.

To use the schema with an editor such as VS Code, add a `$schema` property to
your `intern.json` file that points to the schema, like:

```json
{
  "$schema": "./node_modules/intern/schemas/config.json"
}
```

The schema is also available at
`https://theintern.io/intern/resources/5/schemas/config.json`.

## Environment-specific config

Tests can run in two basic environments: Node and browsers. By default,
“resource” properties (`suites`, `plugins`, `reporters`, and `loader`) in a
config file apply to both environments. This means that if the same config is
used to run tests in a browser and in Node, the same resources will be loaded in
both environments. In some cases this isn’t desirable because tests may load
application code that depends on environment-specific properties or features,
such as the DOM. Intern’s config provides `node` and `browser` properties for
this use case. These properties specify resources that will only be loaded in
the given environment. The values in these properties will be used in addition
to any corresponding values in the base config. For example, when running in a
Node environment, suites in the base `suites` list as well as in the
`node.suites` list will be executed.

> ⚠️ Note that this is different than the `mode` property in Intern 3, which had
> values of “client” or “runner”. Intern 3’s mode indicated whether tests were
> being run in unit test or functional test mode, but it was sometimes used as
> an environment check due to the fact that functional tests always run in Node.

## Properties

A number of config properties are applicable whether Intern is running in Node
or directly in a browser. Some of the more common ones are listed below.

> 💡 See
> [the API docs](https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/config)
> for a complete list of the available config properties.

| Property                   | Description                                                                             | Default                 |
| :------------------------- | :-------------------------------------------------------------------------------------- | :---------------------- |
| [bail]                     | When true, stop testing after the first failure                                         | `false`                 |
| [baseline]                 | When true, run benchmark tests in baseline mode                                         | `false`                 |
| [benchmark]                | When true, run benchmark tests (if loaded)                                              | `false`                 |
| [debug]                    | If true, display runtime messages to the console                                        | `false`                 |
| [defaultTimeout]           | The time, in ms, before an async test times out                                         | 30000                   |
| [filterErrorStack]         | If true, filter non-application code lines out of stack traces                          | `false`                 |
| [grep]                     | Regular expression used to filter which suites and tests are run                        | `/.*/`                  |
| [loader]                   | An optional loader script and options                                                   | `{ script: 'default' }` |
| [plugins]                  | A list of Intern extensions to load before tests begin                                  | `[]`                    |
| [reporters]                | A list of reporters to use                                                              | `[]`                    |
| [suites]                   | A list of suite paths or [globs](#suite-glob-expressions) to load unit tests from       | `[]`                    |
| [warnOnUnhandledRejection] | If `true` or a regex matching a unhandled rejection, emit a warning instead of an error | `undefined`             |
| [warnOnUncaughtException]  | If `true` or a regex matching an uncaught error, emit a warning instead of an error     | `undefined`             |

Some properties are only meaningful for Node or WebDriver tests:

| Property             | Description                                                                                                                                                      | Default                     |
| :------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------- |
| [capabilities]       | Default capabilities to be used for WebDriver sessions                                                                                                           | `{ 'idle-timeout': 60 }`    |
| [coverage]           | An array of paths or globs to collect coverage data for                                                                                                          | `[]`                        |
| [environments]       | Browser + OS combinations to be tested using WebDriver                                                                                                           | `[]`                        |
| [functionalRetries]  | rerun failed sessions when at least one other session succeeds                                                                                                   | `0`                         |
| [functionalSuites]   | Suites to run in WebDriver mode                                                                                                                                  | `[]`                        |
| [functionalTimeouts] | Timeouts used in functional tests                                                                                                                                | `{ connectTimeout: 30000 }` |
| [leaveRemoteOpen]    | If true, leave remote browsers open after testing has finished                                                                                                   | `false`                     |
| [maxConcurrency]     | The maximum number of sessions to drive concurrently                                                                                                             | `Infinity`                  |
| [serveOnly]          | When true, Intern will start its instrumenting web server but not run tests                                                                                      | `false`                     |
| [serverPort]         | The port the instrumenting server should listen on                                                                                                               | `9000`                      |
| [serverUrl]          | A URL a remote executor can use to reach the local Intern                                                                                                        | `http://localhost:9000`     |
| [tsconfig]           | Optional path to a tsconfig.json for ts-node. Uses the project's by default. Set to `false` to explicitly prevent registering tsnode in projects with TypeScript |                             |
| [tunnel]             | The name of a tunnel to use for WebDriver tests                                                                                                                  | `selenium`                  |
| [tunnelOptions]      | Options to use for the WebDriver tunnel                                                                                                                          | `{ tunnelId: Date.now() }`  |

The environment-specific properties come into play when Intern is running in
that environment:

| Property | Description                                                                            | Default |
| :------- | :------------------------------------------------------------------------------------- | :------ |
| browser  | Resources (loader, plugins, reporters, suites) that only apply to browser tests        | `{}`    |
| node     | Resources (loader, plugins, reporters, suites, tsconfig) that only apply to Node tests | `{}`    |

There are also several properties that are handled by the user interface that
aren’t directly involved in the testing process. These properties are ignored if
set programmatically.

| Property    | Description                                                        |
| :---------- | :----------------------------------------------------------------- |
| description | A short string describing a test config                            |
| [extends]   | Another config or config file that the config extends              |
| help        | Display a help message                                             |
| showConfig  | When true, show the resolved configuration and exit                |
| showConfigs | When true, show information about the currently loaded config file |

### Suite glob expressions

Suites may be specified as file paths or using glob expressions. Globbing is
handled with the [glob](https://github.com/isaacs/node-glob) Node package.

```json5
{
  suites: ['tests/unit/**/*.js', 'tests/integration/foo.js']
}
```

Intern also understands glob-based exclusion using the `!` modifier:

```json5
{
  suites: ['tests/**/*.js', '!tests/functional/**']
}
```

Note that using globs with the browser client _requires_ that Intern’s own
server be used to serve test suites. This is because the browser client on its
own has no way to resolve file system globs; it passes the to the Intern server
running on Node, which resolves them and returns list of file paths.

### extends

If the `extends` property is set in a base config, it must be the path to a
different config file. At run time, the properties from the config file with the
`extends` value will be mixed into the properties from the config file being
extended.

If the `extends` property is set in a child config, it must be the name of a
existing child config, or an array of such names. When a child config extends
multiple other child configs, properties from the right-most config being
extended will override properties from configs to the left.

```json5
{
  configs: {
    a: {
      /* ... */
    },
    b: {
      /* ... */
    },
    c: {
      /* ... */
    },
    d: {
      extends: ['a', 'c']
      /* ... */
    }
  }
}
```

Given the config above, and assuming child config “d” is selected, the following
resolution process will occur:

1.  Child “c” will be mixed into child “a”
2.  Child “d” will be mixed into the result of 1
3.  The result of 2 will be mixed into the base config

## Displaying config information

Intern provides two ways to display information about its configuration:
`describe` and the `--show-config` option.

The `describe` command will display information about a config file. It print
the config files top-level description and a list of any child configs it
contains, along with their descriptions.

The `--show-config` option can be provided with the `run` command. It will
display the fully resolved configuration that would be used to run tests, but it
won't actually run any tests.

### show-config

Passing the `--show-config` option to Intern will cause Intern to dump the
resolved configuration to the console.

```
$ npx intern --show-show
{
    "bail": false,
    "baseline": false,
    "benchmark": false,
    "reporters": [
        {
            "name": "console"
        }
    ]
}
```

A property name can be provided to the option, in which case only that
property’s value will be displayed:

```
$ npx intern run --show-config=baseline
false
```

### describe

The `describe` command will cause Intern to show information about a given
config file. Intern will print the value of the current config file’s
`description` property, and then list all child configs contained in the config
file. For example, with a config file containing the following data:

```json5
{
  description: 'Default test suite',
  configs: {
    webdriver: {
      description: 'Run webdriver tests'
    },
    ci: {
      description: 'Run tests on a CI server'
    }
  }
}
```

running Intern with the `describe` command would display the following text:

```
$ npx intern describe
Default test suite

Configs:
  webdriver  (Run webdriver tests)
  ci         (Run tests on a CI server)
```

Describe may also be passed a `-c` or `--config` option to describe a specific
file.

## Config structure

The config structure is a simple JSON object, so all of its property values must
be serializable. (RegExp objects are serialized to strings.)

```json5
{
  // General properties
  bail: false,
  baseline: false,
  suites: ['tests/unit/*.js'],

  // Browser and node specific resources
  browser: {
    suites: ['tests/unit/dom_stuff.js']
  },
  node: {
    suites: ['tests/unit/dom_stuff.js']
  },

  configs: {
    // Child configs have the same structure as the main config
    ci: {
      bail: true,
      'suites+': ['tests/unit/other.js']
    }
  }
}
```

There are four general sections to a config:

- **General properties**: this includes everything but "browser", "configs", and
  "node"
- **Node-specific resources**: resource properties ("loader", "plugins",
  "reporters", "suites") that apply only to Node environments.
- **Browser-specific resources**: resource properties ("loader", "plugins",
  "reporters", "suites") that apply only to browser environments.
- **Child configs**: named configs in the "configs" object; each of these can
  have any config properties except "configs" (i.e., general properties, Node
  resources, and browser resources).

## Sources of configuration information

When run from the command line or using the browser runner, Intern takes in
configuration data from several sources. In order of increasing precedence, they
are:

1.  [Config file](#config-file)
2.  [Environment variable](#environment-variable)
3.  [Command line or query args](#command-line-or-query-args)
4.  [Programmatically](#programmatically)

Multiple configuration methods may be used during a single run of Intern. The
configuration will be fully resolved before tests are executed.

### Config File

An Intern config file is a JSON file specifying config properties, for example:

```json5
{
  environments: [{ browserName: 'chrome' }],
  suites: ['tests/unit/all.js']
}
```

By default, intern will try to load a file named `intern.json` from the current
working directory. A different config file can be specified by passing a
`config` property to the Node or browser runners.

A child config can be selected by adding `@<child>` to the config file name. For
example, to load a child config named “ci” from the default config file, you
could run:

```sh
$ npx intern --config @ci
```

To load a config named “remote” from a config file named “intern-local.json”,
run:

```sh
$ npx intern --config intern-local.json@remote
```

### Command line

Config properties may be provided as command line options. Option names are
based on config property names, but camelCase property names are kebab-case.

```sh
$ npx intern --grep 'run.*' --no-suites
```

Object values may be input as serialized strings (e.g.,
`-- environments '{"browserName":"chrome"}'`). Array values may be set by
repeating a property (e.g., `--suites "foo.js" --suites "bar.js"`). Array
properties, such as `--suites`, may be set to an empty value with a `no` prefix:
`--no-suites`.

### Environment variable

In a Node environment, Intern may also be configured using an `INTERN_ARGS`
environment variable. This variable may contain a space-separate list of
`<property>=<value>` pairs. For example, these two executions of Intern are
equivalent:

```sh
$ npx intern --grep 'run.*' --leave-remote-open
```

```sh
export INTERN_ARGS="grep=run.* leaveRemoteOpen"
$ npx intern
```

### Query args

Query args work very similarly to command line args. They have the same format,
but with URL query arg separators, and escaping of special characters as
necessary.

```
http://localhost:8080/node_modules/intern/?grep=run.*&suites=
```

### Programmatically

When using Intern programmatically, it may be configured with its `configure`
method.

```ts
intern.configure({ grep: /run.*/, suites: [] });
```

The `configure` method may be called any number of times before the testing
process is started. It accepts a config-like structure -- the object should
contain config properties, but the value types are flexible (like the values
that can be passed on the command line).

## Configuring loaders

Intern uses loader scripts to communicate with various loaders so test suites of
any module format may be used. Loader scripts are simply standalone scripts
(like plugins) that connect Intern with a loader. Intern has a number of [loader
scripts] available to help it work with popular loaders and formats. If no
loader script is defined, then Intern will use the default loader script for the
environment (CommonJS in Node and &lt;script> tag injection in the browser).

To use one of Intern's pre-defined loader scripts, simply specify it's name. The
loader script will expect the loader package to be installed in `node_modules`
using NPM.

```json5
{
  browser: {
    loader: 'dojo'
  }
}
```

Additional options may be provided through the options parameter. These options
are passed through to the registered loader script.

```json5
{
  browser: {
    loader: {
      script: './support/my/custom/loader.js',
      options: { basePath: '_build' }
    }
  }
}
```

It's useful to think of Intern's loader scriptss as glue code that Intern uses
to load configured suites within the environment. When Intern needs to load a
module (i.e. a test suite) it hands off a list of these modules to the loader
script and waits for the loading process to be handled by the loader. For more
information and options look at the [loader scripts] in the Intern repository.

## Configuring plugins

Plugins are standalone scripts or modules that are loaded by Intern. Standalone
plugins (ones that do not require a loader) are ran in-order early in the Intern
lifecycle before practically anything else. This is to allow plugins a chance to
listen to Intern events and augment the environment as early as possible. Module
plugins have a `useLoader` property and are loaded later in Intern's lifecycle
with the configured loader. Any number of plugins may be added to Intern.

```json5
{
  plugins: [
    'node_modules/babel-register/lib/node.js',
    {
      script: 'tests/support/mongodbAccess.js',
      options: { dbUrl: 'https://testdb.local' }
    },
    {
      script: 'tests/support/dojoMocking.js',
      useLoader: true
    }
  ]
}
```

Similar to loader scripts, plugin configurations with `options` are passed to
the plugin when it is registered with Intern.

## Configuration resolution

At runtime, the environment-specific resources and any
[active child configs](#config-file) will be mixed into the resolved config. In
general, properties from more specific sources will override properties from
lower precedence sources. The order of precedence, from lowest to highest, is:

1.  A config being extended by the current config
2.  The current config
3.  An active child config in the current config

So if you load the config `intern-myenv.json@wd`, and `intern-myenv.json`
extends `intern.json`, Intern will first load `intern.json`, then mix in
properties from `intern-myenv.json`, and finally mix in properties from the `wd`
child config.

There are a few exceptions:

1.  The "node" and "browser" properties in a child config are shallowly mixed
    into "node" and "browser" in the base config. For example, if "node" in the
    base config looks like:
    ```json5
    "node": {
        "suites": [ "tests/unit/foo.js" ],
        "plugins": [ "tests/plugins/bar.js" ]
    }
    ```
    and "node" in a child config looks like:
    ```json5
    "node": {
        "suites": [ "tests/unit/baz.js" ],
    }
    ```
    then the value of node in the resolved config (assuming the child config is
    active) will be:
    ```json5
    "node": {
        // node.suites from the child overrides node.suites from the base config
        "suites": [ "tests/unit/baz.js" ],
        // node.plugins from the base config remains
        "plugins": [ "tests/plugins/bar.js" ]
    }
    ```
2.  Resource arrays in "node" or "browser" ("plugins", "reporters", "suites"),
    will be used in addition to the corresponding resource arrays in the base
    config. For example, if the base config has:
    ```json5
    "suites": [ "tests/unit/foo.js" ]
    ```
    and the "node" section has:
    ```json5
    "suites": [ "tests/unit/bar.js" ]
    ```
    both sets of suites will be loaded when running on Node.
3.  Some properties can be extended (rather than replaced) by adding a '+' to
    the property name. For example, if the base config has:
    ```json5
    "suites": [ "tests/unit/foo.js" ]
    ```
    and a child config has:
    ```json5
    "suites+": [ "tests/unit/bar.js" ]
    ```
    the resolved value of suites will be:
    ```json5
    "suites": [ "tests/unit/foo.js", "tests/unit/bar.js" ]
    ```
    In general, properties that have array or object values can be extended,
    including:
    - benchmarkConfig
    - browser
    - capabilities
    - coverage
    - environments
    - functionalSuites
    - functionalTimeouts
    - instrumenterOptions
    - node
    - reporters
    - suites
    - tunnelOptions

[bail]: https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/bail
[baseline]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/baseline
[benchmark]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/benchmark
[capabilities]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/capabilities
[coverage]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/coverage
[debug]: https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/debug
[defaulttimeout]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/defaulttimeout
[environments]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/environments
[extends]: #extends
[filtererrorstack]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/filtererrorstack
[functionalretries]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/functionalretries
[functionalsuites]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/functionalsuites
[functionaltimeouts]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/functionaltimeouts
[grep]: https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/grep
[leaveremoteopen]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/leaveremoteopen
[loader scripts]: https://github.com/theintern/intern/tree/master/src/loaders
[loader]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/loader
[maxconcurrency]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/maxconcurrency
[plugins]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/plugins
[reporters]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/reporters
[serveonly]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/serveonly
[serverport]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/serverport
[serverurl]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/serverurl
[suites]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/suites
[tsconfig]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/tsconfig
[tunnel]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/tunnel
[tunneloptions]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/tunneloptions
[warnonunhandledrejection]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/warnOnUnhandledRejection
[warnonuncaughtexception]:
  https://theintern.io/docs.html#Intern/4/api/lib%2Fcommon%2Fconfig/warnOnUncaughtException
