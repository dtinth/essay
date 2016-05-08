
# adder

```js
// add.js
export default (a, b) => a + b
```

And write a test for it:

```js
// add.test.js
import add from './add'

it('should add two numbers', () => {
  assert(add(1, 2) === 3)
})
```
