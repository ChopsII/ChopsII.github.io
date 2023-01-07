# Why I Separate Testing Code from the Source Code Repository

I've recently been working on a testing framework library, and in the course of doing so, have written a lot of generic library code that should probably be split out into its own library, and consumed by the testing framework as a dependency of the testing framework.

Said generic library code, however, also needs to be tested. And I just so happen to have a testing framework library available that needs to be validated through a ~real use case.

So I set up the dependency where the Generic Library depends on the Test Framework Library.

![Oh no! The libraries depend on each other!](<./dep loop.png> "Oh no! The libraries depend on each other!")

But the Test Framework Library already depends on the Generic Library! I have a dependency loop, which is Very Bad (tm)

Looking a bit closer, I can see that the only reason that Generic Library needs to depend on Testing Framework Library is for the code testing the Generic Library. 

![Only parts of the source code within the library depend on the other library](<./structure exposed.png> "Only parts of the source code within the library depend on the other library")

So if I move that out, the Generic Library no longer depends on the Test Framework Library at all, solving the dependency loop.


![Separating the testing code into its own repo resolves the loops](<./deps loop resolved.png> "Separating the testing code into its own repo resolves the loops")

## A secondary reason

There's another reason to separate the testing code from the source code: consumers of the library don't usually need to have a copy of the testing code; that should be handled by the merging processes / Continuous Integration setup.

Having said that, there's at least one fairly significant caveat which I'm not entirely comfortable with my solution to just yet - keeping source code in synch with the code that tests it.

## Other caveats

1. The "secondary reason" is generally only an issue if the library is consumed via checking out the entire repository, rather than using some dependency fetching and management solution that can retrieve only the parts of the dependency that is actually depended on. For example, Conan.io seems to have the ability to include only the actual functional source of a library in the package it deploys (as long as the package maintainer sets up that feature)

2. This separation is really only applicable to projects that may be consumed by other projects - though that's the majority of the stuff I work on in personal projects

3. This separation _might_ only really be a problem for libraries that are consumed by testing frameworks that I'm using to test said library. Haven't quite thought that through yet. I _think_ it's applicable whenever two libraries' tests depend on the other library's implementation, but the libraries' implementations themselves don't depend on each other's implementation.
