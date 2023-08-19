#+HUGO_DRAFT: false
#+TITLE: AWS lambdas with shadow-cljs
#+DATE: 2023-08-19T13:19:07-07:00

A while back [I wrote about writing AWS lambdas with ClojureScript]({{< relref "/posts/aws-lambdas-with-clojurescript.md" >}}). There&rsquo;s surprisingly little information on the topic, which could mean that people are able to figure out their way around the ClojureScript ecosystem (great!) or that few people are interested in writing AWS lambdas with ClojureScript. Recently the author of shadow-cljs published a [repository on github](https://github.com/thheller/lambda-cljs) explaining the steps he would take to write an AWS lambda using shadow-cljs, prompting me to review my original approach.

This is an updated article centering specifically around the usage of [shadow-cljs](https://shadow-cljs.github.io/docs/UsersGuide.html) to achieve the same goals I set for the original article:

1.  Compile ClojureScript to run in nodejs locally.
2.  Use npm dependencies.
3.  Connect the REPL to Emacs/CIDER.
4.  Deploy and run as AWS lambda.

In the original article I chose to use the ClojureScript compiler directly; while this is a valid approach, I don&rsquo;t think it&rsquo;s the best way to write ClojureScript especially if you plan to use npm libraries.

Let&rsquo;s revisit the steps to develop AWS lambdas locally in ClojureScript, this time with the help of shadow-cljs.


<a id="orgaefdece"></a>

# Step 0: Install shadow-cljs

Shadow-cljs is both a **Clojure** library (not ClojureScript) **and** a npm package implementing a convenient command line utility. This approach is interesting and somewhat common in the Clojure ecosystem: build your tools as libraries first, as it allows them to be more easily integrated into existing workflows. This is all [explained in the shadow-cljs guide](https://shadow-cljs.github.io/docs/UsersGuide.html#_high_level_overview).

I will be using the command line tool throughout the examples here.


<a id="org25f4e71"></a>

# Step 1: Compile ClojureScript to run in nodejs

Start with a default shadow-cljs project:

```sh
npx create-cljs-project hello-lambda-cljs
```

The command above will take care of creating a ClojureScript project and installing the shadow-cljs tooling. Most importantly, the project contains a `shadow-cljs.edn` file used to manage both ClojureScript and npm dependencies, as well as configure the builds to be used by shadow-cljs.

Here&rsquo;s a full `shadow-cljs.edn` suitable for lambda development:

```clojure
;; shadow-cljs configuration
{:source-paths
 ["src"]
    
 :dependencies
 []
    
 :builds
 {:lambda
  {:target :node-library
   :output-to "out/index.js"
   :exports {:handler hello-lambda-cljs.lambda/handler}}}}
```

Understanding the `:builds` section is critical for effectively using the shadow-cljs tool, and its [user guide](https://shadow-cljs.github.io/docs/UsersGuide.html#_build_configuration) makes an excellent job explaining the various options. For brevity&rsquo;s sake I&rsquo;ll point out the following important pieces:

1.  The name of the build `:lambda` can be any keyword; the name `:lambda` doesn&rsquo;t have any special meaning for lambda creation.
2.  The target must be `:node-library` which the shadow-cljs guide refers to as &ldquo;Output code suitable for use as a node library&rdquo;.
3.  The exports section is a map of keyword to fully qualified symbols. The name `:handler` here is important, as that&rsquo;s what is expected by the AWS lambda runtime.

With the configuration above we only need a source file to compile. Place the following in `src/hello_lambda_cljs/lambda.cljs`:

```clojure
(ns hello-lambda-cljs.lambda)
    
(defn handler
  "Lambda entrypoint"
  [_event _ctx]
  (println "Executing handler")
  (js/Promise. (fn [resolve reject] (resolve #js {:hello "lambda"}))))
```

AWS lambda expects your function to return a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) object, so that&rsquo;s what our `handler` does.

To build once we can use the command line tool:  `npx shadow-cljs compile lambda` and this will generate the file `out/index.js` as specified in the configuration. In order to run the compiled file locally:

```sh
node -e 'require("./out/index").handler()'
```

It works because we told shadow-cljs to *export* our handler function. Another way to achieve the same is to use `npx run-func out/index.js handler` with the added benefit that it will resolve the returned promise, which can be useful when testing what the actual lambda invocation will return.

Being able to compile and run locally is great, but we can do much better. shadow-cljs can run a process that continuously watches and compiles the code as it changes:

```sh
npx shadow-cljs watch lambda
```

Crucially, the *watch* process also includes a nREPL server that we can connect to using CIDER.


<a id="org8a43986"></a>

# Step 2: The REPL

One of the main selling points of using Clojure(Script) is the developer experience; being able to grow your program along with your understanding of the problem at hand is, what I would consider, it&rsquo;s killer application. Fortunately shadow-cljs already includes all the machinery necessary to start a REPL and evaluate code from our editor of choice.

Running `npx shadow-cljs watch lambda` should output something like:

```sh
shadow-cljs - server version: 2.25.2 running at http://localhost:9630
shadow-cljs - nREPL server started on port 63036
shadow-cljs - watching build :lambda
```

From Emacs you can `M-x cider-connect-cljs` and select a &ldquo;shadow&rdquo; process, choose the build (lambda in our case) and it should connect to the running nREPL server, but it&rsquo;s not yet ready to evaluate ClojureScript code, as we need a suitable JS runtime process which we obtain by running `node out/index.js`.

To summarize:

1.  Run the watch process: `npx shadow-cljs watch lambda`
2.  Connect a JS runtime: `node out/index.js`
3.  Connect your nREPL client. If using Emacs: `M-x cider-connect-cljs`

Getting a ClojureScript up and running has never been easier!


<a id="org4b98f9b"></a>

# Step 3: npm dependencies

Another huge benefit of using shadow-cljs is managing npm dependencies. Depending only in ClojureScript libraries is a luxury not everyone can afford, and if you&rsquo;re writing AWS lambdas, most likely you will use other AWS services, and that means the AWS SDK for JavaScript, which is **not** available as a ClojureScript library. For ClojureScript libraries it&rsquo;s as simple as declaring them directly in the `:dependencies` vector in `shadow-cljs.edn`:

```clojure
{;; other content
    
 :dependencies [[funcool/promesa "11.0.674"]
                [funcool/httpurr "2.0.0"]
                [com.github.seancorfield/honeysql "2.4.1045"]
                [com.cognitect/transit-cljs "0.8.280"]]
    
 ;; builds, etc
}
```

But what about npm dependencies?

In shadow-cljs you manage npm dependencies as you would with any JavaScript project: by installing them using npm. Every shadow-cljs project will also contain a `package.json` file declaring npm dependencies; the `package.json` file is an artifact of the node/npm world, and shadow-cljs will manage this file for us.

Let&rsquo;s suppose our lambda needs to connect to a MySQL server. There&rsquo;s a npm package [mysqljs](https://github.com/mysqljs/mysql) that we can use from our ClojureScript project via JavaScript interop. To use it we can follow the instructions in its project page: `npm install mysqljs/mysql` and then require it in our code:

```clojure
(ns lms-simple-lambda.lambda
  (:require ["mysql" :as mysql]))
    
(def conn (.createConnection mysql #js {:host     "127.0.0.1"
                                        :user     "root"
                                        :password "root"
                                        :database "mydb"}))
(.connect conn)
(.query conn "SELECT * FROM users"
        (fn [err rows]
          ;; do something with your rows
          ;; or handle the error
          ))
```

Restarting the shadow-cljs watch process will cause it to detect the new dependency and do the necessary plumbing so that the `:require` statement above works. Unfortunately not all npm libraries are as straightforward as this one.


<a id="org21a43bc"></a>

## Detour: JavaScript dependencies in practice

Thomas Heller, the author of shadow-cljs has written about the many difficulties of handling the various types of JavaScript packages and how to support them from ClojureScript using shadow-cljs. The article [JS Dependencies: In Practice](https://code.thheller.com/blog/shadow-cljs/2017/11/10/js-dependencies-in-practice.html) goes to great detail, and I find myself referencing its examples constantly. I recommend you save, bookmark, learn, memorize or otherwise devise a way to have this information handy at all times.

A good example is the AWS SDK for JavaScript v3. The main change going from v2 to v3 is everything is modularized so you can require only what you *actually need*, reducing the final bundle size significantly. This is something we definitely want in our lambda functions.

Following the [package instructions](https://github.com/aws/aws-sdk-js-v3) we install the library:

```sh
npm install @aws-sdk/client-dynamodb
```

The instructions for JavaScript are:

```javascript
const { DynamoDBClient, ListTablesCommand } = require("@aws-sdk/client-dynamodb");
    
(async () => {
  const client = new DynamoDBClient({ region: "us-west-2" });
  const command = new ListTablesCommand({});
  try {
    const results = await client.send(command);
    console.log(results.TableNames.join("\n"));
  } catch (err) {
    console.error(err);
  }
})();
```

So `const { DynamoDBClient, ListTablesCommand } = require("@aws-sdk/client-dynamodb");` according to the [JS dependencies in practice](https://code.thheller.com/blog/shadow-cljs/2017/11/10/js-dependencies-in-practice.html) article, this style of import statements should be translated to:

```clojure
(ns lms-simple-lambda.lambda
  (:require ["mysql" :as mysql]
            ["@aws-sdk/client-dynamodb" :refer [DynamoDBClient ListTablesCommand]]))
    
(def ddb-client (DynamoDBClient. #js {:region "us-east-1"}))
(.send ddb-client (ListTablesCommand.))
```

While shadow-cljs makes it much more easy to require and use npm libraries, using those libraries means JavaScript interop. ClojureScript has great facilities for interacting its host environment, however this requires a solid understanding of the intricacies of the JavaScript ecosystem. The fact that I manage to import and use those libraries with my very limited knowledge of CommonJS, ES6 modules and the like is a testament to how useful and streamlined the shadow-cljs story is.


<a id="org5dac304"></a>

# Step 4: packaging

When you&rsquo;re done doing live coding in your connected, fast-feedback REPL environment it&rsquo;s time to package and deploy as an AWS lambda function. Here shadow-cljs can also help us by doing a release build:

```sh
npx shadow-cljs release lambda --config-merge '{:output-to "dist/index.js"}'
cp package.json package-lock.json dist
cd dist
npm install --omit=dev
rm package-lock.json
zip -r lambda.zip .
```

In the script above we override the output directory so it goes to `dist/` instead of `out/` and we&rsquo;ll use that as the root directory. The final step is to copy the node artifacts, install dependencies with npm this time in the root directory, and zip everything up in a final package ready to be deployed to AWS lambda.


<a id="org60f0560"></a>

# Extras


<a id="orgc5443a0"></a>

## clojure-lsp can&rsquo;t find npx

In my particular local configuration I use `rtx` for installing various runtime environments (like node) and `clojure-lsp` as LSP server. clojure-lsp tries to invoke `npx shadow-cljs classpath` to get the project classpath, but it can&rsquo;t find the `npx` binary. I solved this issue by creating a `.lsp/config.edn` file in my project directory with the following:

```clojure
{:project-specs [{:project-path "shadow-cljs.edn"
                  :classpath-cmd ["/full/path/to/your/rtx/installs/node/18.16.1/bin/npx" "shadow-cljs" "classpath"]}]}
```

<a id="orgad02a0a"></a>

## Default cider nREPL connection options

Cider supports setting default settings for when you connect to nREPL via a `.dir-locals.el` file:

```emacs-lisp
((nil . ((cider-default-cljs-repl . shadow)
         (cider-shadow-default-options . ":lambda"))))
```

This is not specific to shadow-cljs, but it helps us tell cider which type of ClojureScript REPL we are dealing with, as well as the name of our build so we don&rsquo;t have to select it every time we connect to the nREPL server.
