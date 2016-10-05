---
layout: post
title: "Build a command-line interface app using node.js and dastoor"
description: ""
category: "node.js"
tags: [nodejs, cli, command-line, terminal]
---
{% include JB/setup %}

While building a command-line interface using node for myself ([onetime-cli](https://github.com/mvalipour/onetime-cli)),
I felt the need for a lean, convention-based and well-structured tool for building command-line interface apps.

This is how [dastoor](https://github.com/mvalipour/dastoor) (in Farsi  meaning "command") was born.

In this article I will take you through how to create a simple (with the potential to become complicated) CLI.

<!--more-->

The example that we will implement is an app with a command to sum any number of input arguments.

To start, create an empty nodejs app and install dastoor:

```bash
mkdir my-cli
cd my-cli
npm init
npm install --save dastoor
```

Now add `index.js` with the following code:

```javascript
#! /usr/bin/env node
var dastoor = require('dastoor'),
    runner  = new dastoor.Runner(),

    app     = require('./commands');

runner.run(app, process.argv.splice(2));
```

This code creates an instance of `Runner` and pass input arguments and app (which we are going to define below) to it.  
this is enough for our simple CLI to be wired.

Then create a `commands.js` file to define your command structure in.  
As your app becomes bigger and bigger you can break this file down into many other files.

```javascript
var cli = require('dastoor').builder;

var root = cli.node('my-cli', function() {
    console.log('hello world!');
});

module.exports = root;
```

OK, it's time to see some output.  
Before we would be able to run our CLI from terminal, we need to register it with npm. To do so add the following to your `package.json`:

```javascript
  "bin": {
    "my-cli": "index.js"
  }
```

This will tell npm that `my-cli` is a command we are adding to terminal. When users install your cli via `npm install -g`,
npm will add `my-cli` as a terminal command.

On dev machine however, you would have to manually do the linking by running the following command from your app directory:

```bash
npm link
```

Now by running

```bash
my-cli
```

You should get:

```bash
hello world!
```

Now let's add some real command to do the sum. We are going to add it to `commands.js` file:

```javascript
cli.node('my-cli.sum', { terminal: true})
.controller(function(args) {
   var res = args.initial || args.i || 0;
   args._.forEach(function (i) {
       res += +i;
   });

   console.log('Sum: ' + res);
});
```

The first line tells dastoor that we are defining the `sum` command as a sub-command of `my-cli`.
`terminal: true` tells the runner that this is a terminal node (i.e. doesn't have any sub-nodes).
We need to specify this otherwise runner will try to interpret our numbers (in `my-cli sum 1 2 3` command) as sub-command names.

Then we call into `.controller()` function to define the controller function of our sum command -- i.e. where we do the sum.
`args` input is the [minimist](https://github.com/substack/minimist) parsed version of all terminal arguments after `my-cli sum`.
which in this instance is all of our numbers in `_` (free parameters) and `i` (named parameter).

We simply sum all of the numbers and the `i` parameter and print the result in the output.

Now entering this into terminal:

```bash
my-cli sum 1 2 3 -i 100
```

will output:

```bash
Sum: 106
```

Let's add some help to our command. Add the following method call to the end of the `.node('my-cli.sum)` command:

```javascript
.help({
    description: 'sums numbers',
    options: [{
        name: '-i, --initial',
        description: 'initial value'
    }],
    usage: [
        'e.g. my-cli sum 1 2 3 -i 100'
    ]
});
```

This will tell dastoor all it needs to know to render help output.

```bash
my-cli sum -h
my-cli sum --help
```

```bash
>   sums numbers

    options:
      -i, --initial        initial value
      -h, --help           show this help

    usage:
      e.g. my-cli sum 1 2 3 -i 100
```

Similarly for `.node('my-cli')`:

```javascript
.help('my test cli')
```

```bash
my-cli --help
```

```bash
>   my test cli

    options:
      -h, --help           show this help

    usage:
      my-cli sum                 sums numbers
```

I'm going to stop here with the example but there are all sorts of possibilities with dastoor.
For more dastoor checkout the [API Reference](https://github.com/mvalipour/dastoor/wiki/API-Reference)

The full source for the sample CLI in this article can be found on [github -- mvalipour/dastoor-sample](https://github.com/mvalipour/dastoor-sample).

## Conclusion

[dastoor](https://github.com/mvalipour/dastoor) helps you build a CLI tool very easily
and with most of things like help and argument parsing, coming out-of-the-box.

Also given, the API style it has (inspired by angular.js) it structures our app
in a way to keep the code required to wire-up a command in one place, resulting in
more cocise code structure.
