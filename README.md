# cypress-grep
[![ci status][ci image]][ci url] [![badges status][badges image]][badges url] [![renovate-app badge][renovate-badge]][renovate-app] ![cypress version](https://img.shields.io/badge/cypress-7.3.0-brightgreen)
> Filter tests using substring

```shell
# run only tests with "hello" in their names
npx cypress run --env grep=hello

  ✓ hello world
  - works
  - works 2 @tag1
  - works 2 @tag1 @tag2

  1 passing (38ms)
  3 pending
```

All other tests will be marked pending, see why in the [Cypress test statuses](https://on.cypress.io/writing-and-organizing-tests#Test-statuses) blog post.

If you have multiple spec files, all specs will be loaded, and every test will be filtered the same way, since the grep is run-time operation and cannot eliminate the spec files without loading them. If you want to run only specific tests, use the built-in [--spec](https://on.cypress.io/command-line#cypress-run-spec-lt-spec-gt) CLI argument.

Watch the video [intro to cypress-grep plugin](https://www.youtube.com/watch?v=HS-Px-Sghd8)

## Install and use

Assuming you have Cypress installed, add this module as a dev dependency

```shell
# using NPM
npm i -D cypress-grep
# using Yarn
yarn add -D cypress-grep
```

**required:** load this module from the [support file](https://on.cypress.io/writing-and-organizing-tests#Support-file) or at the top of the spec file if not using the support file.

```js
// cypress/support/index.js
// load and register the grep feature
// https://github.com/bahmutov/cypress-grep
require('cypress-grep')()
```

**optional:** load and register this module from the [plugin file](https://on.cypress.io/writing-and-organizing-tests#Plugins-file)

```js
// cypress/plugins/index.js
module.exports = (on, config) => {
  // optional: register cypress-grep plugin code
  // https://github.com/bahmutov/cypress-grep
  require('cypress-grep/src/plugin')(config)
}
```

The plugin code will print a little message on load, for example

```shell
$ npx cypress run --env grep=hello
cypress-grep: tests with "hello" in their names
```

## Filter

You can filter tests to run using part of their title via `grep`, and via explicit tags via `grepTags` Cypress environment variables.

Most likely you will pass these environment variables from the command line. For example, to only run tests with "login" in their title and tagged "smoke", you would run:

```shell
$ npx cypress run --env grep=login,grepTags=smoke
```

You can use any way to modify the environment values `grep` and `grepTags`, except the run-time `Cypress.env('grep')` (because it is too late at run-time). You can set the `grep` value in the `cypress.json` file to run only tests with the substring `viewport` in their names

```json
{
  "env": {
    "grep": "viewport"
  }
}
```

You can also set the `env.grep` object in the plugin file, but remember to return the changed config object:

```js
// cypress/plugin/index.js
module.exports = (on, config) => {
  config.env.grep = 'viewport'
  return config
}
```

You can also set the grep and grepTags from the DevTools console while running Cypress in the interactive mode `cypress open`, see [DevTools Console section](#devtools-console).

### grep by test title

```shell
# run all tests with "hello" in their title
$ npx cypress run --env grep=hello
# run all tests with "hello world" in their title
$ npx cypress run --env grep="hello world"
# run all tests WITHOUT "hello world" in their title
$ npx cypress run --env grep="-hello world"
```

## Filter with tags

You can select tests to run or skip using tags by passing `--env grepTags=...` value.

```
# enable the tests with tag "one" or "two"
--env grepTags="one two"
# enable the tests with both tags "one" and "two"
--env grepTags="one+two"
# enable the tests with "hello" in the title and tag "smoke"
--env grep=hello,grepTags=smoke
```

### Tags in the test config object

Cypress tests can have their own [test config object](https://on.cypress.io/configuration#Test-Configuration), and when using this plugin you can put the test tags there, either as a single tag string or as an array of tags.

```js
it('works as an array', { tags: ['config', 'some-other-tag'] }, () => {
  expect(true).to.be.true
})

it('works as a string', { tags: 'config' }, () => {
  expect(true).to.be.true
})
```

You can run both of these tests using `--env grepTags=config` string.

### TypeScript users

Because the Cypress test config object type definition does not have the `tags` property we are using above, the TypeScript linter will show an error. Just add an ignore comment above the test:

```js
// @ts-ignore
it('runs on deploy', { tags: 'smoke' }, () => {
  ...
})
```

This package comes with [src/index.d.ts](./src/index.d.ts) definition file that adds the property `tags` to the Cypress test overrides interface. Include this file in your specs or TS config settings. For example, you can load it using a reference comment

```js
// cypress/integration/my-spec.js
/// <reference types="cypress-grep" />
```

## Test suites

The tags are also applied to the "describe" blocks with some limitations:

- you can only use the config object tags

```js
describe('block with config tag', { tags: '@smoke' }, () => {
})
```

- currently only the invert tag to skip the blog has meaningful effect. For example you can skip the above suite of tests by using `--env grepTags=-@smoke` value. Keep an eye on issue [#22](https://github.com/bahmutov/cypress-grep/issues/22) for the full support implementation.

See the [cypress/integration/describe-tags-spec.js](./cypress/integration/describe-tags-spec.js) file.

### AND tags

Use `+` to require both tags to be present

```
--env grepTags=@smoke+@fast
```

### Invert tag

You can skip running the tests with specific tag using the invert option: prefix the tag with the character `-`.

```
# do not run any tests with tag "@slow"
--env grepTags=-@slow
```

### OR tags

You can run tests that match one tag or another using spaces. Make sure to quote the grep string!

```
# run tests with tags "@slow" or "@critical" in their names
--env grepTags='@slow @critical'
```

## General advice

- keep it simple.
- I like using `@` as tag prefix to make the tags searchable

```js
// ✅ good practice
describe('auth', { tags: '@critical' }, () => ...)
it('works', { tags: '@smoke' }, () => ...)
it('works quickly', { tags: ['@smoke', '@fast'] }, () => ...)

// 🚨 NOT GOING TO WORK
// ERROR: treated as a single tag,
// probably want an array instead
it('works', { tags: '@smoke @fast' }, () => ...)
```

Grepping the tests

```shell
# run the tests by title
$ npx cypress run --env grep="works quickly"
# run all tests tagged @smoke
$ npx cypress run --env grepTags=@smoke
# run all tests except tagged @smoke
$ npx cypress run --env grepTags=-@smoke
```

## DevTools console

You can set the grep string from the DevTools Console. This plugin adds method `Cypress.grep` and `Cypress.grepTags` to set the grep strings and restart the tests

```js
// filter tests by title substring
Cypress.grep('hello world')
// filter tests by tag string
// in this case will run tests with tag @smoke OR @fast
Cypress.grep(null, '@smoke @fast')
// run tests tagged @smoke AND @fast
Cypress.grep(null, '@smoke+@fast')
// run tests with title containing "hello" and tag @smoke
Cypress.grep('hello', '@smoke')
```

- to remove the grep strings enter `Cypress.grep()`

## Debugging

This module uses [debug](https://github.com/visionmedia/debug#readme) to log verbose messages. To enable debug console messages, from the DevTools console set `localStorage.debug='cypress-grep'` and run the tests again.

![Debug messages](./images/debug.png)

## Examples

- [cypress-grep-example](https://github.com/bahmutov/cypress-grep-example)

## See also

- [cypress-select-tests](https://github.com/bahmutov/cypress-select-tests)
- [cypress-skip-test](https://github.com/cypress-io/cypress-skip-test)

## Migration guide

### from v1 to v2

In v2 we have separated grepping by part of the title string from tags.

**v1**

```
--env grep="one two"
```

The above scenario was confusing - did you want to find all tests with title containing "one two" or did you want to run tests tagged `one` or `two`?

**v2**

```
# enable the tests with string "one two" in their titles
--env grep="one two"
# enable the tests with tag "one" or "two"
--env grepTags="one two"
# enable the tests with both tags "one" and "two"
--env grepTags="one+two"
# enable the tests with "hello" in the title and tag "smoke"
--env grep=hello,grepTags=smoke
```

## Small print

Author: Gleb Bahmutov &lt;gleb.bahmutov@gmail.com&gt; &copy; 2021

- [@bahmutov](https://twitter.com/bahmutov)
- [glebbahmutov.com](https://glebbahmutov.com)
- [blog](https://glebbahmutov.com/blog)

License: MIT - do anything with the code, but don't blame me if it does not work.

Support: if you find any problems with this module, email / tweet /
[open issue](https://github.com/bahmutov/cypress-grep/issues) on Github

## MIT License

Copyright (c) 2021 Gleb Bahmutov &lt;gleb.bahmutov@gmail.com&gt;

Permission is hereby granted, free of charge, to any person
obtaining a copy of this software and associated documentation
files (the "Software"), to deal in the Software without
restriction, including without limitation the rights to use,
copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the
Software is furnished to do so, subject to the following
conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES
OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
OTHER DEALINGS IN THE SOFTWARE.

[ci image]: https://github.com/bahmutov/cypress-grep/workflows/ci/badge.svg?branch=main
[ci url]: https://github.com/bahmutov/cypress-grep/actions
[badges image]: https://github.com/bahmutov/cypress-grep/workflows/badges/badge.svg?branch=main
[badges url]: https://github.com/bahmutov/cypress-grep/actions
[renovate-badge]: https://img.shields.io/badge/renovate-app-blue.svg
[renovate-app]: https://renovateapp.com/
