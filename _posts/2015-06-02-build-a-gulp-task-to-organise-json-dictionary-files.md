---
layout: post
title: "Build a gulp task to organise JSON dictionary files"
description: ""
category: "front-end"
tags: [gulp, angular, translation, JSON]
---
{% include JB/setup %}

Recently I've been building an angular app for a local language (farsi).
To inject translated text in my template files, I use
[angular-template](https://github.com/angular-translate/angular-translate)
which is an awesome
library that provides you with directives and filters to translate text
in template files.

One of the ways to feed translated text into `angular-template` is to
store translations into a JSON dictionary file and link it to the translator.
(using [this](https://github.com/angular-translate/bower-angular-translate-loader-static-files) plugin.)

```language-javascript
{
    "-- Select --": "-- انتخاب کنید --",
    "About": "درباره ما",
    "Action": "عملیات"
}
```

```language-javascript
.config(function($translateProvider){
  $translateProvider.useStaticFilesLoader({
    prefix: '/lang/',
    suffix: '.json'
  });

  $translateProvider.preferredLanguage('fa-IR');
})
```

However on a decent-sized app, you will soon hit into difficulty organizing
your dictionary files.

There is however always something to do to make life easier:

<!--more-->

![Gulp! All the tasks!](http://i.imgur.com/tQCKC5O.jpg)

## Solution

We need to build a gulp task that...

- accepts a `phrase` and a `translation` and adds it to the list
in the right alphabetical order.

In order capture input into our gulp task, we will be utilizing
command line arguments.

Luckily there is an `npm` package called [yargs](https://www.npmjs.com/package/yargs) that parses command line arguments
into a ready-to-use object.

```language-javascript
var argv = require('yargs').argv;
```

Now it's pretty easy to add few lines of code to build the `add-translation`
task:

```language-javascript
var fs = require('fs');
var _ = require('underscore');

gulp.task('add-translation', function(){
  var phrase = argv.p;
  var translation = argv.t;

  if(!phrase || !translation) throw 'provide both phrase and translation';

  var path = './res/lang/fa-ir.json';
  var data = JSON.parse(fs.readFileSync(path));
  data[phrase] = translation;

  var keys = _.chain(data).keys().sort().compact().value();
  var result = {};
  _.each(keys, function(i){
    result[i] = data[i];
  });

  fs.writeFileSync(path, JSON.stringify(result, null, 4));
});
```

First two lines, captures `phrase` and `translation` variables from command line args and ensures they are not left empty.

Then we load the target dictionary file (`fa-IR` in this case), parse it as json
and add an entry to it for the new `phrase`.

Finally, we use `underscore` to sort the object (by it's keys) and write the result
back into the dictionary file.

Once this is added to the `gulpfile.js`, it can be used like this in the terminal:

```language-bash
$ gulp add-translation -p "some phrase" -t "یک عبارت"
```

Enjoy!
