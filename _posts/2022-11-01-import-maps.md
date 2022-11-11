---
layout: post
title: "Javascript Import-maps is enabled in Firefox 108"
date: 2022-11-01 00:00:00 -0000
categories: import-maps
---


## Background: JavaScript Modules
If you don't know JavaScript modules, you can read our MDN docs for
[JavaScript Modules] first, and there are also some articles on Mozilla Hacks,
like [ES6 In Depth: Modules], and [ES modules: A cartoon deep-dive].
If you are already familiar with it, then you are probably familiar with
 [static import] and [dynamic imports] in your script.
As a quick refresher for anyone who is new:

```
// In a module script you can do a static import like so:
<script type="module">
import moment from "/node_modules/moment/src/moment.js";
</script>
```
```
// In a classic or a module script, you can do a dynamic import like so.
<script>
import("/node_modules/moment/src/moment.js");
</script>
```

Notice that in both static import and dynamic import above, you need to provide
a string literal with either the absolute path or the relative path of the
module script, so the [host] can know where the module script is located.

This string literal is called **[Module Specifier]** in ECMAScript specification[^1].

One subtle thing about **Module Specifier** is that each host has its own module
resolution algorithm to interpret the module specifier, for example, Node.js has
its own [Resolver Algorithm Specification], whereas browsers also have their
[Resolve A Module Specifier Specification]. The main difference between the two
algorithms is the resolution of the **bare specifier**, which is a module specifier
that is neither an absolution URL nor a relative URL.


### History: Modules between Node.js and ECMAScript
When Node.js 4.x was released, it had its own module system called
"CommonJS modules", and it has various ways to import a module, for example:
- Using a relative path or an absolution path.
- Using a core module name, like require("http")
- Using file modules.
- Using folders as modules.

Detailed can be found in [Node.js v4.x modules documentation].

And later when ECMAScript Modules were merged into HTML, they decided only
relative URLs and absolute URLs were allowed, bare specifiers were excluded at
that time, see [HTML PR 443]. Because bare specifiers could have some security 
concerns and would require more complex design in web standards.

After ECMAScript Modules becomes an official standard, now Node.js also wanted
to implement that, so Node.js added ESM modules implementation in
[Node.js v12 modules]. But in Node.js' implementation, it also borrowed the
concept of bare specifier from CommonJS, see [import specifier] from Node.js
documentation.


### Resolving a bare specifier
The following code will import a built-in module 'moment' on Node.js. However,
**it won't work for browsers that don't support Import-Maps**, unless you use some
transpiler like webpack and Babel.

```
// Import a bare specifier 'moment'.
// Valid on Node.js, but for browsers that don't support Import-maps, it will fail.
import moment from 'moment';
```

And this is a pretty common issue for web developers and newbies, they want to
use a JavaScript module in their webpages, but it turned out the module is a
Node.js module so they now need to spend more time to transpile it.

And Import-maps is the feature to reduce the gap of resolving module specifiers
between different Javascript runtimes, like Node.js and browsers, and it gives
us the ergonomics of bare specifiers, while also ensuring that the security
properties of URLs are preserved. This is what the proposal does at a
fundamental level for most use cases.

## Introduction to Import-Maps
Let's explain what's Import-maps and how you should use it in your web apps.

### Module Specifier remapping
With Import-maps is supported in Firefox now, you can do the followings on
Firefox:

```
// In a module script.
<script type="module">
import moment from "moment";
</script>
```
```
// In a classic or module script.
<script>
import("moment");
</script>
```

To make the resolution of *moment* work in Firefox, we need to provide the
location of the module script of the module specifier 'moment'.
This is where "Import-maps" comes into play.

To create an import map, you need to add a script tag whose type is
"**importmap**" to your HTML document[^2]. The body of the script tag is a JSON
object that maps the module specifier into the URL.

```
<!-- In the HTML document -->
<script type="importmap">
{
  "imports": {
     "moment": "/node_modules/moment/src/moment.js"
  }
}
</script>
```

