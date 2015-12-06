---
layout: post
title: "Configure a telegram bot webhook into an existing express app"
description: ""
category: "node.js"
tags: [nodejs, heroku, telegram]
---
{% include JB/setup %}

Recently I wrote about [How to build a telegram bot using node.js and heroku](/node.js/2015/11/10/build-telegram-bot-nodejs-heroku/). In the article we use polling to listen to incoming messages. This approach has few problems though...

<!--more-->

1. heroku may/will shut down the web-server after a period of inactivity that will result in your polling loop to shut down too -- hence missing incoming traffic.

2. polling loop is slightly naive in handling/discarding incoming messages. (see [infinite loop issue](https://github.com/yagop/node-telegram-bot-api/issues/36)).

3. user message will have a delay in processing up to the interval specified (seconds at least) which may become annoying in message-heavy bots.

## Webhook

Luckily solution is easy. With webhooks telegram does you the favor of calling into your API whenever a new message comes. and it does it in a fairly reliable fashion. (at least far more reliable that what a polling loop does).

In our case we already have an express web app and so wish to add an api end-point for the webhook and configure telegram to use it.

Simply change the setup of your express web app to include the following:

```language-javascript
var bodyParser = require('body-parser');
app.use(bodyParser.json());

app.post('/' + token, function (req, res) {
  bot.processUpdate(req.body);
  res.sendStatus(200);
});
```

The first two line enables the json body parser to allow reading payload on incoming request with `content-type=application/json`. (note that telegram makes a `POST` request to the api with the standard message metadata as request body in `json` format).

Then we listen to the `/<token>` path for `POST` requests and simply call into the bot's message handler.

Next, we need to configure webhook on our telegram bot:

```language-javascript
var bot = new Bot(token);
bot.setWebHook('https://my-web-root.com/' + bot.token);
```

This line will let telegram know about the api you wish messages be sent to.

## Development vs Production

As cool as webhooks are, unfortunately you can't utilize them on development machine -- given telegram servers can't make a request to the local server running on your machine.

The solution I came up with is to fall-back to the polling model for the development environment:

```language-javascript
var bot;

if(process.env.NODE_ENV === 'production') {
  bot = new Bot(token);
  bot.setWebHook('https://my-web-root.com/' + bot.token);
}
else {
  bot = new Bot(token, { polling: true });
}
```

when heroku starts the app, `NODE_ENV` environment variable is set to `production` and therefore the if block is hit.

## Conclusion

Although the `node-telegram-bot-api` library supports wiring up an `http(s)` listener (via `webhook: true` option), I decided to hook the bot into my own express app for various reasons:

1. I already have a web-app for monitoring/health-checking purposes, plus I may add some additional functionality for my app via a web-interface.

2. most importantly, I have two bots that I wish to wire-up in the same app (they have a lot to share) and as of today, the library doesn't support joining up multiple bots to share the same https server.

Therefore I decided to pass empty options to the bot and call into it from my express api handler.

P.S.

A complete source code of this tutorial can be found on the following GitHub repository:

[https://github.com/mvalipour/telegram-bot-webhook](https://github.com/mvalipour/telegram-bot-webhook)
