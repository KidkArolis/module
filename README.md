# module

This is an experiment that demonstrates, through working code, a module system
for ECMAScript (ES). It builds on the the `System` API from the current ES
proposal, along with some other work from the ES module effort so far, but makes
a couple of baseline modifications in an effort to reduce the amount of effort
and learning it will take for people to implement and use the current ES module
effort.

It uses `module` as the API space, because it allows for specifying dependencies
by using a declarative sounding name, and all IDs lookups are based on the
module name-to-path semantics. `System` is vague too, as it could indicate
the browser or operating system API.

However, much of the mechanics of `module` is based on the current design
thoughts around `System`, so the name change is really surface syntax, and
could easily be changed based on feedback.

ModuleLoader is still used

## ES module work reused

* The ModuleLoader API and hooks.
* The concept of a "mutable slot" that is used for module export values.
* Realms should also be a construct that is used with modules, but not
demonstrated here, as it is more about the environment that surrounds the
module system.

## Differences from current ES module effort

* No import/export language syntax. APIs are used instead.

It seems the primary goal for the import/export syntax was to get a link
behavior to check export names earlier, and to allow other "compile time"
features later.

However, this new syntax has nontrivial cost:

* It requires top level syntax forms that lead to also needing changes to HTML
to support this work. This increases the learning curve for HTML, and needing
to couple the ES and HTML changes together is a sign of a more complicated
design.
* It is hard to support a REPL or jsbin-style "just type JS" with the syntax
forms.
* There are complications around default export and named export, and how that
looks like to `System.import`: for example, it would be ugly if a user had to
reference a `.default` property in the import success callback, where in the
import syntax form it is not necessary.
* The syntax encourages the desire for lexical modules. This is very hard to
reconcile with the dynamic parts of ES module loading.
* This is also evident for the need of a per-module `System` construct, to
make sure modules loaded via `System.import` are stored in the correct
module loader instance. Plus, the practical usefulness, as demonstrated in AMD,
of a module being able to know its module name and address (`module.id` and
`module.uri` in AMD, originally specified in CommonJS modules).
* Because of the uncertainty around lexical modules, the bundling API is not
optimal: passing JS strings to `System.define` looks ugly, and causes
practical complications around concatenation and minification steps used in
code today.

On the proposed benefits:

**Export name checking is a very shallow benefit**

It is expected that once modules are available in the language, the use of
default exports, and splitting modules into fine grained, single export modules
will be very common. This will be done for practical concerns:

* It is just easier to reason about the code while developing it
* For browser bundling, it creates very clear, easy boundaries to make it easy
to exclude code from a collection of modules to the amount that is actually
used. "Advanced" minifiers like closure compiler are difficult to use to
get the same benefit. Existing AMD module use bears this out.

Plus, the export checking will not help with any secondary level property
references. For example, if the default export is a constructor function,

On an `import *` or export `* capability`, these have not been needed in
CommonJS/Node/AMD systems to build large codebases. If this capability is
really desired, code editors can provide shortcuts to do the bulk export
property references.

**Adding other compile time features later are still possible**

This is just a sketch, using the experimental sweet.js macro work as an example.
The API syntax shown is just used as illustration, not a strong recommendation.

The main idea though is to rely on a reader for JS to pull out sections from
a function that need to wait for final compilation until later, and then in
the place of that token stream, use a function placeholder that has the reader
tokens attached to it, and that function is not fully compiled until later when
any dependency compile time forms are known.

Example:

```javascript
/***** module 'a' ******/
// get dependencies
var c = module('c');

// create exportable macros
module.exportMacro('swap', {/* macro definition here*/});
module.exportMacro('unless', {/* macro definition here*/});

// export runtime module value for a
module.export({
  name: 'a'
});

/***** module 'b' ******/
// b just wants to use some macros from 'a'
module.useMacro('a', 'swap', 'unless');


// export runtime module value for b
module.set(function b(x) {
  return true unless x > 42;
  return false;
});
```

