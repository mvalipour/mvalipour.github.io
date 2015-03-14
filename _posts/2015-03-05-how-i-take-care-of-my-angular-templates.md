---
layout: post
title: "how I use gulp to take care of my angular template urls"
description: ""
category: "front-end"
tags: [angular, gulp]
---
{% include JB/setup %}

While developing an angular SPA (single page app), I felt the need to find a shortcut to avoid repeating myself when it comes to the `templateUrl` of the ui-router states.

Given that I'm already using gulp to:

- squash all javascript files into a single `.js` file (to avoid having numerous `<script>` tags in my html file) and
- squash all template files into a `template-cache.js` file (to one request per-template on client)

I found that there can be a neat trick added to the gulp to shorten the `templateUrl`s in my states.

<!--more-->

## Application Structure

I use [Y140  - LIFT principal](https://github.com/johnpapa/angular-styleguide#application-structure-lift-principle) standard for my application structure.


```language-bash
/app
  /todos
    index.js
    index.html
  /customers
    index.js
    index.html
    add.html
  /...
```

This basically means that each module has a folder of itself that contains:

- `index.js` that contains the module definition, ui-router state configuration, controllers (if small enough) and so on.
- `index.html` that contains the partial html for the main view of the module -- e.g. the list of it's a collection module, etc.

## Template urls

In the `index.js` when we configure the ui-router states, we need to provide the `templateUrls` that with this structure it will usually end-up being long paths (relative to the root of the application).

To solve this issue, I have defined two conventions:

- `>` to point to `<current folder>/index.html`
- `>/some-path` means the file is relative to this path. -- at `<current folder>/some-path`

**Note** that this convention is not suitable if you do not squash your templates into a template cache file.

Our gulp squash task replaces `>` (with folder path) in `.js` files before the squash happens.

```language-javascript
var jsFiles = ['./app/**/index.js', './app/**/*.js'];

gulp.task('squash_js', function(){

    return gulp.src(jsFiles)
      .pipe(gulpTap(function (file) {
          var relativePath = path.relative(file.cwd, path.dirname(file.path));
          var newPath = relativePath.replace(/\\/g, '/');

          var content = file.contents.toString();
          content = content.replace(/'>\//g, '\'' + newPath + '/');
          content = content.replace(/'>'/g, '\'' + newPath + '/index.html\'');

          var newBuffer = new Buffer(content);
          file.contents = newBuffer;
      }))
      .pipe(gulpIf(isProduction, uglify()))
      .pipe(gulp.dest('.dist/app.js'));
});
```

**Bonus:** by setting `isProduction` to `true` in the above code, the squashed js file will also be uglified.

Now if you use this in your ui-router:

```language-javascript
  ...
    templateUrl: '>',
    templateUrl: '>/add.html',
  }
```

gulp converts it to:

```language-javascript
  ...
    templateUrl: 'app/customers/index.html',
    templateUrl: 'app/customers/add.html',
  }
```

## Template file cache

It is important that the path we use in our template cache file is compatible with the path we create for `templateUrl`s.

For clarity, here is how my template cache task looks like:

```language-javascript
var templateFiles = ['./app/**/*.html'];

gulp.task('squash_template', function () {
    gulp.src(templateFiles)
        .pipe(templateCache('templates.js', { module: 'portal.templates', standalone: true, root: 'angular'}))
        .pipe(gulp.dest('.dist/templates.js'));
});

```

Here I'm using [gulp-angular-templatecache](https://www.npmjs.com/package/gulp-angular-templatecache) library and I configure it to create a new angular module (named `portal.templates`).