When the browser tries to resolve a *Module Specifier*,
it will check if an Import-map exists in the HTML document and try to get the corresponding
URL of the module specifier. If it doesn’t find one, it will try to resolve the
specifier as a URL.


In our example with the "moment" library, it will try to find the entry whose key is "moment", and get the
value "/node_modules/moment/src/moment.js" from the Import-map.

What about more complex use cases? For example, browsers provide caching for
all files with the same name, so your websites will load faster. But what if we
update our module? In this case we would have to do “cache busting”. That is, we
rename the file we are loading. The name will be appended with the hash of
the file's content. In the above example, moment.js could become moment-1234abcd.js,
where the "1234abcd" is the hash number of the content of moment.js.

```
// Static import
<script type="module">
import moment from "/node_modules/moment/src/moment-1234abcd.js";
</script>

// Dynamic import
<script>
import("/node_modules/moment/src/moment-1234abcd.js");
</script>
```
This is quite a pain to do by hand! Instead of modifying all the files that would import the cached module script,
you could use Import-maps to keep track of the hashed module script.

```
<!--
An import map example to map the module specifier to the actual cached file
in the HTML document
-->
<script type="importmap">
{
  "imports": {
    "moment": "/node_modules/moment/src/moment-1234abcd.js"
  }
}
</script>
```

### Prefix remapping via a trailing slash '/'
Import-maps also allow you to remap the prefix of the module specifier, provided
that the entry in the Import-map ends with a trailing slash '**/**'.


```
// In the HTML document.
<script type="importmap">
{
  "imports": {
    "app/": "/js/app/"
  }
}
</script>
```

```
// In a Javascript module script.
<script type="module">
import foo from "app/foo.js";
</script>
```

In this example, There isn't an entry "app/foo.js" in the Import-map. However,
there's an entry "app/"(notice it ends with a slash '**/**'), so the "app/foo.js"
will be resolved to "/js/app/foo.js".

This feature is quite useful when the module contains several sub-modules, or
when you're about to test multiple versions of the external module,

```
<!-- In the HTML document -->
<script type="importmap">
{
  "imports": {
    "feature/": "/js/module/feature/",
    "app/": /js/app@4.0/",
  }
}
</script>
```

### Sub-folders need different versions of the external module.
Import-Maps provide another mapping called "**scopes**". It allows you to use the 
specific mapping table according to the URL of the module script. For example,


```
// In the HTML document.
<script type="importmap">
{
  "scopes": {
    "/foo/": {
      "app.mjs": "/js/app-1.mjs"
    },
    "/bar/": {
      "app.mjs": "/js/app-2.mjs"
    }
  }
}
</script>
```


In this example, the *scopes* map has two entries:
1. "/foo/" -> *Module specifier map 1*
2. "/bar/" -> *Module specifier map 2*

For the module scripts located in "/foo/", the "app.mjs" will be resolved to "/js/app-1.mjs",
whereas for those located in "/bar/", "app.mjs" will be resolved to "js/app-2.mjs".


```
// In /foo/foo.js
import app from "app.mjs"; // Will import "/js/app-1.mjs"
```


```
// In /bar/bar.js
import app from "app.mjs"; // Will import "/js/app-2.mjs"
```

---
## Explanation in depth
Let's explain the terms first.
The string literal "app.mjs" in the above examples is called *[Module Specifier]* in ECMA-Script,
and the map which maps "app.mjs" to a URL is called *[Module Specifier Map]*.

An import map is an object with two optional items:
- **_imports_**, which is a *module specifier map*.
- **_scopes_**, which is a map of URLs to *module specifier maps*.

So an import map could be thought of as:
- A top-level module specifier map called "**_imports_**".
- A map of module specifier maps called "**_scopes_**", could override the top-level module specifier map according to the location of the referrer.

If we put it into a graph

```
Module Specifier Map:
  +------------------+-----+
  | Module Specifier | URL |
  +------------------+-----+
  |  ......          | ... |
  +------------------+-----+
```

```
Import Map:
  imports:
    Top-level Module Specifier Map

  scopes:
    +-------+------------------------+
    | URL   | Module Specifier Map   |
    +-------+------------------------+
    | ...   | ...                    |
    +-------+------------------------+
```

### Validation of entries when parsing the import map
The format of the import map text has some requirements:
- A valid JSON string.
- The parsed JSON string must be a JSON object.
- The *imports* and *scopes* must be JSON objects as well.
- The values in *scopes* must be JSON objects since they should be the type of *Module Specifier Maps*.

Failing to meet any one of the above requirements will result in a failure to parse the import map,
and a **SyntaxError**/**TypeError** will be thrown.[^3]