The reader parses 'a' and sees the module APIs in play, and the
`module.exportMacro` use. It pulls out the token sections for
`module.exportMacro`, and since module bodies in the `module` approach are just
functions, it annotates the function with the pieces of information:

* module dependencies referenced
* set of macros found.

The module system can then use this annotated function for the "factory
function" used for module 'a'.

When 'b' is parsed by the reader, it notices the `module.exportMacro` reference,
so it does not let the function continue to the grammar parsing stage, and
instead, creates a function placeholder, perhaps with a new type of
PartialParsedFunction, and that PartialParsedFunction is annotated with these
pieces of information:

* module dependencies referenced (it would include 'a' since )
* reader tokens waiting for final compile forms to be available.

The module loader runs a's factory function to create the runtime exports for
a. Now that it is complete, the module loader comes to b's factory function.

Since that factory function is a PartialParsedFunction, it uses the macros
needed for it from the set it knows that were found from 'a', then completes
the final parsing of the function to a regular function and then executes it.

## ES spec changes

`module` relies on the following ES spec changes to work:

* Defining the ModuleLoader API (similar amounts of work as existing proposal).
* Uses parsing for a module API use instead of new language tokens.
* Uses the mutable slot from existing proposal, and expands on it.

And later, when compile-time forms, like macros, might be used, a reader
concept and something like the PartialParsedFunction described above. However,
those do not need to be specified now. Hopefully enough has been illustrated to
show that they could be supported later through without needing to cause
backward-incompatible changes to the module APIs.

So for ES6, the new work is primarily around the mutable slot work. Work for it
was already required for ES6, but the `module` proposal expands on it a bit.

But it is important to note that no new HTML changes are needed, and while this
should not be a primary goal, enough of the module approach could be polyfilled
sooner to get more people to try it out before it has be considered done.

## mutable slots

The current ES proposal relies on a "mutable slot" concept for exports. This
allows circular dependencies to work well. That concept is reused for `module`,
and expanded a bit, to first level properties in the module export value.

How I have viewed the mutable slot, in terms of the existing ES module proposal.
Suppose a module imports paint from a brush module.

```javascript
import paint from 'brush';

// some time later
paint();
```

What I envision happens in the engine when `paint` is referenced: it results in
a call similar to `(System.get('brush').paint)`.

So mutable exports for the existing ES module proposal could be polyfilled today
by using an AST parser, like Esprima, to replace all references fo `paint`
(except for the initial import), with `(System.get('brush').paint)`, then
evaluate the code.

This view sees a mutable export reference similar to a getter.

For this `module` proposal, this is same mechanism would be used, except using
a module API with destructuring, instead of syntax:

```javascript
var { paint } = module('brush');

// some time later
paint();
```

In a polyfill, this would work by replacing the `paint` reference in the AST
with `(module('brush').paint)`.

## Differences from `System`

* No System.import, only module.load.

createHooks, although nested system loaders, loader plugins make more sense.

## `module` description

### Basic import and export

The module system is based on parsing for dependencies in a module, loading
and executing dependencies before executing the current module.

The module API indicates dependencies by using `module(StringLiteral)`. A module
sets the module export using `module.export(value)`, where `value` is the export
value. It must be set before the end of the module body execution.

```javascript
// A module that depends on two other modules
var { paint } = module('brush');
var pixelate = module('pixelate');

var someData = {};

pixelate(someData);

module.export({
  paint: paint,
  pixelate: pixelate
});
```

### Multiple, local modules

Multiple modules can be inlined in a file by using `module.define`, where
the module body is stored in a function.

```javascript
module.define('colorize', function(module) {
  module.export(function colorize() {});
});

module.define('blur', function(module) {
  module.export(function blur() {});
});

module.define('effects', function(module) {
  module.set({
    colorize: module('colorize'),
    blur: module('blur')
  })
});
```

For files that contain one module, `module.define` is not used, but the load
can conceptually treat this as executing a `module.define` with the contents
of the file as the body of the factory function passed to `module.define`.

Each module gets its own instance of `module` that is customized to that
module.

`module.define` modules can be nested, and the nested modules are only visible
to each other on the same level, and to the parent module containing them.

