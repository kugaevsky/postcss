# PostCSS

PostCSS is a framework for CSS postprocessors,
to modify CSS by your JS function.

It takes care about most of common CSS tool task:

1. parses CSS;
2. gives you usable JS API to edit CSS node tree;
3. saves modified node tree to new CSS;
4. generates (or modify exists) source map for your changes;

You can use this framework to write you own:

* CSS minifier or beautifizer.
* Grunt plugin to generate sprites, include `data-uri` images or any other work.
* Text editor plugin to automate CSS routine.
* Command-line CSS tool.

Sponsored by [Evil Martians](http://evilmartians.com/).

## Build with PostCSS

* [Autoprefixer](https://github.com/ai/autoprefixer)
* [grunt-pixrem](https://github.com/robwierzbowski/grunt-pixrem)

## Quick Example

Let’s fix forgotten `content` property in `::before` and `::after`:

```js
var postcss = require('postcss');

var contenter = postcss(function (css) {
    css.eachRule(function (rule) {
        if ( rule.selector.match(/::(before|after)/) ) {
            // In every ::before/::after rule

            // Did we forget content property?
            var good = rule.some(function (i) { return i.prop == 'content'; });

            if ( !good ) {
                // Add content: '' if we forget it
                rule.prepend({ prop: 'content', value: '""' });
            }

        }
    });
});
```

And then CSS with forgotten `content`:

```css
a::before {
    width: 10px;
    height: 10px
}
```

will be fixed by our new `contenter`:

```js
var fixed = contenter.process(css).css;
```

to:

```css
a::before {
    content: "";
    width: 10px;
    height: 10px
}
```

## Features

### Source Map

PostCSS generates source map for it’s transformations:

```js
result = processor.process(css, { map: true, from: 'from.css', to: 'to.css' });
result.css // String with processed CSS
result.map // Source map
```

And modify source map from previous step (like Sass preprocessor):

```js
var sassMap = fs.readFileSync('from.sass.map');
processor.process(css, { map: sassMap, from: 'from.sass.css', to: 'to.css' });
```

### Preserves code formatting and indentations

PostCSS will not change any byte of rule if you didn’t modify node:

```js
postcss(function (css) { }).process(css).css == css;
```

And when you modify CSS nodes, PostCSS will try to copy coding style:

```js
contenter.process("a::before{color: black}")
// a::before{content: '';color: black}

contenter.process("a::before {\n  color: black;\n  }")
// a::before {
//   content: '';
//   color: black;
//   }
```

## Why PostCSS Better Than …

### Preprocessors

Preprocessors (like Sass or Stylus) give us special language with variables,
mixins, statements and compile it to CSS. Compass, nib and other mixins
libraries use this languages to work with prefixes, sprites and inline images.

But Sass and Stylus languages was created to be syntax-sugar for CSS.
Writing really complicated programs on preporcessor languages is very difficult.
[Autoprefixer] is totally impossible on Sass.

With PostCSS you can work with CSS by comfort and powerful JS or CoffeeScript.
You can do really magic things with wide range of [npm] libraries.

But postprocessors are not a enemy for preprocessors. Sass and Stylus is still
best way to add reability and some sugar to CSS syntax. You can easily
combine preprocessors and postprocessors.

[Autoprefixer]: https://github.com/ai/autoprefixer
[npm]:          https://npmjs.org/

### RegExp

Some Grunt plugins modify CSS by regular expressions. But CSS parser and
node tree is much safer way to edit CSS. Also regexps will broke source map
from preprocessors.

### CSS Parsers

There are a lot of good CSS parsers, like [Gonzales]. But they help you only
with first step.

Instead of them, PostCSS also gives you useful high level API (for example,
safe iterators) and will generate source map for changes (or modify exists
source map from Sass).

[Gonzales]: https://github.com/css/gonzales

### Rework

[Rework] was a first CSS postprocessors framework and very similar to PostCSS.

But Rework hasn’t high level API and will destroy your CSS code style and indentations. So we can’t use it in text editor plugins.

Instead of it, PostCSS will preserves all spaces and code formatting. If you
didn’t change rule, output will be byte-to-byte equal.

[Rework]: https://github.com/visionmedia/rework

## Usage

### Processor

Function `postcss(fn)` creates processor by your function:

```js
var postcss = require('postcss');

var processor = postcss(function (css) {
    // Code to modify CSS
});
```

If you want to combine several processors (and parse CSS only once),
you can create empty processor and add several functions by `use(fn)` method:

```js
var all = postcss().
          use(prefixer).
          use(minifing);
```

Processor function can just change current CSS node tree:

```js
postcss(function (css) {
    css.append( /* new rule */ )
});
```

or create totally new CSS root and return it:

```js
postcss(function (css) {
    var newCSS = postcss.root()
    // Add rules and declarations
    return newCSS;
});
```

Processor will transform some CSS by `process(css, opts)` method:

```js
var doubler = postcss(function (css) {
    // Clone each declaration
    css.eachDecl(function (decl) {
        decl.parent.prepend( decl.clone() );
    });
});

var css    = "a { color: black; }";
var result = processor.process(css);

result.css //=> "a { color: black; color: black; }"
```

You can set original CSS filename by `from` options and syntax error messages
will be much helpful:

```js
var wrong = "a {";
processor.process(wrong, { from: 'main.css' });
//=> Can't parse CSS: Unclosed block at line 1:1 in main.css
```
### Source Map

PostCSS will generate source map, if you set true to `map` option
in `process(css, opts)` method.

You must set input and output CSS files paths (by `from` and `to` options)
to generate correct map.

```js
var result = processor.process(css, {
    map:  true,
    from: 'main.css',
    to:   'main.out.css'
});

result.map //=> '{"version":3,"file":"main.out.css","sources":["main.css"],"names":[],"mappings":"AAAA,KAAI"}'

fs.writeFileSync('main.out.map', result.map);
```

PostCSS can also modify previous source map (for example, from Sass
compilation). So, if you compile: Sass to CSS and then minify CSS
by postprocessor, final source map will contains mapping from Sass code
to minified CSS.

Just set original source map content (in string on in JS object)
to `map` option:

```js
var result = minifier.process(css, {
    map:   fs.readFileSync('main.sass.map'),
    from: 'main.sass.css',
    to:   'main.min.css'
});

result.map //=> Source map from main.sass to main.min.css
```

### Nodes

Processor function will receive `Root` node with CSS node tree inside.

```js
var processor = postcss(function (cssRoot) {
});
```

There are 3 types of nodes can be inside: `AtRule`, `Rule` and `Declaration`.
All nodes contain `toString()` and `clone()` methods.

You can parse CSS and get `Root` node by `postcss.parse(css, opts)` method:

```js
var postcss = require('postcss');

var cssRoot = postcss.parse('a { }');
```

### Whitespaces

All nodes (exclude `Root`) has `before` property with spaces and comments,
which was before node.

Nodes with children (`Root`, `AtRule` and `Rule`) contain also `after` property
with spaces after last child and before `}` or end of file.

```js
var root = postcss.parse("a {\n  color: black;\n}\n");

root.after                    //=> "\n" from end of file
root.rules[0].after           //=> "\n" before }
root.rules[0].decls[0].before //=> "\n  " before color: black
```

So, the simplest way to minify CSS is to clean `before` and `after` properties:

```js
var minifier = postcss(function (css) {
    css.eachDecl(function (decl) {
        decl.before = '';
    });
    css.eachRule(function (rule) {
        rule.before = '';
        rule.after  = '';
    });
    css.eachAtRule(function (atRule) {
        atRule.before = '';
        atRule.after  = '';
    });
});

var css = "a{\n  color:black\n}\n";
minifier.process(css).css //=> "a{color:black}"
```

### Raw Properties

Some CSS values (like selectors, at-rule params and declaration values) can
contain comments. PostCSS will clean them for you:

```js
var root = postcss.parse("a /**/ b {}");
var ab   = root.rules[0];

ab.selector //=> 'a  b' trimmed and cleaned from comments
```

But PostCSS saves origin raw content to stringify it to CSS, if you didn’t
set new value. As you remember, PostCSS try to save origin CSS byte-to-byte,
when it is possible:

```js
ab.toString() //=> 'a /**/ b {}' with comment

ab.selector = '.link b';
ab.toString() //=> '.link b' you change value and magic was gone
```

### Containers

`Root`, `AtRule` and `Rule` nodes can contain children in `rules` or `decls`
property.

There are common method to work with children:

* `append(newChild)` to add child to end of children list.
* `prepend(newChild)` to add child to start of children list.
* `insertBefore(existsChild, newChild)` to insert new child before some
   extists child.
* `insertAfter(existsChild, newChild)` to insert new child before some
   extists child.
* `remove(child)` to remove child.
* `index(child)` to return child index.
* `some(fn)` to return true if `fn` return true on any of children.
* `every(fn)` to return true if `fn` return true on all of children.

Methods `insertBefore`, `insertAfter` and `remove` can receive child node
or child index number as exists child argument, but index is much faster.

### Children

`AtRule`, `Rule` and `Declaration` nodes will be inside other nodes.

All children contain `parent` property with parent node:

```js
rule.decls[0].parent == rule;
```

All children has `removeSelf()` method:

```js
rule.decls[0].removeSelf();
```

But `remove(index)` in parent with child index is much faster:

```js
rule.each(function (decl, i) {
    rule.remove(i);
});
```

### Iterators

All containers node has `each` method, to iterate it children nodes:

```js
root = postcss.parse('a { color: black; display: none }');

root.each(function (rule, i) {
    console.log(rule.selector, i); // Will log "a 0"
});

root.rules[0].each(function (decl, i) {
    console.log(decl.prop, i); // Will log "color 0" and "display 1"
});
```

Instead of simple `for` or `Array#forEach()` this iterator is safe.
You can change children inside iteration and it will fix current index:

```js
rule.rules.forEach(function (decl, i) {
    rule.prepend( decl.clone() );
    // Will be infinity cycle, because on prepend current declaration become
    // second and next index will go to current declaration again
});

rule.each(function (decl, i) {
    rule.prepend( decl.clone() );
    // Will work correct (once clone each declaration), because after prepend
    // iterator index will be recalculated
});
```

Because CSS is nested structure, PostCSS contains recursive iterator
by node type:

```js
root.eachDecl(function (decl, i) {
    // Each declaration inside root
});

root.eachRule(function (rule, i) {
    // Each rule inside root and any nested at-rules
});

root.eachAtRule(function (atRule, i) {
    // Each at-rule inside root and any nested at-rules
});
```

### Root Node

`Root` node contains all CSS tree. It children can be only `AtRule` or `Rule`
nodes in `rules` property.

You can create new root by shortcut:

```js
var root = postcss.root();
```

Method `toString()` will stringify all current CSS:

```js
root = postcss.parse(css);
root.toString() == css;
```

### AtRule Node

`AtRule` has two own property: `name` and `params`.

```css
@charset 'utf-8';

@font-face {
    font-family: 'Cool'
}

@media print {
    img { display: none }
}
```

As you see, some doesn’t contain any children (like `@charset` or `@import`),
some of at-rules can contain only declarations (like `@font-face` or `@page`),but most of them can contain rules and nested at-rules (like `@media`,
`@keyframes` and other).

Parser select `AtRule` content type by it name. If you create `AtRule` node manually, it will detect own content type by new child type on first `append`
or other add method call:

```js
var atRule = postcss.atRule({ name: '-x-animations' });
atRule.rules        //=> undefined
atRule.decls        //=> undefined

atRule.append( postcss.rule({ selector: 'from' }) );
atRule.rules.length //=> 1
atRule.decls        //=> undefined
```

You can create new at-rule by shortcut:

```js
var atRule = postcss.atRule({ name: 'charset', params: 'utf-8' });
```

### Rule Node

`Rule` node has `selector` property and contains `Declaration` childs
in `decls` property:

```css
a {
    color: black;
}
```

You can miss `Declaration` constructor in `append` and other add methods:

```js
rule.append({ prop: 'color', value: 'black' });
```

Property `semicolon` marks, does last declaration in rule should have semicolon:

```js
var root = postcss.parse('a { color: black }');
root.rules[0].semicolon //=> false

var root = postcss.parse('a { color: black; }');
root.rules[0].semicolon //=> true
```

You can create new rule by shortcut:

```js
var rule = postcss.rule({ selector: 'a' });
```

### Declaration Node

`Declaration` node has `prop` and `value` properties.

```css
color: black
```

You can create new declaration by shortcut:

```js
var decl = postcss.decl({ prop: 'color', value: 'black' });
```

Or use short form in `append()` and other add methods:

```js
rule.append({ prop: 'color', value: 'black' });
```
