CSS MQPacker
============

Pack same CSS media query rules into one using PostCSS


SYNOPSIS
--------

A CSS file processed with a CSS pre-processor may have same queries that can
merge:

```css
.foo::before {
  content: "foo on small";
}

@media screen and (min-width: 769px) {
  .foo::before {
    content: "foo on medium";
  }
}

.bar::before {
  content: "bar on small";
}

@media screen and (min-width: 769px) {
  .bar::before {
    content: "bar on medium";
  }
}
```

This PostCSS plugin packs exactly same queries (and optionally sorts) like this:

```css
.foo::before {
  content: "foo on small";
}

.bar::before {
  content: "bar on small";
}

@media screen and (min-width: 769px) {
  .foo::before {
    content: "foo on medium";
  }
  .bar::before {
    content: "bar on medium";
  }
}
```


INSTALL
-------

    $ npm install css-mqpacker


USAGE
-----

Of course, this package can be used as PostCSS plugin:

```javascript
#!/usr/bin/env node

"use strict";

var fs = require("fs");
var postcss = require("postcss");

var css = fs.readFileSync("from.css", "utf8");
postcss([
  require("autoprefixer-core")(),
  require("css-mqpacker")()
]).process(css).then(function (result) {
  console.log(result.css);
});
```


### As standard Node.js package

Read `from.css`, process its content, and output processed CSS to STDOUT.

```javascript
#!/usr/bin/env node

"use strict";

var fs = require("fs");
var mqpacker = require("css-mqpacker");

var original = fs.readFileSync("from.css", "utf8");
var processed = mqpacker.pack(original, {
  from: "from.css",
  map: {
    inline: false
  },
  to: "to.css"
});
console.log(processed.css);
```


### As CLI Program

This package also installs a command line interface.

    $ node ./node_modules/.bin/mqpacker --help
    Usage: mqpacker [options] INPUT [OUTPUT]
    
    Description:
      Pack same CSS media query rules into one using PostCSS
    
    Options:
      -s, --sort       Sort “min-width” queries.
          --sourcemap  Create source map file.
      -h, --help       Show this message.
          --version    Print version information.
    
    Use a single dash for INPUT to read CSS from standard input.
    
    Examples:
      $ mqpacker fragmented.css
      $ mqpacker fragmented.css > packed.css

When PostCSS failed to parse INPUT, CLI shows a CSS parse error in GNU error
format instead of Node.js stack trace.

The `--sort` option does not currently support a custom function.


OPTIONS
-------

### sort

By default, CSS MQPacker pack and order media queries as they are defined. See
also [The "First Win" Algorithm][1]. If you want to sort queries automatically,
pass `sort: true` to this module.

```javascript
postcss([
  mqpacker({
    sort: true
  })
]).process(css);
```

Currently, this option only supports `min-width` queries with specific units
(`ch`, `em`, `ex`, `px`, and `rem`). If you want to do more, you need to create
your own sorting function and pass it to this module like this:

```javascript
postcss([
  mqpacker({
    sort: function (a, b) {
      return a.localeCompare(b);
    }
  })
]).process(css);
```

In this example, all your queries will sort by A-Z order.

This sorting function directly pass to `Array#sort()` method of an array of all
your queries.


API
---

### pack(css, [options])

Packs media queries in `css`.

The second argument is optional. The `options` are:

- [options][2] mentioned above
- the second argument of [PostCSS’s `process()` method][3]

You can specify both at the same time.

```javascript
var fs = require("fs");
var mqpacker = require("css-mqpacker");

var css = fs.readFileSync("from.css", "utf8");
var result = mqpacker.pack(css, {
  from: "from.css",
  map: {
    inline: false
  },
  sort: true,
  to: "to.css"
});
fs.writeFileSync("to.css", result.css);
fs.writeFileSync("to.css.map", result.map);
```


NOTES
-----

With CSS MQPacker, the processed CSS is always valid CSS, but you and your
website user will get unexpected results. This section explains how CSS MQPacker
works and what you should keep in mind.


### CSS Cascading Order

CSS MQPacker changes rulesets’ order. This means the processed CSS will have an
unexpected cascading order. For example:

```css
@media (min-width: 640px) {
  .foo {
    width: 300px;
  }
}

.foo {
  width: 400px;
}
```

Becomes:

```css
.foo {
  width: 400px;
}

@media (min-width: 640px) {
  .foo {
    width: 300px;
  }
}
```

`.foo` is always `400px` in original CSS, but `300px` if viewport is wider than
`640px' with processed CSS.

This does not occur on small project. On large project, however, this could
occur frequently. For example, if you want to override a CSS framework (like
Bootstrap) component declaration, your whole CSS code will be something similar
to above example. To avoid this problem, you should pack only CSS you write, and
then concaenate with a CSS framework.


### The "First Win" Algorithm

CSS MQPacker is implemented with the "first win" algorithm. This means:

```css
.foo {
  width: 10px;
}

@media (min-width: 640px) {
  .foo {
    width: 150px;
  }
}

.bar {
  width: 20px;
}

@media (min-width: 320px) {
  .bar {
    width: 200px;
  }
}

@media (min-width: 640px) {
  .bar {
    width: 300px;
  }
}
```

Becomes:

```css
.foo {
  width: 10px;
}

.bar {
  width: 20px;
}

@media (min-width: 640px) {
  .foo {
    width: 150px;
  }
  .bar {
    width: 300px;
  }
}

@media (min-width: 320px) {
  .bar {
    width: 200px;
  }
}
```

This breaks cascading order of `.bar`, and `.bar` will be displayed in `200px`
instead of `300px` even if a viewport wider than `640px`.

I suggest defining a query order on top of your CSS:

```css
@media (min-width: 320px) { /*! Wider than 320px */ }
@media (min-width: 640px) { /*! Wider than 640px */ }
```

If you use only simple `min-width` queries, [the `sort` option][4] can help.


### Multiple Classes

CSS MQPacker works only with CSS. This may break CSS applying order to an
elements that have multiple classes.

```css
@media (min-width: 320px) {
  .foo {
    width: 100px;
  }
}

@media (min-width: 640px) {
  .bar {
    width: 200px;
  }
}

@media (min-width: 320px) {
  .baz {
    width: 300px;
  }
}
```

Becomes:

```css
@media (min-width: 320px) {
  .foo {
    width: 100px;
  }
  .baz {
    width: 300px;
  }
}

@media (min-width: 640px) {
  .bar {
    width: 200px;
  }
}
```

The result looks good. However, if an HTML element has `class="bar baz"` and
viewport width larger than `640px`, that element `width` incorrectly set to
`200px` instead of `300px`. This problem cannot be resolved only with CSS, so be
careful!


LICENSE
-------

MIT: http://hail2u.mit-license.org/2014


[1]: #the-first-win-algorithm
[2]: #options
[3]: http://api.postcss.org/global.html#processOptions
[4]: #sort
