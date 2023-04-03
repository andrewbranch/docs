# Theory

## Scripts and modules in JavaScript

In the early days of JavaScript, when the language only ran in browsers, there were no modules, but it was still possible to split the JavaScript for a web page into multiple files by using multiple `script` tags in HTML:

```html
<html>
  <head>
    <script src="a.js"></script>
    <script src="b.js"></script>
  </head>
  <body></body>
</html>
```

This approach had some downsides, especially as web pages grew larger and more complex. In particular, all scripts loaded onto the same page share the same scope—appropriately called the “global scope”—meaning the scripts had to be very careful not to overwrite each others’ variables and functions.

Any system that solves this problem by giving files their own scope while still providing a way to make bits of code available to other files can be called a “module system.” (It may sound obvious to say that each file in a module system is called a “module,” but the term is often used to contrast with _script_ files, which run outside a module system, in a global scope.)

> There are [many module systems](https://github.com/myshov/history-of-javascript/tree/master/4_evolution_of_js_modularity), and TypeScript [supports emitting several](https://www.typescriptlang.org/tsconfig/#module), but this documentation will focus on the two most important systems today: ECMAScript modules (ESM) and CommonJS (CJS).
>
> ECMAScript Modules (ESM) is the module system built into the language, supported in modern browsers and in Node since v12. It uses dedicated `import` and `export` syntax:
>
> ```js
> // a.js
> export default "Hello from a.js";
> ```
>
> ```js
> // b.js
> import a from "./a.js";
> console.log(a); // 'Hello from a.js'
> ```
>
> CommonJS (CJS) is the module system that originally shipped in Node, before ESM was part of the language specification. It’s still supported in Node alongside ESM. It uses plain JavaScript objects and functions named `module` and `require`:
>
> ```js
> // a.js
> module.exports = "Hello from a.js";
> ```
>
> ```js
> // b.js
> const a = require("./a");
> console.log(a); // 'Hello from a.js'
> ```

Accordingly, when TypeScript detects that a file is a CommonJS or ECMAScript module, it starts by assuming that file will have its own scope. Beyond that, though, the compiler’s job gets a little more complicated.

## TypeScript’s job concerning modules

The TypeScript compiler’s chief goal is to look at input code and tell the author about problems the output code might encounter at runtime. To do this, the compiler needs to know some things about the code’s intended runtime environment—what globals are available, for example. When the input code is a module, there are several additional questions the compiler needs to answer in order to do its job. Let’s use a few lines of input code as an example to think about all the information needed to analyze it:

```ts
import sayHello from "greetings";
sayHello("world");
```

To check this file, the compiler needs to know the type of `sayHello` (is it a function that can accept one string argument?), which opens quite a few additional questions:

1. What kind of module does the module system expect to find at the location where I output this file?
2. What output JavaScript code should I emit for this input?
3. When the module system loads that output code, where will it look to find the module specified by `"greetings"`? Will the lookup succeed?
4. What kind of module is the file resolved by that lookup?
5. Does the module system allow the kind of module detected in (1) to reference the kind of module detected in (4) with the syntax decided in (2)?
6. Once the `"greetings"` module has been analyzed, what piece of that module is bound to `sayHello`?

Notice that all of these questions depend on characteristics of the _host_—the system that ultimately consumes the output code to direct its module loading behavior, typically either a runtime (like Node) or bundler (like Webpack). The ECMAScript specification defines how ESM imports and exports match up with each other, but it doesn’t specify how the file lookup in (3), known as _module resolution_, happens, and it doesn’t say anything about other module systems like CommonJS. So runtimes and bundlers, especially those that want to support both ESM and CJS, have a lot of freedom to design their own rules. Consequently, the way TypeScript should answer the questions above can vary dramatically depending on where the code is intended to run. There’s no single right answer, so the compiler must be told the rules through configuration options.

The other key idea to keep in mind is that TypeScript always has to think about these questions in terms of its _output_ JavaScript files, not its _input_ TypeScript (or JavaScript!) files. Over the course of this document, we’ll examine a few concrete examples of where this matters. But for now, we can summarize TypeScript’s job when it comes to modules in those terms:

Understand the **rules of the host** enough

1. to compile files into a valid **output module format**,
2. to ensure that imports in those **outputs** will **resolve successfully**, and
3. to know what **type** to assign to **imported names**.

## Who is the host?

Before we move on, it’s worth making sure we’re on the same page about the term _host_, because it will come up frequently. We defined it before as “the system that ultimately consumes the output code to direct its module loading behavior.” In other words, it’s the system outside of TypeScript that TypeScript’s module analysis tries to model:

- When the output code (whether produced by `tsc` or a third-party transpiler) is run directly in a runtime like Node, the runtime is the host.
- When there is no “output code” because a runtime consumes TypeScript files directly, the runtime is still the host.
- When a bundler consumes TypeScript inputs or outputs and produces a bundle, the bundler is the host, because it looked at the original set of imports/requires, looked up what files they referenced, and produced a new file or set of files where the original imports and requires are erased or transformed beyond recognition. (That bundle itself might comprise modules, and the runtime that runs it will be its host, but TypeScript doesn’t know about anything that happens post-bundler.)
- If another transpiler, optimizer, or formatter runs on TypeScript’s outputs, it’s _not_ a host that TypeScript cares about, as long as it leaves the imports and exports it sees alone.
- When loading modules in a web browser, the behaviors TypeScript needs to model are actually split between the web server and the module system running in the browser. The browser’s JavaScript engine (or a script-based module-loading framework like RequireJS) controls what module formats are accepted, while the web server decides what file to send when one module triggers a request to load another.

## The module output format

In any project, the first question about modules we need to answer is what kinds of modules the host expects, so TypeScript can set its output format for each file to match. Sometimes, the host only _supports_ one kind of module—ESM in the browser, or CJS in Node v11 and earlier, for example. Node v12 and later accepts both CJS and ES modules, but uses file extensions and `package.json` files to determine what format each file should be, and throws an error if the file’s contents don’t match the expected format.

The `module` compiler option provides this information to the compiler. Its primary purpose is to control the module format of any JavaScript that gets emitted during compilation, but it also serves to inform the compiler about how the module kind of each file should be detected, how different module kinds are allowed to import each other, and whether features like `import.meta` and top-level `await` are available. So, even if a TypeScript project is using `noEmit`, choosing the right setting for `module` still matters. As we established earlier, the compiler needs an accurate understanding of the module system so it can type check (and provide IntelliSense for) imports. [TODO: link to choosing compiler opitons guide]

The available `module` settings are

- **`node16`**: Reflects the module system of Node v16+, which supports ES modules and CJS modules side-by-side with particular interoperability and detection rules.
- **`nodenext`**: Currently identical to `node16`, but will be a moving target reflecting the latest Node versions as Node’s module system evolves.
- **`es2015`**: Reflects the ES2015 language specification for JavaScript modules (the version that first introduced `import` and `export` to the language).
- **`es2020`**: Adds support for `import.meta` and `export * as ns from "mod"` to `es2015`.
- **`es2022`**: Adds support for top-level `await` to `es2020`.
- **`esnext`**: Currently identical to `es2022`, but will be a moving target reflecting the latest ECMAScript specifications, as well as module-related Stage 3+ proposals that are expected to be included in upcoming specification versions.

[TODO: list the other modes here]

> Node’s rules for module kind detection and interoperability make it incorrect to specify `module` as `esnext` or `commonjs` for projects that run in Node, even if all files emitted by `tsc` are ESM or CJS, respectively. The only correct `module` settings for projects that intend to run in Node are `node16` and `nodenext`, because these are the only settings that encode these rules. While the emitted JavaScript for an all-ESM Node project might look identical between compilations using `esnext` and `nodenext`, the type checking can differ. [TODO: link to more info on `nodenext`]

### Module kind detection

Node understands both ES modules and CJS modules, but the format of each file is determined by its file extension and the `type` field of the first `package.json` file found in a search of the file’s directory and all ancestor directories. `.mjs` and `.cjs` files are always interpreted as ES modules and CJS modules, respectively. `.js` files are interpreted as ES modules if the nearest `package.json` file contains a `type` field with the value `"module"`. If there is no `package.json` file, or if the `type` field is missing or has any other value, `.js` files are interpreted as CJS modules. If a file is determined to be an ES module by these rules, Node will not inject the CommonJS `module` and `require` objects into the file’s scope during evaluation, so a file that tries to use them will cause a crash. Conversely, if a file is determined to be a CJS module, `import` and `export` declarations in the file will cause a syntax error crash.

When the `module` compiler option is set to `node16` or `nodenext`, TypeScript applies this same algorithm to the project’s _input_ files to determine the module kind of each corresponding _output_ file. Let’s look at how module kinds are detected in an example project that uses `--module nodenext`:

| Input file name                  | Contents               | Output file name | Module kind | Reason                                  |
| -------------------------------- | ---------------------- | ---------------- | ----------- | --------------------------------------- |
| `/package.json`                  | `{}`                   |                  |             |                                         |
| `/main.mts`                      |                        | `/main.mjs`      | ESM         | File extension                          |
| `/utils.cts`                     |                        | `/utils.cjs`     | CJS         | File extension                          |
| `/example.ts`                    |                        | `/example.js`    | CJS         | No `"type": "module"` in `package.json` |
| `/node_modules/pkg/package.json` | `{ "type": "module" }` |                  |             |                                         |
| `/node_modules/pkg/index.d.ts`   |                        |                  | ESM         | `"type": "module"` in `package.json`    |
| `/node_modules/pkg/index.d.cts`  |                        |                  | CJS         | File extension                          |

When the input file extension is `.mts` or `.cts`, TypeScript knows to treat that file as an ES module or CJS module, respectively, because Node will have the same interpretation of the corresponding output file due to its `.mjs` or `.cjs` extension. When the input file extension is `.ts`, TypeScript has to consult the nearest `package.json` file to determine the module kind, because this is what Node will do when it encounters the output `.js` file. Notice that the same rules apply to the `.d.cts` and `.d.ts` declaration files in the `pkg` dependency: though they will not produce an output file as part of this compilation, the presence of a `.d.ts` file _implies_ the existence of a corresponding `.js` file—perhaps created when the author of the `pkg` library ran `tsc` on an input `.ts` file of their own—which Node must interpret as an ES module, due to its `.js` extension and the presence of the `"type": "module"` field in `/node_modules/pkg/package.json`.

The detected module kind of input files is used by TypeScript to ensure it emits the output syntax that Node expects in each output file. If TypeScript were to emit `/example.js` with `import` and `export` statements in it, Node would crash when parsing the file. If TypeScript were to emit `/main.mjs` with `require` calls, Node would crash during evaluation. Beyond emit, the module kind is also used to determine rules for type checking and module resolution, which we’ll discuss in the following sections.

It’s worth mentioning again that TypeScript’s behavior in `--module node16` and `--module nodenext` is entirely motivated by Node’s behavior. Since TypeScript’s goal is to catch potential runtime errors at compile time, it needs a very accurate model of what will happen at runtime. This fairly complex set of rules for module kind detection is _necessary_ for checking code that will run in Node, but is likely to be _entirely incorrect_ if applied to non-Node hosts. [TODO: link to guide on choosing compiler options]

### Input module syntax

It’s important to note that the _input_ module syntax seen in input source files is somewhat decoupled from the output module syntax emitted to JS files. That is, a file with an ESM import:

```ts
import { sayHello } from "greetings";
sayHello("world");
```

might be emitted in ESM format exactly as-is, or might be emitted as CommonJS:

```ts
Object.defineProperty(exports, "__esModule", { value: true });
const greetings_1 = require("greetings");
(0, greetings_1.sayHello)("world");
```

depending on the `module` compiler option (and any applicable [module kind detection](#module-kind-detection) rules, if the `module` option supports more than one kind of module). In general, this means that looking at the contents of an input file isn’t enough to determine whether it’s an ES module or a CJS module.

> Today, most TypeScript files are authored using ESM syntax (`import` and `export` statements) regardless of the output format. This is largely a legacy of the long road ESM has taken to widespread support. ECMAScript modules were standardized in 2015, were supported in most browsers by 2017, and landed in Node v12 in 2019. During much of this window, it was clear that ESM was the future of JavaScript modules, but very few runtimes could consume it. Tools like Babel made it possible for JavaScript to be authored in ESM and downleveled to another module format that could be used in Node or browsers. TypeScript followed suit, adding support for ES module syntax and softly discouraging the use of the original CommonJS-inspired `import fs = require("fs")` syntax in [the 1.5 release](https://devblogs.microsoft.com/typescript/announcing-typescript-1-5/).
>
> The upside of this “author ESM, output anything” strategy was that TypeScript could use standard JavaScript syntax, making the authoring experience familiar to newcomers, and (theoretically) making it easy for projects to start targeting ESM outputs in the future. There are three significant downsides, which became fully apparent only after ESM and CJS modules were allowed to coexist and interoperate in Node:
>
> 1. Early assumptions about how ESM/CJS interoperability would work in Node were wrong, and today, interoperability rules differ between Node and bundlers. Consequently, the configuration space for modules in TypeScript is large.
> 2. When the syntax in input files all looks like ESM, it’s easy for an author or code reviewer to lose track of what kind of module a file is at runtime. And because of Node’s interoperability rules, what kind of module each file is became very important.
> 3. When input files are written in ESM, the syntax in type declaration outputs (`.d.ts` files) looks like ESM too. But because the corresponding JavaScript files could have been emitted in any module format, TypeScript can’t tell what kind of module a file is just by looking at the contents of its type declarations. And again, because of the nature of ESM/CJS interoperability, TypeScript _has_ to know what kind of module everything is in order to provide correct types and prevent imports that will crash.
>
> In TypeScript 5.0, a new compiler option called `verbatimModuleSyntax` was introduced to help TypeScript authors know exactly how their `import` and `export` statements will be emitted. When enabled, the flag requires imports and exports in input files to be written in the form that will undergo the least amount of transformation before emit. So if a file will be emitted as ESM, imports and exports must be written in ESM syntax; if a file will be emitted as CJS, it must be written in the CommonJS-inspired TypeScript syntax (`import fs = require("fs")` and `export = {}`). This setting is particularly recommended for Node projects that use mostly ESM, but have a select few CJS files. It is not recommended for projects that currently target CJS, but may want to target ESM in the future.

### ESM and CJS interoperability

Can an ES module `import` a CommonJS module? If so, does a default import select `module.exports` or `module.exports.default`? Can a CommonJS module `require` an ES module? CommonJS isn’t part of the ECMAScript specification, so runtimes, bundlers, and transpilers have been free to make up their own answers to these questions since ESM was standardized in 2015. Today, interoperability rules between ESM and CJS for most runtimes and bundlers broadly fall into one of three categories:

1. **ESM-only.** Some runtimes, like browser engines, only support what’s actually a part of the language: ECMAScript Modules.
2. **Bundler-like.** Before any major JavaScript engine could run ES modules, Babel allowed developers to write them by transpiling them to CommonJS. The way these ESM-transpiled-to-CJS files (indicated with an `__esModule` property on `module.exports`) interacted with hand-written-CJS files implied a set of permissive interoperability rules that have become the de facto standard for bundlers and transpilers.
3. **Node.** In Node, CommonJS modules cannot load ES modules synchronously (with `require`); they can only load them asynchronously with dynamic `import()` calls. ES modules can default-import CJS modules, which always binds to `module.exports`. (This means that a default import of a Babel-like CJS output with `__esModule` behaves differently between Node and some bundlers.)

TypeScript needs to know which of these rule sets to assume in order to provide correct types on (particularly `default`) imports and to error on imports that will crash at runtime. When the `module` compiler option is set to `node16` or `nodenext`, Node’s rules are enforced. All other settings assume bundler-like rules. (While using `--module esnext` does prevent you from _writing_ CommonJS modules, it does not prevent you from _importing_ them as dependencies. There’s currently no TypeScript setting that can guard against an ES module importing a CommonJS module, as would be appropriate for direct-to-browser code.)

> #### `allowSyntheticDefaultImports` and `esModuleInterop`
>
> [TODO]

### Module specifiers are not transformed

While the `module` compiler option can transform imports and exports in input files to different module formats in output files, the module _specifier_ (the string `from` which you `import`, or pass to `require`) is always emitted as-written. For example, an input like:

```ts
import { add } from "./math.js";
add(1, 2);
```

might be emitted as either:

```ts
import { add } from "./math.js";
add(1, 2);
```

or:

```ts
const math_1 = require("./math.js");
math_1.add(1, 2);
```

depending on the `module` compiler option, but the module specifier will always be `"./math.js"`. There is no compiler option that enables transforming, substituting, or rewriting module specifiers. Consequently, module specifiers must be written in a way that works for the code’s target runtime or bundler, and it’s TypeScript’s job to understand those _output_-relative specifiers. The process of finding the file referenced by a module specifier is called _module resolution_.

## Module resolution

Let’s return to our [first example](#typescripts-job-concerning-modules) and review what we’ve learned about it so far:

```ts
import sayHello from "greetings";
sayHello("world");
```

So far, we’ve discussed how the host’s module system and TypeScript’s `module` compiler option might impact this code: we know that the input syntax looks like ESM, but the output format depends on the `module` compiler option and potentially the file extension and `package.json` `"type"` field; we also know that what `sayHello` gets bound to, and even whether the import is even allowed, may vary depending on the module kinds of this file and the target file. But we haven’t yet discussed how to _find_ the target file.

### Module resolution is host-defined

While the ECMAScript specification defines how to parse and interpret `import` and `export` statements, it leaves module resolution up to the host. If you’re creating a hot new JavaScript runtime, you’re free to create a module resolution scheme like:

```ts
import monkey from "🐒"; // Looks for './eats/bananas.js'
import cow from "🐄";    // Looks for './eats/grass.js'
import lion from "🦁";   // Looks for './eats/you.js'
```

and still claim to implement “standards-compliant ESM.” Needless to say, TypeScript would have no idea what types to assign to `monkey`, `cow`, and `lion` without built-in knowledge of this runtime’s module resolution algorithm. Just as `module` informs the compiler about the host’s expected module format, `moduleResolution`, along with a few customization options, specify the algorithm the host uses to resolve module specifiers to files.

> #### Why doesn’t TypeScript delegate module resolution to third-party plugins?
>
> [TODO]