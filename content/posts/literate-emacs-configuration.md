+++
title = "Literate Emacs Configuration"
author = ["Cesar Olea"]
date = 2020-09-28T13:02:00-07:00
draft = false
+++

I've migrated my Emacs configuration to use a literate programming
style, after several unsuccessful attempts at better organization, and
going through many Emacs
[configuration
bankruptcies](https://www.emacswiki.org/emacs/DotEmacsBankruptcy).

The big plus is obviously that the configuration file itself is in an
Org file. But another big plus is that Gihub renders the file directly
in the repository. It makes it easier for me to document all my
settings, but also people can pick and choose pieces of my
configuration straight from the Github page.

One (maybe dealbreaker) caveat is that Emacs will take longer to
start. That is because instead of just interpreting the configuration
file, it now has to untangle the Org file first, build the final
configuration file and then execute it. Since I use Emacs as a daemon
it doesn't matter (it is only started once, at system startup) but if
you use Emacs as a traditional application then I would recommend to
untangle the file manually, and doing so every time you add a new
section to the Org configuration.

My configuration can be found in
[my Github repository](https://github.com/cesarolea/emacs).
