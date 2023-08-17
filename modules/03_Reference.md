# Reference

## Module syntax

TODO: copy and edit from existing website

## The `module` compiler option

This section discusses the details of each `module` compiler option value. See the [_Module output format_](./01_Theory.md#the-module-output-format) theory section for more background on what the option is and how it fits into the overall compilation process. In brief, the `module` compiler option was historically only used to control the output module format of emitted JavaScript files. The more recent `node16` and `nodenext` values, however, describe a wide range of characteristics of Node.js’s module system, including what module formats are supported, how the module format of each file is determined, and how different module formats interoperate.

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

This section describes module resolution features and processes shared by multiple `moduleResolution` modes, then specifies the details of each mode. See the [_Module resolution_](./01_Theory.md#module-resolution) theory section for more background on what the option is and how it fits into the overall compilation process. In brief, `moduleResolution` controls how TypeScript resolves _module specifiers_ (string literals in `import`/`export`/`require` statements) to files on disk, and should be set to match the module resolver used by the target runtime or bundler.

### Common features and processes

#### File extension substitution

TypeScript always wants to resolve internally to a file that can provide type information, while ensuring that the runtime or bundler can use the same path to resolve to a file that provides a JavaScript implementation. For any module specifier that would, according to the `moduleResolution` algorithm specified, trigger a lookup of a JavaScript file in the runtime or bundler, TypeScript will first try to find a TypeScript implementation file or type declaration file with the same name and analagous file extension.

| Runtime lookup | TypeScript lookup #1 | TypeScript lookup #2 | TypeScript lookup #3 | TypeScript lookup #4 | TypeScript lookup #5 |
| -------------- | -------------------- | -------------------- | -------------------- | -------------------- | -------------------- |
| `/mod.js`      | `/mod.ts`            | `/mod.tsx`           | `/mod.d.ts`          | `/mod.js`            | `./mod.jsx`          |
| `/mod.mjs`     | `/mod.mts`           | `/mod.d.mts`         | `/mod.mjs`           |                      |                      |
| `/mod.cjs`     | `/mod.cts`           | `/mod.d.cts`         | `/mod.cjs`           |                      |                      |

Note that this behavior is independent of the actual module specifier written in the import. This means that TypeScript can resolve to a `.ts` or `.d.ts` file even if the module specifier explicitly uses a `.js` file extension:

```ts
import x from "./mod.js";
// Runtime lookup: "./mod.js"
// TypeScript lookup #1: "./mod.ts"
// TypeScript lookup #2: "./mod.d.ts"
// TypeScript lookup #3: "./mod.js"
```

See [_TypeScript imitates the host’s module resolution, but with types_](./01_Theory.md#typescript-imitates-the-hosts-module-resolution-but-with-types) for an explanation of why TypeScript’s module resolution works this way.

#### Relative file path resolution

All of TypeScript’s `moduleResolution` algorithms support referencing a module by a relative path that includes a file extension (which will be substituted according to the [rules above](#file-extension-substitution)):

```ts
// @Filename: a.ts
export {};

// @Filename: b.ts
import {} from "./a.js"; // ✅ Works in every `moduleResolution`
```

#### Extensionless relative paths

In some cases, the runtime or bundler allows omitting a `.js` file extension from a relative path. TypeScript supports this behavior where the `moduleResolution` setting and the context indicate that the runtime or bundler supports it:

```ts
// @Filename: a.ts
export {};

// @Filename: b.ts
import {} from "./a";
```

If TypeScript determines that the runtime will perform a lookup for `./a.js` given the module specifier `"./a"`, then `./a.js` will undergo [extension substitution](#file-extension-substitution), and resolve to the file `a.ts` in this example.

Extensionless relative paths are not supported in `import` paths in Node.js, and are not always supported in file paths specified in package.json files. TypeScript currently never supports omitting a `.mjs`/`.mts` or `.cjs`/`.cts` file extension, even though some runtimes and bundlers do.

#### Directory modules (index file resolution)

In some cases, a directory, rather than a file, can be referenced as a module. In the simplest and most common case, this involves the runtime or bundler looking for an `index.js` file in a directory. TypeScript supports this behavior where the `moduleResolution` setting and the context indicate that the runtime or bundler supports it:

```ts
// @Filename: dir/index.ts
export {};

// @Filename: b.ts
import {} from "./dir";
```

If TypeScript determines that the runtime will perform a lookup for `./dir/index.js` given the module specifier `"./dir"`, then `./dir/index.js` will undergo [extension substitution](#file-extension-substitution), and resolve to the file `dir/index.ts` in this example.

Directory modules may also contain a package.json file, where resolution of the [`"main"` and `"types"`](#packagejson-main-and-types) fields are supported, and take precedence over `index.js` lookups. The [`"typesVersions"`](#packagejson-typesversions) field is also supported in directory modules.

Note that directory modules are not the same as [`node_modules` packages](#node_modules-package-lookups) and only support a subset of the features available to packages, and are not supported at all in some contexts.  Node.js considers them a [legacy feature](https://nodejs.org/dist/latest-v20.x/docs/api/modules.html#folders-as-modules).

#### `baseUrl`

TODO

#### `paths`

TODO


#### `node_modules` package lookups

Node.js treats module specifiers that aren’t relative paths, absolute paths, or URLs as references to packages that it looks up in `node_modules` subdirectories. Bundlers conveniently adopted this behavior to allow their users to use the same dependency management system, and often even the same dependencies, as they would in Node.js. All of TypeScript’s `moduleResolution` options except `classic` support `node_modules` lookups. (`classic` supports lookups in `node_modules/@types` when other means of resolution fail, but never looks for packages in `node_modules` directly.) Every `node_modules` package lookup has the following structure (beginning after higher precedence bare specifier rules, like `paths`, `baseUrl`, self-name imports, and package.json `"imports"` lookups have been exhausted):

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

Note that the presence of `"exports"` prevents any subpaths not explicitly listed or matched by a pattern in `"exports"` from being resolved.

##### Example: subpaths, conditions, and extension substitution

Scenario: `"pkg/subpath"` is requested with conditions `["types", "node", "require"]` (determined by `moduleResolution` setting and the context that triggered the module resolution request) in a package directory with the following package.json:

```json
{
  "name": "pkg",
  "exports": {
    ".": {
      "import": "./index.mjs",
      "require": "./index.cjs"
    },
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
8. Return the path `"./types/subpath/index.d.mts"` if the file exists, `undefined` otherwise.

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
7. Return the path `"./ts4.6/subpath/index.d.ts"` if the file exists, `undefined` otherwise.

##### Example: subpath patterns

Scenario: `"pkg/wildcard.js"` is requested with conditions `["types", "node", "import"]` (determined by `moduleResolution` setting and the context that triggered the module resolution request) in a package directory with the following package.json:

```json
{
  "name": "pkg",
  "type": "module",
  "exports": {
    "./*.js": {
      "types": "./types/*.d.ts",
      "default": "./dist/*.js"
    }
  }
}
```

Resolution process within the package directory:

1. Does `"exports"` exist? **Yes.**
2. Does `"exports"` have a `"./wildcard.js"` entry? **No.**
3. Does any key with a `*` in it match `"./wildcard.js"`? **Yes, `"./*.js"` matches and sets `wildcard` to be the substitution.**
4. The value at `exports["./*.js"]` is an object—it must be specifying conditions.
5. Does the first condition `"types"` match this request? **Yes.**
6. In `./types/*.d.ts`, replace `*` with the substitution `wildcard`. **`./types/wildcard.d.ts`**
7. Does the path `"./types/wildcard.d.ts"` have a recognized TypeScript file extension? **Yes, so don’t use extension substitution.**
8. Return the path `"./types/wildcard.d.ts"` if the file exists, `undefined` otherwise.

##### Example: `"exports"` block other subpaths

Scenario: `"pkg/dist/index.js"` is requested in a package directory with the following package.json:

```json
{
  "name": "pkg",
  "main": "./dist/index.js",
  "exports": "./dist/index.js"
}
```

Resolution process within the package directory:

1. Does `"exports"` exist? **Yes.**
2. The value at `exports` is a string—it must be a file path for the package root (`"."`).
3. Is the request `"pkg/dist/index.js"` for the package root? **No, it has a subpath `dist/index.js`.**
4. Resolution fails; return `undefined`.

Without `"exports"`, the request could have succeeded, but the presence of `"exports"` prevents resolving any subpaths that cannot be matched through `"exports"`.

#### package.json `"typesVersions"`

A [`node_modules` package](#node_modules-package-lookups) or [directory module](#directory-modules-index-file-resolution) may specify a `"typesVersions"` field in its package.json to redirect TypeScript’s resolution process according to the TypeScript compiler version, and for `node_modules` packages, according to the subpath being resolved. This allows package authors to include new TypeScript syntax in one set of type definitions while providing another set for backward compatibility with older TypeScript versions (through a tool like [downlevel-dts](https://github.com/sandersn/downlevel-dts)). `"typesVersions"` is supported in all `moduleResolution` modes; however, the field is not read in situations when [package.json `"exports"`](#packagejson-exports) are read.

##### Example: redirect all requests to a subdirectory

Scenario: a module imports `"pkg"` using TypeScript 5.2, where `node_modules/pkg/package.json` is:

```json
{
  "name": "pkg",
  "version": "1.0.0",
  "types": "./index.d.ts",
  "typesVersions": {
    ">=3.1": {
      "*": ["ts3.1/*"]
    }
  }
}
```

Resolution process:

1. (Depending on compiler options) Does `"exports"` exist? **No.**
2. Does `"typesVersions"` exist? **Yes.**
3. Is the TypeScript version `>=3.1`? **Yes. Remember the mapping `"*": ["ts3.1/*"]`.**
4. Are we resolving a subpath after the package name? **No, just the root `"pkg"`.**
5. Does `"types"` exist? **Yes.**
6. Does any key in `"typesVersions"` match `./index.d.ts`? **Yes, `"*"` matches and sets `index.d.ts` to be the substitution.**
7. In `ts3.1/*`, replace `*` with the substitution `./index.d.ts`: **`ts3.1/index.d.ts`**.
8. Does the path `./ts3.1/index.d.ts` have a recognized TypeScript file extension? **Yes, so don’t use extension substitution.**
9. Return the path `./ts3.1/index.d.ts` if the file exists, `undefined` otherwise.

##### Example: redirect requests for a specific file

Scenario: a module imports `"pkg"` using TypeScript 3.9, where `node_modules/pkg/package.json` is:

```json
{
  "name": "pkg",
  "version": "1.0.0",
  "types": "./index.d.ts",
  "typesVersions": {
    "<4.0": { "index.d.ts": ["index.v3.d.ts"] }
  }
}
```

Resolution process:

1. (Depending on compiler options) Does `"exports"` exist? **No.**
2. Does `"typesVersions"` exist? **Yes.**
3. Is the TypeScript version `<4.0`? **Yes. Remember the mapping `"index.d.ts": ["index.v3.d.ts"]`.**
4. Are we resolving a subpath after the package name? **No, just the root `"pkg"`.**
5. Does `"types"` exist? **Yes.**
6. Does any key in `"typesVersions"` match `./index.d.ts`? **Yes, `"index.d.ts"` matches.**
7. Does the path `./index.v3.d.ts` have a recognized TypeScript file extension? **Yes, so don’t use extension substitution.**
8. Return the path `./index.v3.d.ts` if the file exists, `undefined` otherwise.

#### package.json `"main"` and `"types"`

If a directory’s [package.json `"exports"`](#packagejson-exports) field is not read (either due to compiler options, or because it is not present, or because the directory is being resolved as a [directory module](#directory-modules-index-file-resolution) instead of a [`node_modules` package](#node_modules-package-lookups)) and the module specifier does not have a subpath after the package name or package.json-containing directory, TypeScript will attempt to resolve from these package.json fields, in order, in an attempt to find the main module for the package or directory:

- `"types"`
- `"typings"` (legacy)
- `"main"`

The declaration file found at `"types"` is assumed to be an accurate representation of the implementation file found at `"main"`. If `"types"` and `"typings"` are not present or cannot be resolved, TypeScript will read the `"main"` field and perform [extension substitution](#file-extension-substitution) to find a declaration file.

When publishing a typed package to npm, it’s recommended to include a `"types"` field even if [extension substitution](#file-extension-substitution) or [package.json `"exports"`](#packagejson-exports) make it unnecessary, because npm shows a TS icon on the package registry listing only if the package.json contains a `"types"` field.

#### Package-relative file paths

If neither [package.json `"exports"`](#packagejson-exports) nor [package.json `"typesVersions"`](#packagejson-typesversions) apply, subpaths of a bare package specifier resolve relative to the package directory, according to applicable [relative path](#relative-file-path-resolution) resolution rules. In modes that respect [package.json `"exports"`], this behavior is blocked by the mere presence of the `"exports"` field in the package’s package.json, even if the import fails to resolve through `"exports"`, as demonstrated in [an example above](#example-exports-block-other-subpaths). On the other hand, if the import fails to resolve through `"typesVersions"`, a package-relative file path resolution is attempted as a fallback.

When package-relative paths are supported, they resolve under the same rules as any other relative path considering the `moduleResolution` mode and context. For example, in [`--moduleResolution nodenext`](#node16-nodenext-1), [directory modules](#directory-modules-index-file-resolution) and [extensionless paths](#extensionless-relative-paths) are only supported in `require` calls, not in `import`s:

```ts
// @Filename: module.mts
import "pkg/dist/foo";                // ❌ import, needs `.js` extension
import "pkg/dist/foo.js";             // ✅
import foo = require("pkg/dist/foo"); // ✅ require, no extension needed
```

#### package.json `"imports"` and self-name imports

When `moduleResolution` is set to `node16`, `nodenext`, or `bundler`, and `resolvePackageJsonImports` is not disabled, TypeScript will attempt to resolve import paths beginning with `#` through the the `"imports"` field of the nearest ancestor package.json of the importing file. Similarly, when [package.json `"exports"` lookups](#packagejson-exports) are enabled, TypeScript will attempt to resolve import paths beginning with the current package name—that is, the value in the `"name"` field of the nearest ancestor package.json of the importing file—through the `"exports"` field of that package.json. Both of these features allow files in a package to import other files in the same package, replacing a relative import path.

TypeScript follows Node.js’s resolution algorithm for [`"imports"`](https://nodejs.org/api/packages.html#subpath-imports) and [self references](https://nodejs.org/api/packages.html#self-referencing-a-package-using-its-name) exactly up until a file path is resolved. At that point, TypeScript’s resolution algorithm forks based on whether the package.json containing the `"imports"` or `"exports"` being resolved belongs to a `node_modules` dependency or the local project being compiled (i.e., its directory contains the tsconfig.json file for the project that contains the importing file):

- If the package.json is in `node_modules`, TypeScript will apply [extension substitution](#file-extension-substitution) to the file path if it doesn’t already have a recognized TypeScript file extension, and check for the existence of the resulting file paths.
- If the package.json is part of the local project, an additional remapping step is performed in order to find the _input_ TypeScript implementation file that will eventually produce the output JavaScript or declaration file path that was resolved from `"imports"`. Without this step, any compilation that resolves an `"imports"` path would be referencing output files from the _previous compilation_ instead of other input files that are intended to be included in the current compilation. This remapping uses the `outDir`/`declarationDir` and `rootDir` from the tsconfig.json, so using `"imports"` usually requires an explicit `rootDir` to be set.

This variation allows package authors to write `"imports"` and `"exports"` fields that reference only the compilation outputs that will be published to npm, while still allowing local development to use the original TypeScript source files.

##### Example: local project with conditions

Scenario: `"/src/main.mts"` imports `"#utils"` with conditions `["types", "node", "import"]` (determined by `moduleResolution` setting and the context that triggered the module resolution request) in a project directory with a tsconfig.json and package.json:

```json5
// tsconfig.json
{
  "compilerOptions": {
    "moduleResolution": "node16",
    "resolvePackageJsonImports": true,
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```

```json5
// package.json
{
  "name": "pkg",
  "imports": {
    "#utils": {
      "import": "./dist/utils.d.mts",
      "require": "./dist/utils.d.cts"
    }
  }
}
```

Resolution process:

1. Import path starts with `#`, try to resolve through `"imports"`.
2. Does `"imports"` exist in the nearest ancestor package.json? **Yes.**
3. Does `"#utils"` exist in the `"imports"` object? **Yes.**
4. The value at `imports["#utils"]` is an object—it must be specifying conditions.
5. Does the first condition `"import"` match this request? **Yes.**
6. Should we attempt to map the output path to an input path? **Yes, because:**
   - Is the package.json in `node_modules`? **No, it’s in the local project.**
   - Is the tsconfig.json within the package.json directory? **Yes.**
7. In `./dist/utils.d.mts`, replace the `outDir` prefix with `rootDir`. **`./src/utils.d.mts`**
8. Replace the output extension `.d.mts` with the corresponding input extension `.mts`. **`./src/utils.mts`**
9. Return the path `"./src/utils.mts"` if the file exists.
10. Otherwise, return the path `"./dist/utils.d.mts"` if the file exists.

##### Example: `node_modules` dependency with subpath pattern

Scenario: `"/node_modules/pkg/main.mts"` imports `"#internal/utils"` with conditions `["types", "node", "import"]` (determined by `moduleResolution` setting and the context that triggered the module resolution request) with the package.json:

```json5
// /node_modules/pkg/package.json
{
  "name": "pkg",
  "imports": {
    "#internal/*": {
      "import": "./dist/internal/*.mjs",
      "require": "./dist/internal/*.cjs"
    }
  }
}
```

Resolution process:

1.  Import path starts with `#`, try to resolve through `"imports"`.
2.  Does `"imports"` exist in the nearest ancestor package.json? **Yes.**
3.  Does `"#internal/utils"` exist in the `"imports"` object? **No, check for pattern matches.**
4.  Does any key with a `*` match `"#internal/utils"`? **Yes, `"#internal/*"` matches and sets `utils` to be the substitution.**
5.  The value at `imports["#internal/*"]` is an object—it must be specifying conditions.
6.  Does the first condition `"import"` match this request? **Yes.**
7.  Should we attempt to map the output path to an input path? **No, because the package.json is in `node_modules`.**
8.  In `./dist/internal/*.mjs`, replace `*` with the substitution `utils`. **`./dist/internal/utils.mjs`**
9.  Does the path `./dist/internal/utils.mjs` have a recognized TypeScript file extension? **No, try extension substitution.**
10. Via [extension substitution](#file-extension-substitution), try the following paths, returning the first one that exists, or `undefined` otherwise:
    1. `./dist/internal/utils.mts`
    2. `./dist/internal/utils.d.mts`
    3. `./dist/internal/utils.mjs`

### `node16`, `nodenext`

`--moduleResolution node16` and `nodenext` must be pared with their [corresponding `module` value](#node16-nodenext). They are currently identical, but if Node.js makes significant changes to its module system in the future, `node16` will be frozen while `nodenext` will be updated to reflect the new behavior.

These modes reflect the module resolution behavior of Node.js v12 and later, where the resolution algorithm for ECMAScript imports is significantly different from the algorithm for CommonJS `require` calls. For each module specifier being resolved, the syntax and the [module format of the importing file](#module-format-detection) are first used to determine whether the module specifier will be in an `import` or `require` in the emitted JavaScript. That information is then passed into the module resolver to determine which resolution algorithm to use (and whether to use the `"import"` or `"require"` condition for package.json [`"exports"`](#packagejson-exports) or [`"imports"`](#packagejson-imports-and-self-name-imports)).

> TypeScript files that are [determined to be in CommonJS format](#module-format-detection) may still use `import` and `export` syntax by default, but the emitted JavaScript will use `require` and `module.exports` instead. This means that it’s common to see `import` statements that are resolved using the `require` algorithm. If this causes confusion, the `verbatimModuleSyntax` compiler option can be enabled, which prohibits the use of `import` statements that would be emitted as `require` calls.

Note that dynamic `import()` calls are always resolved using the `import` algorithm, according to Node.js’s behavior. However, `import()` types are resolved according to the format of the importing file (for backward compatibility with existing CommonJS-format type declarations):

```ts
// @Filename: module.mts
import x from "./mod.js";             // `import` algorithm due to file format (emitted as-written)
import("./mod.js");                   // `import` algorithm due to syntax (emitted as-written)
type Mod = typeof import("./mod.js"); // `import` algorithm due to file format
import mod = require("./mod");        // `require` algorithm due to syntax (emitted as `require`)

// @Filename: commonjs.cts
import x from "./mod";                // `require` algorithm due to file format (emitted as `require`)
import("./mod.js");                   // `import` algorithm due to syntax (emitted as-written)
type Mod = typeof import("./mod");    // `require` algorithm due to file format
import mod = require("./mod");        // `require` algorithm due to syntax (emitted as `require`)
```

#### Supported features

| | `import` | `require` |
|-| -------- | --------- |
| [`node_modules` package lookups](#node_modules-package-lookups) | ✅ | ✅ |
| [package.json `"exports"`](#packagejson-exports) | ✅ matches `types`, `node`, `import` | ✅ matches `types`, `node`, `require` |
| [package.json `"imports"` and self-name imports](#packagejson-imports-and-self-name-imports) | ✅ matches `types`, `node`, `import` | ✅ matches `types`, `node`, `require` |
| [package.json `"typesVersions"`](#packagejson-typesversions) | ✅ | ✅ |
| [Package-relative paths](#package-relative-file-paths) | ✅ when `exports` not present | ✅ when `exports` not present |
| [Full relative paths](#relative-file-path-resolution) | ✅ | ✅ |
| [Extensionless relative paths](#extensionless-relative-paths) | ❌ | ✅ |
| [Directory modules](#directory-modules-index-file-resolution) | ❌ | ✅ |
