# An Introduction To Utilizing Public-Key Cryptography In Javascript

## Open Cryptochat - A Tutorial

Cryptography is important.  Without encryption, the internet as we know it would not be possible - data sent online would be as vulnerable to interception as a message shouted across a crowded room.  Cryptography is also a major topic in current events, increasingly playing a central role in [law enforcement investigations](https://en.wikipedia.org/wiki/FBI%E2%80%93Apple_encryption_dispute) and [government legislation](https://www.politico.com/tipsheets/morning-cybersecurity/2017/11/10/texas-shooting-could-revive-encryption-legislation-223290).

Encryption is an invaluable tool for journalists, activists, nation-states, businesses, and everyday people who need to protect their data from the ever-present threat of hackers, spies, and advertising agencies.

An understanding of how to utilize strong encryption is essential for modern software development.  We will not be delving much into the underlying math and theory of cryptography for this tutorial; instead, the focus will be on how to harness these techniques for your own applications.

![Screenshot 5](https://cdn.patricktriest.com/blog/images/posts/e2e-chat/screenshot_5.png)

In this tutorial, we will walk through the basic concepts and implementation of an end-to-end 2048-bit [RSA encrypted](https://en.wikipedia.org/wiki/RSA_(cryptosystem)) messenger. We'll be utilizing [Vue.js](https://vuejs.org/) for coordinating the frontend functionality along with a [Node.js](https://nodejs.org/en/) backend using [Socket.io](https://socket.io/) for sending messages between users.

- Live Preview - https://chat.patricktriest.com
- Github Repository - https://github.com/triestpa/Open-Cryptochat

The concepts that we are covering in this tutorial are implemented in Javascript and are mostly intended to be platform-agnostic.  We will be building a traditional browser-based web app, but you can adapt this code to work within a pre-built desktop (using [Electron](https://electronjs.org/)) or mobile ( [React Native](https://facebook.github.io/react-native/), [Ionic](https://ionicframework.com/), [Cordova](https://cordova.apache.org/)) application binary if you are concerned about browser-based application security.[^1]  Likewise, implementing similar functionality in another programming language should be relatively straightforward since most languages have reputable open-source encryption libraries available; the base syntax will change but the core concepts remain universal.

> Disclaimer - This is meant to be a primer in end-to-end encryption implementation, not a definitive guide to building the Fort Knox of browser chat applications. I've worked to provide useful information on adding cryptography to your Javascript applications, but I cannot 100% guarantee the security of the resulting app.  There's a lot that can go wrong at all stages of the process, especially at the stages not covered by this tutorial such as setting up web hosting and securing the server(s).  If you are a security expert, and you find vulnerabilities in the tutorial code, please feel free to reach out to me by email (patrick.triest@gmail.com) or in the comments section below.

## 1 - Project Setup

### 1.0 - Install Dependencies

You'll need to have [Node.js](https://nodejs.org/en/) (version 6 or higher) installed in order to run the backend for this app.

Create an empty directory for the project and add a `package.json` file with the following contents.

```json
{
  "name": "open-cryptochat",
  "version": "1.0.0",
  "node":"8.1.4",
  "license": "MIT",
  "author": "patrick.triest@gmail.com",
  "description": "End-to-end RSA-2048 encrypted chat application.",
  "main": "app.js",
  "engines": {
    "node": ">=7.6"
  },
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "4.15.3",
    "socket.io": "2.0.3"
  }
}
```
<br>

Run `npm install` on the command line to install the two Node.js dependencies.

### 1.1 - Create Node.js App

Create a file called `app.js`, and add the following contents.

```javascript
const express = require('express')

// Setup Express server
const app = express()
const http = require('http').Server(app)

// Attach Socket.io to server
const io = require('socket.io')(http)

// Serve web app directory
app.use(express.static('public'))

// INSERT SOCKET.IO CODE HERE

// Start server
const port = process.env.PORT || 3000
http.listen(port, () => {
  console.log(`Chat server listening on port ${port}.`)
})
```
<br>

This is the core server logic.  Right now, all it will do is start a server and make all of the files in the local `/public` directory accessible to web clients.

> In production, I would strongly recommend serving your frontend code separately from the Node.js app, using battle-hardened server software such [Apache](https://httpd.apache.org/) and [Nginx](https://www.nginx.com/), or hosting the website on file storage service such as [AWS S3](https://aws.amazon.com/s3/).  For this tutorial, however, using the Express static file server is the simplest way to get the app running.

### 1.2 - Add Frontend

Create a new directory called `public`.  This is where we'll put all of the frontend web app code.

##### 1.2.0 - Add HTML Template
Create a new file, `/public/index.html`, and add these contents.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Open Cryptochat</title>
    <meta name="description" content="A minimalist, open-source, end-to-end RSA-2048 encrypted chat application.">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <link href="https://fonts.googleapis.com/css?family=Montserrat:300,400" rel="stylesheet">
    <link href="https://fonts.googleapis.com/css?family=Roboto+Mono" rel="stylesheet">
    <link href="/styles.css" rel="stylesheet">
  </head>
  <body>
    <div id="vue-instance">
      <!-- Add Chat Container Here -->
      <div class="info-container full-width">
        <!-- Add Room UI Here -->
        <div class="notification-list" ref="notificationContainer">
          <h1>NOTIFICATION LOG</h1>
          <div class="notification full-width" v-for="notification in notifications">
            <div class="notification-timestamp">{{ notification.timestamp }}</div>
            <div class="notification-message">{{ notification.message }}</div>
          </div>
        </div>
        <div class="flex-fill"></div>
        <!-- Add Encryption Key UI Here -->
      </div>
      <!-- Add Bottom Bar Here -->
    </div>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/vue/2.4.1/vue.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/2.0.3/socket.io.slim.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/immutable/3.8.1/immutable.min.js"></script>
    <script src="/page.js"></script>
  </body>
</html>
```
<br>

This template sets up the baseline HTML structure and downloads the client-side JS dependencies.  It will also display a simple list of notifications once we add the client-side JS code.

##### 1.2.1 - Create Vue.js App

Add the following contents to a new file, `/public/page.js`.

```javascript
/** The core Vue instance controlling the UI */
const vm = new Vue ({
  el: '#vue-instance',
  data () {
    return {
      cryptWorker: null,
      socket: null,
      originPublicKey: null,
      destinationPublicKey: null,
      messages: [],
      notifications: [],
      currentRoom: null,
      pendingRoom: Math.floor(Math.random() * 1000),
      draft: ''
    }
  },
  created () {
    this.addNotification('Hello World')
  },
  methods: {
    /** Append a notification message in the UI */
    addNotification (message) {
      const timestamp = new Date().toLocaleTimeString()
      this.notifications.push({ message, timestamp })
    },
  }
})
```
<br>

This script will initialize the Vue.js application and will add a "Hello World" notification to the UI.

##### 1.2.2 - Add Styling

Create a new file, `/public/styles.css` and paste in the following stylesheet.

<style>
.language-css {
height: 500px;
}
</style>
```css
/* Global */
:root {
  --black: #111111;
  --light-grey: #d6d6d6;
  --highlight: yellow;
}

body {
  background: var(--black);
  color: var(--light-grey);
  font-family: 'Roboto Mono', monospace;
  height: 100vh;
  display: flex;
  padding: 0;
  margin: 0;
}

div { box-sizing: border-box; }
input, textarea, select { font-family: inherit; font-size: small; }
textarea:focus, input:focus { outline: none; }

.full-width { width: 100%; }
.green { color: green; }
.red { color: red; }
.yellow { color: yellow; }
.center-x { margin: 0 auto; }
.center-text { width: 100%; text-align: center; }

h1, h2, h3 { font-family: 'Montserrat', sans-serif; }
h1 { font-size: medium; }
h2 { font-size: small; font-weight: 300; }
h3 { font-size: x-small; font-weight: 300; }
p { font-size: x-small; }

.clearfix:after {
   visibility: hidden;
   display: block;
   height: 0;
   clear: both;
}

#vue-instance {
  display: flex;
  flex-direction: row;
  flex: 1 0 100%;
  overflow-x: hidden;
}

/** Chat Window **/
.chat-container {
  flex: 0 0 60%;
  word-wrap: break-word;
  overflow-x: hidden;
  overflow-y: scroll;
  padding: 6px;
  margin-bottom: 50px;
}

.message > p { font-size: small; }
.title-header > p {
  font-family: 'Montserrat', sans-serif;
  font-weight: 300;
}

/* Info Panel */
.info-container {
  flex: 0 0 40%;
  border-left: solid 1px var(--light-grey);
  padding: 12px;
  overflow-x: hidden;
  overflow-y: scroll;
  margin-bottom: 50px;
  position: relative;
  justify-content: space-around;
  display: flex;
  flex-direction: column;
}

.divider {
  padding-top: 1px;
  max-height: 0px;
  min-width: 200%;
  background: var(--light-grey);
  margin: 12px -12px;
  flex: 1 0;
}

.notification-list {
  display: flex;
  flex-direction: column;
  overflow: scroll;
  padding-bottom: 24px;
  flex: 1 0 40%;
}

.notification {
  font-family: 'Montserrat', sans-serif;
  font-weight: 300;
  font-size: small;
  padding: 4px 0;
  display: inline-flex;
}

.notification-timestamp {
  flex: 0 0 20%;
  padding-right: 12px;
}

.notification-message { flex: 0 0 80%; }
.notification:last-child {
  margin-bottom: 24px;
}

.keys {
  display: block;
  font-size: xx-small;
  overflow-x: hidden;
  overflow-y: scroll;
}

.keys > .divider {
  width: 75%;
  min-width: 0;
  margin: 16px auto;
}

.key { overflow: scroll; }

.room-select {
  display: flex;
  min-height: 24px;
  font-family: 'Montserrat', sans-serif;
  font-weight: 300;
}

#room-input {
    flex: 0 0 60%;
    background: none;
    border: none;
    border-bottom: 1px solid var(--light-grey);
    border-top: 1px solid var(--light-grey);
    border-left: 1px solid var(--light-grey);
    color: var(--light-grey);
    padding: 4px;
}

.yellow-button {
  flex: 0 0 30%;
  background: none;
  border: 1px solid var(--highlight);
  color: var(--highlight);
  cursor: pointer;
}

.yellow-button:hover {
  background: var(--highlight);
  color: var(--black);
}

.yellow > a { color: var(--highlight); }

.loader {
    border: 4px solid black;
    border-top: 4px solid var(--highlight);
    border-radius: 50%;
    width: 48px;
    height: 48px;
    animation: spin 2s linear infinite;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

/* Message Input Bar */
.message-input {
  background: none;
  border: none;
  color: var(--light-grey);
  width: 90%;
}

.bottom-bar {
  border-top: solid 1px var(--light-grey);
  background: var(--black);
  position: fixed;
  bottom: 0;
  left: 0;
  padding: 12px;
  height: 48px;
}

.message-list {
  margin-bottom: 40px;
}
```
<br>

We won't really be going into the CSS, but I can assure you that it's all fairly straight-forward.

For the sake of simplicity, we won't bother to add a build system to our frontend.  A build system, in my opinion, is just not really necessary for an app this simple (the total gzipped payload of the completed app is under 100kb).  You are very welcome (and encouraged, since it will allow the app to be backwards compatible with outdated browsers) to add a build system such as [Webpack](https://webpack.js.org/), [Gulp](https://gulpjs.com/), or [Rollup](https://rollupjs.org/) to the application if you decide to fork this code into your own project.

### 1.3 - Try it out

Try running `npm start` on the command-line.  You should see the command-line output `Chat server listening on port 3000.`.  Open `http://localhost:3000` in your browser, and you should see a very dark, empty web app displaying "Hello World" on the right side of the page.

![Screenshot 1](https://cdn.patricktriest.com/blog/images/posts/e2e-chat/screenshot_1.png)

## 2 - Basic Messaging
Now that the baseline project scaffolding is in place, we'll start by adding basic (unencrypted) real-time messaging.

### 2.0 - Setup Server-Side Socket Listeners
In `/app.js`, add the follow code directly below the `// INSERT SOCKET.IO CODE HERE` marker.

```javascript
/** Manage behavior of each client socket connection */
io.on('connection', (socket) => {
  console.log(`User Connected - Socket ID ${socket.id}`)

  // Store the room that the socket is connected to
  let currentRoom = 'DEFAULT'

  /** Process a room join request. */
  socket.on('JOIN', (roomName) => {
    socket.join(currentRoom)

    // Notify user of room join success
    io.to(socket.id).emit('ROOM_JOINED', currentRoom)

    // Notify room that user has joined
    socket.broadcast.to(currentRoom).emit('NEW_CONNECTION', null)
  })

  /** Broadcast a received message to the room */
  socket.on('MESSAGE', (msg) => {
    console.log(`New Message - ${msg.text}`)
    socket.broadcast.to(currentRoom).emit('MESSAGE', msg)
  })
})
```
<br>

This code-block will create a connection listener that will manage any clients who connect to the server from the front-end application.  Currently, it just adds them to a `DEFAULT` chat room, and retransmits any message that it receives to the rest of the users in the room.

### 2.1 - Setup Client-Side Socket Listeners

Within the frontend, we'll add some code to connect to the server.  Replace the `created` function in `/public/page.js` with the following.

```javascript
created () {
  // Initialize socket.io
  this.socket = io()
  this.setupSocketListeners()
},
```
<br>

Next, we'll need to add a few custom functions to manage the client-side socket connection and to send/receive messages.  Add the following to `/public/page.js` inside the `methods` block of the Vue app object.

```javascript
/** Setup Socket.io event listeners */
setupSocketListeners () {
  // Automatically join default room on connect
  this.socket.on('connect', () => {
    this.addNotification('Connected To Server.')
    this.joinRoom()
  })

  // Notify user that they have lost the socket connection
  this.socket.on('disconnect', () => this.addNotification('Lost Connection'))

  // Display message when recieved
  this.socket.on('MESSAGE', (message) => {
    this.addMessage(message)
  })
},

/** Send the current draft message */
sendMessage () {
  // Don't send message if there is nothing to send
  if (!this.draft || this.draft === '') { return }

  const message = this.draft

  // Reset the UI input draft text
  this.draft = ''

  // Instantly add message to local UI
  this.addMessage(message)

  // Emit the message
  this.socket.emit('MESSAGE', message)
},

/** Join the chatroom */
joinRoom () {
  this.socket.emit('JOIN')
},

/** Add message to UI */
addMessage (message) {
  this.messages.push(message)
},
```
<br>

### 2.2 - Display Messages in UI

Finally, we'll need to provide a UI to send and display messages.

In order to display all messages in the current chat, add the following to `/public/index.html` after the `<!-- Add Chat Container Here -->` comment.

```html
<div class="chat-container full-width" ref="chatContainer">
  <div class="message-list">
    <div class="message full-width" v-for="message in messages">
      <p>
      > {{ message }}
      </p>
    </div>
  </div>
</div>
```
<br>

To add a text input bar for the user to write messages in, add the following to `/public/index.html`, after the `<!-- Add Bottom Bar Here -->` comment.

```html
<div class="bottom-bar full-width">
  > <input class="message-input" type="text" placeholder="Message" v-model="draft" @keyup.enter="sendMessage()">
</div>
```
<br>

Now, restart the server and open `http://localhost:3000` in two separate tabs/windows.  Try sending messages back and forth between the tabs.  In the command-line, you should be able to see a server log of messages being sent.

![Screenshot 2](https://cdn.patricktriest.com/blog/images/posts/e2e-chat/screenshot_2.png)
![Screenshot 3](https://cdn.patricktriest.com/blog/images/posts/e2e-chat/screenshot_3.png)

## Encryption 101

Cool, now we have a real-time messaging application.  Before adding end-to-end encryption, it's important to have a basic understanding of how asymmetric encryption works.

####  Symetric Encryption & One Way Functions

Let's say we're trading secret numbers.  We're sending the numbers through a third party, but we don't want the third party to know which number we are exchanging.

In order to accomplish this, we'll exchange a shared secret first - let's use `7`.

To encrypt the message, we'll first multiply our shared secret (`7`) by a random number `n`, and add a value `x` to the result.  In this equation, `x` represents the number that we want to send and `y` represents the encrypted result.

`(7 * n) + x = y`

We can then use modular arithmetic in order to transform an encrypted input into the decrypted output.

`y mod 7 = x`

Here, `y` as the exposed (encrypted) message and `x` is the original unencrypted message.

If one of us wants to exchange the number `2`, we could compute `(7*4) + 2` and send `30` as a message.  We both know the secret key (`7`), so we'll both be able to calculate `30 mod 7` and determine that `2` was the original number.

The original number (`2`), is effectively hidden from anyone listening in the middle since the only message passed between us was `30`.  If a third party is able to retrieve both the unencrypted result (`30`) and the encrypted value (`2`), they would still not know the value of the secret key.  In this example, `30 mod 14` and `30 mod 28` are also equal to `2`, so an interceptor could not know for certain whether the secret key is `7`, `14`, or `28`, and therefore could not dependably decipher the next message.

Modulo is thus considered a "one-way" function since it cannot be trivially reversed.

Modern encryption algorithms are, to vastly simplify and generalize, very complex applications of this general principle.  Through the use of large prime numbers, modular exponentiation, long private keys, and multiple rounds of cipher transformations, these algorithms generally take a very inconvenient amount a time (1+ million years) to crack.

> Quantum computers could, theoretically, crack these ciphers more quickly.  You can read more about this [here](https://www.infoworld.com/article/3040991/security/mits-new-5-atom-quantum-computer-could-make-todays-encryption-obsolete.html).  This technology is still in its infancy, so we probably don't need to worry about encrypted data being compromised in this manner just yet.

The above example assumes that both parties were able to exchange a secret (in this case `7`) ahead of time.  This is called *symmetric encryption* since the same secret key is used for both encrypting and decrypting the message.  On the internet, however, this is often not a viable option - we need a way to send encrypted messages without requiring offline coordination to decide on a shared secret.  This is where asymmetric encryption comes into play.

#### Public Key Cryptography

In contrast to symmetric encryption, public key cryptography (asymmetric encryption) uses pairs of keys (one public, one private) instead of a single shared secret - *public keys* are for encrypting data, and *private keys* are for decrypting data.

A *public key* is like an open box with an unbreakable lock.  If someone wants to send you a message, they can place that message in your public box, and close the lid to lock it.  The message can now be sent, to be delivered by an untrusted party without needing to worry about the contents being exposed.  Once I receive the box, I'll unlock it with my *private key* - the only existing key which can open that box.

Exchanging *public keys* is like exchanging those boxes - each private key is kept safe with the original owner, so the contents of the box are safe in transit.

This is, of course, a bare-bones simplification of how public key cryptography works.  If you're curious to learn more (especially regarding the history and mathematical basis for these techniques) I would strongly recommend starting with these two videos.

<iframe width="560" height="315" src="https://www.youtube.com/embed/YEBfamv-_do" frameborder="0" gesture="media" allow="encrypted-media" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://www.youtube.com/embed/wXB-V_Keiu8" frameborder="0" gesture="media" allow="encrypted-media" allowfullscreen></iframe>

## 3 - Crypto Web Worker

Cryptographic operations tend to be computationally intensive.  Since Javascript is single-threaded, doing these operations on the main UI thread will cause the browser to freeze for a few seconds.

> Wrapping the operations in a promise will not help, since promises are for managing asynchronous operations on a single-thread, and do not provide any performance benefit for computationally intensive tasks.

In order to keep the application performant, we will use a [Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers) to perform cryptographic computations on a separate browser thread.  We'll be using [JSEncrypt](https://github.com/travist/jsencrypt), a reputable Javascript RSA implementation originating from Stanford.  Using JSEncrypt, we'll create a few helper functions for encryption, decryption, and key pair generation.

### 3.0 - Create Web Worker To Wrap the JSencrypt Methods

Add a new file called `crypto-worker.js` in the `public` directory.  This file will store our web worker code in order to perform encryption operations on a separate browser thread.

```javascript
self.window = self // This is required for the jsencrypt library to work within the web worker

// Import the jsencrypt library
self.importScripts('https://cdnjs.cloudflare.com/ajax/libs/jsencrypt/2.3.1/jsencrypt.min.js');

let crypt = null
let privateKey = null

/** Webworker onmessage listener */
onmessage = function(e) {
  const [ messageType, messageId, text, key ] = e.data
  let result
  switch (messageType) {
    case 'generate-keys':
      result = generateKeypair()
      break
    case 'encrypt':
      result = encrypt(text, key)
      break
    case 'decrypt':
      result = decrypt(text)
      break
  }

  // Return result to the UI thread
  postMessage([ messageId, result ])
}

/** Generate and store keypair */
function generateKeypair () {
  crypt = new JSEncrypt({default_key_size: 2056})
  privateKey = crypt.getPrivateKey()

  // Only return the public key, keep the private key hidden
  return crypt.getPublicKey()
}

/** Encrypt the provided string with the destination public key */
function encrypt (content, publicKey) {
  crypt.setKey(publicKey)
  return crypt.encrypt(content)
}

/** Decrypt the provided string with the local private key */
function decrypt (content) {
  crypt.setKey(privateKey)
  return crypt.decrypt(content)
}
```
<br>

This web worker will receive messages from the UI thread in the `onmessage` listener, perform the requested operation, and post the result back to the UI thread.  The private encryption key is never directly exposed to the UI thread, which helps to mitigate the potential for key theft from a [cross-site scripting (XSS) attack](https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)).

### 3.1 - Configure Vue App To Communicate with Web Worker

Next, we'll configure the UI controller to communicate with the web worker.  Sequential call/response communications using event listeners can be painful to synchronize.  To simplify this, we'll create a utility function that wraps the entire communication lifecycle in a promise.  Add the following code to the `methods` block in `/public/page.js`.

```javascript
/** Post a message to the web worker and return a promise that will resolve with the response.  */
getWebWorkerResponse (messageType, messagePayload) {
  return new Promise((resolve, reject) => {
    // Generate a random message id to identify the corresponding event callback
    const messageId = Math.floor(Math.random() * 100000)

    // Post the message to the webworker
    this.cryptWorker.postMessage([messageType, messageId].concat(messagePayload))

    // Create a handler for the webworker message event
    const handler = function (e) {
      // Only handle messages with the matching message id
      if (e.data[0] === messageId) {
        // Remove the event listener once the listener has been called.
        e.currentTarget.removeEventListener(e.type, handler)

        // Resolve the promise with the message payload.
        resolve(e.data[1])
      }
    }

    // Assign the handler to the webworker 'message' event.
    this.cryptWorker.addEventListener('message', handler)
  })
}
```
<br>

This code will allow us to trigger an operation on the web worker thread and receive the result in a promise.  This can be a very useful helper function in any project that outsources call/response processing to web workers.

## 4 - Key Exchange

In our app, the first step will be generating a public-private key pair for each user.  Then, once the users are in the same chat, we will exchange *public keys* so that each user can encrypt messages which only the other user can decrypt.  Hence, we will always encrypt messages using the recipient's *public key*, and we will always decrypt messages using the recipient's *private key*.

### 4.0 - Add Server-Side Socket Listener To Transmit Public Keys

On the server-side, we'll need a new socket listener that will receive a public-key from a client and re-broadcast this key to the rest of the room.  We'll also add a listener to let clients know when someone has disconnected from the current room.

Add the following listeners to `/app.js` within the `io.on('connection', (socket) => { ... }` callback.

```javascript
/** Broadcast a new publickey to the room */
socket.on('PUBLIC_KEY', (key) => {
  socket.broadcast.to(currentRoom).emit('PUBLIC_KEY', key)
})

/** Broadcast a disconnection notification to the room */
socket.on('disconnect', () => {
  socket.broadcast.to(currentRoom).emit('USER_DISCONNECTED', null)
})
```
<br>

### 4.1 - Generate Key Pair In Vue App

Next, we'll replace the `created` function in `/public/page.js` to initialize the web worker and generate a new key pair.

```javascript
async created () {
  this.addNotification('Welcome! Generating a new keypair now.')

  // Initialize crypto webworker thread
  this.cryptWorker = new Worker('crypto-worker.js')

  // Generate keypair and join default room
  this.originPublicKey = await this.getWebWorkerResponse('generate-keys')
  this.addNotification('Keypair Generated')

  // Initialize socketio
  this.socket = io()
  this.setupSocketListeners()
},
```
<br>

We are using the [async/await syntax](https://blog.patricktriest.com/what-is-async-await-why-should-you-care/) to receive the web worker promise result with a single line of code.

### 4.2 - Add Public Key Helper Functions

We'll also add a few new functions to `/public/page.js` for sending the public key, and to trim down the key to a human-readable identifier.

```javascript
/** Emit the public key to all users in the chatroom */
sendPublicKey () {
  if (this.originPublicKey) {
    this.socket.emit('PUBLIC_KEY', this.originPublicKey)
  }
},

/** Get key snippet for display purposes */
getKeySnippet (key) {
  return key.slice(400, 416)
},
```
<br>

### 4.3 - Send and Receive Public Key

Next, we'll add some listeners to the client-side socket code, in order to send the local public key whenever a new user joins the room, and to save the public key sent by the other user.

Add the following to `/public/page.js` within the `setupSocketListeners` function.

```javascript
// When a user joins the current room, send them your public key
this.socket.on('NEW_CONNECTION', () => {
  this.addNotification('Another user joined the room.')
  this.sendPublicKey()
})

// Broadcast public key when a new room is joined
this.socket.on('ROOM_JOINED', (newRoom) => {
  this.currentRoom = newRoom
  this.addNotification(`Joined Room - ${this.currentRoom}`)
  this.sendPublicKey()
})

// Save public key when received
this.socket.on('PUBLIC_KEY', (key) => {
  this.addNotification(`Public Key Received - ${this.getKeySnippet(key)}`)
  this.destinationPublicKey = key
})

// Clear destination public key if other user leaves room
this.socket.on('user disconnected', () => {
  this.notify(`User Disconnected - ${this.getKeySnippet(this.destinationKey)}`)
  this.destinationPublicKey = null
})
```
<br>

### 4.4 - Show Public Keys In UI

Finally, we'll add some HTML to display the two public keys.

Add the following to `/public/index.html`, directly below the `<!-- Add Encryption Key UI Here -->` comment.

```html
<div class="divider"></div>
<div class="keys full-width">
  <h1>KEYS</h1>
  <h2>THEIR PUBLIC KEY</h2>
  <div class="key red" v-if="destinationPublicKey">
    <h3>TRUNCATED IDENTIFIER - {{ getKeySnippet(destinationPublicKey) }}</h3>
    <p>{{ destinationPublicKey }}</p>
  </div>
  <h2 v-else>Waiting for second user to join room...</h2>
  <div class="divider"></div>
  <h2>YOUR PUBLIC KEY</h2>
  <div class="key green" v-if="originPublicKey">
    <h3>TRUNCATED IDENTIFIER - {{ getKeySnippet(originPublicKey) }}</h3>
    <p>{{ originPublicKey }}</p>
  </div>
  <div class="keypair-loader full-width" v-else>
    <div class="center-x loader"></div>
    <h2 class="center-text">Generating Keypair...</h2>
  </div>
</div>
```
<br>

Try restarting the app and reloading `http://localhost:3000`.  You should be able to simulate a successful key exchange by opening two browser tabs.

![Screenshot 4](https://cdn.patricktriest.com/blog/images/posts/e2e-chat/screenshot_4.png)

> Having more than two pages with web app running will break the key-exchange.  We'll fix this further down.

## 5 - Message Encryption

Now that the key-exchange is complete, encrypting and decrypting messages within the web app is rather straight-forward.

### 5.0 - Encrypt Message Before Sending

Replace the `sendMessage` function in `/public/page.js` with the following.

```javascript
/** Encrypt and emit the current draft message */
async sendMessage () {
  // Don't send message if there is nothing to send
  if (!this.draft || this.draft === '') { return }

  // Use immutable.js to avoid unintended side-effects.
  let message = Immutable.Map({
    text: this.draft,
    recipient: this.destinationPublicKey,
    sender: this.originPublicKey
  })

  // Reset the UI input draft text
  this.draft = ''

  // Instantly add (unencrypted) message to local UI
  this.addMessage(message.toObject())

  if (this.destinationPublicKey) {
    // Encrypt message with the public key of the other user
    const encryptedText = await this.getWebWorkerResponse(
      'encrypt', [ message.get('text'), this.destinationPublicKey ])
    const encryptedMsg = message.set('text', encryptedText)

    // Emit the encrypted message
    this.socket.emit('MESSAGE', encryptedMsg.toObject())
  }
},
```
<br>

### 5.1 - Receive and Decrypt Message

Modify the client-side `message` listener in `/public/page.js` to decrypt the message once it is received.

```javascript
// Decrypt and display message when received
this.socket.on('MESSAGE', async (message) => {
  // Only decrypt messages that were encrypted with the user's public key
  if (message.recipient === this.originPublicKey) {
    // Decrypt the message text in the webworker thread
    message.text = await this.getWebWorkerResponse('decrypt', message.text)

    // Instantly add (unencrypted) message to local UI
    this.addMessage(message)
  }
})
```
<br>

### 5.2 - Display Message List

Modify the message list UI in `/public/index.html` (inside the `chat-container`) to display the decrypted message and the abbreviated public key of the sender.

```html
<div class="message full-width" v-for="message in messages">
  <p>
    <span v-bind:class="(message.sender == originPublicKey) ? 'green' : 'red'">{{ getKeySnippet(message.sender) }}</span>
    > {{ message.text }}
  </p>
</div>
```
<br>

### 5.3 - Try It Out

Try restarting the server and reloading the page at `http://localhost:3000`.  The UI should look mostly unchanged from how it was before, besides displaying the public key snippet of whoever sent each message.

![Screenshot 5](https://cdn.patricktriest.com/blog/images/posts/e2e-chat/screenshot_5.png)
![Screenshot 6](https://cdn.patricktriest.com/blog/images/posts/e2e-chat/screenshot_6.png)

In command-line output, the messages are no longer readable - they now display as garbled encrypted text.

## 6 - Chatrooms

You may have noticed a massive flaw in the current app - if we open a third tab running the web app then the encryption system breaks.  Asymmetric-encryption is designed to work in one-to-one scenarios; there's no way to encrypt the message *once* and have it be decryptable by *two* separate users.

This leaves us with two options -

1. Encrypt and send a separate copy of the message to each user, if there is more than one.
1. Restrict each chat room to only allow two users at a time.

Since this tutorial is already quite long, we'll be going with second, simpler option.

### 6.0 - Server-side Room Join Logic

In order to enforce this new 2-user limit, we'll modify the server-side socket `JOIN` listener in `/app.js`, at the top of socket connection listener block.

```javascript
// Store the room that the socket is connected to
// If you need to scale the app horizontally, you'll need to store this variable in a persistent store such as Redis.
// For more info, see here: https://github.com/socketio/socket.io-redis
let currentRoom = null

/** Process a room join request. */
socket.on('JOIN', (roomName) => {
  // Get chatroom info
  let room = io.sockets.adapter.rooms[roomName]

  // Reject join request if room already has more than 1 connection
  if (room && room.length > 1) {
    // Notify user that their join request was rejected
    io.to(socket.id).emit('ROOM_FULL', null)

    // Notify room that someone tried to join
    socket.broadcast.to(roomName).emit('INTRUSION_ATTEMPT', null)
  } else {
    // Leave current room
    socket.leave(currentRoom)

    // Notify room that user has left
    socket.broadcast.to(currentRoom).emit('USER_DISCONNECTED', null)

    // Join new room
    currentRoom = roomName
    socket.join(currentRoom)

    // Notify user of room join success
    io.to(socket.id).emit('ROOM_JOINED', currentRoom)

    // Notify room that user has joined
    socket.broadcast.to(currentRoom).emit('NEW_CONNECTION', null)
  }
})
```
<br>

This modified socket logic will prevent a user from joining any room that already has two users.

### 6.1 - Join Room From The Client Side

Next, we'll modify our client-side `joinRoom` function in `/public/page.js`, in order to reset the state of the chat when switching rooms.

```javascript
/** Join the specified chatroom */
joinRoom () {
  if (this.pendingRoom !== this.currentRoom && this.originPublicKey) {
    this.addNotification(`Connecting to Room - ${this.pendingRoom}`)

    // Reset room state variables
    this.messages = []
    this.destinationPublicKey = null

    // Emit room join request.
    this.socket.emit('JOIN', this.pendingRoom)
  }
},
```
<br>

### 6.2 - Add Notifications

Let's create two more client-side socket listeners (within the `setupSocketListeners` function in `/public/page.js`), to notify us whenever a join request is rejected.

```javascript
// Notify user that the room they are attempting to join is full
this.socket.on('ROOM_FULL', () => {
  this.addNotification(`Cannot join ${this.pendingRoom}, room is full`)

  // Join a random room as a fallback
  this.pendingRoom = Math.floor(Math.random() * 1000)
  this.joinRoom()
})

// Notify room that someone attempted to join
this.socket.on('INTRUSION_ATTEMPT', () => {
  this.addNotification('A third user attempted to join the room.')
})
```
<br>

### 6.3 - Add Room Join UI

Finally, we'll add some HTML to provide an interface for the user to join a room of their choosing.

Add the following to `/public/index.html` below the `<!-- Add Room UI Here -->` comment.

```html
<h1>CHATROOM</h1>
<div class="room-select">
  <input type="text" class="full-width" placeholder="Room Name" id="room-input" v-model="pendingRoom" @keyup.enter="joinRoom()">
  <input class="yellow-button full-width" type="submit" v-on:click="joinRoom()" value="JOIN">
</div>
<div class="divider"></div>
```
<br>

### 6.4 - Add Autoscroll

An annoying bug remaining in the app is that the notification and chat lists do not yet auto-scroll to display new messages.

In `/public/page.js`, add the following function to the `methods` block.

```javascript
/** Autoscoll DOM element to bottom */
autoscroll (element) {
  if (element) { element.scrollTop = element.scrollHeight }
},
```
<br>

To auto-scroll the notification and message lists, we'll call `autoscroll` at the end of their respective `add` methods.

```javascript
/** Add message to UI and scroll the view to display the new message. */
addMessage (message) {
  this.messages.push(message)
  this.autoscroll(this.$refs.chatContainer)
},

/** Append a notification message in the UI */
addNotification (message) {
  const timestamp = new Date().toLocaleTimeString()
  this.notifications.push({ message, timestamp })
  this.autoscroll(this.$refs.notificationContainer)
},
```
<br>

### 6.5 - Try it out

That was the last step!  Try restarting the node app and reloading the page at `localhost:3000`.  You should now be able to freely switch between rooms, and any attempt to join the same room from a third browser tab will be rejected.

![Screenshot 7](https://cdn.patricktriest.com/blog/images/posts/e2e-chat/screenshot_7.png)

## 7 - What next?

Congrats! You have just built a completely functional end-to-end encrypted messaging app.

Github Repository - https://github.com/triestpa/Open-Cryptochat
Live Preview - https://chat.patricktriest.com

Using this baseline source code you could deploy a private messaging app on your own servers.  In order to coordinate which room to meet in, one slick option could be using a time-based pseudo-random number generator (such as [Google Authenticator](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2&hl=en)), with a shared seed between you and a second party (I've got a Javascript "Google Authenticator" clone tutorial in the works - stay tuned).

### Further Improvements

There are lots of ways to build up the app from here:

- Group chats, by storing multiple public keys, and encrypting the message for each user individually.
- Multimedia messages, by encrypting a byte-array containing the media file.
- Import and export key pairs as local files.
- Sign messages with the private key for sender identity verification.  This is a trade-off because it increases the difficulty of fabricating messages, but also undermines the goal of "deniable authentication" as outlined in the [OTR messaging standard](https://en.wikipedia.org/wiki/Off-the-Record_Messaging).
- Experiment with different encryption systems such as:
  - [**AES**](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) - Symmetric encryption, with a shared secret between the users.  This is the only publicly available algorithm that is in use by the NSA and US Military.
  - [**ElGamal**](https://en.wikipedia.org/wiki/ElGamal_encryption) - Similar to RSA, but with smaller cyphertexts, faster decryption, and slower encryption.  This is the core algorithm that is used in [PGP](https://en.wikipedia.org/wiki/Pretty_Good_Privacy).
  - Implement a [**Diffie-Helman**](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) key exchange.  This is a technique of using asymmetric encryption (such as ElGamal) to exchange a shared secret, such as a symmetric encryption key (for AES).  Building this on top of our existing project and exchanging a new shared secret before each message is a good way to improve the security of the app (see [Perfect Forward Security](https://en.wikipedia.org/wiki/Forward_secrecy)).
- Build an app for virtually any use-case where intermediate servers should never have unencrypted access to the transmitted data, such as password-managers and P2P (peer-to-peer) networks.
- Refactor the app for [React Native](https://facebook.github.io/react-native/), [Ionic](https://ionicframework.com/), [Cordova](https://cordova.apache.org/), or [Electron](https://electronjs.org/) in order to provide a secure pre-built application bundle for mobile and/or desktop environments.

Feel free to comment below with questions, responses, and/or feedback on the tutorial.

[^1]:**Security Implications Of Browser Based Encryption**<br><br>Please remember to be careful. The use of these protocols in a browser-based Javascript app is a great way to experiment and understand how they work in practice, but this app is not a suitable replacement for established, peer-reviewed encryption protocol implementations such as [OpenSSL](https://en.wikipedia.org/wiki/OpenSSL) and [GnuPG](https://en.wikipedia.org/wiki/GNU_Privacy_Guard).<br><br> Client-side browser Javascript encryption is a controversial topic among security experts due to the vulnerabilities present in web application delivery versus pre-packaged software distributions that run outside the browser.  Many of these issues can be mitigated by utilizing HTTPS to prevent man-in-the-middle resource injection attacks, and by avoiding persistent storage of unencrypted sensitive data within the browser, but it is important to stay aware of potential vulnerabilities in the web platform.