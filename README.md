# `@aspect_rules_js` escaping sandbox

Files seem to be picking up `type: "module"` from the root `package.json`, even
when that file is _never_ depended upon.

Currently `bin.mjs` imports `lib.js` (both ESM format), but `lib.js` should be
treated as CommonJS by default, since there is no dependency on `package.json`.
However it fails with:

```
$ bazel build //:gen
# ...
import { hello } from './lib.js';
         ^^^^^
SyntaxError: The requested module './lib.js' does not provide an export named 'hello'
    at ModuleJob._instantiate (node:internal/modules/esm/module_job:123:21)
    at async ModuleJob.run (node:internal/modules/esm/module_job:189:5)
    at async Promise.all (index 0)
    at async ESMLoader.import (node:internal/modules/esm/loader:530:24)
    at async loadESM (node:internal/process/esm_loader:91:5)
    at async handleMainPromise (node:internal/modules/run_main:65:12)
```

`lib.js` is being interpreted as an ES module because of a phantom dependency on
`package.json`. Removing `type: "module"` fixes the issue.

The `BUILD.bazel` file is set up so both `bin.mjs` and `lib.js` can be renamed
easily to experiment with different configurations.

My observations:

*   Using `lib.mjs` or `lib.cjs` works regardless of what `package.json` says.
    *   I think that's expected given that Node always prefers the file extension,
        so it isn't even looking for the `package.json` at all.
*   Renaming `bin.mjs` to `bin.js` also makes it dependent on `package.json`.
    *   So technically the import isn't even necessary to reproduce this issue.
    *   A truly "minimal" example is just using a `js_binary()` with a `*.js`
        entry point.
*   Using `bin.cjs` and `lib.mjs` fails because you can't directly `require()` an
    ES module (`ERR_REQUIRE_ESM`).
    *   I think that's a constraint in Node that we wouldn't expect to work in
        `@aspect_rules_js`.
*   Using `bin.cjs` and `lib.js` with `"type": "module"` fails with a slightly
    different error (`SyntaxError: Unexpected token 'export'`).
    *   This makes me think this case is actually not checking the
        `package.json` because it is attempting to load as CommonJS and
        encountering a syntax error.
    *   I wonder if this is because it isn't finding the `package.json` or
        because it knows you can't import ESM from CommonJS and isn't bothering
        to look?
*   I'm also able to reproduce either with `bazel run //:bin` **OR**
    `bazel build //:gen`.
    *   Not sure exactly how the sandboxes are different between those cases.
*   I tried using `NODE_DEBUG=*` to see if I could observe the `package.json`
    read, but it doesn't log anything about a `package.json`, even when Node is
    clearly relying on it.
