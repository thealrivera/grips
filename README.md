# grips Templating Engine

**NOTE: this project used to be called "HandlebarJS". To avoid confusion with the newer but more popular "Handlebars" templating engine from @wycats, I'm renaming this project to "grips".**

grips is a simple templating engine written in JavaScript. It's designed to work either/both in the browser or on the server, with the same code-base and the same template files.

grips takes as its only input data in a simple JSON data-dictionary format, and returns output from the template(s) selected.

grips will "compile" requested templates into executable JavaScript functions, which take the JSON data dictionary as input and return the output. The compilation of templates can either be JIT (at request time) or pre-compiled in a build process.

## Examples

The examples/ directory has several sample template files. Take a look at "tmpl.master.html" and "tmpl.index.html" in particular for good general real-world looking examples. "test.html" is more esoteric and shows off more complexities of the syntax nuances.

## Templating sytax

### Define a named template section (aka, "partial")

	{$: "#xxx" }
		...
		...
	{$}

### Define a named template section (partial), with local variable assignment(s)
  `$` is the data object (context) passed into the partial.

	{$: "#xxx" |
		x = $.val_1 |
		y = $.val_2 ? "#yyy" : "#zzz"
	}
		...
		...
	{$}

### Conditional Assignments

	{$: "#bar" |
		baz = $.baz ? "yes" : ""
	}
		baz: {$= baz $}
	{$}

  Also, the `else` condition of a conditional can be omitted and defaults to `""`.

	{$: "#bar" |
		baz = $.baz ? "yes"
	}
		baz: {$= baz $}
	{$}

### Include data variable

	{$= $.val_1 $}  <!-- OR -->  {$= myval $}

### Static include template section

	{$= @"#yyy" $}

### Dynamic include template section from variable

	{$= @$.val_1 $}  <!-- OR -->  {$= @myval $}

### Loop on data variable (array or plain key/value object)
  `$$` iteration binding includes: 
    `index` (numeric), `key` (string: property name or index),
    `value`, `first`, `last`, `odd`, and `even`

	{$* $.val_1 }
		...
		{$= $$.value $}  <!-- include the loop iteration value -->
		...
	{$}

### Loop on data variable, with loop iteration local variable assignment(s)

	{$* $.val_1 |
		rowtype = $$.odd ? "#oddrow" : "#evenrow" | 
	    someprop = $$.value.someProp ? "#hassomeprop"
	}
		...
		{$= @rowtype $}  <!-- include local variable -->
		...
		{$= @someprop $}
		...
		{$= $$.value.otherProp $}  <!-- include loop iteration value variable -->
		...
	{$}

### Range Literals

    {$* [2..7] } <!-- let's count from 2 to 7 -->
    	counting: {$= $$.value $}
    {$}

    {$* [4..-3] } <!-- count down from 4 to -3 -->
    	counting: {$= $$.value $}
    {$}

### Set Literals

    {$* ["Jan", "Feb", "Mar"] } <!-- show the months in Q1 -->
    	month: {$= $$.value $}
    {$}

### Precomputing hash literals
  Hash literals can be pre-computed against a defined set or range of values. In the below examples, the value of `$.myradio` will be compared to all values in the hash (0,1,2 or "low","medium","high"). The results of the comparison and conditional assignment are stored in a local variable hash. For instance, in the below example, part of what the syntax implies is: `checked[1] = ($.myradio === 1) ? "checked" : ""`.

	{$: "#bar" |
		checked[0..2] = $.myradio ? "checked"
	}
		<input type="radio" name="myradio" value="0" {$= checked[0] $}>
		<input type="radio" name="myradio" value="1" {$= checked[1] $}>
		<input type="radio" name="myradio" value="2" {$= checked[2] $}>
	{$}

  Using a loop the above can be even more terse:

	{$: "#bar" |
		checked[0..2] = $.myradio ? "checked"
	}
		{$* [0..2] }
			<input type="radio" name="myradio" value="{$= $$.value $}" {$= checked[$$.value] $}>
		{$}
	{$}

  Here's pre-computed against a set-literal:

	{$: "#bar" |
		checked["low","medium","high"] = $.myradio ? "checked"
	}
		<input type="radio" name="myradio" value="low" {$= checked["low"] $}>
		<input type="radio" name="myradio" value="medium" {$= checked["medium"] $}>
		<input type="radio" name="myradio" value="high" {$= checked["high"] $}>
	{$}

