---
title: NodeJS, Express and JWT
author: Ali Heidari
layout: post
---
Step by step guide to use JWT as authenticator for a MVC Node application.

## Background

Hello everyone,

I was in the middle of an open-source project named PipesHub which uses JWT for authentication of clients, so i decided to share it with you; I talk about it in another post.
To going forward, you need to know what is jwt and why we need it? I am going to explain everything in brief, if you aren't familiar with one of these subjects, suggest you study a little about each of them.

### Why we need JWT

JSON Web Token is a token format to authentication and authorization of software system which communicates through the network. You can see the standard here <https://tools.ietf.org/html/rfc7519>. It consisted of 3 parts, headers + payload + signature which encoded by Base64 and separated by a '.'(dot).
If implemented correctly, it is a Secure proven way to transfer the authentication key with a stable structure lets you identify a consumer.

## Let's make something

First of all, do we have the Node?

```bash
~$ node -v
v13.12.0
```

If not, go to the official node website to install it on your machine.
<https://nodejs.org/en/>

Make a fresh project

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

### JSON Web Token

To implement JWT we use a module named [Jose](https://www.npmjs.com/package/jose). Install it using NPM.

```bash
nodeApp$ npm i jose --save
```

Create a folder named module then add authenticator.js.

```bash
nodeApp/modules$ mkdir modules && cd modules && touch authenticator.js
```

```JavaScript
const jose = require('jose');
const {
    JWK,
    JWT,
} = jose;
```

#### Private key

The whole story is you generate the private key and public key to sign and verify token, but words are something, codes are something else.

Let's generate the private key first.

```JavaScript
let privateKey;
async function init(){
  privateKey = await JWK.generate("RSA", 2048, {
      use: 'sig'
  });
}
```

We created a key using RSA encryption with the default length of 2048. It is needed when signs or verify the key, but we only sign. The generates method is asynchronous, so it must place in an async function. To make the synchronous call, you can use generateSync.

Let's use it to sign a token.

#### JWT Sign

```JavaScript
JWT.sign({
    'aid': 'some-key'
}, privateKey, {
    algorithm: 'PS512',
    audience: aud,
    issuer: iss
});
```

The sign accepts payload containing claims-set as the first parameter. You can use either conventional claims from [here](https://www.iana.org/assignments/jwt/jwt.xhtml) or use your own. In this example 'aid' used as a claim. Then specify the private key we generated before, And the last parameter is options like what algorithm to use and some essential claims which will replace it by values set in the payload.

You can read the docs here:
[sign](https://github.com/panva/jose/blob/master/docs/README.md#jwtsignpayload-key-options)

#### Public key

We can use the same private key or create a public key that is safe to provide for clients to verify. Let's modify init function.

```JavaScript
let privateKey, publicKey;
async function init(){
  privateKey = await JWK.generate("RSA", 2048, {
      use: 'sig'
  });
    let jwk = privateKey.toJWK();
    publicKey = JWK.asKey(jwk);
}
```

Use the generator function of the private key to create a public key.

#### JWT Verify

```JavaScript
const verified = JWT.verify(
    jwt,
    JWK.asKey(publicKey.toPEM())
console.log(verified);
```

#### Authenticator module

Now we know how to use jose, let's involve it in our app.
Open authenticator.js, we added jose to it before.
**This is final code**

```JavaScript
const jose = require('jose');
const {
    JWK,
    JWT,
} = jose;

let privateKey, publicKey;
/**
 * Generate private and public key
 */
const init = async () => {
    privateKey = await JWK.generate("RSA", 2048, {
        use: 'sig'
    });
    let jwk = privateKey.toJWK();
    publicKey = JWK.asKey(jwk);
}
/**
 * Sign the token
 */
const sign = (iss, aud) =>
    JWT.sign({
        'aid': 'some-key'
    }, privateKey, {
        algorithm: 'PS512',
        audience: aud,
        issuer: iss
    });
/**
 * Verify the request
 */
const verify = function (req, res, next) {
    if (exports.exceptions.indexOf(req.url) < 0){
        if(req.headers.authorization==undefined){
            next(new Error('No token provided'));
            return;
        }
        const jwt = req.headers.authorization.substr(7);
        const verified = JWT.verify(
            jwt,
            JWK.asKey(publicKey.toPEM())
        );
    }
    next();
};


/**
 * Exports
 */
exports.init = () => init();
exports.sign = (iss, aud) => sign(iss, aud);
exports.verify = (req, res, next) => verify(req, res, next);
exports.exceptions = ['/users/auth'];
```

We added init, sign, and verify functions and made them accessible from outside of the module using **exports** keyword.
**exceptions** is an array to exclude routes from the verification process. Inside verify function We Check if the requested route is in the exceptions array then expect the token from authorization header. It makes the client request valid while the 'token' sent through the header and verified by the public key successfully.
Because the authorization header contains a bearer schema, The substr removes the first 7 characters.

Open app.js and add these lines before routing.
Add verify method as middleware to express.

```JavaScript
const auth = require("./modules/authenticator");

//...
auth.init();
app.use(auth.verify);
//...
```

Let's use JWT.

Add these lines to ./routes/users.js

```JavaScript
//...
const auth = require("../modules/authenticator");
//...

//...
router.post('/auth', function (req, res, next) {
    if(req.body.pass=='123')
        res.end(auth.sign('localhost', 'some-uid'));
    else
        res.send('Invalid pass');
});
//...
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

You should see an error which says:
**No token provided**

I test it with Postman, so be sure if you have other tool or method to send test requests set the settings correctly.

1. <http://localhost:3000/users/auth> for address
2. Make sure it sets on **POST** request
3. Choose x-www-form-urlencoded in body tab
4. Set Key = pass And Value = 123 (Use wrong pass to see error)
5. Send request

If you enter pass incorrect, you get **Invalid pass**, but if you enter correct pass you should get the token. Copy it because you need it for further requests.

### Test token

Create new tab in post man.

1. <http://localhost:3000/> for address
2. Make sure it sets on **GET** request
3. Choose Authorization tab
4. From the type list select Bearer
5. Paste the token you received from the previous step in textbox
6. Send request

You should see **Express welcome message**

If you enter wrong token, you get **JWT is malformed**

Ok, We done here! You got the idea, create amazing stuffs.

## Conclusion

In this article we learned:

- How to make a NodeJs app
- MVC pattern using ExpressJS
- ExpressJS Middleware
- Authentication using JSON Web Token.

I hope it helped you. Please leave a comment and share your ideas to make it better.
