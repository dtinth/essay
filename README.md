
# essay
[![NPM version][npm-svg]][npm]
[![Build status][travis-svg]][travis]
[![Code coverage][codecov-svg]][codecov]

[travis]: https://travis-ci.org/dtinth/essay
[travis-svg]: https://img.shields.io/travis/dtinth/essay.svg?style=flat
[npm]: https://www.npmjs.com/package/essay
[npm-svg]: https://img.shields.io/npm/v/essay.svg?style=flat
[codecov]: https://codecov.io/gh/dtinth/essay/src/master/README.md
[codecov-svg]: https://img.shields.io/codecov/c/github/dtinth/essay.svg

__Generate your JavaScript library out of an essay!__

- __Write code in your README.md file__ in fenced code blocks, with a comment indicating the file’s name.

- __Write code using ES2015 syntax__. `essay` will use [Babel](https://babeljs.io/) to transpile them to ES5.

- __Test your code__ using [Mocha](https://mochajs.org/) and [power-assert](https://github.com/power-assert-js/power-assert).

- __Measures your code coverage__. `essay` generates [code coverage report for your README.md file][codecov] using [Istanbul](https://github.com/gotwarlost/istanbul) and [babel-plugin-\_\_coverage\_\_](https://github.com/dtinth/babel-plugin-__coverage__).

- __Examples__ of JavaScript libraries/articles written using `essay`:

  | Project | Description |
  | ------- | ----------- |
  | [circumstance](https://github.com/dtinth/circumstance) | BDD for your pure state-updating functions (e.g. Redux reducers) |
  | [positioning-strategy](https://github.com/taskworld/positioning-strategy) | A library that implements an algorithm to calculate where to position an element relative to another element |
  | [code-to-essay](https://github.com/zugarzeeker/code-to-essay) | Turns your code into an essay-formatted Markdown file. |
  | [timetable-calculator](https://github.com/zugarzeeker/timetable-calculator) | An algorithm for computing the layout for a timetable, handling the case where items are overlapped. |
  | [impure](https://github.com/taskworld/impure) | A simple wrapper object for non-deterministic code (like IO monads in Haskell) |

  - Using __essay__ to write a library/article? Feel free to add your link here: please submit a PR!


## synopsis

For example, you could write your library in a fenced code block in your README.md file like this:

```js
// examples/add.js
export default (a, b) => a + b
```

And also write a test for it:

```js
// examples/add.test.js
import add from './add'

// describe block called automatically for us!
it('should add two numbers', () => {
  assert(add(1, 2) === 3)
})
```


### getting started

1. Make sure you already have your README.md file in place and already initialized your npm package (e.g. using `npm init`).

2. Install `essay` as your dev dependency.

    ```
    npm install --save-dev essay
    ```

3. Ignore these folders in `.gitignore`:

    ```
    src
    lib
    lib-cov
    ```

4. Add `"files"` array to your `package.json`:

    ```json
    "files": [
      "lib",
      "src"
    ]
    ```

5. Add the `"scripts"` to your `package.json`:

    ```json
    "scripts": {
      "prepublish": "essay build",
      "test": "essay test"
    },
    ```

6. Set your main to the file you want to use, prefixed with `lib/`:

    ```json
    "main": "lib/index.js",
    ```


### building

When you run:

```
npm run prepublish # -> essay build
```

These code blocks will be extracted into its own file:

```
src
└── examples
    ├── add.js
    └── add.test.js
lib
└── examples
    ├── add.js
    └── add.test.js
```

The `src` folder contains the code as written in README.md, and the `lib` folder contains the transpiled code.


### testing

When you run:

```
npm test # -> essay test
```

All the files ending with `.test.js` will be run using Mocha framework. `power-assert` is included by default (but you can use any assertion library you want).

```
  examples/add.test.js
    ✓ should add two numbers
```

Additionally, test coverage report for your README.md file will be generated.



## development

You need to use `npm install --force`, because I use `essay` to write `essay`,
but `npm` doesn’t want to install a package as a dependency of itself
(even though it is an older version).



## commands

### `essay build`

```js
// cli/buildCommand.js
import obtainCodeBlocks from '../obtainCodeBlocks'
import dumpSourceCodeBlocks from '../dumpSourceCodeBlocks'
import transpileCodeBlocks from '../transpileCodeBlocks'
import getBabelConfig from '../getBabelConfig'

export const command = 'build'
export const description = 'Builds the README.md file into lib folder.'
export const builder = (yargs) => yargs
export const handler = async (argv) => {
  const babelConfig = getBabelConfig()
  const targetDirectory = 'lib'
  const codeBlocks = await obtainCodeBlocks()
  await dumpSourceCodeBlocks(codeBlocks)
  await transpileCodeBlocks({ targetDirectory, babelConfig })(codeBlocks)
}
```


### `essay test`

```js
// cli/testCommand.js
import obtainCodeBlocks from '../obtainCodeBlocks'
import dumpSourceCodeBlocks from '../dumpSourceCodeBlocks'
import transpileCodeBlocks from '../transpileCodeBlocks'
import getTestingBabelConfig from '../getTestingBabelConfig'
import runUnitTests from '../runUnitTests'

export const command = 'test'
export const description = 'Runs the test.'
export const builder = (yargs) => yargs
export const handler = async (argv) => {
  const babelConfig = getTestingBabelConfig()
  const targetDirectory = 'lib-cov'
  const codeBlocks = await obtainCodeBlocks()
  await dumpSourceCodeBlocks(codeBlocks)
  await transpileCodeBlocks({ targetDirectory, babelConfig })(codeBlocks)
  await runUnitTests(codeBlocks)
}
```

### `essay lint`

```js
// cli/lintCommand.js
import obtainCodeBlocks from '../obtainCodeBlocks'
import runLinter from '../runLinter'

export const command = 'lint'
export const description = 'Runs the linter.'
export const builder = (yargs) => yargs
export const handler = async (argv) => {
  const codeBlocks = await obtainCodeBlocks()
  await runLinter(codeBlocks, argv)
}
```


## obtaining code blocks

This IO function reads your README.md and extracts the fenced code blocks.

```js
// obtainCodeBlocks.js
import extractCodeBlocks from './extractCodeBlocks'
import fs from 'fs'

export async function obtainCodeBlocks () {
  const readme = fs.readFileSync('README.md', 'utf8')
  const codeBlocks = extractCodeBlocks(readme)
  return codeBlocks
}

export default obtainCodeBlocks
```


### code block extraction

This function takes a string (representing your README.md) and extracts the fenced code block. Returns an object of code block entries, which contains the contents and line number in README.md.

See the test (below) for more details:

```js
// extractCodeBlocks.js
export function extractCodeBlocks (data) {
  const codeBlocks = { }
  const regexp = /(`[`]`js\s+\/\/\s*(\S+).*\n)([\s\S]+?)`[`]`/g
  data.replace(regexp, (all, before, filename, contents, index) => {
    if (codeBlocks[filename]) throw new Error(filename + ' already exists!')

    // XXX: Not the most efficient way to find the line number.
    const line = data.substr(0, index + before.length).split('\n').length

    codeBlocks[filename] = { contents, line }
  })
  return codeBlocks
}

export default extractCodeBlocks
```

Here’s the test for this function:

```js
// extractCodeBlocks.test.js
import extractCodeBlocks from './extractCodeBlocks'

const END = '`' + '`' + '`'
const BEGIN = END + 'js'

const example = [
  'Hello world!',
  '============',
  '',
  BEGIN,
  '// file1.js',
  'console.log("hello,")',
  END,
  '',
  '- It should work in lists too!',
  '',
  '  ' + BEGIN,
  '  // file2.js',
  '  console.log("world!")',
  '  ' + END,
  '',
  'That’s it!'
].join('\n')

const blocks = extractCodeBlocks(example)

it('should extract code blocks into object', () => {
  assert.deepEqual(Object.keys(blocks).sort(), [ 'file1.js', 'file2.js' ])
})

it('should contain the code block’s contents', () => {
  assert(blocks['file1.js'].contents.trim() === 'console.log("hello,")')
  assert(blocks['file2.js'].contents.trim() === 'console.log("world!")')
})

it('should contain line numbers', () => {
  assert(blocks['file1.js'].line === 6)
  assert(blocks['file2.js'].line === 13)
})
```


## dumping code blocks to source files

```js
// dumpSourceCodeBlocks.js
import forEachCodeBlock from './forEachCodeBlock'
import saveToFile from './saveToFile'
import path from 'path'

export const dumpSourceCodeBlocks = forEachCodeBlock(async ({ contents }, filename) => {
  const targetFilePath = path.join('src', filename)
  await saveToFile(targetFilePath, contents)
})

export default dumpSourceCodeBlocks
```


## transpilation using Babel

```js
// transpileCodeBlocks.js
import forEachCodeBlock from './forEachCodeBlock'
import transpileCodeBlock from './transpileCodeBlock'

export function transpileCodeBlocks (options) {
  return forEachCodeBlock(transpileCodeBlock(options))
}

export default transpileCodeBlocks
```


### transpiling an individual code block

To speed up transpilation,
we’ll skip the transpilation process if the source file has not been modified
since the corresponding transpiled file has been generated (similar to make).

```js
// transpileCodeBlock.js
import path from 'path'
import fs from 'fs'
import { transformFileSync } from 'babel-core'

import saveToFile from './saveToFile'

export function transpileCodeBlock ({ babelConfig, targetDirectory } = { }) {
  return async function (codeBlock, filename) {
    const sourceFilePath = path.join('src', filename)
    const targetFilePath = path.join(targetDirectory, filename)
    if (await isAlreadyUpToDate(sourceFilePath, targetFilePath)) return
    const { code } = transformFileSync(sourceFilePath, babelConfig)
    await saveToFile(targetFilePath, code)
  }
}

async function isAlreadyUpToDate (sourceFilePath, targetFilePath) {
  if (!fs.existsSync(targetFilePath)) return false
  const sourceStats = fs.statSync(sourceFilePath)
  const targetStats = fs.statSync(targetFilePath)
  return targetStats.mtime > sourceStats.mtime
}

export default transpileCodeBlock
```


## babel configuration

```js
// getBabelConfig.js
export function getBabelConfig () {
  return {
    presets: [
      require('babel-preset-es2015'),
      require('babel-preset-stage-2')
    ],
    plugins: [
      require('babel-plugin-transform-runtime')
    ]
  }
}

export default getBabelConfig

```

### additional options for testing

```js
// getTestingBabelConfig.js
import getBabelConfig from './getBabelConfig'

export function getTestingBabelConfig () {
  const babelConfig = getBabelConfig()
  return {
    ...babelConfig,
    plugins: [
      require('babel-plugin-__coverage__'),
      require('babel-plugin-espower'),
      ...babelConfig.plugins
    ]
  }
}

export default getTestingBabelConfig
```


## running unit tests

It’s quite hackish right now, but it works.

```js
// runUnitTests.js
import fs from 'fs'
import saveToFile from './saveToFile'
import mapSourceCoverage from './mapSourceCoverage'

export async function runUnitTests (codeBlocks) {
  // Generate an entry file for mocha to use.
  const testEntryFilename = './lib-cov/_test-entry.js'
  const entry = generateEntryFile(codeBlocks)
  await saveToFile(testEntryFilename, entry)

  // Initialize mocha with the entry file.
  const Mocha = require('mocha')
  const mocha = new Mocha({ ui: 'bdd' })
  mocha.addFile(testEntryFilename)

  // Now go!!
  prepareTestEnvironment()
  await runMocha(mocha)
  await saveCoverageData(codeBlocks)
}

function runMocha (mocha) {
  return new Promise((resolve, reject) => {
    mocha.run(function (failures) {
      if (failures) {
        reject(new Error('There are ' + failures + ' test failure(s).'))
      } else {
        resolve()
      }
    })
  })
}

function generateEntryFile (codeBlocks) {
  const entry = [ '"use strict";' ]
  for (const filename of Object.keys(codeBlocks)) {
    if (filename.match(/\.test\.js$/)) {
      entry.push('describe(' + JSON.stringify(filename) + ', function () {')
      entry.push('  require(' + JSON.stringify('./' + filename) + ')')
      entry.push('})')
    }
  }
  return entry.join('\n')
}

function prepareTestEnvironment () {
  global.assert = require('power-assert')
}

async function saveCoverageData (codeBlocks) {
  const coverage = global['__coverage__']
  if (!coverage) return
  const istanbul = require('istanbul')
  const reporter = new istanbul.Reporter()
  const collector = new istanbul.Collector()
  const synchronously = true
  collector.add(mapSourceCoverage(coverage, {
    codeBlocks,
    sourceFilePath: fs.realpathSync('README.md'),
    targetDirectory: fs.realpathSync('src')
  }))
  reporter.add('lcov')
  reporter.add('text')
  reporter.write(collector, synchronously, () => { })
}

export default runUnitTests
```


## running linter
```js
// runLinter.js
import fs from 'fs'
import saveToFile from './saveToFile'
import padRight from 'lodash/padEnd'
import flatten from 'lodash/flatten'
import isEmpty from 'lodash/isEmpty'
import compact from 'lodash/compact'
import { CLIEngine } from 'eslint'
import Table from 'cli-table'

export const countLinterErrors = (results) => results[0].messages.length

export const runESLint = (contents, fix) => {
  const cli = new CLIEngine({ fix, globals: ['describe', 'it', 'should'] })
  const report = cli.executeOnText(contents)
  return report.results
}

export const formatLinterErrorsColumnMode = (errors, resetStyle = {}) => {
  if (isEmpty(errors)) return ''
  const table = new Table({ head: ['Where', 'Path', 'Rule', 'Message'], ...resetStyle })
  errors.map((error) => table.push([
    error.line + ':' + error.column,
    error.filename,
    error.ruleId,
    error.message
  ]))
  return table.toString()
}

const formatLinterSolution = (line, filename, output) => ({
  line,
  filename,
  fixedCode: output
})

const formatLinterError = (line, filename, error) => {
  error.line += line - 1
  error.filename = filename
  return error
}

export const isFix = (options) => !!options._ && options._[0] === 'fix'

export const generateCodeBlock = (code, filename) => {
  const END = '`' + '`' + '`'
  const BEGIN = END + 'js'
  return [
    BEGIN,
    '// ' + filename,
    code + END
  ].join('\n')
}

export const fixLinterErrors = async (errors, codeBlocks, targetPath = 'README.md') => {
  let readme = fs.readFileSync(targetPath, 'utf8')
  errors.map(({ filename, fixedCode }) => {
    const code = codeBlocks[filename].contents
    if (fixedCode) {
      readme = readme.split(generateCodeBlock(code, filename)).join(generateCodeBlock(fixedCode, filename))
    }
  })
  await saveToFile(targetPath, readme)
}

export const mapLinterErrorsToLine = (results, line, filename) => (
  results.map(({ messages, output }) => ({
    solutions: [formatLinterSolution(line, filename, output)],
    remainingErrors: messages.map((error) => formatLinterError(line, filename, error))
  })).reduce((prev, cur) => ({
    solutions: compact([...prev.solutions, ...cur.solutions]),
    remainingErrors: compact([...prev.remainingErrors, ...cur.remainingErrors])
  }), { solutions: [], remainingErrors: [] })
)

export async function runLinter (codeBlocks, options) {
  let remainingErrors = []
  let solutions = []
  const fix = isFix(options)
  Object.keys(codeBlocks).map(filename => {
    const { contents, line } = codeBlocks[filename]
    const results = runESLint(contents, fix)
    const linterResults = mapLinterErrorsToLine(results, line, filename)
    solutions = [...solutions, ...linterResults.solutions]
    remainingErrors = [...remainingErrors, ...linterResults.remainingErrors]
  })
  if (fix) await fixLinterErrors(solutions, codeBlocks)
  console.error(formatLinterErrorsColumnMode(remainingErrors))
}

export default runLinter
```

And its tests

```js
// runLinter.test.js
import {
  runESLint,
  mapLinterErrorsToLine,
  countLinterErrors,
  formatLinterErrorsColumnMode,
  isFix,
  generateCodeBlock
} from './runLinter'

it('should map linter errors back to line in README.md', () => {
  const linterResults = [
    { messages: [{ message: 'message-1', line: 2 }], output: 'fixed-code' }
  ]
  const { solutions, remainingErrors } = mapLinterErrorsToLine(linterResults, 5, 'example.js')
  assert.deepEqual(remainingErrors, [{
    line: 6,
    filename: 'example.js',
    message: 'message-1'
  }])
  assert.deepEqual(solutions, [{
    line: 5,
    filename: 'example.js',
    fixedCode: 'fixed-code'
  }])
})

it('should format linter errors on a column mode', () => {
  const errors = [{
    line: 5,
    column: 10,
    message: 'message',
    filename: 'example.js',
    ruleId: 'ruleId-1'
  }]
  const resetStyle = { style: { head: [], border: [] } }
  const table = formatLinterErrorsColumnMode(errors, resetStyle)
  assert(table === [
    '┌───────┬────────────┬──────────┬─────────┐',
    '│ Where │ Path       │ Rule     │ Message │',
    '├───────┼────────────┼──────────┼─────────┤',
    '│ 5:10  │ example.js │ ruleId-1 │ message │',
    '└───────┴────────────┴──────────┴─────────┘'
  ].join('\n'))
  const tableNoErrors = formatLinterErrorsColumnMode([], resetStyle)
  assert(tableNoErrors === '')
})

it('should trigger fix mode when \'fix\' option is given', () => {
  assert(isFix({ _: ['fix'] }) === true)
  assert(isFix({}) === false)
})

it('should insert javascript code block', () => {
  assert(generateCodeBlock('const x = 5\n', 'example.js') === [
    '`' + '`' + '`js',
    '// example.js',
    'const x = 5',
    '`' + '`' + '`'
  ].join('\n'))
})

it('should execute on text by `eslint-cli` and `eslint-config-standard`', () => {
  assert(countLinterErrors(runESLint('1 === 2\n')) === 0)
  assert(countLinterErrors(runESLint('1 === 2')) === 1)
  assert(countLinterErrors(runESLint('1 == 2')) === 2)
})

```


### the coverage magic

This module rewrites the coverage data so that you can view the coverage report for README.md. It’s quite complex and thus deserves its own section.

```js
// mapSourceCoverage.js
import path from 'path'
import { forOwn } from 'lodash'

export function mapSourceCoverage (coverage, {
  codeBlocks,
  sourceFilePath,
  targetDirectory
}) {
  const result = { }
  const builder = createReadmeDataBuilder(sourceFilePath)
  for (const key of Object.keys(coverage)) {
    const entry = coverage[key]
    const relative = path.relative(targetDirectory, entry.path)
    if (codeBlocks[relative]) {
      builder.add(entry, codeBlocks[relative])
    } else {
      result[key] = entry
    }
  }
  if (!builder.isEmpty()) result[sourceFilePath] = builder.getOutput()
  return result
}

function createReadmeDataBuilder (path) {
  let nextId = 1
  let output = {
    path,
    s: { }, b: { }, f: { },
    statementMap: { }, branchMap: { }, fnMap: { }
  }
  let empty = true
  return {
    add (entry, codeBlock) {
      const id = nextId++
      const prefix = (key) => `${id}.${key}`
      const mapLine = (line) => codeBlock.line - 1 + line
      const map = mapLocation(mapLine)
      empty = false
      forOwn(entry.s, (count, key) => {
        output.s[prefix(key)] = count
      })
      forOwn(entry.statementMap, (loc, key) => {
        output.statementMap[prefix(key)] = map(loc)
      })
      forOwn(entry.b, (count, key) => {
        output.b[prefix(key)] = count
      })
      forOwn(entry.branchMap, (branch, key) => {
        output.branchMap[prefix(key)] = {
          ...branch,
          line: mapLine(branch.line),
          locations: branch.locations.map(map)
        }
      })
      forOwn(entry.f, (count, key) => {
        output.f[prefix(key)] = count
      })
      forOwn(entry.fnMap, (fn, key) => {
        output.fnMap[prefix(key)] = {
          ...fn,
          line: mapLine(fn.line),
          loc: map(fn.loc)
        }
      })
    },
    isEmpty: () => empty,
    getOutput: () => output
  }
}

function mapLocation (mapLine) {
  return ({ start = { }, end = { } }) => ({
    start: { line: mapLine(start.line), column: start.column },
    end: { line: mapLine(end.line), column: end.column }
  })
}

export default mapSourceCoverage
```

And its tests is quite… ugh!

```js
// mapSourceCoverage.test.js
import mapSourceCoverage from './mapSourceCoverage'
import path from 'path'

const loc = (startLine, startColumn) => (endLine, endColumn) => ({
  start: { line: startLine, column: startColumn },
  end: { line: endLine, column: endColumn }
})
const coverage = {
  '/home/user/essay/src/hello.js': {
    path: '/home/user/essay/src/hello.js',
    s: { 1: 1 },
    b: { 1: [ 1, 2, 3 ] },
    f: { 1: 99, 2: 30 },
    statementMap: {
      1: loc(2, 15)(2, 30)
    },
    branchMap: {
      1: { line: 4, type: 'switch', locations: [
        loc(5, 10)(5, 20),
        loc(7, 10)(7, 25),
        loc(9, 10)(9, 30)
      ] }
    },
    fnMap: {
      1: { name: 'x', line: 10, loc: loc(10, 0)(10, 20) },
      2: { name: 'y', line: 20, loc: loc(20, 0)(20, 15) }
    }
  },
  '/home/user/essay/src/world.js': {
    path: '/home/user/essay/src/world.js',
    s: { 1: 1 },
    b: { },
    f: { },
    statementMap: { 1: loc(1, 0)(1, 30) },
    branchMap: { },
    fnMap: { }
  },
  '/home/user/essay/unrelated.js': {
    path: '/home/user/essay/unrelated.js',
    s: { 1: 1 },
    b: { },
    f: { },
    statementMap: { 1: loc(1, 0)(1, 30) },
    branchMap: { },
    fnMap: { }
  }
}
const codeBlocks = {
  'hello.js': { line: 72 },
  'world.js': { line: 99 }
}
const getMappedCoverage = () => mapSourceCoverage(coverage, {
  codeBlocks,
  sourceFilePath: '/home/user/essay/README.md',
  targetDirectory: '/home/user/essay/src'
})

it('should combine mappings from code blocks', () => {
  const mapped = getMappedCoverage()
  assert(Object.keys(mapped).length === 2)
})

const testReadmeCoverage = (f) => () => (
  f(getMappedCoverage()['/home/user/essay/README.md'])
)

it('should have statements', testReadmeCoverage(entry => {
  const keys = Object.keys(entry.s)
  assert(keys.length === 2)
  assert.deepEqual(keys, [ '1.1', '2.1' ])
}))
it('should have statementMap', testReadmeCoverage(({ statementMap }) => {
  assert(statementMap['1.1'].start.line === 73)
  assert(statementMap['1.1'].end.line === 73)
  assert(statementMap['2.1'].start.line === 99)
  assert(statementMap['2.1'].end.line === 99)
}))
it('should have branches', testReadmeCoverage(({ b }) => {
  assert(Array.isArray(b['1.1']))
}))
it('should have branchMap', testReadmeCoverage(({ branchMap }) => {
  assert(branchMap['1.1'].locations[2].start.line === 80)
  assert(branchMap['1.1'].line === 75)
}))
it('should have functions', testReadmeCoverage(({ f }) => {
  assert(f['1.1'] === 99)
  assert(f['1.2'] === 30)
}))
it('should have function map', testReadmeCoverage(({ fnMap }) => {
  assert(fnMap['1.1'].loc.start.line === 81)
  assert(fnMap['1.2'].line === 91)
}))

```


## acceptance test

```js
// acceptance.test.js
import mkdirp from 'mkdirp'
import fs from 'fs'
import * as buildCommand from './cli/buildCommand'
import * as testCommand from './cli/testCommand'
import * as lintCommand from './cli/lintCommand'

it('works', async () => {
  const example = fs.readFileSync('example.md', 'utf8')
  const eslintrc = fs.readFileSync('.eslintrc', 'utf8')
  await runInTemporaryDir(async () => {
    fs.writeFileSync('README.md', example.replace('a + b', 'a+b'))
    fs.writeFileSync('.eslintrc', eslintrc)
    await buildCommand.handler({ })
    assert(fs.existsSync('src/add.js'))
    assert(fs.existsSync('lib/add.js'))
    await testCommand.handler({ })
    assert(fs.existsSync('coverage/lcov.info'))
    assert(fs.readFileSync('README.md', 'utf8') !== example)
    await lintCommand.handler({ _: ['fix'] })
    assert(fs.readFileSync('README.md', 'utf8') === example)
  })
})

async function runInTemporaryDir (f) {
  const cwd = process.cwd()
  const testDirectory = '/tmp/essay-acceptance-test'
  mkdirp.sync(testDirectory)
  try {
    process.chdir(testDirectory)
    await f()
  } finally {
    process.chdir(cwd)
  }
}
```


## miscellaneous

### command line handler

```js
// cli/index.js
import * as buildCommand from './buildCommand'
import * as testCommand from './testCommand'
import * as lintCommand from './lintCommand'

export function main () {
  // XXX: Work around yargs’ lack of default command support.
  const commands = [ buildCommand, testCommand, lintCommand ]
  const yargs = commands.reduce(appendCommandToYargs, require('yargs')).help()
  const registry = commands.reduce(registerCommandToRegistry, { })
  const argv = yargs.argv
  const command = argv._.shift() || 'build'
  const commandObject = registry[command]
  if (commandObject) {
    const subcommand = commandObject.builder(yargs.reset())
    Promise.resolve(commandObject.handler(argv)).catch(e => {
      setTimeout(() => { throw e })
    })
  } else {
    yargs.showHelp()
  }
}

function appendCommandToYargs (yargs, command) {
  return yargs.command(command.command, command.description)
}

function registerCommandToRegistry (registry, command) {
  return Object.assign(registry, {
    [command.command]: command
  })
}
```


### saving file

This function wraps around the normal file system API, but provides this benefits:

- It will first read the file, and if the content is identical, it will not re-write the file.

- It displays log message on console.

```js
// saveToFile.js
import fs from 'fs'
import path from 'path'
import mkdirp from 'mkdirp'

export async function saveToFile (filePath, contents) {
  mkdirp.sync(path.dirname(filePath))
  const exists = fs.existsSync(filePath)
  if (exists) {
    const existingData = fs.readFileSync(filePath, 'utf8')
    if (existingData === contents) return
  }
  console.log('%s %s…', exists ? 'Updating' : 'Writing', filePath)
  fs.writeFileSync(filePath, contents)
}

export default saveToFile
```


### run a function for each code block

```js
// forEachCodeBlock.js
export function forEachCodeBlock (fn) {
  return async function (codeBlocks) {
    const filenames = Object.keys(codeBlocks)
    for (const filename of filenames) {
      await fn(codeBlocks[filename], filename, codeBlocks)
    }
  }
}

export default forEachCodeBlock
```
