# Export statements

## Status

Stage: 0

## Motivation

Consider having a big function.

You could break up your function, so your definitions and return statement are closer, but then your definitions can't reference each other. This pain is **particularly noticable when most of the definitions in the function are themselves functions**.

## Description

Permit export modifiers on variable declarations within the blocks of function bodies. If the function exits without an explicit return, it will return a new object with properties for every exported identifier, so this snippet:

```js
export function makeBigThing(initialCount) {
  if (initialCount < 0) {
    return;
  }

  export const [bMultiplier, setMultiplier] = Behavior(initialCount);
  const [eIncrement, dispatchIncrement] = Revent();
  export { dispatchIncrement };
  export const bSum = eIncrement
    .reduce((acc, n) => acc + n, 0)
    .liftA2(bMultiplier, (sum, count) => sum + count);
}
```

is exactly equivalent to:

```js
export function makeBigThing(initialCount) {
  if (initialCount <= 0) {
    return;
  }

  const [bMultiplier, setMultiplier] = Behavior(initialCount);
  const [eIncrement, dispatchIncrement] = Revent();
  const bSum = eIncrement
    .reduce((acc, n) => acc + n, 0)
    .liftA2(bMultiplier, (sum, mul) => sum * mul);

  return {
    bMultiplier,
    setMultiplier,
    dispatchIncrement,
    bSum,
  };
}
```

Exports in async and generator functions are what would be expected.

- **Locality**

Having a big object return at the end of the function is the best existing alternative in my mind, however the loss of locality is very painful. Your diffs are split up and it's significantly harder to understand the public interface of any particular section.

- **Symmetry with module scope**

If you have a lot of code at the module scope, but then need to lift it to a function, supporting exports within functions would be good for symmetry. Unexpectedly needing to add a non-static input would require a less drastic change.

One weakness is that `export ... from` would still be bad syntax. While a variant of import declarations, code may vary reasonably use `export const bMessagesLeft = bSum` and `export { bSum as bMessagesLeft } from '...'.` interchangeably. Authors would still need to rewrite these cases.

- **Works well with destructing**

Any existing userland solution cannot work with destructuring anywhere near as well as expanding export modifiers would.

- **Easy for type inference**

## Comparison

### Status quo

I see two viable existing approaches:

- splitting up your function
- creating sub-objects that can be spreaded over kept as-is in the return

### Object mutation

Building the return by mutating an object does work well for locality, but doesn't work with type inference or type safety at all (when considering TypeScript). Moreover, either the names need to be redeclared twice (o.bSum = bSum)
- Doesn't work well with destructuring
- (Considering TypeScript) doesn't work with type inference or type safety at all
- Either the names need to be redeclared twice (o.bSum = bSum) or the values are only referred to from the built-up object, (o.bSum = eIncrement.reduce(...)), leaking irrelevant knowledge into consumers and increasing diff churn
