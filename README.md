# system

This is an experiment that demonstrates, through working code, a module system
for ECMAScript (ES). It builds on the the `System` API from the current ES
proposal, along with some other work from the ES module effort so far, but makes
a couple of baseline modifications in an effort to reduce the amount of effort
and learning it will take for people to implement and use the current ES module
effort.

The `System` API was chosen also to hopefully make it easier to understand the
symantics of this experiment for people who are already familiar with the
current state of the ES module effort. However, it is expected that once this
experiment is understood, the API names could change. This experiment also uses
`system` instead of `System` since it is not a constructor.

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
* There are complications around default export and named export, and how that
looks like to `System.import`: for example, it would be ugly if a user had to
reference a `.default` property in the import success callback, where in the
import syntax form it is not necessary.
* The syntax encourages the desire for lexical modules. This is very hard to
reconcile with the dynamic parts of ES module loading.
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
var c = system.get('c');

// create exportable macros
system.setMacro('swap', {/* macro definition here*/});
system.setMacro('unless', {/* macro definition here*/});

// export runtime module value for a
system.set({
  name: 'a'
});

/***** module 'b' ******/
// b just wants to use some macros from 'a'
system.getMacro('a', 'swap', 'unless');


// export runtime module value for b
system.set(function b(x) {
  return true unless x > 42;
  return false;
});
```

The reader parses 'a' and sees the module APIs in play, and the
`system.setMacro` use. It pulls out the token sections for `system.setMacro`,
and since module bodies in the `system` approach are just functions, it
annotates the function with the pieces of information:

* module dependencies referenced
* set of macros found.

The module system can then use this annotated function for the "factory
function" used for module 'a'.

When 'b' is parsed by the reader, it notices the `system.getMacro` reference,
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

`system` relies on the following ES spec changes to work:

* Defining the ModuleLoader API (similar amounts of work as existing proposal).
* Uses parsing for a module API use instead of new language tokens.
* Uses the mutable slot from existing proposal, and expands on it.

And later, when compile-time forms, like macros, might be used, a reader
concept and something like the PartialParsedFunction described above. However,
those do not need to be specified now. Hopefully enough has been illustrated to
show that they could be supported later through without needing to cause
backward-incompatible changes to the module APIs.

So for ES6, the new work is primarily around the mutable slot work. Work for it
was already required for ES6, but the `system` proposal expands on it a bit.

But it is important to note that no new HTML changes are needed, and while this
should not be a primary goal, enough of the module approach could be polyfilled
to get more people to try it out before it is done.

## mutable slots

xx

## Differences from `System`

No System.import, system.load instead
createHooks


----

So creating a load, it should favor direct system.define() for one (parsed out when getting the factory function, otherwise, ask for parent load, and if it does not
have one, go up to topmost to get one. So, only get an intermediate one if a load
has already been registered.

And loader should create a local load, and local module table entry, for any module,
so that other modules loaded in that system scope get the same value always. Hmm,
but that means an intermediate could still intercept? Need a way to make sure get top
level load, but then fix value at that system level to the final resolved value.

* but what if an exterior function does a system.setMacro?
  system.define() is the boundary.

* what about loads of groups of modules, system.define, with a system.get?
  * allow system.get at top levels, that is the indicator that a custom
    system is needed.

* race conditions where things attached to the fetch event callbacks or load
  event callbacks are lost once the object transfers to module defined

* how to do cycles. always transform source to funky gets?

* TODO: need a timeout on loads, since waiting on promises for lifecycle, need to shut down if taking too long.

Notes

* does not implement realm stuff.
* I prefer `return` to set module export, instead of system.set(), as it more correctly enforces "end of factory function means export is considered set" but right now JS grammar does not consider a top level `return` as valid.
