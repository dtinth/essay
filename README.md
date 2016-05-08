
# essay

Generate your JavaScript library out of an essay!
Write your JavaScript library and explain your reasoning in a `README.md` file.

For example, you could write your library like this:

```js
// examples/add.js
export default (a, b) => a + b
```

And write a test for it:

```js
// examples/add.test.js
import add from './add'

// describe block called automatically for us!
it('should add two numbers', () => {
  assert(add(1, 2) === 3)
})
```


## usage

1. Create your README.md with fenced code blocks.

2. Run `essay`.

3. It should generate files in your "./lib" directory.



## development

You need to use `npm install --force`, because I use `essay` to write `essay`,
but `npm` doesn’t want to install a package as a dependency of itself
(even though it is an older version).



## implementation

### CLI

```js
// cli/index.js
import * as buildCommand from './buildCommand'
import * as testCommand from './testCommand'

export function main () {
  // XXX: Work around yargs’ lack of default command support.
  const commands = [ buildCommand, testCommand ]
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


#### build command

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


#### test command

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


### obtaining code blocks

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

This function extracts all the code blocks:

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

Let’s test it!

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


### dumping code blocks to source files

We’re going to build each file one by one.

```js
// dumpSourceCodeBlocks.js
import forEachCodeBlock from './forEachCodeBlock'
import extractCodeBlockToSourceFile from './extractCodeBlockToSourceFile'

export const dumpSourceCodeBlocks = forEachCodeBlock(extractCodeBlockToSourceFile)

export default dumpSourceCodeBlocks
```

Now, to write each code block to a file:

```js
// extractCodeBlockToSourceFile.js
import path from 'path'

import saveToFile from './saveToFile'

export async function extractCodeBlockToSourceFile ({ contents }, filename) {
  const targetFilePath = path.join('src', filename)
  await saveToFile(targetFilePath, contents)
}

export default extractCodeBlockToSourceFile
```

### transpilation using Babel

```js
// transpileCodeBlocks.js
import forEachCodeBlock from './forEachCodeBlock'
import transpileCodeBlock from './transpileCodeBlock'

export function transpileCodeBlocks (options) {
  return forEachCodeBlock(transpileCodeBlock(options))
}

export default transpileCodeBlocks
```

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

```js
// getBabelConfig.js
export function getBabelConfig () {
  return {
    presets: [
      require('babel-preset-es2015'),
      require('babel-preset-stage-2'),
    ],
    plugins: [
      require('babel-plugin-transform-runtime')
    ]
  }
}

export default getBabelConfig
```

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


### running unit tests

```js
// runUnitTests.js
import fs from 'fs'
import saveToFile from './saveToFile'
import mapSourceCoverage from './mapSourceCoverage'

export async function runUnitTests (codeBlocks) {
  const Mocha = require('mocha')
  const mocha = new Mocha({ ui: 'bdd' })
  const testEntryFilename = './lib-cov/_test-entry.js'
  const entry = generateEntryFile(codeBlocks)
  await saveToFile(testEntryFilename, entry)
  mocha.addFile(testEntryFilename)
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

#### the coverage magic

This module rewrites coverage data to be in README.md!

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
      2: { name: 'y', line: 20, loc: loc(20, 0)(20, 15) },
    },
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


### utilities

This function saves files to the file system. Just a utility functions.

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

This function runs something for each code block:

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