### "Extend" (inherit from) another template collection

	{$+ "collection-id-or-filename" $}

### Raw un-parsed section

	{$%
		this stuff {$= won't be parsed $}, just passed through raw
	%$}

### Template comment block

	{$/
		comments get removed in parsing
	/$}

## Using the JavaScript API

The two most typical tasks for the JavaScript API are `compileCollection()` and `render()`.

"grips" organizes template partials by grouping them together in collections. A single collection is an arbitrary grouping of one or more partials, but it usually will correspond to a single template file. The collection ID is arbitrary, but again, will usually be the filename of the template file.

You can render an individual partial, but you compile a collection of one or more partials.

### Compiling a collection
To compile a collection of partials, call `compileCollection(templateStr, collectionID,[build=true])`. 

`templateStr` is the string representation of your collection of template partials. `collectionID` should be the same as any other references to the collection by ID, such as other absolute template includes, or template extend directives, in other collections.

`build`, which defaults to true, is an optional boolean flag. If `true`, it will build/interpret the compiled template function representation, so that it's ready to render. `compileCollection()` also returns you the string value of that compiled template function, so you can choose to store it in a file during a build process, etc.

A collection ID is the first part of a canonical template ID (`foo` in `"foo#bar"`, whereas `#bar` is the partial ID). For example, the `{$+ ... $}` collection extend tag takes only the collection ID (without any `#bar` partial ID).

Ex:

```
grips.compileCollection("{$: '#bar' } Hello {$= $.name $}! {$}", "foo");
```

For convenience, if you want to compile several collections at once, use `compile(sources, [build=true])`. `sources` is an object whose keys are collection names and values are collection template sources.

Ex:

```
grips.compile({
	foo: "{$: '#bar' } Hello {$= $.name $}! {$}"
});
```

### Rendering a partial
In an environment where one or more collections have been built (aka, interpreted/executed), they can be rendered by calling `render(templateID, data)`.

To render a partial, you refer canonically to its `templateID` by both the partial ID and the collection in which it lives. For example: `"foo#bar"`, where `foo` is the collection ID and `#bar` is the partial ID.

Ex:

```
var markup = grips.render("foo#bar", {name: "World"});
```

### Other methods
Since you can pre-compile templates during a build process and store them in files for later use in production, you can call `buildCollection()` (or `build()`) to interpret/execute a compiled template function's source retrieved from a file.

For instance, if you had the compiled functions for the `foo` collection in source form (from a file), you can build them (so they're ready for rendering) by either `eval()`ing them yourself, or with `buildCollection(collectionID, compiledSource)`.

Ex:

```
eval(fooCompiledSource);

// or, better:

grips.buildCollection("foo", fooCompiledSource);
```

And if you have all your collections in one big string, you can just call `build(compiledSourceBundle)`.

Ex:


```
eval(compiledSource);

// or, better:

grips.build(compiledSource);
```

## Building grips

grips comes with a node JavaScript tool called "build", in the root directory, which when run will generate the files you need to use grips, in a directory called "deploy".

There are several options for the build tool:

```
usage: build [opt, ...]

options:
--help       show this help
--verbose    display progress
--all        build all options (ignores --runtime and --nodebug)
--runtime    builds only the stripped down runtime (no compiler)
--nodebug    strip debug code (smaller files with less graceful error handling)
--minify     minify all built files with uglify.js (creates *.min.js files)
```

By default, the build tool is silent, meaning it outputs nothing to the console unless there are errors. If you'd like verbose output, pass the `--verbose` flag. Unless you pass the `--runtime`, the build tool will generate both the "full" compiler and the base runtime. `--all` overrides/ignores `--runtime` (and `--nodebug`).

Also by default, the build tool will generate only the debug versions of the files (with extra/verbose error handling, etc). For production use, pass the `--nodebug` flag, and the debug code will be stripped. Or, pass the `--all` flag and both debug and non-debug files will be built.

To also produce the minified versions of all files being built (suitable for production deployment), pass the `--minify` flag. Note: to use minification, the node.js "uglify-js" package should be installed (via npm). (`npm install 'uglify-js'`)

## License

grips (c) 2012 Kyle Simpson | http://getify.mit-license.org/