```
<!-- In the HTML document -->
<script>
window.onerror = (e) => {
  // SyntaxError will be thrown.
};
</script>
<script type="importmap">
NOT_A_JSON_STRING
</script>
```

```
<!-- In another HTML document -->
<script>
window.onerror = (e) => {
  // TypeError will be thrown.
};
</script>
<script type="importmap">
{
  "imports" : "NOT_A_OBJ"
}
</script>
```

After the validation of JSON is done, parsing the import map will check
whether the values(URLs) in the Module specifier maps are valid.

If the map contains an invalid URL, the value of the entry in the module specifier map
will be marked as invalid. Later when the browser is trying to resolve
the module specifier, if the resolution result is the invalid value, the
resolution will fail and throw a **TypeError**.

```
<!-- In the HTML document -->
<script type="importmap">
{
  "imports": {
    "foo": "INVALID URL"
  }
}
</script>

<script>
// Notice that TypeError is thrown when trying to resolve the specifier with
// an invalid URL.
import("foo").catch((err) => {
  // TypeError will be thrown.
});
</script>
```

### Resolution precedence
When the browser is trying to resolve the module specifier, it will
find out the most specific *Module Specifier Map* to use, depending on the URL
of the referrer.

The precedence order of the *Module Specifier Maps* from high to low is:
1. **scopes**
2. **imports**

After the most specific *Module Specifier Map* is determined, then the resolving will
iterate the parsed module specifier map to find out the best match of the module
specifier:
1. The entry whose key equals the module specifier.
2. The entry whose key has the **longest common prefix** with the module
specifier provided the key ends with a trailing slash '/'.


```
<!-- In the HTML document -->
<script type="importmap">
{
  "imports": {
    "a/": "/js/test/a/",
    "a/b/": "/js/dir/b/"
  }
}
</script>

```

```
// In a module script.
import foo from "a/b/c.js"; // will import "/js/dir/b/c.js"
```

Notice that although the first entry "a/" in the import map could be used to resolve "a/b/c.js",
however, there is a better match below "a/b/" since it has a longer common prefix
of the module specifier. So "a/b/c.js" will be resolved to "js/dir/b/c.js", instead
of "/js/test/a/b/c.js".

Details can be found in [Resolve A Module Specifier Specification].

### Limitations of Import-maps
Currently, there are some limitations of the Import-maps, but these may be lifted
in the future:
- Only one Import-map is supported
  - Processing the first import-map script tag will disallow the following import maps from being processed.
    Those import map script tags won't be parsed and the onerror handlers will be called.
    Note that even if the first import map is failed to parse, those import maps
    afterward still won't be processed.
- Not supported for external import-maps. See [issue 235].
- Import-maps won't be processed if the module loading has been started.
- Not supported for workers/worklets. See [issue 2].

---
## Common problems when using import-maps
There are some common problems when you use Import-maps **incorrectly**:
- Invalid JSON format
  - Check the '**[Validation of entries when parsing the import map]**' section
  above. If the validation of the import map failed, a SyntaxError or a
  TypeError will be thrown when parsing the import map text.

- The module specifier cannot be resolved, but the import map seems correct:
This is one of the most common problems when using Import-maps. The import map
tag needs to be loaded **before** any module load happens, which includes:
  - Inline/External module load.
  - Static import/Dynamic import of Javascript modules.
  - Preload the module script in `<modulepreload>`.

- Unexpected resolution
  - See the '**[Resolution precedence]**' part above, and check if there is another specifier key
that takes higher precedence than the specifier key you thought.


