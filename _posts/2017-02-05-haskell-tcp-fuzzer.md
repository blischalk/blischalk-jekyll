---
layout: post
title: "Haskell TCP Fuzzer"
description: "An example TCP fuzzer written in Haskell"
tags: [haskell, fuzzer]
---

Code for this project can be found on Github [here](https://github.com/blischalk/haskell-fuzzer).

I have been spending a lot of time lately programming in C and the
verbosity and memory management have ben wearing on me. In my day job
I have worked mostly in Clojure for the past few years so working on
projects in C has felt a bit clunky. I have really been missing all of
the great functional programming niceties I have come to enjoy from Clojure.

So why am I playing with Haskell here? Well I also dabble in Haskell
as it has a lot of really cool features without the JVM start up
time. So, I wanted to exercise that interest a bit. So how can we
write a fuzzer in Haskell?

For those of you unfamiliar with a fuzzer, it is essentially a program
that sends data to another program in attempts to trigger a buffer
overflow exception. Using the fuzzer you increase the size of the
payload until you can get the program under analysis to
crash. Normally in the security world fuzzers are written in Python,
Ruby, or Perl an look something like this:

{% highlight python %}
#!/usr/bin/python

import socket
import os
import sys
CRASH     = "\x41" * 7
EIP = "\x42\x42\x42\x42"
SHELLCODE = "\x43" * 43


buffer="HELP " + CRASH + EIP + SHELLCODE + "\r\n"

print "[*] Sending evil HTTP request"

expl = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
expl.connect(("192.168.99.102", 4321))
data = expl.recv(1024)
print "\n[*] Server Banner: %s" % data
print "\n[*] Buffer: %s" % buffer
expl.send(buffer)
data1 = expl.recv(1024)
print "\n[*] Server Response: %s" % data1
expl.close()


{% endhighlight %}

Essentially what this does is open a tcp socket connnection to a
machine and then sends and receives data. The send and receive data
part is often different from application to application as some
applications will send first and others will wait to receive first. So
how might we open a tcp socket connection in Haskell and send some
data?

Through a short Google search I found this
[tcp library](https://hackage.haskell.org/package/network-simple-0.4.0.5/docs/Network-Simple-TCP.html).
This seems like it will be helpful.

Using [stack](https://docs.haskellstack.org/en/stable/README/) we set
ourselves up with a skeleton project following the quickstart:

We also need to update our `fuzz.cabal` and `stack.yaml` with some dependencies:

### stack.yaml

{% highlight yaml %}
# Packages to be pulled from upstream that are not in the resolver (e.g., acme-missiles-0.3)
extra-deps: [network-simple-0.4.0.5]
{% endhighlight %}

### fuzz.cabal

{% highlight haskell %}

library
  hs-source-dirs:      src
  exposed-modules:     Lib
  build-depends:       base >= 4.7 && < 5
                     , bytestring
                     , network-simple
  default-language:    Haskell2010

executable fuzz-exe
  hs-source-dirs:      app
  main-is:             Main.hs
  ghc-options:         -threaded -rtsopts -with-rtsopts=-N
  build-depends:       base
                     , bytestring
                     , fuzz
                     , network-simple
  default-language:    Haskell2010


{% endhighlight %}

The two dependencies we have added are `network-simple` which is the
TCP library we found earlier and `bytestring` which is a library
included with Haskell we need to require into our project.

Next we write our code:

{% highlight haskell %}
module Lib
    ( fuzz
    ) where

import qualified Data.ByteString as BS
import Network.Simple.TCP

host :: String
host = "127.0.0.1"

port :: String
port = "4444"

payload :: BS.ByteString
payload = BS.pack $ take 500 $ repeat 0x41

fuzz :: IO ()
fuzz = do
  putStrLn $ "About to fuzz " ++ host
  connect host port $ \(connectionSocket, remoteAddr) -> do
    putStrLn $ "Connection established to " ++ show remoteAddr
    send connectionSocket payload

{% endhighlight %}

If we startup a netcat listener using `nc -nlvp 4444` in one terminal:

{% highlight bash %}
:~$ nc -nlvp 4444
{% endhighlight %}

And compile our program with stack doing in a separate terminal:

{% highlight bash %}
stack build
{% endhighlight %}

We can execute our new program with `stack exec fuzz-exe` and should
see some output on the netcat side:


{% highlight bash %}
:~$ nc -nlvp 4444
Connection from 127.0.0.1:53340
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
{% endhighlight %}

And it works! we see our payload of A's gets sent accross the network.
