---
layout: post
title: Handling Errors in Node.js
author: Sean Massa
categories: [dev]
blurb:
  The async callback standard in Node.js suggests that the first parameter of the callback
  is an error object. If that's null, you can move along. If it's not, you have to
  figure out what to do. Let's take a look at our options!
published: false
---

## logging

The first thing we should investigate is
how we can format our error for log output.
There are a lot of logging libraries out there
that will handle this for you,
but I want to take a look at everything involved.

Let's first try simply sending the error to `stderr`.

```coffeescript
console.error (new Error 'something broke')
# [Error: something broke]
```

That only output the actual error message.
What about the stack trace?

```coffeescript
console.error (new Error 'something broke').stack
###
Error: something broke
  at [eval]:1:16
  at Object.<anonymous> ([eval]-wrapper:6:22)
  at Module._compile (module.js:449:26)
  at evalScript (node.js:283:25)
  at startup (node.js:77:7)
  at node.js:628:3
###
```

The stack property contains the message as well as the stack.
If we only log this value, we get both!
But what about other properties that might exist on the error object?

```coffeescript
console.error JSON.stringify (new Error 'something broke')
# '{}'
```

That's empty, of course.
This is JavaScript-land, after all.
JSON.stringify accepts more arguments.
We can pass filters in the second argument.

```coffeescript
error = new Error 'something broke'
console.error JSON.stringify error, ['stack', 'message']
###
{
  "stack":"Error: something broke\n  at Object.<anonymous> (/home/smassa/test.coffee:5:9, <js>:7:11)\n  at Object.<anonymous> (/home/smassa/test.coffee:6:1, <js>:12:3)\n  at Module._compile (module.js:449:26)\n  at runModule (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:101:17)\n  at runMain (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:94:10)\n  at processInput (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:272:7)\n  at /home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:286:16\n  at fs.readFile (fs.js:176:14)\n  at Object.oncomplete (fs.js:297:15)\n",
  "message":"something broke"
}
###
```

Now, if you are wondering why filtering works when not filtering already shows nothing--
like I said, JavaScript-land.

But what happens if the error object has other properties we care about?

```coffeescript
error = new Error 'something broke'
error.inner = new Error 'some original error'
error.code = '500B'
JSON.stringify error, ['stack', 'message', 'inner']
###
{
  "stack":"Error: something broke\n  at Object.<anonymous> (/home/smassa/test.coffee:5:9, <js>:7:11)\n  at Object.<anonymous> (/home/smassa/test.coffee:8:1, <js>:16:3)\n  at Module._compile (module.js:449:26)\n  at runModule (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:101:17)\n  at runMain (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:94:10)\n  at processInput (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:272:7)\n  at /home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:286:16\n  at fs.readFile (fs.js:176:14)\n  at Object.oncomplete (fs.js:297:15)\n",
  "message":"something broke",
  "inner":{
    "stack":"Error: some original error\n  at Object.<anonymous> (/home/smassa/test.coffee:7:15, <js>:9:17)\n  at Object.<anonymous> (/home/smassa/test.coffee:8:1, <js>:16:3)\n  at Module._compile (module.js:449:26)\n  at runModule (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:101:17)\n  at runMain (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:94:10)\n  at processInput (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:272:7)\n  at /home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:286:16\n  at fs.readFile (fs.js:176:14)\n  at Object.oncomplete (fs.js:297:15)\n",
    "message":"some original error"
  },
  "code":"500B"
}
###
```

This works, but requires you to list all potential extra properties in the filter list.
That means this won't be a great general purpose solution.
Let's revisit our original attempt with a twist.

```coffeescript
error = new Error 'something broke'
error.inner = new Error 'some original error'
error.code = '500B'
JSON.stringify error
###
{
  "code":"500B",
  "inner":{}
}
###
```

The normal property `code` was found properly,
`stack` and `error` are still missing as we expect,
but `inner` shows up with no content.
This makes sense as well--
the value of `inner` is an Error object.

So, if we don't expect to have nested Error objects--
which is not an unreasonable expectation--
we could get a pretty complete error log with the following.

Let's also pull in [prettyjson](https://npmjs.org/package/prettyjson)
for better formatted output.

