# React-native tech stack with TypeScript

react-native, expo, typescript, tslint, redux, jest, enzyme, snapshots

## Bootstrap: create-react-native-app

The project was bootstrapped with [create-react-native-app](https://facebook.github.io/react-native/blog/2017/03/13/introducing-create-react-native-app.html), which defaults to using [expo](https://expo.io/) for running your app on a phone or phone emulator, [jest](https://facebook.github.io/jest/) and react-test-renderer for testing, flow for typesafety and npm. Expo gives you access to a lot of [features](https://expo.io/features) on the phone, such as the camera, location, etc. I swithced from flow to TypeScript, see below. I also prefer yarn to npm, it's a easy switch to make since they both work off of the same `package.json` file format.

## Store: redux

I'm using redux for local store, because the [flux architecture](https://www.startuprocket.com/articles/evolution-toward-one-way-data-flow-a-quick-introduction-to-redux) with unidirectional flow of information just makes so much sense. These are the packages to install:

```
yarn add redux react-redux 
yarn add --dev redux-devtools
```

There is always quite a bit of boiler plate needed. I started out writing all the boilerplate for a very simple hello-world app, then I tweaked it to a more DRY architecture, which I wrote up [separately](https://github.com/pg-irc/pathways-backend/blob/master/dry-redux-with-typescript.md).

## Typescript

I'm using TypeScript to squash bugs before they happen. With version 2.8, there is `ReturnType<>` which can be used to [simplify](https://medium.com/@martin_hotell/improved-redux-type-safety-with-typescript-2-8-2c11a8062575) a lot of the Redux boilerplate. With `Readonly<>`, TypeScript can enforce immutability at compile time, so there should be no need for `immutable.js`. I haven't worked with `immutable.js`, but it [seems](https://medium.com/@AlexFaunt/immutablejs-worth-the-price-66391b8742d4) to be good to avoid if possible.

Due to a collision between node and TypeScript both defining `require`, a [post-install script](https://github.com/aws/aws-amplify/issues/281#issuecomment-370049435) is needed that removes the node implementation. I created a file `bin/postinstall.sh` with this line in it:

```
sed -i -e "s/\(^declare var require: NodeRequire;\)/\/\/\1/g" node_modules/\@types/node/index.d.ts
```

and added this to `package.json`:

```json
"scripts": {
    "postinstall": "bin/postinstall.sh"
}
```

I then followed [this excellent write-up](https://medium.com/@rintoj/react-native-with-typescript-40355a90a5d7),  skipping straight to "Setup TypeScript" and followed along as far as "Refactor iOS to TypeScript". Then I renamed all my `.js` files to `.ts`, fixed all the errors from `yarn tsc` and finally removed the `.flowconfig` file that `create-react-native-app` created to complete the move from flow to TypeScript.

Configuration in `tsconfig.json` is locked down to make the compiler as thorough as practical, including setting `noImplicitAny`, `noImplicitReturns`, `noUnusedParameters`  and `noUnusedLocals`  to `true`. I ran into [this problem](https://github.com/DefinitelyTyped/DefinitelyTyped/issues/24573) with a library that the TypeScript compiler didn't like, it helped to set `skipLibCheck` to true in `tsconfig.json`.

## TsLint

TsLint also helps squash a lot of bugs before they happen. I again prefer pretty locked down settings, including disallowing `any`, but follow your preferences. [tslint-immutable](https://github.com/jonaskello/tslint-immutable) is a set of additional tslint rules that enforce functional programming style and immutability that lines up really nicey with redux. Make sure the version of tslint-immutable is [compatible](https://github.com/jonaskello/tslint-immutable) with the version of tslint you're using. I installed with

```
yarn add --dev tslint-immutable
```

and added this script to `package.json`:

```json
"scripts": { 
    "lint": "tslint --rules-dir node_modules/tslint-immutable/rules/ src/**/*.ts"
}
```

## Testing

A number of packages are needed for testing. The "16" in `enzyme-adapter-react-16` refers to the `react` version.

```
yarn add --dev babel-jest enzyme enzyme-adapter-react-16 enzyme-to-json jest react-dom ts-jest
yarn add --dev @types/enzyme @types/enzyme-adapter-react-16
```

As suggested by `create-react-native-app` I use jest as the test library. For testing react, I'm [also](https://www.codementor.io/vijayst/unit-testing-react-components-jest-or-enzyme-du1087lh8) using [enzyme](http://airbnb.io/projects/enzyme/). To make enzyme work, a simple setup file is needed, I named it `src/application/__tests__/setup_tests.ts` with this content:

```TypeScript
import Enzyme from 'enzyme';
import EnzymeAdapter from 'enzyme-adapter-react-16';

Enzyme.configure({
    adapter: new EnzymeAdapter()
});
```

and pointed at it from `packages.json` like so:

```json
"jest": {
  "setupTestFrameworkScriptFile": "<rootDir>/lib/application/__tests__/setup_tests.js",
}
```

`setup_tests.ts` is compiled like all TypeScript files, so `packages.json` is refering to the compiled js file that ends up in the `/lib` folder. BTW, the output location `lib` for compiled files is set in `tsconfig.json` in `outDir` under `compilerOptions`.

Jest complains about the lack of an ErrorBoundary component, so I put one in (`src/application/error_boundary.ts`), don't know if it works yet, jest is still complaining. I will update when I figure out what the problem is.

## Snapshot testing

I have one working snapshot test. This needs a setting  `snapshotSerializers` under jest in `package.json`. I found quit a few articles with incorrect information on this, this is how I got it working:

```json
"jest": {
    "snapshotSerializers": [
        "enzyme-to-json/serializer"
    ],
}
```

I'm looking forward to using snapshots for some real world testing to see if it lives up to the hype.

TypeScript and snapshots don't play really well together, since the snapshots end up being written to the `/lib` folder next to the compiled js code. This means you can no longer just nuke /lib to clean, so I had to change the `clean` script in `package.json`:

```
"scripts": {
    "clean": "rimraf lib/**/*.js lib/**/*.js.map",
}
```

I also had to update `.gitignore` so that I could commit the snapshot files, so rather than ignoring all of `/lib`, I have:

```
/lib/**/*js
/lib/**/*map
```

The snapshots are written to a folder called `__snapshots__`, so for consistency I've had to stick with the conventional name `__tests__` for test folders. I was hoping to buck that convention and use `tests`, but that's probably just good for my OCD.

