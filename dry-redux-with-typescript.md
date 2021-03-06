# Dry redux with TypeScript

THe most common complaint about redux is the amount of boiler plate code needed to make actions, action creators, action type constants and reducers. Another complaint is related to bugs that happen because state is changed when it should't be. TypeScript offers tools that can help alleviate both these problems.

## Design

It is common to structure redux codebases by putting all the reducers together in one file and all the actions together in another. I prefer a different layout where all the actions and reducer working with the same data go together in one file. With this arrangement, all reducer functions are called `reducer()`. I hope this lessens the tendency to conflate the reducer with the data it operates on, which I find confusing in some discussions of redux.

The action type constants go in a single shared file, since this makes it less likely to inadvertently duplicate action type strigs.

## Defining redux actions and store using ReturnType<T>

The `ReturnType<T>` helper takes a function and returns the type returned by that function. We can use this to define all the action types [in terms of the action creator functions](https://medium.com/@martin_hotell/improved-redux-type-safety-with-typescript-2-8-2c11a8062575).

Redux encourages us to work in a functional style where values are computed once and never changed (mutated) later. In Javascript everything is mutable, so it's easy to introduce bugs by changing state where we shouldn't. [Immutable.js](https://facebook.github.io/immutable-js/) is a widely used JS library designed to stop us from changing state. However, it requires you to always keep converting between the special immutable data structures and plain javascript object. Typescript has the `readonly` modifier and the `Readonly<T>` helper. These enforce immutability at compile time with no run time impact, which means that unlike with `immutable.js` we can work with plain javascript objects throughout.

With these tools, the typical `increment()` action and action generator can be defined without any repetion:

```ts
export const increment = (store: Store) => (
    { type: constants.SET_COUNTER, payload: { value: store.value + 1 } }
);

export type SetCounterAction = Readonly<ReturnType<typeof increment>>;
```

## Defining the Store type

The same approach can be used to define the Store type for the counter. Now the `Store` is returned by the reducer, so it would be nice to be able to say

```ts
export type Store = Readonly<ReturnType<typeof reducer>>; // won't work
```

Unfortunately, the reducer also takes a Store as argument, so this creates a circular dependency. Instead, we can have a helper function which builds the default (empty) store, and use that function to define the Store type and also call it in the reducer:

```ts
const buildDefaultStore = () => (
    { value: 0 }
);

export type Store = Readonly<ReturnType<typeof buildDefaultStore>>;

export const reducer = (store: Store = buildDefaultStore(), action?: SetCounterAction): Store => {
   ...
}
```

If you're using TsLint with the `typedef:arrow-call-signature` rule enabled, TsLint will complain about `increment()` and `buildDefaultStore()`, because neither function define a return type. They can't because we're using each to define the type using `ReturnType<T>`. Put the comment `// tslint:disable-next-line:typedef` above the functions to make those warnings go away.

If we try to always define functions in the "correct" order, i.e. always defining them before referencing them, the store files can be a bit messy, with action creators and action types interspersed. Javascript hoisting allows us to reference types before they are defined so that we can arrange the different elements of the file in any order we like, such as putting the types up top, followed by the actions, and the reducer at the end.

## Discriminated unions for types

Ideally, we want the action types to be [discriminated or "tagged" unions](https://blog.mariusschulz.com/2016/11/03/typescript-2-0-tagged-union-types). In order to demonstrate the value of this, I will add a bit of functionality to the counter example, with one more action type called `reset()` with no payload:

```ts
export const reset = () => (
    { type: constants.RESET_COUNTER }
);

export type ResetCounterAction = Readonly<ReturnType<typeof reset>>;
```

We now have to handle the two different action types in the reducer:

```ts
export const reducer = (store: Store = buildDefaultStore(), 
                        action?: SetCounterAction | ResetCounterAction): Store => {
    if (!action) {
        return store;
    }
    switch (action.type) {
        case constants.SET_COUNTER:
            return { ...store, value: action.payload.value };
        case constants.RESET_COUNTER:
            return { ...store, value: 0 };
        default:
            return store;
    }
};

```
This gives a compilation error:

```
src/stores/counter.ts(32,46): error TS2339: Property 'payload' does not exist on type 
'Readonly<{ type: string; payload: { value: number; }; }> | Readonly<{ type: string; }>'.
```

I know that the payload is there when the type is equal to `constants.SET_COUNTER`, but how to explain this to the compiler? We _could_ cast the action to the correct type in the reducer, but there is a much better way: with [this helper function](https://medium.com/@martin_hotell/improved-redux-type-safety-with-typescript-2-8-2c11a8062575) the compiler error goes away:

```
export interface Action<T extends string> { type: T };
export interface ActionWithPayload<T extends string, P> { type: T, payload: P };

export function makeAction<T extends string>(type: T): Action<T>;
export function makeAction<T extends string, P>(type: T, payload: P): ActionWithPayload<T, P>;
export function makeAction<T extends string, P>(type: T, payload?: P) {
    return payload === undefined ? { type } : { type, payload };
};

```

When actions are created using `makeAction()`, the compiler knows that when the _value_ of `action.type` is `constants.SET_COUNTER`, then the _type_ of payload is `{ value: number }`, but when the _value_ of `action.type` is `constants.RESET_COUNTER`, then there is no payload. No casts needed!

So this is the counter store in its entirety:

```ts
import * as constants from '../application/constants';
import * as helpers from '../application/helpers/redux-helpers';

export type Store = Readonly<ReturnType<typeof buildDefaultStore>>;
export type SetCounterAction = Readonly<ReturnType<typeof increment>>;
export type ResetCounterAction = Readonly<ReturnType<typeof reset>>;

// tslint:disable-next-line:typedef
export const increment = (store: Store) => (
    helpers.makeAction(constants.SET_COUNTER, { value: store.value + 1 })
);

export const decrement = (store: Store): SetCounterAction => (
    helpers.makeAction(constants.SET_COUNTER, { value: store.value - 1 })
);

// tslint:disable-next-line:typedef
export const reset = () => (
    helpers.makeAction(constants.RESET_COUNTER)
);

// tslint:disable-next-line:typedef
const buildDefaultStore = () => (
    { value: 0 }
);

export const reducer = (store: Store = buildDefaultStore(), 
                        action?: SetCounterAction | ResetCounterAction): Store => {
    if (!action) {
        return store;
    }
    switch (action.type) {
        case constants.SET_COUNTER:
            return { ...store, value: action.payload.value };
        case constants.RESET_COUNTER:
            return { ...store, value: 0 };
        default:
            return store;
    }
};
```

## Scaling up the store

Separating concerns in the store leads to a root `Store` type with a number of sub-stores for different parts of the application domain. This means that the call to `combineReducers()` could end up taking a lot of arguments. 

I would like to have a single file that combines all the sub-stores into one application store type and root reducer and does nothing else. This file should be the only place in the architecture that knows about all the different sub-stores. This file should also not know anything about the details inside each of the sub-stores. This approach looks like it should be able to scale up nicely:

```ts
import { combineReducers } from 'redux';
import * as counter from './counter';
import * as message from './message';

export interface Store {
    readonly counterInStore: counter.Store;
    readonly messageInStore: message.Store;
}

export const rootReducer = combineReducers({
    counterInStore: counter.reducer,
    messageInStore: message.reducer,
});
```

