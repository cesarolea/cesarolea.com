#+HUGO_DRAFT: false
#+TITLE: Build Your Own Nativecomp PGTK Emacs
#+DATE: 2021-04-14T12:30:39-07:00

I've [written before]({{< relref "/posts/emacs-native-compile.org" >}}) on what a gamechanger Emacs with native compiled lisp and PGTK frontend is. However it's a bit of a hassle to install the toolchain and compile it yourself.

If you are running Ubuntu 20.04 as I am and you would like to get your hands on a deb package you can install, now's easier than ever. You can build your own using [my fork](https://github.com/cesarolea/emacs-gcc-pgtk) of [konstare's Docker based build for Ubuntu 20.10](https://github.com/konstare/emacs-gcc-pgtk).

The only dependency is Docker. With Docker in your system it is as easy as:

```
git clone https://github.com/cesarolea/emacs-gcc-pgtk
cd emacs-gcc-pgtk
./build.sh
```

Sit back and let the process finish. Once it's done, the deb package can be installed by `sudo dpkg -i deploy/emacs-gcc-pgtk_28.0.50.deb`.
