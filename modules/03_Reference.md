# Reference

## Module syntax

TODO: copy and edit from existing website

## The `module` compiler option

This section discusses the details of each `module` compiler option value. See the [Module output format](./01_Theory.md#the-module-output-format) theory section for more background on what the option is and how it fits into the overall compilation process. In brief, the `module` compiler option was historically only used to control the output module format of emitted JavaScript files. The more recent `node16` and `nodenext` values, however, describe a wide range of characteristics of Node.js’s module system, including what module formats are supported, how the module format of each file is determined, and how different module formats interoperate.

### `node16`, `nodenext`

Node.js supports both CommonJS and ECMAScript modules, with specific rules for which format each file can be and how the two formats are allowed to interoperate. `node16` and `nodenext` describe the full range of behavior for Node.js’s dual-format module system, and **emit files in either CommonJS or ESM format**. This is different from every other `module` option, which are runtime-agnostic and force all output files into a single format, leaving it to the user to ensure the output is valid for their runtime.

> A common misconception is that `node16` and `nodenext` only emit ES modules. In reality, `node16` and `nodenext` describe versions of Node.js that _support_ ES modules, not just projects that _use_ ES modules. Both ESM and CommonJS emit are supported, based on the [detected module format](#module-format-detection) of each file. Because `node16` and `nodenext` are the only `module` options that reflect the complexities of Node.js’s dual module system, the are the **only correct `module` options** for all apps and libraries that are intended to run in Node.js v12 or later, whether they use ES modules or not.

`node16` and `nodenext` are currently identical, with the exception that they [imply different `target` option values](#implied-and-enforced-options). If Node.js makes significant changes to its module system in the future, `node16` will be frozen while `nodenext` will be updated to reflect the new behavior.

#### Module format detection

- `.mts`/`.mjs`/`.d.mts` files are always ES modules.
- `.cts`/`.cjs`/`.d.cts` files are always CommonJS modules.
- `.ts`/`.tsx`/`.js`/`.jsx`/`.d.ts` files are ES modules if the nearest ancestor package.json file contains `"type": "module"`, otherwise CommonJS modules.

The detected module format of input `.ts`/`.tsx`/`.mts`/`.cts` files determines the module format of the emitted JavaScript files. So, for example, a project consisting entirely of `.ts` files will emit all CommonJS modules by default under `--module nodenext`, and can be made to emit all ES modules by adding `"type": "module"` to the project package.json.

#### Interoperability rules

- **When an ES module references a CommonJS module:**
  - The `module.exports` of the CommonJS module is available as a default import to the ES module.
  - Properties (other than `default`) of the CommonJS module’s `module.exports` may or may not be available as named imports to the ES module. Node.js attempts to make them available via [static analysis](https://github.com/nodejs/cjs-module-lexer). TypeScript cannot know from a declaration file whether that static analysis will succeed, and optimistically assumes it will. This limits TypeScript’s ability to catch named imports that may crash at runtime. See [#54018](https://github.com/nodejs/cjs-module-lexer) for more details.
- **When a CommonJS module references an ES module:**
  - `require` cannot reference an ES module. For TypeScript, this includes `import` statements in files that are [detected](#module-format-detection) to be CommonJS modules, since those `import` statements will be transformed to `require` calls in the emitted JavaScript.
  - A dynamic `import()` call may be used to import an ES module. It returns a Promise of the module’s Module Namespace Object (what you’d get from `import * as ns from "./module.js"` from another ES module).

#### Emit

The emit format of each file is determined by the [detected module format](#module-format-detection) of each file. ESM emit is similar to [`--module esnext`](#es2015-es2020-es2022-esnext), but has a special transformation for `import x = require("...")`, which is not allowed in `--module esnext`:

```ts
import x = require("mod");
```

```js
import { createRequire as _createRequire } from "module";
const __require = _createRequire(import.meta.url);
const x = __require("mod");
```

CommonJS emit is similar to [`--module commonjs`](#commonjs), but dynamic `import()` calls are not transformed. Emit here is shown with `esModuleInterop` enabled:

```ts
import fs from "fs"; // transformed
const dynamic = import("mod"); // not transformed
```

```js
"use strict";
var __importDefault = (this && this.__importDefault) || function (mod) {
    return (mod && mod.__esModule) ? mod : { "default": mod };
};
Object.defineProperty(exports, "__esModule", { value: true });
const fs_1 = __importDefault(require("fs")); // transformed
const dynamic = import("mod"); // not transformed
```

#### Implied and enforced options

- `--module nodenext` or `node16` implies and enforces the `moduleResolution` with the same name.
- `--module nodenext` implies `--target esnext`.
- `--module node16` implies `--target es2022`.
- `--module nodenext` or `node16` implies `--esModuleInterop`.

#### Summary

- `node16` and `nodenext` are the only correct `module` options for all apps and libraries that are intended to run in Node.js v12 or later, whether they use ES modules or not.
- `node16` and `nodenext` emit files in either CommonJS or ESM format, based on the [detected module format](#module-format-detection) of each file.
- Node.js’s interoperability rules between ESM and CJS are reflected in type checking.
- ESM emit transforms `import x = require("...")` to a `require` call constructed from a `createRequire` import.
- CommonJS emit leaves dynamic `import()` calls untransformed, so CommonJS modules can asynchronously import ES modules.

### `es2015`, `es2020`, `es2022`, `esnext`

#### Summary

- Use `esnext` with `--moduleResolution bundler` for bundlers, Bun, and tsx.
- Do not use for Node.js. Use `node16` or `nodenext` with `"type": "module"` in package.json to emit ES modules for Node.js.
- `import mod = require("mod")` is not allowed in non-declaration files.
- `es2020` adds support for `import.meta` properties.
- `es2022` adds support for top-level `await`.
- `esnext` is a moving target that may include support for Stage 3 proposals to ECMAScript modules.
- Emitted files are ES modules, but dependencies may be any format.

#### Examples

```ts
import x, { y, z } from "mod";
import * as mod from "mod";
const dynamic = import("mod");
console.log(x, y, z, mod, dynamic);

export const e1 = 0;
export default "default export";
```

```js
import x, { y, z } from "mod";
import * as mod from "mod";
const dynamic = import("mod");
console.log(x, y, z, mod, dynamic);

export const e1 = 0;
export default "default export";
```

### `commonjs`

#### Summary

- You probably shouldn’t use this. Use `node16` or `nodenext` to emit CommonJS modules for Node.js.
- Emitted files are CommonJS modules, but dependencies may be any format.
- Dynamic `import()` is transformed to a Promise of a `require()` call.
- `esModuleInterop` affects the output code for default and namespace imports.

#### Examples

> Output is shown with `esModuleInterop: false`.

```ts
import x, { y, z } from "mod";
import * as mod from "mod";
const dynamic = import("mod");
console.log(x, y, z, mod, dynamic);

export const e1 = 0;
export default "default export";
```

```js
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
exports.e1 = void 0;
const mod_1 = require("mod");
const mod = require("mod");
const dynamic = Promise.resolve().then(() => require("mod"));

console.log(mod_1.default, mod_1.y, mod_1.z, mod);
exports.e1 = 0;
exports.default = "default export";
```

```ts
import mod = require("mod");
console.log(mod);

export = {
    p1: true,
    p2: false
};
```

```js
"use strict";
const mod = require("mod");
console.log(mod);

module.exports = {
    p1: true,
    p2: false
};
```

### `amd`

#### Summary

- Designed for AMD loaders like RequireJS.
- You probably shouldn’t use this. Use a bundler instead.
- Emitted files are AMD modules, but dependencies may be any format.
- Supports `outFile`.

#### Examples

```ts
import x, { y, z } from "mod";
import * as mod from "mod";
const dynamic = import("mod");
console.log(x, y, z, mod, dynamic);

export const e1 = 0;
export default "default export";
```

```js
define(["require", "exports", "mod", "mod"], function (require, exports, mod_1, mod) {
    "use strict";
    Object.defineProperty(exports, "__esModule", { value: true });
    exports.e1 = void 0;
    const dynamic = new Promise((resolve_1, reject_1) => { require(["mod"], resolve_1, reject_1); });

    console.log(mod_1.default, mod_1.y, mod_1.z, mod, dynamic);
    exports.e1 = 0;
    exports.default = "default export";
});
```

### `umd`

#### Summary

- Designed for AMD or CommonJS loaders.
- Does not expose a global variable like most other UMD wrappers.
- You probably shouldn’t use this. Use a bundler instead.
- Emitted files are UMD modules, but dependencies may be any format.

#### Examples

```ts
import x, { y, z } from "mod";
import * as mod from "mod";
const dynamic = import("mod");
console.log(x, y, z, mod, dynamic);

export const e1 = 0;
export default "default export";
```

```js
(function (factory) {
    if (typeof module === "object" && typeof module.exports === "object") {
        var v = factory(require, exports);
        if (v !== undefined) module.exports = v;
    }
    else if (typeof define === "function" && define.amd) {
        define(["require", "exports", "mod", "mod"], factory);
    }
})(function (require, exports) {
    "use strict";
    var __syncRequire = typeof module === "object" && typeof module.exports === "object";
    Object.defineProperty(exports, "__esModule", { value: true });
    exports.e1 = void 0;
    const mod_1 = require("mod");
    const mod = require("mod");
    const dynamic = __syncRequire ? Promise.resolve().then(() => require("mod")) : new Promise((resolve_1, reject_1) => { require(["mod"], resolve_1, reject_1); });

    console.log(mod_1.default, mod_1.y, mod_1.z, mod, dynamic);
    exports.e1 = 0;
    exports.default = "default export";
});
```

## The `moduleResolution` compiler option

This section describes module resolution features and processes shared by multiple `moduleResolution` modes, then specifies the details of each mode. See the [Module resolution](./01_Theory.md#module-resolution) theory section for more background on what the option is and how it fits into the overall compilation process. In brief, `moduleResolution` controls how TypeScript resolves _module specifiers_ (string literals in `import`/`export`/`require` statements) to files on disk, and should be set to match the module resolver used by the target runtime or bundler.

### Common features and processes

#### File extension substitution

TypeScript always wants to resolve internally to a file that can provide type information, while ensuring that the runtime or bundler can use the same path to resolve to a file that provides a JavaScript implementation. For any module specifier that would, according to the `moduleResolution` algorithm specified, trigger a lookup of a JavaScript file in the runtime or bundler, TypeScript will first try to find a TypeScript implementation file or type declaration file with the same name and analagous file extension.

| Runtime lookup | TypeScript lookup #1 | TypeScript lookup #2 | TypeScript lookup #3 |
| -------------- | -------------------- | -------------------- | -------------------- |
| `/mod.js`      | `/mod.ts`            | `/mod.d.ts`          | `/mod.js`            |
| `/mod.mjs`     | `/mod.mts`          | `/mod.d.mts`          | `/mod.mjs`           |
| `/mod.cjs`     | `/mod.cts`          | `/mod.d.cts`          | `/mod.cjs`           |

Note that this behavior is independent of the actual module specifier written in the import. This means that TypeScript can resolve to a `.ts` or `.d.ts` file even if the module specifier explicitly uses a `.js` file extension:

```ts
import x from "./mod.js";
// Runtime lookup: "./mod.js"
// TypeScript lookup #1: "./mod.ts"
// TypeScript lookup #2: "./mod.d.ts"
// TypeScript lookup #3: "./mod.js"
```

#### Relative file path resolution

All of TypeScript’s `moduleResolution` algorithms support referencing a module by a relative path that includes a file extension (which will be substituted according to the [rules above](#file-extension-substitution)):

```ts
// @Filename: a.ts
export {};

// @Filename: b.ts
import {} from "./a.js"; // ✅ Works in every `moduleResolution`
```

#### `baseUrl`

TODO

#### `paths`

TODO

#### `node_modules` package lookups

Node.js treats module specifiers that aren’t relative paths, absolute paths, or URLs as references to packages that it looks up in `node_modules` subdirectories. Bundlers conveniently adopted this behavior to allow their users to use the same dependency management system, and often even the same dependencies, as they would in Node.js. All of TypeScript’s `moduleResolution` options except `classic` support `node_modules` lookups. Every `node_modules` package lookup has the following structure (beginning after higher precedence bare specifier rules, like `paths`, `baseUrl`, self-name imports, and package.json `"imports"` lookups have been exhausted):

1. For each ancestor directory of the importing file, if a `node_modules` directory exists within it:
   1. If a directory with the same name as the package exists within `node_modules`:
      1. Attempt to resolve types from the package directory.
      2. If a result is found, return it and stop the search.
   2. If a directory with the same name as the package exists within `node_modules/@types`:
      1. Attempt to resolve types from the `@types` package directory.
      2. If a result is found, return it and stop the search.
2. Repeat the previous search through all `node_modules` directories, but this time, allow JavaScript files as a result, and do not search in `@types` directories.

All `moduleResolution` modes (except `classic`) follow this pattern, while the details of how they resolve from a package directory, once located, differ, and are explained in the following sections.

#### package.json `"exports"`

When `moduleResolution` is set to `node16`, `nodenext`, or `bundler`, and `resolvePackageJsonExports` is not disabled, TypeScript follows Node.js’s [package.json `"exports"` spec](https://nodejs.org/api/packages.html#packages_package_entry_points) when resolving from a package directory triggered by a [bare specifier `node_modules` package lookup](#node_modules-package-lookups).

TypeScript’s implementation for resolving a module specifier through `"exports"` to a file path follows Node.js exactly. Once a file path is resolved, however, TypeScript will still [try multiple file extensions](#file-extension-substitution) in order to prioritize finding types.

When resolving through [conditional `"exports"`](https://nodejs.org/api/packages.html#conditional-exports), TypeScript always matches the `"types"` and `"default"` conditions if present. Additionally, TypeScript will match a versioned types condition in the form `"types@{selector}"` (where `{selector}` is a `"typesVersions"`-compatible version selector) according to the same version-matching rules implemented in [`"typesVersions"`](#packagejson-typesversions). Other non-configurable conditions are dependent on the `moduleResolution` mode and specified in the following sections. Additional conditions can be configured to match with the `customConditions` compiler option.

##### Example: subpaths, conditions, and extension substitution

Scenario: `"pkg/subpath"` is requested with conditions `["types", "node", "require"]` (determined by `moduleResolution` setting and the context that triggered the module resolution request) in a package directory with the following package.json:

```json
{
  "name": "pkg",
  "exports": {
    "./subpath": {
      "import": "./subpath/index.mjs",
      "require": "./subpath/index.cjs"
    }
  }
}
```

Resolution process within the package directory:

1. Does `"exports"` exist? **Yes.**
2. Does `"exports"` have a `"./subpath"` entry? **Yes.**
3. The value at `exports["./subpath"]` is an object—it must be specifying conditions.
4. Does the first condition `"import"` match this request? **No.**
5. Does the second condition `"require"` match this request? **Yes.**
6. Does the path `"./subpath/index.cjs"` have a recognized TypeScript file extension? **No, so use extension substitution.**
7. Via [extension substitution](#file-extension-substitution), try the following paths, returning the first one that exists, or `undefined` otherwise:
   1. `./subpath/index.cts`
   2. `./subpath/index.d.cts`
   3. `./subpath/index.cjs`

If `./subpath/index.cts` or `./subpath.d.cts` exists, resolution is complete. Otherwise, resolution searches `node_modules/@types/pkg` and other `node_modules` directories in an attempt to resolve types, according to the [`node_modules` package lookups](#node_modules-package-lookups) rules. If no types are found, a second pass through all `node_modules` resolves to `./subpath/index.cjs` (assuming it exists), which counts as a successful resolution, but one that does not provide types, leading to `any`-typed imports and a `noImplicitAny` error if enabled.

##### Example: explicit `"types"` condition

Scenario: `"pkg/subpath"` is requested with conditions `["types", "node", "import"]` (determined by `moduleResolution` setting and the context that triggered the module resolution request) in a package directory with the following package.json:

```json
{
  "name": "pkg",
  "exports": {
    "./subpath": {
      "import": {
        "types": "./types/subpath/index.d.mts",
        "default": "./es/subpath/index.mjs"
      },
      "require": {
        "types": "./types/subpath/index.d.cts",
        "default": "./cjs/subpath/index.cjs"
      }
    }
  }
}
```

Resolution process within the package directory:

1. Does `"exports"` exist? **Yes.**
2. Does `"exports"` have a `"./subpath"` entry? **Yes.**
3. The value at `exports["./subpath"]` is an object—it must be specifying conditions.
4. Does the first condition `"import"` match this request? **Yes.**
5. The value at `exports["./subpath"].import` is an object—it must be specifying conditions.
6. Does the first condition `"types"` match this request? **Yes.**
7. Does the path `"./types/subpath/index.d.mts"` have a recognized TypeScript file extension? **Yes, so don’t use extension substitution.**
8. Return the path `"./types/subpath/index.d.mts"` if it exists, `undefined` otherwise.

##### Example: versioned `"types"` condition

Scenario: using TypeScript 4.7.5, `"pkg/subpath"` is requested with conditions `["types", "node", "import"]` (determined by `moduleResolution` setting and the context that triggered the module resolution request) in a package directory with the following package.json:

```json
{
  "name": "pkg",
  "exports": {
    "./subpath": {
      "types@>=5.2": "./ts5.2/subpath/index.d.ts",
      "types@>=4.6": "./ts4.6/subpath/index.d.ts",
      "types": "./tsold/subpath/index.d.ts",
      "default": "./dist/subpath/index.js"
    }
  }
}
```

Resolution process within the package directory:

1. Does `"exports"` exist? **Yes.**
2. Does `"exports"` have a `"./subpath"` entry? **Yes.**
3. The value at `exports["./subpath"]` is an object—it must be specifying conditions.
4. Does the first condition `"types@>=5.2"` match this request? **No, 4.7.5 is not greater than or equal to 5.2.**
5. Does the second condition `"types@>=4.6"` match this request? **Yes, 4.7.5 is greater than or equal to 4.6.**
6. Does the path `"./ts4.6/subpath/index.d.ts"` have a recognized TypeScript file extension? **Yes, so don’t use extension substitution.**
7. Return the path `"./ts4.6/subpath/index.d.ts"` if it exists, `undefined` otherwise.

#### package.json `"imports"` and self-name imports

TODO

#### package.json `"typesVersions"`

TODO

### `node16`, `nodenext`

`--moduleResolution node16` and `nodenext` must be partnered with [`--module node16` or `nodenext`](#node16-nodenext)