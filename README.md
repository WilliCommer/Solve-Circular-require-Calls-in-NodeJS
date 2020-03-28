# Solve Circular `require()` Calls

## Preface

I ran into a problem of circular `require()` calls and was looking for a perfect solution. My favorite is to export *getter* for late loading. When I read the [stackoverflow post](https://stackoverflow.com/questions/23875233/require-returns-an-empty-object), I took the example to describe my solution.

## Wrong Solution

The example is divided into four files

```javascript
// -- author.js --
var Book = require('./book')
var Author = {
    name: 'Author',
    author : Book
}
module.exports = Author
```

```javascript
// -- books.js --
var Author = require('./author')
var Book = {
    name: 'Book',
    author : Author
}
module.exports = Book

```

```javascript
// -- lib.js --
module.exports = {
    Book: 	require('./book'),
    Author: require('./author'),
}
```

```
// -- test.js --
const lib = require('./lib')
console.log('1: %o',lib.Author.author)
console.log('2: %o',lib.Book.author)
console.log('3: %o',lib)
```

And the result is:

```
1: {}
2: { name: 'Author', author: {} }
3: {
  Book: { name: 'Book', author: { name: 'Author', author: {} } },
  Author: { name: 'Author', author: {} }
}

```

This do not work and the reason is described in [NodeJs Documentation](https://nodejs.org/api/modules.html#modules_cycles).

## Solution

The solution is to move the `require` into the object. You can make `author` a function, but don't want to use brackets when referencing `author`. Use a Getter:

```
// -- author.js --
var Author = {
    name: 'Author',
    get author() { return require('./book') }
}
module.exports = Author

// -- books.js --
var Book = {
    name: 'Book',
    get author() { return require('./author') }
}
module.exports = Book
```

The test run says:

```
1: { name: 'Book', author: [Getter] }
2: { name: 'Author', author: [Getter] }
3: {
  Book: { name: 'Book', author: [Getter] },
  Author: { name: 'Author', author: [Getter] }
}
```

This works fine. But now let's get rid of the Getter. 

## Perfect

There is a trick, called *Lazy Values*.

```
// -- author.js --
var Author = {
    name: 'Author',
    get author() { 
        var Book = require('./book')
        Object.defineProperty(this, 'author', { value: Book });
        return Book 
    }
}
module.exports = Author

// -- books.js --
var Book = {
    name: 'Book',
    get author() { 
        var Author = require('./author')
        Object.defineProperty(this, 'author', { value: Author });
        return Author 
    }
}
module.exports = Book
```

When `author` is read for the first time, it changes into a normal variable. The output:

```
1: { name: 'Book', author: [Getter] }
2: { name: 'Author', author: { name: 'Book', author: [Circular] } }
3: {
  [Author]: { name: 'Author', author: { name: 'Book', author: [Circular] } },
  [Book]: { name: 'Book', author: { name: 'Author', author: [Circular] } }
}
```

This is perfect and brings a performance advantage when initializing the app.  When I build libraries (index.js), I like to use a export function and `lib.js` can be changed to:

```
// -- lib.js --
addModule('Author', './author')
addModule('Book', './book')

function addModule (name, file, mod = module.exports) {
    Object.defineProperty(mod, name, { get: getIt, configurable: true });

    function getIt () {
        var re = require(file)
        Object.defineProperty(this, name, { value: re });
        return re
    }
}
```

`addModule` adds a module as *Lazy Value* to `exports`. If you chose to make it a module, remember to add `exports` as parameter.

```
// -- lib.js --
const addModule = require('./add-module')
const list = [
    ['Author',  './author'],
    ['Book',    './book']
].forEach( (args) => addModule(...args, exports))
```

 I love JavaScript 💘



## References

- [Mention in Node Documentation](https://nodejs.org/api/modules.html#modules_cycles)
- [Stackoverflow: Require returns an empty object](https://stackoverflow.com/questions/23875233/require-returns-an-empty-object)

