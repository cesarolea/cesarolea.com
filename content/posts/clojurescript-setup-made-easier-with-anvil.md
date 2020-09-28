---
title: "ClojureScript Setup Made Easier With Anvil"
date: 2020-09-24T11:58:00-07:00
draft: false
---

Some time ago [I wrote a series posts](https://blog.devz.mx/clojurescript-sin-atajos-fase-1/) in the [devz.mx](https://devz.mx) blog on setting up a ClojureScript project from scratch. I did it for learning how to do it myself. There’s multiple leiningen templates that will set it up for you, but I always fell short when wanting to set up a more complex environment such as a browser connected REPL, code evaluation from your editor, setting up multiple ClojureScript libraries, and more.

From the learning perspective, it was a success. I was able to learn the ins and outs of setting up a new ClojureScript project, and used it for my personal projects. Eventually I decided to wrap everything in a leiningen template and publish it for everyone else to use. Enter [anvil](https://github.com/cesarolea/anvil-lein-template).

Anvil is a leiningen template for creating a mostly bare-bones ClojureScript template. The purpose is to get you up and running with an efficient workflow with as little boilerplate code as possible. The extras are up to you. Anvil is opinionated in what exactly is an efficient workflow, and its geared towards people working with Emacs and familiar with Clojure (not just ClojureScript).

What Anvil gives you out of the box is:

- Hot reloading of Web assets (html, JavaScript, CSS and ClojureScript) with Figwheel.
- Linting support with clj-kondo.
- A networked REPL that allows you to evaluate code from your connected editor, and also a browser connected REPL that allows you to evaluate code in your browser.
- An option to support Reagent and Re-Frame.
- An option to use a virtualized environment with Docker.

## Workflow

The workflow is the same if you choose to use Docker or go native:

1. Create a new project with `lein new anvil my-awesome-project`. You can add additional options to support docker, reagent and/or reframe: `lein new anvil my-awesome-project +docker +reagent +reframe`
2. Change to the project directory and start a REPL with lein repl
3. Connect your editor to the running REPL. In Emacs/Cider this is done with `cider-connect-clj`
4. Once in the Clojure REPL evaluate `(start)` and it should start the figwheel process. Navigate to `localhost:9500` in your browser, it should connect and at that moment the ClojureScript REPL is available and connected to your editor.

When you are ready to go to production, a `lein fig:prod` will compile everything for you with all optimizations turned on.

If this sounds like something you might want to use, there’s more instructions and options in [the Github page](https://github.com/cesarolea/anvil-lein-template). The template is published to Clojars, so it should be readily available for you to use.

Happy Clojuring!
