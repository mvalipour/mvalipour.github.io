---
layout: post
title: "how I use gulp to take care of my angular templates"
description: ""
category: "front-end"
tags: [angular, gulp]
---
{% include JB/setup %}

OK, so there is basically two main things I need to take care of automatically while developing my angular single page app:

1. Resolving relative paths used in angular modules with absolute ones.

1. Squashing all partial html files into a single `js` file and adding them to angular's template-cache.

<!--more-->

### relative template urls

I organize my angular modules like this:

```
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

So in general, each module has a folder that contains:

- `index.js` that contains the module definition, ui-router state configuration, controllers (if small enough) and so on.
- `index.html` that contains the partial html for the main view of the module -- e.g. the list of it's a collection module, etc.

In the `index.js` when we configure the ui-router states, we need to provide the `templateUrls` that with this structure it will usually end-up being very long absolute paths.

To solve this issue, I have two conventions for templateUrls to make them as short as possible:

- `>` means our template is at `<current folder>/index.html`
- `>/some-path` means the file is relative to this path. -- at `<current folder>/some-path`

Given the fact that I'm using gulp to squash all `.js` files into a single file, it's then easy to replace `>` with the appropriate value in gulp:

```language-javascript
var jsFiles = ['./angular/**/index.js', './angular/**/*.js'];

gulp.task('main_portal_build_js', function(){

    return gulp.src(input)
      .pipe(gulpTap(function (file) {
          var relativePath = path.relative(file.cwd, path.dirname(file.path));
          var newPath = relativePath.replace(/\\/g, '/');

          var content = file.contents.toString();
          content = content.replace(/'>\//g, '\'' + newPath + '/');
          content = content.replace(/'>'/g, '\'' + newPath + '/index.html\'');

          var newBuffer = new Buffer(content);
          file.contents = newBuffer;
      }))
      .pipe(sourcemaps.init())
          .pipe(gulpIf(isProduction, ngAnnotate({
              add: true,
              single_quotes: true,
          })))
          .pipe(concat(output.fileName + '.js'))
      .pipe(sourcemaps.write())
      .pipe(gulpIf(isProduction, uglify()))
      .pipe(gulp.dest(output.folder))
      .pipe(livereload());
});
```

### template cache

When developing a single page application, it's hard to imagine not to squash all partial `.html` template files into angular template cache file. -- otherwise you will end-up bombarding your server for partials files.
