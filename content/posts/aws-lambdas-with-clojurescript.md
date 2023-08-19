+++
title = "Writing AWS lambdas with ClojureScript"
author = ["Cesar Olea"]
date = 2021-01-12T20:01:00-07:00
+++

## Deprecation notice! {#note}
This article has been superseeded by [writing AWS lambdas with shadow-cljs]({{< relref "/posts/writing-aws-lambdas-with-shadow-cljs.org" >}}). Most of the information contained here should still work, but I don't recommend following this approach as shadow-cljs presents a much better development story. Still some of the information presented here is still valid and valuable, such as the use of core.async.

## Rationale {#rationale}

When I created [the anvil template](https://github.com/cesarolea/anvil-lein-template), I did it to learn how to create a
ClojureScript project from scratch, tailoring it to my exact needs for
**frontend development**. Those needs can be summarized in three key
points:

1.  Code reloading.
2.  A browser connected REPL.
3.  Connectivity to the REPL from Emacs/CIDER.

At the time I was doing a lot of frontend ClojureScript development in
the form of animations with [Quil](http://quil.info) (see my
[raycaster
demo](https://blog.cesarolea.com/posts/raycasting-demo/index.html) and the [old-school fire effect](https://blog.cesarolea.com/posts/efecto-de-fuego-con-clojurescript/index.html)) and the pattern for the
template emerged from those projects, though it can also be used for
more "serious" Web frontend development.

Nowadays I'm not doing so much frontend development, but there's a
specific need where ClojureScript would really shine: backend
development with AWS lambda.


### Why ClojureScript and not Clojure {#why-clojurescript-and-not-clojure}

I do maintain some JVM (Clojure) based lambdas at work. The
development experience is fantastic and deploying couldn't be
easier: create an uberjar, and ship it to AWS. Couple this with the
fact that the AWS SDK for Java is one if not the best SDK out
there, and the overall experience is hard to beat.

The problem is lambda cold starts. AWS lambda will create an
execution environment for your code, download the package, start
the JVM and execute it. This whole process is in the order of
seconds, which is nothing short of amazing, but can have big
implications if some of your requests start taking 5 or 10 seconds
instead of milliseconds. In addition, lambdas are billed by the
millisecond and you pay for the cold start time, so it makes sense
from the pricing point of view to reduce cold start time.


## Goals {#goals}

My goals for creating AWS lambdas in ClojureScript are:

1.  Compile ClojureScript to run in nodejs locally.
2.  Use npm dependencies.
3.  Connect the REPL to Emacs/CIDER.
4.  Deploy and run as AWS lambda.


## Step 1: Compile ClojureScript to run in nodejs {#step-1-compile-clojurescript-to-run-in-nodejs}

Start with a default Clojure (that's right, Clojure and not
ClojureScript) project.

```sh
lein new app hello-lambda-cljs
```

It generates a project with the following structure:

<a id="orga130b84"></a>
```sh
.
├── CHANGELOG.md
├── LICENSE
├── README.md
├── doc
│   └── intro.md
├── project.clj
├── resources
├── src
│   └── hello_lambda_cljs
│       └── core.clj
└── test
    └── hello_lambda_cljs
        └── core_test.clj
```

We'll remove what's not needed:

```sh
cd hello-lambda-cljs
rm -r test resources doc README.md LICENSE CHANGELOG.md src/hello_lambda_cljs/core.clj
touch src/hello_lambda_cljs/core.cljs
```

<a id="orgeeff84b"></a>
```sh
.
├── project.clj
└── src
    └── hello_lambda_cljs
        └── core.cljs
```

We'll use the trusty old `cljsbuild` leiningen plugin. I like this
plugin as it's basically a no-frills gateway into the ClojureScript
compiler.

Here's how we set our `project.clj` to compile our ClojureScript code
for node execution:

```clojure
(defproject hello-lambda-cljs "1.0.0"
  :dependencies [[org.clojure/clojure "1.10.1"]
                 [org.clojure/clojurescript "1.10.758"]]
  :plugins [[lein-cljsbuild "1.1.8"]]
  :cljsbuild
  {:builds [{:source-paths ["src"]
             :compiler {:output-to "hello_lambda_cljs.js"
                        :main hello-lambda-cljs.core
                        :target :nodejs
                        :optimizations :simple}}]})
```

The most important part of this `project.clj` is the `:compiler`
map. This map is passed to the ClojureScript compiler directly, so
it's worth [getting familiar with all the options at your disposal](https://clojurescript.org/reference/compiler-options).

Almost done. Before compiling we need something to compile. Edit
`src/hello_lambda_cljs/core.cljs` with the following code:

```clojure
(ns hello-lambda-cljs.core)

(defn say-hello []
  "Hello World!")

(println (say-hello))
```

The purpose of this very simple ClojureScript program is to verify
that we can compile our project and run the compiled output with
nodejs.

To build: `lein cljsbuild once`. There should be a
`hello_lambda_cljs.js` file in the root of the project. This program
is runnable locally by nodejs but it can't be executed by AWS lambda
yet. We'll get to that in [Step 4](#step-4-deploy-and-run-as-aws-lambda).

```sh
node hello_lambda_cljs.js
Hello guys!
```


## Step 2: Use npm dependencies {#step-2-use-npm-dependencies}

In writing backend JavaScript (ClojureScript in our case) eventually
you'll need to use a JavaScript (not ClojureScript) library, and this
means interacting with npm. This is especially true for AWS lambdas as
more often than not you'll use the AWS SDK for JavaScript to consume
other AWS services from your lambda function.

ClojureScript makes it very easy to use npm libraries. Simply declare
them as npm dependencies in your ClojureScript build definition. In
`project.clj`:

```clojure
(defproject hello-lambda-cljs "1.0.0"
  :dependencies [[org.clojure/clojure "1.10.1"]
                 [org.clojure/clojurescript "1.10.758"]]
  :plugins [[lein-cljsbuild "1.1.8"]]
  :cljsbuild
  {:builds [{:source-paths ["src"]
             :compiler {:output-to "hello_lambda_cljs.js"
                        :main hello-lambda-cljs.core
                        :target :nodejs
                        :optimizations :simple
                        :npm-deps {:luxon "1.25.0"}
                        :install-deps true}}]})
```

Note how two keys were added: `:npm-deps` and
`:install-deps`. Compiling this code now will fetch the `luxon`
library (used as an example) from npm and allow our code to require
it, just as any other ClojureScript library:

```clojure
(ns hello-lambda-cljs.core
  (:require [luxon :refer [DateTime]]))

(defn today-as-string []
  (-> DateTime .local .toString))

(println (today-as-string))
```

Upon compilation, the ClojureScript compiler will download the
dependencies from npm and compile them in such a way that they can be
required by your compiled program. A `node_modules` directory will be
placed in your project root along with familiar npm artifacts
`package.json` and `package-lock.json`.

Note that if you don't `:require` any npm libraries in ClojureScript
code, then the dependencies won't be fetched from npm even if they are
declared as dependencies to the ClojureScript compiler.

Running the newly compiled file:

```sh
node hello_lambda_cljs.js
2021-01-11T10:25:06.099-07:00
```

What about more complex libraries? It's the same. First declare them
in `project.clj`:

```clojure
(defproject hello-lambda-cljs "1.0.0"
  :dependencies [[org.clojure/clojure "1.10.1"]
                 [org.clojure/clojurescript "1.10.758"]]
  :plugins [[lein-cljsbuild "1.1.8"]]
  :cljsbuild
  {:builds [{:source-paths ["src"]
             :compiler {:output-to "hello_lambda_cljs.js"
                        :main hello-lambda-cljs.core
                        :target :nodejs
                        :optimizations :simple
                        :npm-deps {:luxon "1.25.0"
                                   :aws-sdk "2.824.0"}
                        :install-deps true}}]})
```

And require them in code as you usually would with Clojure(Script)
libraries.

```clojure
(ns hello-lambda-cljs.core
  (:require [luxon :refer [DateTime]]
            [aws-sdk :as aws]
            [cljs.pprint :refer [pprint]]))

;; set AWS credentials from profile
(set! (.-credentials aws/config)
      (aws/SharedIniFileCredentials. #js {:profile "test"}))
(def s3 (aws/S3.))

(defn list-buckets []
  (println "Requesting your buckets...")
  (.listBuckets s3 (fn [err data]
                     (if err
                       (println "ERROR: " err)
                       (pprint (js->clj data))))))

(defn today-as-string []
  (-> DateTime .local .toString))

(println (today-as-string))

(list-buckets)
```

A few pointers before running the code. First, a `test` AWS CLI
profile needs to exists in your local computer. Hardcoded here for
illustrative purposes only, but will have to be replaced with a
flexible solution before deploying as AWS lambda.

Second, your `test` profile needs to have permissions to read your S3
buckets. An IAM tutorial is out of the scope of this article, so it is
left as an exercise to the reader.

Third, we are still in "callback hell". This is in part because we are
using the JavaScript AWS SDK library directly. This is something that
can be fixed by using `core.async`. See [Extras](#extras).

Compile and run locally as before, you should get a listing of all
your S3 buckets in a Clojure map:

```sh
{"Buckets"
 [{"Name"
   "some-bucket-1",
   "CreationDate" #inst "2020-08-13T19:22:22.000-00:00"}
  {"Name" "some-other-bucket",
   "CreationDate" #inst "2020-08-21T17:12:16.000-00:00"}
  {"Name"
   "yet-another-bucket",
   "CreationDate" #inst "2020-10-18T16:18:52.000-00:00"}
  {"Name" "and-some-more-buckets",
   "CreationDate" #inst "2020-07-07T15:09:09.000-00:00"}],
 "Owner"
 {"DisplayName" "your-aws-account",
  "ID"
  "some-random-id"}}
```


## Step 3: The REPL {#step-3-the-repl}

While the main objective is to run our ClojureScript code (transpiled
to JavaScript code) in AWS as a lambda function, equally important is
the development experience. Without the REPL this experience would be
significantly hampered.

There are multiple ways to achieve it. CIDER in Emacs supports a
nodejs REPL. The process is very simple, you `M-x cider-jack-in-cljs`,
select node as REPL and it starts node as a subprocess. But I prefer
running the server in a separate terminal, and let CIDER connect to
it.

For the REPL we'll use [shadow-cljs](https://github.com/thheller/shadow-cljs). Shadow-cljs is much more than just
a REPL, but we won't be using any of its other capabilities here.

To install shadow-cljs, in your project root `npm install -D
shadow-cljs`. It will add shadow-cljs as a development
dependency. Next we need to add a configuration file for shadow-cljs
`shadow-cljs.edn` with content:

```clojure
{:dependencies [[cider/cider-nrepl "0.25.6"]]}
```

Your project tree should look like this:

```sh
.
├── project.clj
├── shadow-cljs.clj
└── src
    └── hello_lambda_cljs
        └── core.cljs
```

Running the REPL is as easy as executing `npx shadow-cljs
node-repl`. After a while it will respond with:

```sh
shadow-cljs - server version: 2.11.13 running at http://localhost:9630
shadow-cljs - nREPL server started on port 39049
cljs.user=> shadow-cljs - #4 ready!
```

To connect to this REPL from Emacs/CIDER `M-x cider-connect-cljs` and
follow the prompts. It will ask for:

-   Host: `localhost`
-   Port: In this case `39049`
-   Type of REPL: `shadow`
-   Shadow build: `node-repl`

It should connect and be able to use it just as you would with a
Clojure REPL.

{{<figure src="/ox-hugo/cider-clojurescript-repl.png">}}


## Step 4: Deploy and run as AWS lambda {#step-4-deploy-and-run-as-aws-lambda}

Before deploying as AWS lambda there's two things we need to fix:

1.  The profile credentials need to be set only if running
    locally. When running as lambda it should pick its credentials from
    the role assigned to it.
2.  There's no AWS lambda entry point.

We can use the presence of the `AWS_PROFILE` environment variable as a
flag to either set the credentials ourselves, or let the SDK take the
lambda role.

<a id="org3104e37"></a>
```clojure
(ns hello-lambda-cljs.core
  (:require [aws-sdk :as aws]
            [cljs.pprint :refer [pprint]]))

;; set AWS credentials from profile
(when (-> js/process .-env .-AWS_PROFILE)
  (set! (.-credentials aws/config)
        (aws/SharedIniFileCredentials.
         #js {:profile (-> js/process .-env .-AWS_PROFILE)})))
(def s3 (aws/S3.))

(defn list-buckets []
  (println "Requesting your buckets...")
  (.listBuckets s3 (fn [err data]
                     (if err
                       (println "ERROR: " err)
                       (pprint (js->clj data))))))

(list-buckets)
```

With the code above, if the environment variable `AWS_PROFILE` is not
set, then the SDK will follow it's own authentication chain. When
running locally we will set the `AWS_PROFILE` to our IAM profile and
when running in a lambda we will simply not set it, allowing the role
assigned to the lambda to take over. This fixes issue #1.

Issue #2 requires us to specify the lambda entry point, its
**handler**. The handler is the function that the AWS lambda runtime
will execute and needs to have a specific signature.

```clojure
(ns hello-lambda-cljs.core
  (:require [aws-sdk :as aws]
            [cljs.pprint :refer [pprint]]))

;; set AWS credentials from profile
(when (-> js/process .-env .-AWS_PROFILE)
  (set! (.-credentials aws/config)
        (aws/SharedIniFileCredentials.
         #js {:profile (-> js/process .-env .-AWS_PROFILE)})))
(def s3 (aws/S3.))

(defn list-buckets []
  (println "Requesting your buckets...")
  (.listBuckets s3 (fn [err data]
                     (if err
                       (println "ERROR: " err)
                       (pprint (js->clj data))))))

(list-buckets)

(defn handler
  "Lambda main entry point"
  [event context callback]
  (do
    (pprint event)
    (callback nil
              (clj->js {:status 200
                        :body "Hello from AWS Lambda in ClojureScript!"
                        :headers {}}))))

(set! (.-exports js/module) #js {:handler handler})
```

The relevant code is at the bottom. First we create a new function
`handler` with [the 3 arg signature specified by AWS lambda](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-handler.html). Then we
set this handler as a ES6 module export as required by the lambda
runtime. This fixes issue #2.

Compile, package and deploy to AWS. Note how permissions on the
JavaScript file are set to execute for all. If this is not set, the
lambda runtime won't be able to execute our handler and fail with a
generic "EACCESS" error.

```sh
lein cljsbuild once
chmod 755 hello_lambda_cljs.js
zip -r hello-lambda-cljs.zip hello_lambda_cljs.js node_modules
aws lambda update-function-code --function-name hello-lambda-cljs --zip-file fileb://hello-lambda-cljs.zip --profile test
```

The last step above assumes a lambda already exists with function name
`hello-lambda-cljs`. The most critical part of the lambda
configuration is **the handler**. In our case set the handler to `hello_lambda_cljs.handler`.

{{<figure src="/ox-hugo/lambda-execute.png">}}

Note in the screenshot above how there's an Access Denied error. This
is because the role my lambda has doesn't have access to read all
buckets. This is easily solvable by adding the required IAM permission
to the lambda role.

```js
{
    "Sid": "VisualEditor2",
    "Effect": "Allow",
    "Action": "s3:ListAllMyBuckets",
    "Resource": "*"
}
```


## Conclusion {#conclusion}

Writing AWS lambdas in ClojureScript is possible by transpiling
ClojureScript to JavaScript, and desirable due to lower cold start
times compared to Clojure and the JVM.

The ClojureScript tooling has matured enough to use this approach in
production, beyond proof of concepts. Its ability to require
JavaScript libraries from npm opens up the whole garden. [ClojureScript
is not an Island](https://clojurescript.org/news/2017-07-12-clojurescript-is-not-an-island-integrating-node-modules).

## Gotchas {#gotchas}

In researching for this article I came across many pages using the AWS
SDK as proof that their lambda were up and running and using npm
packages. When trying to replicate in my own environment I couldn't
get the same results. More specifically: I could get the AWS SDK to
work correctly, but not other npm libraries. It would work locally
running with node, but not in AWS.

The reason is [the lambda runtime in AWS has the AWS SDK built in](https://aws.amazon.com/premiumsupport/knowledge-center/lambda-layer-aws-sdk-latest-version/). You might include it in your project and use it correctly, thinking you are using the version you packages but that is not true. The proof is using a different npm library (in our case luxon) and get it to work properly.

I tried using both shadow-npm and figwheel-main to create the package for executing as a lambda using npm packages, but wasn't successful. I'm not saying it's not possible, just that I couldn't get it to work. It would work locally, but not in AWS.

In the end the method I present here is tried and tested and works with every npm library I've used. That being said, I prefer ClojureScript native libraries, and I think CLJSJS still has its place. Maybe a good compromise would be if you depend on a handful of npm libraries, learn how to package them for CLJSJS, submit it to their repository and maintain it!

## Extras {#extras}

While the [goals above](#goals) have been met, there's still a few things that
can make the experience better.


### Escaping callback hell through core.async {#escaping-callback-hell-through-core-dot-async}

One of the main selling points of ClojureScript are the consistent
syntax of a lisp vs the quirks of JavaScript, and a way out of
callback hell thanks to core.async.

A core.async tutorial is out of the scope of this article, but
we'll see how we can use it to escape callback hell while working
with the AWS SDK.

Full disclosure: replacing a single callback with core.async is not
a good idea. If there's just a few callbacks in your project with
no coordination, then using callbacks is fine. When callbacks need
to be coordinated, that's when core.async starts to shine.

The `list-buckets` function above can be rewritten with core.async:

```clojure
(defn <<< [f & args]
  (let [c (chan)]
    (apply f (concat args [#(put! c [%1 %2])]))
    c))

(go (pprint (<! (<<< #(.listBuckets s3 %)))))
```

The `<<<` function takes a function `f` and its arguments, and it
applies `f` to the list of arguments BUT it adds one more argument
to the end: an anonymous function that puts the return values of `f`
to a channel.

Conveniently, most of the AWS SDK functions use the pattern of
requiring a callback as the last argument that takes two arguments:
`error` and `data` to indicate an error or the returned data
respectively.

This allows us to write asynchronous code as if it was a regular
synchronous invocation:

```clojure
(defn handler
  "Lambda main entry point"
  [_ _ callback]
  (go
    (let [[error buckets] (<! (<<< #(.listBuckets s3 %)))]
      (callback nil
                (clj->js {:status 200
                          :body {:s3-buckets buckets
                                 :error error}
                          :headers {}})))))
```

Again, this is a very simple example and the gain is not very
obvious. But when your application starts scaling up and
coordination is required, that's when core.async shines.

Think coordinating three processes: a DynamoDB request, publishing a
message to Kinesis and downloading a file from S3 and doing some
data crunching with it. All three have different running times, and
you need to return your response only when all three are
done. Possible in JavaScript? yes, but not pretty. With core.async
we can have coordination without callbacks.

```clojure
(def c (chan))

(defn simulated-request [c request-type]
  (go
    (let [seconds (* (rand 5) 1000)
          _ (<! (timeout seconds))]
      (>! c {:response request-type :time seconds}))))

(defn process-actions [c]
  (go-loop [responses []]
    (let [{:keys [response time] :as r} (<! c)]
      (println "I'm done:" response ". Took" time "ms.")
      (if (= (count responses) 2)
        (do
          (println "All done! This was the result:")
          (pprint (conj responses r)))
        (recur (conj responses r))))))

(simulated-request c :dynamodb-request)
(simulated-request c :kinesis-push)
(simulated-request c :s3-download)

(process-actions c)
```

The result should be something like:

```sh
I'm done: :kinesis-push . Took 1037.6134152549344 ms.
I'm done: :s3-download . Took 2960.371038999945 ms.
I'm done: :dynamodb-request . Took 4736.172511271778 ms.
All done! This was the result:
[{:response :kinesis-push, :time 1037.6134152549344}
 {:response :s3-download, :time 2960.371038999945}
 {:response :dynamodb-request, :time 4736.172511271778}]
```

This coordination without callbacks can be leveraged in your lambda
functions.


### Packaging for smaller file size {#packaging-for-smaller-file-size}

`node_modules` is notorious for its big size. While a typical
uberjar will very likely be bigger than a comparable ClojureScript
zip file including its npm dependencies, there are actions we can
take to reduce the final package size.

> The lambda cold start time is directly proportional to the
> deployment zip archive.

1.  Use production dependencies. Run `npm install --production` in
    your project root.
2.  Use a tool to remove unnecessary files. I've used [node-prune](https://github.com/tj/node-prune) and
    it can significantly reduce your `node_modules` directory size.
3.  Declare development dependencies as such. Run `npm install -D
          name-of-library` or edit `project.json` directly.

With the actions above you're realistically looking into a ~30%
reduction in size of the final bundle.


### Automating everything with make {#automating-everything-with-make}

Leiningen is my tool of choice when working with Clojure(Script)
projects. There are however, other project related tasks that should
fall outside the responsibilities of leiningen. In writing lambda
functions with ClojureScript, tasks such as:

-   Creating the final bundle for deployment.
-   Pruning the size of the `node_modules` directory.
-   Deploying the lambda.
-   Cleaning up compilation artifacts and npm dependencies.

Make is exactly the tool for the job. Zipping up your project and
running `aws lambda update-function-code ...` by hand gets old real
quick.


### Leiningen template {#leiningen-template}

I might write a leiningen template similar to [anvil](http://github.com/cesarolea/anvil-lein-template) if I end up
writing many ClojureScript lambdas with these same patterns.


## Credits {#credits}

-   [This article in dev.to](https://dev.to/beders/developing-testing-and-deploying-aws-lambda-functions-written-in-clojurescript-284l). I've had the idea of leveraging
    ClojureScript for AWS lambda development for a few years now, but
    there was always _something_ that prevented me from proceeding. The
    article provided much guidance, but also the reassurance that there
    was someone out there that got it to work.
-   [PurelyFunctional.tv](https://purelyfunctional.tv/mini-guide/core-async-code-style/) article on core.async guide. I straight up
    lifted the `<<<` function and some ideas from the linked
    article. The site is a treasure trove!
