---
title: Building an AngularJS app
author: Tobias Davis
layout: post
---

This post is a description of an in-house, custom-built architecture
that we use to make our [AngularJS](https://angularjs.org/) app. Ours
is a single-page app with a few dozen screens, using a couple dozen
third-party JS libraries. Since we started building the app
several years ago, we've gathered a lot of experience in ways
to structure things to be more maintainable.

Initially, we started developing our app using [ngbp](https://github.com/ngbp/ngbp),
a boilerplate starter for AngularJS-based sites. The starting
settings are reasonable, and will get you started in a short
time using pretty good defaults.

However, over the years, as we added several utilities and
third-party modules, we started to find that maintainability
was difficult in certain cases, adding true ES6 support
would have been difficult, and other similar issues brought
us to designing a new build system.

# Overview

We use [browserify](http://browserify.org/) to bundle everything, and
a bunch of plugins to transpile ES6 and do other things. We use
[Bootstrap](https://getbootstrap.com/) and LESS for styles.

# The JS Side

Each of our Angular files (e.g. directives, factories, etc.) are inside
a JS file ending in `*.angular.js`, written as a CommonJS module which
exports (at least) a single function. The only thing passed to the function
is a variable which is equivalent to the global `window.angular`.

An example file might look like this (using ES6):

```js
module.exports = function(app) {
	app.factory('logPageLoad', $http => {
		return (page, details) => {
			return $http.post(`/rest/log_page_load`, {
				page: page,
				details: details || null
			}).then(result => result.data)
		}
	})
}
```

Things to note:

* The variable `app` is equivalent to the `window.angular` that is
	available when you source the Angular JS file.
* Our build process adds the array annotation `['$http', function($http) { ... }]`
	for factories.
* The module name is not defined in this file, which means
	dependency definitions of the app aren't defined here.

The equivalent long-form way to write this would be (in ES5):

```js
// not included in the file
var app = angular.module('myApp', [])

// this is the equivalent part
app.factory('logPageLoad', [ '$http', function($http) {
	return function logPageLoad(page, details) {
		return $http.post('/rest/log_page_load', {
			page: page,
			details: details || null
		}).then(function(result) {
			return result.data
		})
	}
}])
```

Our build process grabs all the `*.angular.js` files, and creates
a temporary file that looks like this:

```js
var app = angular.module('ourApp', [
	'ui.router', // https://github.com/angular-ui/ui-router
	'ui.bootstrap' // https://github.com/angular-ui/bootstrap
])

;[
	require('./path/to/log-page-load.angular.js'),
	require('./path/to/other.angular.js') // etc.
].forEach(fn => fn(app))
```

Then we use browserify to put it all together, and that
looks something like this:

```js
const builder = browserify({
	entries: [
		// this is the temp file reference
		'./angular-bootstrapper.tmp.js'
	],
	cache: {},
	packageCache: {},
	debug: true
}).transform(envify(process.env)).transform('babelify', {
	presets: 'es2015'
}).transform('stringify')

if (dev) {
	builder.plugin(require('errorify'))
	builder.plugin(require('watchify'))
} else {
	builder.transform(ngAnnotate)
}

const stream = builder.bundle()
const writer = fs.createWriteStream('./bundle.js')
stream.pipe(writer)
```

# CommonJS and Angular Modules

Since our build runs through browserify, our Angular modules can
use CommonJS `require`, like this simple example:

```
const transform = require('transform-some-data')

module.exports = function(app) {
	app.filter('doSomething', () => {
		return (input) => transform(input)
	})
}
```

This means we write CommonJS modules for most of the data tranformation
that we do on the client side, so we end up writing a lot of modules
that get `require`'d instead of injected into the Angular global namespace.

One thing that can become frustrating is relative paths, so that
you end up with these two requiring the same file:

* `const transform = require('../../../transform-some-data.js')`
* `const transform = require('./path/to/transform-some-data.js')`

This is awkward, primarily because it means that refactoring
anything becomes a dangerous act. What we did to resolve this
problem is have our folder structure look something like this:

```txt
webapp/
	package.json
	node_modules/
		# modules added by the normal `npm install --save`
	src/
		index.html
		my-state.angular.js
		node_modules/
			edatasource/
				package.json
				thing.js
```

Now inside `my-state.angular.js` we can reference third-party
modules this way:

```js
const stuff = require('npm-module')
```

And we can reference our own custom JS modules this way, without
any relative path:

```js
const thing = require('edatasource/thing.js')
```

That's the core of it!