```coffeescript
prettyjson = require('prettyjson')                                                                                                                                                        
formatJson = (object) ->
  # adds 4 spaces in front of each line
  json = prettyjson.render(object)
  json = json.split('\n').join('\n    ')
  "    #{json}"

error = new Error 'something broke'
error.code = '500B'
error.severity = 'high'

stack = error.stack.trim()
metadata = formatJson error
"#{stack}\n  Metadata:\n#{metadata}"
###
Error: something broke
  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:11:11, <js>:14:11)
  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:17:1, <js>:20:3)
  at Module._compile (module.js:449:26)
  at runModule (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:101:17)
  at runMain (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:94:10)
  at processInput (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:272:7)
  at /home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:286:16
  at fs.readFile (fs.js:176:14)
  at Object.oncomplete (fs.js:297:15)
  Metadata:
    code:     500B
    severity: high
###
```

However, if we do expect to have nested error objects,
we need to go a little further.
Until this requirement,
I wanted to stay away from modifying standard object prototypes.
If that doesn't bother, read on!

It [turns out](http://stackoverflow.com/a/18391400/106)
that we can tell `Error` objects how to serialize themselves to json
by setting their `toJSON` properties.

```coffeescript
Object.defineProperty Error.prototype, 'toJSON',
  configurable: true
  value: ->
    alt = {}
    storeKey = (key) ->
      alt[key] = this[key]

    Object.getOwnPropertyNames(this).forEach(storeKey, this)
    alt
```

Now, when we serialize our `Error` objects, we get:

```coffeescript
error = new Error 'something broke'
error.inner = new Error 'some inner thing broke'
error.code = '500c'
error.severity = 'high'

JSON.stringify error 
###
{
  "message":"something broke",
  "code":"500c",
  "severity":"high",
  "inner":{
    "message":"some inner thing broke",
    "stack":"Error: some inner thing broke\n  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:16:15, <js>:20:17)\n  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:20:1, <js>:24:3)\n  at Module._compile (module.js:449:26)\n  at runModule (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:101:17)\n  at runMain (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:94:10)\n  at processInput (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:272:7)\n  at /home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:286:16\n  at fs.readFile (fs.js:176:14)\n  at Object.oncomplete (fs.js:297:15)\n"
  },
  "stack":"Error: something broke\n  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:15:13, <js>:19:11)\n  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:20:1, <js>:24:3)\n  at Module._compile (module.js:449:26)\n  at runModule (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:101:17)\n  at runMain (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:94:10)\n  at processInput (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:272:7)\n  at /home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:286:16\n  at fs.readFile (fs.js:176:14)\n  at Object.oncomplete (fs.js:297:15)\n"
}
###
```

This won't work directly wth prettyjson,
but we can do something messy:

```coffeescript
prettyjson = require 'prettyjson'
Object.defineProperty Error.prototype, 'toJSON',
  configurable: true
  value: ->
    alt = {}
    storeKey = (key) ->
      alt[key] = this[key]

    Object.getOwnPropertyNames(this).forEach(storeKey, this)
    alt
                                                                                                                                                                                          
error = new Error 'something broke'
error.inner = new Error 'some inner thing broke'
error.code = '500c'
error.severity = 'high'

prettyjson.render JSON.parse JSON.stringify error
###
severity: high                                                                                                                                                                     [0/150]
inner: 
  stack:   Error: some inner thing broke
  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:17:15, <js>:21:17)
  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:21:1, <js>:25:3)
  at Module._compile (module.js:449:26)
  at runModule (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:101:17)
  at runMain (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:94:10)
  at processInput (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:272:7)
  at /home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:286:16
  at fs.readFile (fs.js:176:14)
  at Object.oncomplete (fs.js:297:15)

  message: some inner thing broke
stack:    Error: something broke
  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:16:13, <js>:20:11)
  at Object.<anonymous> (/home/smassa/source/demo/blog/test.coffee:21:1, <js>:25:3)
  at Module._compile (module.js:449:26)
  at runModule (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:101:17)
  at runMain (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/run.js:94:10)
  at processInput (/home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:272:7)
  at /home/smassa/.nvm/v0.8.25/lib/node_modules/coffee-script-redux/lib/cli.js:286:16
  at fs.readFile (fs.js:176:14)
  at Object.oncomplete (fs.js:297:15)

message:  something broke
code:     500c
###
```

Which is not too bad.
If that doesn't overly disgust you,
this may be the solution you are looking for.
If it does, just go one solution back!



## long stack traces
`npm install long-stack-traces`
https://github.com/tlrobinson/long-stack-traces



## process

```
process.on 'uncaughtException', (error) ->
  log.error(error)
```


## domains


## express error middleware




