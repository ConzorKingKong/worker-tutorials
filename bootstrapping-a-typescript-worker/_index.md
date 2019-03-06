Cloudflare [Workers](https://developers.cloudflare.com/workers/about/) allows you to quickly deploy Javascript code to our 150+ data centers around the world and execute very close to your end-user. The edit/compile/debug story is already pretty amazing using the [Workers IDE](https://dash.cloudflare.com/workers) with integrated Chrome Dev Tools. However, for those hankering for some [Typescript](https://www.typescriptlang.org/) and an IDE with static analysis, autocomplete and that jazz, follow along to see one way to set up a Typescript project with [Webstorm](https://www.jetbrains.com/webstorm/) and npm run upload your code straight to the edge.

## Pre Requisites

My environment looks like this:

- macOS High Sierra
- node v8.11.3
- npm v5.6.0
- Webstorm v2018.1.3

You'll also need a [Cloudflare domain](https://support.cloudflare.com/hc/en-us/articles/201720164) and to [activate Workers](https://www.cloudflare.com/a/workers) on it.

I'll be using cryptoserviceworker.com

I'll also use Yeoman to build our initial scaffolding. Install it with `npm install yo -g`

## Getting Started

Let's start with a minimal node app with a "hello world" class and a test.

```bash
mkdir cryptoserviceworker && cd cryptoserviceworker
npm install generator-node-typescript -g
yo node-typescript
```

That generator creates the following directory structure:

```bash
drwxr-xr-x   16 steve  staff     512 Jun 18 20:40 .
drwxr-xr-x   10 steve  staff     320 Jun 18 20:35 ..
-rw-r--r--    1 steve  staff     197 Jun 18 20:40 .editorconfig
-rw-r--r--    1 steve  staff      96 Jun 18 20:40 .gitignore
-rw-r--r--    1 steve  staff     147 Jun 18 20:40 .npmignore
-rw-r--r--    1 steve  staff     267 Jun 18 20:40 .travis.yml
drwxr-xr-x    5 steve  staff     160 Jun 18 20:40 .vscode
-rw-r--r--    1 steve  staff    1066 Jun 18 20:40 LICENSE
-rw-r--r--    1 steve  staff    2071 Jun 18 20:40 README.md
drwxr-xr-x    4 steve  staff     128 Jun 18 20:40 __tests__
drwxr-xr-x  479 steve  staff   15328 Jun 18 20:40 node_modules
-rw-r--r--    1 steve  staff  244624 Jun 18 20:40 package-lock.json
-rw-r--r--    1 steve  staff    1506 Jun 18 20:40 package.json
drwxr-xr-x    4 steve  staff     128 Jun 18 20:40 src
-rw-r--r--    1 steve  staff     454 Jun 18 20:40 tsconfig.json
-rw-r--r--    1 steve  staff      73 Jun 18 20:40 tslint.json
```

It includes default settings, a task runner, an initial Typescript config and more. We won't use all of it, but it's a good starting point.

## First Test

If we take a look at the contents of `src/greeter.ts`, we'll see it's a very Typescript implementation of hello world.

```bash
$ cat greeter.ts 
export class Greeter {
  private greeting: string;

  constructor(message: string) {
    this.greeting = message;
  }

  public greet(): string {
    return `Bonjour, ${this.greeting}!`;
  }
}
```

Because Yeoman has set up our test infrastructure, we should be able exercise the code using the greeter test in `__tests__/greeter-spec.ts`

```javascript
import { Greeter } from '../src/greeter';

test('Should greet with message', () => {
  const greeter = new Greeter('friend');
  expect(greeter.greet()).toBe('Bonjour, friend!');
});
```

This generator uses jest. It's installed locally, but let's install it globally for convenience and run it!

```bash
npm install jest -g
jest
 PASS  __tests__/greeter-spec.ts
 PASS  __tests__/index-spec.ts

Test Suites: 2 passed, 2 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.867s
Ran all test suites.
```

OK, so we have a testable Typescript template. Let's fire up Webstorm and write some code!

## Hello, World with Workers

A hello world implementation in Typescript might look something like this:

![Hello world first attempt](https://blog.cloudflare.com/content/images/2018/06/typescript-hello-world.png)

Webstorm doesn't like it as you can see from the red error highlights. Even though Request and Response are part of the Service Worker API and will be available to us in the V8 runtime, Typescript doesn't know about them yet. [node-fetch](https://www.npmjs.com/package/node-fetch) provides an implementation for node, so let’s install that.

```bash
npm install node-fetch
npm install @types/node-fetch
```

That made Webstorm happier. It’s been able to locate the type definitions.

![Hello world with type defs](https://blog.cloudflare.com/content/images/2018/06/typescript-hello-world2.png)

Now let's write a test. Create a new file `tests/worker-spec.ts`:

```javascript
import { Request } from "node-fetch";
import { Worker } from "../src/worker";

test('Should say hello', () => {

  const worker = new Worker();
  const request = new Request("https://cryptoserviceworker.com/");
  const response = worker.handle(request);
  expect(response.status).toEqual(200);
  expect(response.body).toEqual("Hello, world!");
});
```

And delete the other files and tests so we're just working worker.ts and worker-spect.ts

Run `jest`

```bash
 PASS  __tests__/worker-spec.ts
 PASS  __tests__/worker-spec.js

Test Suites: 2 passed, 2 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.213s, estimated 2s
```

OK, so our test passed, but notice it ran both the Typescript and the Javascript? Let's restrict to just Typescript. Go into package.json, locate jest and change

`"testRegex": "(/__tests__/.*|\\.(test|spec))\\.(ts|js)$"`
to
`"testRegex": "(/__tests__/.*)\\-spec.ts$"`

Run it again:

```bash
jest
 PASS  __tests__/worker-spec.ts
  ✓ Should say hello (8ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        1.123s, estimated 2s
Ran all test suites.
```

Better. OK, ship it!

## From Local Typescript to Worker Compatible Javascript.

Let's take a look at `src/worker.js` to see how our Typescript transpiled.

```javascript
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
const node_fetch_1 = require("node-fetch");
class Worker {
    handle(request) {
        return new node_fetch_1.Response('Hello, world!');
    }
}
exports.Worker = Worker;
```

Actually, let's try it in the Cloudflare Workers IDE and try it for real. Go to your [dashboard](https://dash.cloudflare.com/), click the Workers icon and then "Launch Editor"

![Workers Dashboard](https://blog.cloudflare.com/content/images/2018/06/workers-dashboard.png)

First things first, check the canonical Hello World implementation works.

![Hello world in IDE](https://blog.cloudflare.com/content/images/2018/06/hello-world-ide.png)

Awesome, now let's replace it with our "transpiled from Typescript" version:

![Fail 1](https://blog.cloudflare.com/content/images/2018/06/fail1.png)

Fail. OK, so the out of the box "transpiled from typescript" is not going to work. Let's make the changes necessary to get it run manually, then incorporate that into the build process.

**Error #1: Uncaught ReferenceError: exports is not defined at line 2**
That's easy enough, let's add `var exports = {}`. Update Preview.

**Error #2: Uncaught ReferenceError: require is not defined at line 4**

True, we're running in V8 on the Cloudflare Edge and the only code is what we uploaded. There are no "node_modules" to include. Plus, that line was only for dev anyway. Remove it. Update Preview.

**Error #3: No event handlers were registered. This script does nothing.**

Right, we need to invoke the code. Let's add a snippet to the top of the file to actually invoke our worker.

```javascript
addEventListener('fetch', event => {
  let worker = new exports.Worker();
  event.respondWith(worker.handle(event.request));
})
```

**Error #4: Uncaught ReferenceError: node_fetch_1 is not defined**

Right, we removed that because [Response](https://developer.mozilla.org/en-US/docs/Web/API/Request) is a native object when it runs in the context of a worker. So remove the `node_fetch_1` prefix.

**Error #5: exports.__esModule = true does nothing**

So let's remove that.

Success!!

![Success](https://blog.cloudflare.com/content/images/2018/06/hello-world-success.png)

OK, so with some massaging, we got a Worker transpiled from Typescript to execute. We:

- Added a line to create an exports object
- Removed the dev dependency on "node_fetch"
- Removed the exports.__esModule = true line

Let's add that to our build process so we can have "Worker-ready" Javascript every time we make a change to our Typescript.

## Grunt

I'm going to use Grunt to automate that. Here's my new `worker.ts`

```javascript
// --BEGIN PREAMBLE--
/// //Invoke worker
/// var exports = {};
/// addEventListener('fetch', event => {
///   event.respondWith(fetchAndApply(event.request))
/// });
///
/// async function fetchAndApply(request) {
///   let worker = new exports.Worker();
///   return worker.handle(request);
/// }
// --END PREAMBLE--

// --BEGIN COMMENT--
// mock the methods and objects that will be available in the browser
import { Request, Response } from 'node-fetch';
// --END COMMENT--
export class Worker {
  public handle(request: Request) {
    return new Response("Hello, world!")
  }
}
```

I want to uncomment the preamble to invoke our script, comment out the dev dependencies and remove the __esmodule line. Let's install Grunt, a text-replace module and create a `Gruntfile.js`

```bash
npm install grunt-cli -g
npm install grunt --save-dev
npm install grunt-replace --save-dev
touch Gruntfile.js
```

My `Gruntfile.js` looks like this

```javascript
module.exports = function (grunt) {

  grunt.loadNpmTasks('grunt-replace');
  grunt.initConfig({
    replace: {
      comments: {
        options: {
          patterns: [
            {
              /* Comment imports for node during dev */
              match: /--BEGIN COMMENT--[\s\S]*?--END COMMENT--/g,
              replacement: 'Dev environment code block removed by build'
            },
            {
              /* Uncomment preamble for production to process the request */
              match: /\/\/\//mg,
              replacement: ''
            }
          ]
        },
        files: [
          { expand: true, flatten: true, src: ['src/worker.ts'], dest: 'build/' }
        ]
      },
      exports: {
        //remove the exports line that typescript includes without an option to
        //suppress, but is not in the v8 env that workers run in.
        options: {
          patterns: [
            {
              match: /exports.__esModule = true;/g,
              replacement: "// exports line commented by build"
            }
          ]
        },
        files: [
          { expand: true, flatten: true, src: ['build/worker.js'], dest: 'build/' }
        ]
      }
    }
  });

  grunt.registerTask('prepare-typescript', 'replace:comments');
  grunt.registerTask('fix-export', 'replace:exports');
};
```

There are two tasks. The first is the comment/uncomment step that we want before our Typescript is transpiled.

The second is to remove the `exports.__esmodule = true` line

```bash
$ grunt prepare-typescript
Running "replace:comments" (replace) task
>> 11 replacements in 1 file.

Done.
```

If we open `build/worker.ts`, we see this:

```javascript
// --BEGIN PREAMBLE--
 //Invoke worker
 var exports = {};
 addEventListener('fetch', event => {
   event.respondWith(fetchAndApply(event.request))
 });

 async function fetchAndApply(request) {
   let worker = new exports.Worker();
   return worker.handle(request);
 }
// --END PREAMBLE--

// Dev environment code block removed by build
export class Worker {
  public handle(request: Request) {
    return new Response("Hello, world!")
  }
}
```

Opening `build/worker.js` you'll see a whole lot of code generated for handling async functions. That's because we're using the `async` keyword in the preamble.

```javascript
"use strict";
var __awaiter = (this && this.__awaiter) || function (thisArg, _arguments, P, generator) {
    return new (P || (P = Promise))(function (resolve, reject) {
        function fulfilled(value) { try { step(generator.next(value)); } catch (e) { reject(e); } }
        function rejected(value) { try { step(generator["throw"](value)); } catch (e) { reject(e); } }
        function step(result) { result.done ? resolve(result.value) : new P(function (resolve) { resolve(result.value); }).then(fulfilled, rejected); }
        step((generator = generator.apply(thisArg, _arguments || [])).next());
    });
};
var __generator = (this && this.__generator) || function (thisArg, body) {
    var _ = { label: 0, sent: function() { if (t[0] & 1) throw t[1]; return t[1]; }, trys: [], ops: [] }, f, y, t, g;
    return g = { next: verb(0), "throw": verb(1), "return": verb(2) }, typeof Symbol === "function" && (g[Symbol.iterator] = function() { return this; }), g;
    function verb(n) { return function (v) { return step([n, v]); }; }
    function step(op) {
        if (f) throw new TypeError("Generator is already executing.");
        while (_) try {
            if (f = 1, y && (t = y[op[0] & 2 ? "return" : op[0] ? "throw" : "next"]) && !(t = t.call(y, op[1])).done) return t;
            if (y = 0, t) op = [0, t.value];
            switch (op[0]) {
                case 0: case 1: t = op; break;
                case 4: _.label++; return { value: op[1], done: false };
                case 5: _.label++; y = op[1]; op = [0]; continue;
                case 7: op = _.ops.pop(); _.trys.pop(); continue;
                default:
                    if (!(t = _.trys, t = t.length > 0 && t[t.length - 1]) && (op[0] === 6 || op[0] === 2)) { _ = 0; continue; }
                    if (op[0] === 3 && (!t || (op[1] > t[0] && op[1] < t[3]))) { _.label = op[1]; break; }
                    if (op[0] === 6 && _.label < t[1]) { _.label = t[1]; t = op; break; }
                    if (t && _.label < t[2]) { _.label = t[2]; _.ops.push(op); break; }
                    if (t[2]) _.ops.pop();
                    _.trys.pop(); continue;
            }
            op = body.call(thisArg, _);
        } catch (e) { op = [6, e]; y = 0; } finally { f = t = 0; }
        if (op[0] & 5) throw op[1]; return { value: op[0] ? op[1] : void 0, done: true };
    }
};
exports.__esModule = true;
// --BEGIN PREAMBLE--
//Invoke worker
var exports = {};
addEventListener('fetch', function (event) {
    event.respondWith(fetchAndApply(event.request));
});
function fetchAndApply(request) {
    return __awaiter(this, void 0, void 0, function () {
        var worker;
        return __generator(this, function (_a) {
            worker = new exports.Worker();
            return [2 /*return*/, worker.handle(request)];
        });
    });
}
// --END PREAMBLE--
// Dev environment code block removed by build
var Worker = /** @class */ (function () {
    function Worker() {
    }
    Worker.prototype.handle = function (request) {
        return new Response("Hello, world!");
    };
    return Worker;
}());
exports.Worker = Worker;
```

Now let's remove that `exports.__esModule = true` line.

`grunt fix-export`

and now we'll see instead in the worker.js `// exports line commented by build`.

## Put it together

I just want to run `npm run build` and get Worker-friendly Javascript. Let's modify `package.json` to do just that. Change

`"build": "tsc --pretty"` to `"build": "grunt prepare-typescript && tsc build/*.ts --pretty --skipLibCheck; grunt fix-export",`

And run it.

`npm run build` will result in

```javascript
"use strict";
var __awaiter = (this && this.__awaiter) || function (thisArg, _arguments, P, generator) {
    return new (P || (P = Promise))(function (resolve, reject) {
        function fulfilled(value) { try { step(generator.next(value)); } catch (e) { reject(e); } }
        function rejected(value) { try { step(generator["throw"](value)); } catch (e) { reject(e); } }
        function step(result) { result.done ? resolve(result.value) : new P(function (resolve) { resolve(result.value); }).then(fulfilled, rejected); }
        step((generator = generator.apply(thisArg, _arguments || [])).next());
    });
};
var __generator = (this && this.__generator) || function (thisArg, body) {
    var _ = { label: 0, sent: function() { if (t[0] & 1) throw t[1]; return t[1]; }, trys: [], ops: [] }, f, y, t, g;
    return g = { next: verb(0), "throw": verb(1), "return": verb(2) }, typeof Symbol === "function" && (g[Symbol.iterator] = function() { return this; }), g;
    function verb(n) { return function (v) { return step([n, v]); }; }
    function step(op) {
        if (f) throw new TypeError("Generator is already executing.");
        while (_) try {
            if (f = 1, y && (t = y[op[0] & 2 ? "return" : op[0] ? "throw" : "next"]) && !(t = t.call(y, op[1])).done) return t;
            if (y = 0, t) op = [0, t.value];
            switch (op[0]) {
                case 0: case 1: t = op; break;
                case 4: _.label++; return { value: op[1], done: false };
                case 5: _.label++; y = op[1]; op = [0]; continue;
                case 7: op = _.ops.pop(); _.trys.pop(); continue;
                default:
                    if (!(t = _.trys, t = t.length > 0 && t[t.length - 1]) && (op[0] === 6 || op[0] === 2)) { _ = 0; continue; }
                    if (op[0] === 3 && (!t || (op[1] > t[0] && op[1] < t[3]))) { _.label = op[1]; break; }
                    if (op[0] === 6 && _.label < t[1]) { _.label = t[1]; t = op; break; }
                    if (t && _.label < t[2]) { _.label = t[2]; _.ops.push(op); break; }
                    if (t[2]) _.ops.pop();
                    _.trys.pop(); continue;
            }
            op = body.call(thisArg, _);
        } catch (e) { op = [6, e]; y = 0; } finally { f = t = 0; }
        if (op[0] & 5) throw op[1]; return { value: op[0] ? op[1] : void 0, done: true };
    }
};
// exports line commented by build
// --BEGIN PREAMBLE--
//Invoke worker
var exports = {};
addEventListener('fetch', function (event) {
    event.respondWith(fetchAndApply(event.request));
});
function fetchAndApply(request) {
    return __awaiter(this, void 0, void 0, function () {
        var worker;
        return __generator(this, function (_a) {
            worker = new exports.Worker();
            return [2 /*return*/, worker.handle(request)];
        });
    });
}
// --END PREAMBLE--
// Dev environment code block removed by build
var Worker = /** @class */ (function () {
    function Worker() {
    }
    Worker.prototype.handle = function (request) {
        return new Response('Hello, world!');
    };
    return Worker;
}());
exports.Worker = Worker;
```

Paste that into the Workers IDE... works first time.

## Automated upload

It's going to get old uploading from our IDE to the Web IDE every time we want to test a change and we're going to want to auto deploy from CI at some point. Thankfully there's [Workers Configuration API](https://developers.cloudflare.com/workers/api/), which makes it very simple to upload a Worker automatically:

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/zones/:zone_id/workers/script" -H "X-Auth-Email:YOUR_CLOUDFLARE_EMAIL" -H "X-Auth-Key:ACCOUNT_AUTH_KEY" -H "Content-Type:application/javascript" --data-binary "@PATH_TO_YOUR_WORKER_SCRIPT"
```

OK, so we need our zone ID, Cloudflare email, auth key and path to the binary. I'm going to create Grunt task that uses the [dotenv](https://www.npmjs.com/package/dotenv) package to load config from a .env file or environment variables.

Create a `.env` file that looks like this:

```
CF_WORKER_ZONE_ID=xxxxxxxxxxxxxxxxxxx
CF_WORKER_EMAIL=steve@example.com
CF_WORKER_AUTH_KEY=xxxxxxxxxxxxxxxxxx
CF_WORKER_PATH=build/worker.js
```
To locate your zone ID and auth key, go to the dashboard, select your zone and click the "Overview" icon.

![Overview](https://blog.cloudflare.com/content/images/2018/06/overview.png)

The zone ID is right there, then click "Get API key" and choose the "Global API Key" to get the Auth Key.

![API key](https://blog.cloudflare.com/content/images/2018/06/api-key.png)

Fill out your .env with those values and then add the following to your Gruntfile which will:

- Read your config
- Upload to Cloudflare
- Parse any success or error messages.

```javascript
grunt.registerTask('upload-worker', 'Uploads workers to Cloudflare', function(path) {

    require('dotenv').config();
    const fs = require('fs');
    const log = console;

    const done = this.async();
    const conf = readConfig();
    path = path || grunt.option('path') || process.env.CF_WORKER_PATH;
    if (!path) {
      fail("path is required");
    }
    if (!fs.existsSync(path)) {
      fail(`path not found ${path}`);
    }

    let script = fs.readFileSync(path);
    log.info("Uploading...");
    let url = `https://api.cloudflare.com/client/v4/zones/${conf.zoneId}/workers/script`;
    let options = {
      url: url,
      method: 'PUT',
      headers: {
        'Content-Type': 'application/javascript'
      },
      body: script
    };
    invokeApi(options, conf, done);
  });

  function invokeApi(options, conf, done) {

    // Add authentication to the request
    options.headers = options.headers || {};
    Object.assign(options.headers, {
      'X-Auth-Email': conf.email,
      'X-Auth-Key': conf.apiKey,
    });

    request(options, function(error, response) {
      try {
        if (error) {
          log.error(error);
          fail(`API failure ${response.statusCode} error: ${error}`);
          done();
          return;
        }
        let body = JSON.parse(response.body);
        if (body) {
          logResult(body);
        }
        done();
      } catch (e) {
        fail(`Unhandled error. ${e}`);
        done();
      }
    });
  }

  function logResult(body) {
    body.success ? log.error("Status: Success") : log.error("Status: Failed");
    let errors = body.errors || [];
    if (errors) {
      log.info(` Errors: ${errors.length}`);
      for (let e of errors) {
        log.error(` Code: ${e.code} Message: ${e.message}`);
      }
    }
    let messages = body.messages || [];
    if (messages) {
      log.info(` Messages ${messages.length}`);
      for (let msg of messages) {
        log.info(` ${msg}`);
      }
    }
    let result = body.result;
    log.info(" Result");
    log.info(` ${JSON.stringify(result, null, 2)}`);
  }

  function readConfig() {
    let zoneId = grunt.option('zoneId') || process.env.CF_WORKER_ZONE_ID;
    let email = grunt.option('email') || process.env.CF_WORKER_EMAIL;
    let apiKey = grunt.option('apiKey') || process.env.CF_WORKER_AUTH_KEY;

    log.debug("zoneID: " + zoneId);
    log.debug("email: " + email);
    log.debug("apiKey: " + "*".repeat(apiKey.length));

    if (!zoneId || !email || !apiKey) {
      fail("zone id, cloudflare email and api key are required");
    }
    return {
      zoneId: zoneId,
      email: email,
      apiKey: apiKey
    }
  }

  function fail(message) {
    grunt.fail.fatal(message, TASK_FAILED);
  }
```

Finally, let's add a new task to `package.json` so we can just `npm run upload` any time we update our Worker.

`"upload": "grunt upload-worker"`

```bash
npm run upload-worker

Running "upload-worker" task
zoneID: **************
email: steve@example.com
apiKey: *************************************
Uploading...
Status: Success
 Errors: 0
 Messages 0
 Result
 {
  "script"...
 }
 ```

Voila! Script uploaded. OK, so if it's uploaded, we can call call it remotely:

`$ curl https://cryptoserviceworker.com/hello`

Hmmm... nothing. Ah, we haven't actually configured Workers to route any requests to our Worker. You can do this via the API, but since it's a one off, I'll do it in the web IDE.

![Add routes](https://blog.cloudflare.com/content/images/2018/06/routes.png)

And try again:

```bash
$ curl https://cryptoserviceworker.com/hello
Hello, world!
```

Success! OK, so to recap:

- We've bootstrapped a Typescript project using NodeJS and Webstorm
- Written a "Hello, World" worker in Typescript
- Setup build tasks to modify the code for Workers
- Automatically uploading to the Cloudflare edge with `npm run upload`
- ...
- Profit
