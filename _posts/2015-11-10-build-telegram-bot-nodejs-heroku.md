---
layout: post
title: "Build a telegram bot using node.js and heroku"
description: ""
category: "node.js"
tags: [nodejs, heroku, telegram]
---
{% include JB/setup %}

A little while ago I started using [telegram](https://telegram.org/) as my messaging app (admittedly along-side dozen other apps). But it didn't take long till I realized, there is something different about telegram which is really cool. It's [almost] fully open-source and more importantly highly programmable, so much that they natively support bots! and quickly I found myself wanting to write my own bot.

In this article I will take you through the steps of how to write a very simple telegram bot that interacts with the user on telegram and delivers a service. You can use this as a starting-point for building your more complex bots.

<!--more-->

## Register your bot

Like many other platforms, the first thing you need to do is to register your app. Doing so is amazingly simple in telegram and is done via... well... another bot on telegram!

[@BotFather](https://telegram.me/BotFather) is the telegram's designated robot for registering and managing your bots. Simply start chatting with it and you will see all of your options. -- here we focus only on registering our new bot.

Simply type:

```language-bash
/newbot
```

and you will be asked to provide a `name` and `identifier` for your bot. Once done, BotFather will give you a `token` which is what you need to configure in your server component later in this article.

## Domain

For the sake of this tutorial, let's assume we need our bot to support the following two commands:

- `/say_hello <your-name>` to reply `Hello <your-name>`
- `/sum <num1> <num2> ...` to sum up any given numbers and reply the result

## Write bot server

Next, we need to write a node.js program that listens to the incoming communications from bot users and responds to them.

For this we use [node-telegram-bot-api](https://github.com/yagop/node-telegram-bot-api) which is a nicely-done wrapper around telegram's bot api.

```language-bash
npm init
npm install --save node-telegram-bot-api
```

Now create `bot.js` file:

```language-javascript
var token = '<token-from-bot-father>';

var Bot = require('node-telegram-bot-api'),
    bot = new Bot(token, { polling: true });

console.log('bot server started...');
```

This will instantiate a new instance of `Bot` in *polling mode* (i.e. an infinite loop that reads incoming messages as they come) using the token we obtained earlier.

Now, let's write our first command listener:

```language-javascript
bot.onText(/^\/say_hello (.+)$/, function (msg, match) {
  var name = match[1];
  bot.sendMessage(msg.chat.id, 'Hello ' + name + '!').then(function () {
    // reply sent!
  });
});
```

Above code registers a text-listener that calls our callback whenever a *text* message is received matching the given syntax. The reg-ex also extract the name segment out of the message and passes to the callback via the `match` parameter.

Then we send a *text* message back to the sender of the original message (`msg.chat.id` which may be a group or individual chat) with the result.

And similarly our `sum` command:

```language-javascript
bot.onText(/^\/sum((\s+\d+)+)$/, function (msg, match) {
  var result = 0;
  match[1].trim().split(/\s+/).forEach(function (i) {
    result += (+i || 0);
  })
  bot.sendMessage(msg.chat.id, result).then(function () {
    // reply sent!
  });
});
```

This one has a slightly more complicated reg-ex which extracts all of the numbers given as the command parameter, then split the chunk, convert them to int (`+` prefix) and sum-up. Then similar to above, send the result back to the user.

## Run!

Run the bot and bingo!

```language-bash
node ./bot.js
```

Your bot should now be responding to the messages:

![screen](http://i.imgur.com/33QgkP8.jpg)

## Http end-point

There are two reasons why we need to add a http end-point to our node.js bot server:

**First of all** to continue enjoying Heroku for free, we want our polling loop to run for-ever on a web role (as oppose to a paid [Heroku scheduler add-on](https://devcenter.heroku.com/articles/scheduler)). Heroku shuts the web role process down, if no http port is listened from the process within 30 seconds.

**Additionally** I would like to set-up a web monitoring service ([Uptime Robot](https://uptimerobot.com/) provides this for free), to ensure my server is always up and running and never goes to sleep.

So I spin up a simple web server that simply returns app version number as json.

```language-bash
npm install --save express
```

```language-javascript
var express = require('express');
var packageInfo = require('./package.json');

var app = express();

app.get('/', function (req, res) {
  res.json({ version: packageInfo.version });
});

var server = app.listen(process.env.PORT, function () {
  var host = server.address().address;
  var port = server.address().port;

  console.log('Web server started at http://%s:%s', host, port);
});
```

## Deploy

Finally we need to create and deploy our Heroku app.

```language-bash
heroku create
```

This will create a new app in your heroku account (you will be asked to setup one if have no account setup) with a random name. To open the app in your browser simply run:

```language-bash
heroku open
```

Each time you need to deploy the app, simply run the following command and push your latest code to heroku remote repo, which then consequently result in your node.js app building and starting on the Heroku servers.

```language-bash
git push heroku master
```

## Conclusion

This is a minimalistic example to show you how easy and cool it is to create your own telegram bot in no time. You can keep adding more and more complexity and business logic to your server. For example remember users' choice in a database to then report back to them in later commands, etc.

A complete source code of this tutorial can be found on the following GitHub repository:

[https://github.com/mvalipour/telegram-bot-simple](https://github.com/mvalipour/telegram-bot-simple)
