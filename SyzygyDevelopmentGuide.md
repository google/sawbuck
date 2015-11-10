**DEPRECATED: Please use the [new version](https://code.google.com/p/syzygy/wiki/SyzygyDevelopmentGuide) of the page.**


# Introduction #

Syzygy is all about taking apart perfectly good executable binaries, mutating them in interesting ways, and then outputting new, perfectly good executable binaries.
The output binaries are either functionally equivalent, but improved in some way - like e.g. in an improved order, or else contain new functionality, which is the case for instrumented binaries.

This is pretty exacting business, so correctness and quality are very important for Syzygy, hence these policies.

# Details #

## Contribution ##

If you don't work for Google and want to contribute code to Syzygy, you'll need to sign the [individual](http://code.google.com/legal/individual-cla-v1.0.html) or the [corporate](http://code.google.com/legal/corporate-cla-v1.0.html) contributor license.

Syzygy uses the same build tools as Chromium, so you'll need to set up the build environment as described in the [Chromium instructions](http://www.chromium.org/developers/how-tos/build-instructions-windows#TOC-Build-environment).  If you get build errors where 64-bit projects are skipped, it is likely because you failed to install "X64 Compilers and Tools" per the [Chromium instructions](http://www.chromium.org/developers/how-tos/build-instructions-windows#TOC-Build-environment).

Once this is done, you can fetch the sources by doing:
```
> mkdir Syzygy
> cd Syzygy
> gclient config http://sawbuck.googlecode.com/svn/trunk/ --name=src
> gclient sync
```
After this completes, you can open the main solution file at `src\syzygy\syzygy.sln` and build it.

## Coding Guidelines ##

Syzygy code must generally adhere to the [Chromium style guidelines](http://www.chromium.org/developers/coding-style).

All code must be peer-reviewed by one or more other project members before submission.
Once you have code ready for review, use `gcl change my_change_name` to create a change list, then use `gcl upload` to upload it to the code review site. See the [Chromium instructions](http://www.chromium.org/developers/contributing-code#TOC-Upload-your-change-to-Rietveld) for more detail.

All pre-submit checks must pass before committing the change.

All new code must be unittested. See the [testing](#Testing.md) section for details.

New submissions will trigger the continuous builder on [Syzygy's buildbot](http://build.chromium.org/p/client.syzygy/console), which will compile the new contribution, run a suite of smoke tests on it, and generate a code coverage report.
Build or test failures on the buildbot should be fixed immediately, either by reverting the change, or by submitting a fix as soon as possible.

## Testing ##

Syzygy requires all C++ code to have better than 60% unittest coverage, as measured by the Syzygy [coverage builder](http://chromegw.corp.google.com/i/client.syzygy/builders/Syzygy%20Coverage).
After every submission, look at the next coverage report (you should get an email when it completes).
If the new code does not meet the coverage requirement, either revert the change, or submit a fix to improve the unittest coverage.

Any submission with a non-trivial bugfix must add a regression test for the bug.

Any significant feature addition must additionally be tested by an integration test, that tests the feature end-to-end.

## Release criteria ##

TODO(siggi): release criteria.