---
## Chrome shipped this ages ago! What took you so long?
Mozilla started to implement this feature a while ago, see
[Intent to Prototype: Import-maps], but this feature was turned off by default.
At first, Import-maps was an incubated feature proposal in a web community group
called [WICG]. This is different than a standards organizations like [WHATWG] or
[ECMA International]. The feature was initiated by Google so they decided to
ship [Import-maps in Chrome 89]. But it was not a web standard yet so other Browser
vendors didn't prioritize implementing this feature at that time.


Import maps presented an important stepping stone for making the authoring of
the web more accessible. We also heard the many requests/inquiries from web
developers who were interested in seeing this feature land for similar reasons.
After discussing with Google, they agreed to finish this feature and publish it
as a web standard. Recently, the last work to integrate it into the specification
was finished, and [import-maps] has been officially [merged into HTML spec].
With this, we shipped Import-maps unflagged.


---
## Specification link
The specification can be found in [import-maps].

---
## Acknowledgments
Many thanks to [Jon Coppeard], [Yulia Startsev], and [Tooru Fujisawa] for their
contributions to the modules implementations and those code reviews on the Import-maps
implementation in Spidermonkey, and also great thanks to [Domenic Denicola] for
clarifying and explaining the specifications. and thanks to Steven De Tar to
coordinate this project.

TODO: Add people who help to review this post

[JavaScript Modules]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules
[ES6 In Depth: Modules]: https://hacks.mozilla.org/2015/08/es6-in-depth-modules/
[ES modules: A cartoon deep-dive]: https://hacks.mozilla.org/2018/03/es-modules-a-cartoon-deep-dive/
[static import]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import
[dynamic imports]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import
[host]: https://tc39.es/ecma262/#host
[Module specifier]: https://tc39.es/ecma262/#prod-ModuleSpecifier
[import specifier]: https://nodejs.org/api/esm.html#import-specifiers
[ImportSpecifier]: https://tc39.es/ecma262/#prod-ImportSpecifier
[Resolver Algorithm Specification]: https://nodejs.org/api/esm.html#resolver-algorithm-specification
[Resolve A Module Specifier Specification]: https://html.spec.whatwg.org/multipage/webappapis.html#resolve-a-module-specifier
[Node.js v4.x modules documentation]: https://nodejs.org/docs/latest-v4.x/api/modules.html
[Node.js v12 modules]: https://nodejs.org/docs/latest-v12.x/api/esm.html
[HTML PR 443]: https://github.com/whatwg/html/pull/443
[Module Specifier Map]: https://html.spec.whatwg.org/multipage/webappapis.html#module-specifier-map
[Resolution Precedence]: #Resolution-precedence
[Validation of entries when parsing the import map]: #Validation-of-entries-when-parsing-the-import-map
[issue 235]: https://github.com/WICG/import-maps/issues/235
[issue 2]: https://github.com/WICG/import-maps/issues/2
[Import-maps in Chrome 89]: https://chromestatus.com/feature/5315286962012160
[WICG]: https://wicg.io/
[WHATWG]: https://whatwg.org/
[ECMA International]: https://www.ecma-international.org/
[Intent to Prototype: Import-maps]: https://groups.google.com/a/mozilla.org/g/dev-platform/c/tiReRwpIT30
[merged into HTML spec]: https://github.com/whatwg/html/pull/8075
[import-maps]: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps
[Jon Coppeard]: https://hacks.mozilla.org/author/jcoppeardmozilla-com/
[Yulia Startsev]: https://hacks.mozilla.org/author/ystartsevmozilla-com/
[Tooru Fujisawa]: https://github.com/arai-a
[Domenic Denicola]: https://github.com/domenic

---
## Note
[^1]: In Node.js, it's called **[import specifier]**, but in ECMAScript, [ImportSpecifier] has a different meaning.
[^2]: Currently, external import maps are not supported, so you could only specify the import map in an HTML document.
[^3]: If it isn't a valid JSON string, a **SyntaxError** will be thrown. Otherwise, if the parsed strings are not of type JSON objects, a **TypeError** will be thrown.
