
# essay

Write your JavaScript library and explain your reasoning in a `README.md` file.

For example, you could write your library like this:

```js
// examples/add.js
export default (a, b) => a + b
```

And write a test for it:

```js
// examples/add.test.js
import add from 'example'
it('should add two numbers', () => {
  assert.equal(add(1, 2) === 3)
})
```


## Usage

1. Create your README.md with fenced code blocks.

2. Run `essay`.

3. It should generate files in your "./lib" directory.



## Development

You need to use `npm install --force`, because I use `essay` to write `essay`,
but `npm` doesn’t want to install a package as a dependency of itself
(even though it is an older version).



## How it works

### Code block extraction…

This function extracts all the code blocks:

```js
// extractCodeBlocks.js
export function extractCodeBlocks (data) {
  const codeBlocks = { }
  data.replace(/`[`]`js\s+\/\/\s*(\S+).*\n([\s\S]+?)`[`]`/g, (all, filename, contents) => {
    if (codeBlocks[filename]) throw new Error(filename + ' already exists!')
    codeBlocks[filename] = { contents }
    // TODO: add line number
  })
  return codeBlocks
}

export default extractCodeBlocks
```


### CLI

The CLI simply reads README.md and builds them into files:

```js
// cli.js
import fs from 'fs'

import extractCodeBlocks from './extractCodeBlocks'
import buildCodeBlocks from './buildCodeBlocks'

export function main () {
  const readme = fs.readFileSync('README.md', 'utf8')
  const codeBlocks = extractCodeBlocks(readme)
  buildCodeBlocks(codeBlocks)
}
```


### Builder

We’re going to build each file one by one.

```js
// buildCodeBlocks.js
import buildCodeBlock from './buildCodeBlock'

export function buildCodeBlocks (codeBlocks) {
  for (const filename of Object.keys(codeBlocks)) {
    buildCodeBlock(filename, codeBlocks[filename])
  }
}

export default buildCodeBlocks
```


Now, to write each code block to a file:

```js
// buildCodeBlock.js
import path from 'path'
import { transform } from 'babel-core'

import saveToFile from './saveToFile'

export function buildCodeBlock (filename, { contents }) {
  const targetFilePath = path.join('lib', filename)
  const { code } = transform(contents, {
    filename: targetFilePath,
    presets: [
      require('babel-preset-es2015'),
      require('babel-preset-stage-2')
    ]
  })
  saveToFile(targetFilePath, code)
}

export default buildCodeBlock
```

```js
// saveToFile.js
import fs from 'fs'
import path from 'path'
import mkdirp from 'mkdirp'

export function saveToFile (filePath, contents) {
  mkdirp.sync(path.dirname(filePath))
  console.log('Writing %s…', filePath)
  fs.writeFileSync(filePath, contents)
}

export default saveToFile
```