However, given the async resolution of dependencies (even local modules could
need dynamically loaded dependencies), `module()` in the same module cannot be
used to grab those resources and use them in an export for the current function.

Instead, there should be a local module that does a `module.define()` to set up
all of the exported value, and `module.exportLocal()` should be used to
specify what local module to use for the current module export.

```javascript
module.define('colorize', function(module) {
  module.export(function colorize() {});
});

module.define('blur', function(module) {
  module(function blur() {});
});

module.define('effects', function(module) {
  module.export({
    colorize: module('colorize'),
    blur: module('blur')
  })
});

// THE FOLLOWING COMMENTED OUT CODE WOULD FAIL because
// 'effects' may not be available yet, and 'effects' is
// only fully resolved once it is part of an active
// dependency tree.
// var effects = module('effects');
// var $ = module('jquery');

// THIS WILL WORK, define a module that does get those dependencies,
// and then use `module.exportLocal()` to use that module value as
// the export value.
module.define('publicExport', function(module) {
  var effects = module('effects');
  var $ = module('jquery');
  module.export(function applyEffectsToDom(selector) {
    return effects.colorize(effects.blur($(selector)[0]));
  });
});

module.exportLocal('publicExport');
```

If a module does not need a local module for immediate module export, then
`module.exportLocal` does not need to be used. Additionally, if this is not
a module body, but just a script in an HTML script tag, `module.load()` can
be used to get access to these local modules.

When a module is not present at the immediate module's loader level, it is
searched for up the chain of nested module loader instances. If it is not
available in any of the parents, the topmost loader will dynamically load it.

### `module.load`

Instead of a `System.import()`, just a `module.load()` for dynamically using
any modules that may or may not be loaded. This will most commonly be used in
web pages to start module loading, but could also be used inside a module.
Therefore, any local module name resolution should work with this call.

It takes a list of modules and a callback function and returns a Promise.
The success promise value is an array of module exports, but the callback
function receives them as individual callback arguments. The promise returned
will hold any error generated by the callback function, if one is generated.

```javascript
module.load('a', 'b', function(a, b) {
})
.catch(function(error) {
});

```

## Concerns about existing `module` use in JS

While `module` is used today in some code, notably in CommonJS/Node and AMD
module systems, that object is a second-tier object, and it appears as an
object and not a function. Script detection for those module systems usually
use `require`, `exports` and `define` for choosing what module system to use.

So it should be possible to use `module` without too much trouble. However,
a configuration option on the `module` loader could be provided to instruct the
loader to not inject or provide a wrapping that provies this kind of `module`
object if a script was wanting to use `module` in a different fashion.

In short, the benefits of an API approach and the declarative feel of using
`module('id')` are larger benefits than trying to avoid `module` for the case
of some scripts that are a small minority of all scripts and will shrink even
smaller over time.

----

Tests to write:

* return a promise for a module value, make sure that works, promise unwrapping not in the way.
* return a promise for a setFromLocal case too, promise unwrapping not in the way.
* test interop with normal scripts that do globals.

TODOS:

* how to do cycles. always transform source to funky gets?

* TODO: need a timeout on loads, since waiting on promises for lifecycle, need to shut down if taking too long. Call reject on specific modules. Allow for a reset though?

* Hmm, will loader plugins really work? the load() method right now allows fetching some
dependencies to finish loading of the resource. These can get associated with the load for that resource, to allow cycle breaking, in AMD systems. Does that hold together here?

Notes

* I prefer `return` to set module export, instead of system.set(), as it more correctly enforces "end of factory function means export is considered set" but right now JS grammar does not consider a top level `return` as valid.
* It may make sense to have a system.import() for specifying dependencies, and system.get() for just runtime fetching. This would allow system.get() to throw an error a bit sooner than if system.get() that returns a mutable slot might.

Things to consider

* How best to do load cleanups. Given async nature, and code could grab a reference to a load in the middle of something like the pipeline that survives across async calls, need to cautious about cleaning up. Maybe it is enough to do it once all module loading has been known to complete for the current cycle, since the module cache would be warm by then.
