---
title: NodeJS, Express and Markdown
author: Ali Heidari
layout: post
---

Step by step guide to use markdown as a view engine for a MVC Node application.

## Background

Hello everyone,

In this post, we going to make a simple Node app with Express using Markdown as the view engine. I was in the middle of an open-source project named PipesHub which uses markdown to show some simple content, Decided to share this part of it with you. I'll write about PipesHub later.
Did you see the Readme file in Github projects? How clean, simple, and easy to read it is. Those files are written in Markdown.

### ExpressJS

ExpressJS is a Nodejs framework that helps a lot to create a routing capable system. There is a tool offered by the Express team named "Express application generator" which creates a fresh project based on a template that is simple but rich. It provides you all the basic needs.

### What is Markdown

Markdown is a language to make writing easier, As John Gruber says:
>Markdown is a text-to-HTML conversion tool for web writers. Markdown allows you to write using an easy-to-read, easy-to-write plain text format, then convert it to structurally valid XHTML (or HTML).

It is a great language for writers. Because of its simplicity to write a description page that needs no fancy design; That's why I preferred to choose Markdown for my project.

## Let's make something

First of all, do we have the Node?

```bash
~$ node -v
v13.12.0
```

If no, go to the official website of Node to install it on your machine.
<https://nodejs.org/en/>

Make a new project

```bash
~$ mkdir nodeApp && cd nodeApp
```

Install ExpressJS

```bash
~/nodeApp$ npm install express --save
~/nodeApp$ npx express-generator
```

The last command installs an express-generator and runs it. The content of nodeApp folder should be something similar to this:

- nodeApp
  - bin
  - public
  - routes
  - views
  - app.js
  - package.json

If you open the app.js, you can see all basics added already.

### Markdown view engine

I suggest you place your modules in a folder named modules.

```bash
nodeApp$ mkdir modules && cd modules
nodeApp/modules$ touch md_engine.js
```

Now, open md_engine.js with an editor. VSCode is a good choice.
This is how we define a new custom view engine.

```JavaScript
app.engine('extension', function (filePath, options, callback) {
  // Do something
});
```

In the code above, the "app" is an Express object defined in the template.
We are going to export this module as a function named "setEngine" and pass the app object. Only, module knows the detail of the engine

```JavaScript
exports.setEngine = function (app) {
  app.engine('md', function (filePath, options, callback) {
    // Do something
  });
}
```

We created a function can be invoked from the outside of the module. It consists of a setter to set the view engine of ExpressJS. "md" is the extension of our views files.
Ok, now let's read the file passed by Express callback. Remind that Express looks for the requested view file, if it exists, then the callback raises.

```JavaScript
const fs = require('fs');//<<<<<<<<<<<<

exports.setEngine = function (app) {
    app.engine('md', function (filePath, options, callback) {
        fs.readFile(filePath, function (err, content) {//<<<<<<<<<<<<
            if (err)//<<<<<<<<<<<<
                return callback(err);//<<<<<<<<<<<<

            let md = content.toString();
            return callback(null, md);
            // Do something
        });//<<<<<<<<<<<<
    });
}
```

Well, I marked the added lines. It is quite simple, we read the content of the file. In case of error, we inform Express otherwise the content will return. But, there is a mistake, we should not return raw Markdown content, needs to be translated to HTML before presenting it as a response. Indeed, there is a module named [showdown](https://www.npmjs.com/package/showdown). Install it to go forward.

```bash
~/nodeApp$ npm i showdown
```

**Note: Be sure you located in root of app.**

```JavaScript
const fs = require('fs');
const showdown = require('showdown');//<<<<<<<<<<<<
/**
 * Sets the view engine
 */
exports.setEngine = function (app) {
    app.engine('md', function (filePath, options, callback) {
        fs.readFile(filePath, function (err, content) {
            if (err)
                return callback(err);
            let md = content.toString()// Edited
                .replace('?title?', options.title)//<<<<<<<<<<<<
                .replace('?message?', options.message);//<<<<<<<<<<<<
            const converter = new showdown.Converter();//<<<<<<<<<<<<
            let html = converter.makeHtml(md);//<<<<<<<<<<<<
            return callback(null, html);// Edited
        });
    });
    app.set('view engine', 'md');
}
```

It is so easy to use, create an object of Converter then translate Markdown to HTML by makeHtml method.
What are those replaces!?
You can specify some place holders to make dynamic content. I have chosen question marks to wrap variable names because the question mark is not a Markdown reserved character.
Let's see how to use this.

```bash
nodeApp/modules$ cd ../views
nodeApp/views$ touch index.md
```

```markdown
# ?title?

**?message?**
```

**Delete .jade files, we no need them.**
Then open the controller here: nodeApp/routes/index.js

```JavaScript
var express = require('express');
var router = express.Router();

/* GET home page. */
router.get('/', function(req, res, next) {
  res.render('index', { title: 'Express' });
});

module.exports = router;
```

Change the render to:

```JavaScript
  res.render('index', { title: 'Markdown view engine', message: 'Hello from markdown View engine...' });
```

### Display errors

To show an error create a file named error.md in views folder and put the code below in it.

```markdown
# ?message?

**?error.status?**

?error.stack?
```

Ok, let's back to md_engine.js we created in modules folder. Apply changes below:

```JavaScript
const fs = require('fs');
const showdown = require('showdown');
/**
 * Sets the view engine
 */
exports.setEngine = function (app) {
    app.engine('md', function (filePath, options, callback) {
        fs.readFile(filePath, function (err, content) {
            if (err)
                return callback(err);
            let md = content.toString()
                .replace('?title?', options.title)
                .replace('?message?', options.message);
            if(options.error)
              md = md.replace('?error.status?', options.error.status)//<<<<<<<<<<<<
                     .replace('?error.stack?', options.error.stack);//<<<<<<<<<<<<
            const converter = new showdown.Converter();
            let html = converter.makeHtml(md);
            return callback(null, html);
        });
    });
    app.set('view engine', 'md');
}
```

### In the end

One last thing, open app.js in root path, then add this line top of file.

```JavaScript
const engine = require("./modules/md_engine");
```

In app.js replace
  
```JavaScript
app.set('view engine', 'jade');
```

with

```JavaScript
engine.setEngine(app);
```

let's run it.

```bash
nodeApp$ npm start
```

This command runs a script defined in the package.json. Typically, it is something like "node ./bin/www". If you wonder what is www inside the bin folder, just open it using a text editor. You see it runs a node server then imports app.js as a module to execute.

In case you got an error because of missing packages, install them using code below.
**Run it in root project.**

```bash
nodeApp$ npm install
```

If no error occurred, or all been resolved, run the app again.

Now open this address on the browser.

<http://localhost:3000>

You should see this:

![Normal page](/assets/images/posts/node-with-markdown/nodeApp.png)

Enter this and you get error.

<http://localhost:3000/error>

![Error page](/assets/images/posts/node-with-markdown/error.png)

## Conclusion

What we learned:

- How to make a NodeJs app
- ExpressJS Middleware
- Set a custom view engine

I hope it helped you. Please leave a comment and share your ideas to make it better.
