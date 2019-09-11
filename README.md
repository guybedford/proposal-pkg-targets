# Package Targets Proposal

_Contributors: Jan Krems, Guy Bedford, Jamie Kyle_

Different JS runtimes and tools tend to follow their own rules on how to resolve the main entry point of a JavaScript package through the `package.json`.

For example, depending on the tool or runtime you might load the entry point as - `"main"`, `"browser"`, `"react-native"`, `"electron"`, `"module"`, to name a few.

There are a number of problems with continuing to scale this approach including:

1. There is no single resolution algorithm that can explain what will get loaded and why.
1. These resolutions only apply to the entry point, but packages might want to similarly map subpaths as well. Eg the `"browser"` field can be an object to support internal remapping.
1. There is no ability to compose conditions - eg `"module"` + `"browser"` cannot be distinguished from `"module"` + `"node"`.

## Proposal

The proposal is to generalize the concept of a **target** as exactly one of these environment names, and to define a generic resolver that can apply predictable target matching
both for the entry point itself as well as for subpaths through features such as package exports, currently supported under Node.js --experimental-modules.

### Targets

Targets are strings that the resolver knows about and will resolve based on a priority order.

For example, given the list - `targets: ['browser', 'main']`, and the package.json file:

```json
{
  "main": "./index.js",
  "browser": "./index-browser.js"
}
```

The `index-browser.js` file would resolve as the entry point as it has a higher priority than the `"main"`.

It is up to the resolver of the runtime, tool or environment to decide which targets it wants to support, and in what priority order.

#### Proposed Targets

The initial proposed list of targets is the following:

* _main_: The default fallback target for any environment.
* _browser_: A web browser environment, that is not also a Node.js environment (eg Electron does not follow the browser field).
* _electron_: Electron, as defined by the runtime of the Electron project, or any fork of it.
* _react-native_: React Native, as defined by the runtime of the React Native project, or any fork of it.
* _development_: Any environment which can be considered to be running in development mode.
* _production_: Any environment which can be considered to be running in production mode.

_Anyone can define a target name in their tool or workflow._

If, on the other hand, users wish to try and get consensus for a target definition between tools and environments, the spec can provide a simple process for anyone to register a target definition to avoid any possible conflicts of meaning, but this process is entirely optional, and should have strong checks in place to ensure definition quality.

### Target Definitions

Target maps are not always sufficient to provide all the flexibility users might want from conditional resolution.

Sometimes package authors want to carefully define the environments in which target maps should apply.

Parcel 2 has an RFC for a [`"targets"` field in the `package.json`](https://github.com/parcel-bundler/parcel/blob/master/PARCEL_2_RFC.md#targets) which would allow these definitions via eg:

```
{
  "targets": {
    "main": {
      "engines": {
        "node": ">=4.x",
        "electron": ">=2.x"
      },
      "browsers": ["> 1%", "not dead"]
    }
  }
}
```

Where the resolver is informed to resolve the `"main"` target only when the provided conditions match, and not to match it otherwise.

The targets proposal as provided here is designed to be fully compatible with such target definitions, since it is completely up to the resolver which targets it will resolve.

### Target Maps

The support for targets is extended beyond entry points to also support the [Node.js package exports proposal](https://nodejs.org/dist/latest-v12.x/docs/api/esm.html#esm_package_exports).

Given the definition of targets and the ability for a resolver to detect targets in a priority order, we extend the support for targets from the entry point to the package.json `"exports"`
by allowing an `"exports"` target to map into an object:

```json
{
  "exports": {
    "./features/": {
      "browser": "./features-browser/",
      "main": "./features/"
    }
  }
}
```

In the above, when the `"browser"` target is included in the resolver, a request to `pkg/features/x.js` will resolve to `pkg/features-browser/x.js`, while
in other environments it would resolve to `pkg/features/x.js`.

To make the above compatible with Node.js support for `"exports"`, which does not currently support the above proposal, we can use package fallbacks:

```json
{
  "exports": {
    "./features/": [{
      "browser": "./features-browser/"
    }, "./features/"]
  }
}
```

Because the target map is not supported in Node.js, it falls back to using `"./features/"` by default, while resolvers that support the target map can resolve the browser target if it applies.

#### Composition through Nesting

In addition, targets can compose through nesting:

```json
{
  "exports": {
    "./features/": [{
      "browser": {
        "development": "./features-browser-dev/",
        "production": "./features-browser-production/"
      }
    }, "./features/"]
  }
}
```

allowing splitting of `pkg/features/x.js` resolution between browser production and browser development environments.

#### Not Found Target

In addition to string targets it is also possible to define a not found target which indicates that there should be no mapping for the given module:

```js
{
  "browser": false,
  "exports": {
    "./": { browser: false }
  }
}
```

The above implies that a _Module Not Found_ error should be thrown when attempting to resolve the given resolutions for the browser.

### Full Type Definitions

Combining all of the above, the proposal types can be defined as (in TypeScript):

```typescript
type TargetValue = string | false | TargetMap | TargetArray;
interface TargetMap {
  [conditional: string]: TargetValue;
};
interface TargetArray extends Array<TargetValue> {};
```

where the individual targets are defined as type `TargetValue`, and the _exports_ field is defined as a map of target values:

```typescript
type PackageExports = string | false | {
  [key: string]: TargetValue
};
```


