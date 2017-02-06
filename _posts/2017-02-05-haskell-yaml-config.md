---
layout: post
title: "Haskell YAML Config"
description: "A quick example of reading a yaml config file in Haskell."
tags: [haskell, yaml, config]
---

Source code for this blog post can be found
[here](https://github.com/blischalk/haskell-yaml-config).

One of my favorite ways of learning a programming language is to just
create little projects for myself that I want to learn. The process of
doing the research to learn how to achieve a particular programming
task I often find more enjoyable than trying to understand all of the
in's and out's of a language. That said, one project I embarked on to
get better at Haskell was reading a yaml config file. Pretty much all
programming projects I have worked on have had some sort of
configuration so I felt that this little project would be well worth
the effort.


## Libraries

As always, the first place we start looking is for libraries. Out of
the gate we find the [yaml](https://hackage.haskell.org/package/yaml)
and [yaml-config](https://hackage.haskell.org/package/yaml-config)
libraries. The more general `yaml` library seems to specifically deal
with yaml data. On the otherhand, the `yaml-config` library seems to
go a step further and deals with loading a config file from
disk. Looking at it's `.cabal` file it seems that unsurprisingly
`yaml-config` uses the `yaml` library behind the scenes. I always like
to work at the highest level of abstraction if possible and it seems
that `yaml-config` may do exactly what we are looking for so lets
start there.

## yaml-config

We'll start off by creating a basic project with stack:

{% highlight bash %}
stack new readyamlconfig
stack setup
cd readyamlconfig
stack build
:~/shared/haskell/readyamlconfig$ stack exec readyamlconfig-exe
someFunc
{% endhighlight %}

We can see that executing the basic stub program with `stack exec
readyamlconfig-exe` outputs `someFunc`. This means our scaffolded
program is ready for us to write some code.

The next thing we'll want to do is go ahead and add the `yaml-config`
library dependency to our `readyamlconfig.cabal` and `stack.yaml`
files.

### readyamlconfig.cabal

{% highlight haskell %}

library
  hs-source-dirs:      src
  exposed-modules:     Lib
  build-depends:       base >= 4.7 && < 5
                     , yaml-config
  default-language:    Haskell2010

executable readyamlconfig-exe
  hs-source-dirs:      app
  main-is:             Main.hs
  ghc-options:         -threaded -rtsopts -with-rtsopts=-N
  build-depends:       base
                     , readyamlconfig
                     , yaml-config
  default-language:    Haskell2010

{% endhighlight %}

### stack.yaml

{% highlight yaml %}
extra-deps: [yaml-config-0.4.0]
{% endhighlight %}

Then we rebuild the project with `stack build` to acquire the dependencies.

Next we create a basic `yaml` config file:

### config.yaml

{% highlight yaml %}
config:
  items:
    - thing one
    - thing two
{% endhighlight %}

Now lets write some code:

{% highlight haskell %}
{-# LANGUAGE OverloadedStrings #-}
module Lib
    ( someFunc
    ) where

import Data.Yaml.Config (load, subconfig, lookupDefault)

someFunc :: IO ()
someFunc = do
  configFile <- load "./config.yaml"
  cfg <- subconfig "config" configFile
  let items = lookupDefault "items" [] cfg

  mapM_ putStrLn items

{% endhighlight %}

This library seems to have a dependency on OverloadedStrings so I have
added that from the example on the github page. The code is all pretty
straight forward. We load up the file, grab the "config" key and get
the collection of "items". `lookupDefault` is used here as an example
of how a default can be specified if a key is not found in a config
file.
