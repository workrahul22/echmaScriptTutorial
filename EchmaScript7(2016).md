
## includes() method 

this method determines whether an array includes a certain value among its entries, returning true or false as appropriate.

### Example

```JavaScript
var array1 = [1, 2, 3];

console.log(array1.includes(2));
// expected output: true

var pets = ['cat', 'dog', 'bat'];

console.log(pets.includes('cat'));
// expected output: true

console.log(pets.includes('at'));
// expected output: false
```

### Syntex
arr.includes(valueToFind[, fromIndex])

### Parameters
    - valueToFind : The value to search for.

    Note: When comparing strings and characters, includes() is case-sensitive.
fromIndex Optional
    The position in this array at which to begin searching for valueToFind; the first character to be searched is found at fromIndex for positive values of fromIndex, or at array.length + fromIndex for negative values of fromIndex (using the absolute value of fromIndex as the number of characters from the end of the array at which to start the search). Defaults to 0.

Return value
Section

A Boolean which is true if the value valueToFind is found within the array (or the part of the array indicated by the index fromIndex, if specified). Values of zero are all considered to be equal regardless of sign (that is, -0 is considered to be equal to both 0 and +0), but false is not considered to be the same as 0.

Note: Technically speaking, includes() uses the sameValueZero algorithm to determine whether the given element is found.
Examples
Section

```JavaScript
[1, 2, 3].includes(2);     // true
[1, 2, 3].includes(4);     // false
[1, 2, 3].includes(3, 3);  // false
[1, 2, 3].includes(3, -1); // true
[1, 2, NaN].includes(NaN); // true
```

fromIndex is greater than or equal to the array length
Section

If fromIndex is greater than or equal to the length of the array, false is returned. The array will not be searched.

```JavaScript
var arr = ['a', 'b', 'c'];

arr.includes('c', 3);   // false
arr.includes('c', 100); // false
```

Computed index is less than 0
Section

If fromIndex is negative, the computed index is calculated to be used as a position in the array at which to begin searching for valueToFind. If the computed index is less or equal than -1 * array.length, the entire array will be searched.

```JavaScript
// array length is 3
// fromIndex is -100
// computed index is 3 + (-100) = -97

var arr = ['a', 'b', 'c'];

arr.includes('a', -100); // true
arr.includes('b', -100); // true
arr.includes('c', -100); // true
arr.includes('a', -2); // false
```

includes() used as a generic method
Section

includes() method is intentionally generic. It does not require this value to be an Array object, so it can be applied to other kinds of objects (e.g. array-like objects). The example below illustrates includes() method called on the function's arguments object.

```JavaScript
(function() {
  console.log([].includes.call(arguments, 'a')); // true
  console.log([].includes.call(arguments, 'd')); // false
})('a','b','c');
```

ES2016 feature: exponentiation operator (**)
[2016-02-01] dev, javascript, esnext, es2016
(Ad, please don’t block)

The exponentiation operator (**) is an ECMAScript proposal by Rick Waldron. It is at stage 4 (finished) and part of ECMAScript 2016.
## An infix operator for exponentiation  

### ** is an infix operator for exponentiation:

```JavaScript
x ** y
```

produces the same result as
```Javascript
Math.pow(x, y)
```

Examples:

```JavaScript
let squared = 3 ** 2; // 9

let num = 3;
num **= 2;
console.log(num); // 9
```
