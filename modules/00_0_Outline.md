1. Introduction
   1. Who is this for?
1. Theory
   1. Scripts and modules in JavaScript
   1. What is TypeScript’s job concerning modules?
   1. Who is the host?
   1. Module emit
      1. Input syntax is (somewhat) decoupled from output
      1. Note: `verbatimModuleSyntax`
      1. Note: top-level `await`
      1. Module format interop
      1. Module specifiers are not transformed
   1. Module resolution
      1. Module resolution is host-defined
      1. TypeScript “mirrors” the host resolver
         1. Module specifiers reference the host’s source files
         1. Declaration files are substituted for JS files
         1. Note: non-JS files supported with `allowArbitraryExtensions`
      1. Common resolution features
         1. Omittable extensions and directory index files
         1. node_modules and package.json
            1. @types
            1. package.json `"exports"`
      1. Bundlers and other Node-like non-Node hosts
         1. `noEmit` and `allowImportingTsExtensions`
         1. Customization options
   1. Caveats / limitations?
      1. Why module specifiers are not transformed
      1. Why dual-emit is not supported and sometimes impossible
      1. Future work? Browser, import maps, URLs?
1. Guides
   1. Choosing compiler options
   1. Troubleshooting dependency issues
   1. Publishing a library
1. Reference
   1. Module syntax
      1. ESM (link elsewhere?)
      1. CJS in JavaScript
      1. CJS in TypeScript
      1. Type-only imports and exports
      1. Ambient module declarations & augmentations?
    1. `module`
       1. `commonjs`
       1. `es2015`+
       1. `node16`/`nodenext`
    1. `moduleResolution`
       1. `node16`/`nodenext`
       1. `bundler`
    1. `allowJs` and `maxNodeModuleJsDepth`
    1. package.json
       1. `"main"`, `"types"`
       1. `"typesVersions"`
       1. `"exports"`
       1. `"type"`
       1. `"imports"`
    1. Customization compiler options
       1. `paths`
       1. `baseUrl`
       1. `resolvePackageJsonImports`
       1. `resolvePackageJsonExports`
       1. `customConditions`
       1. `allowImportingTsExtensions`
       1. `resolveJsonModule`
       1. `rootDirs`
       1. `typeRoots`
       1. `moduleSuffixes`
1. Appendices
   1. ESM/CJS Interoperability