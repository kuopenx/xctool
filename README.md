
# xctool

__xctool__ is a replacement for Apple's __xcodebuild__ that makes it
easier to build and test iOS and Mac products.  It's especially helpful
for continuous integration.

It's a drop-in replacement for __xcodebuild__ that adds a
few extra features:

* **Runs the same tests as Xcode.app.**

  Surprisingly, Apple's command-line _xcodebuild_ tool does not run your
product's tests the same way as _Xcode.app_.  xcodebuild doesn't
understand which targets in your scheme are test targets, which test
suites or cases you've disabled, or how to run simulator-based,
application tests.

  If you use [application
tests](http://developer.apple.com/library/mac/#documentation/developertools/Conceptual/UnitTesting/08-Glossary/glossary.html#//apple_ref/doc/uid/TP40002143-CH8-SW1),
you've probably seen xcodebuild skipping them with this message:
	
	```
	Skipping tests; the iPhoneSimulator platform does not currently support
	application-hosted tests (TEST_HOST set).
	```

  *xctool* looks at your Xcode scheme and is able to reproduce the same
test run you would get with Xcode.app via _Cmd-U_ or 
_Product &rarr; Test_, including running application
tests that require the iOS simulator.

* **Structured output of build and test results.**

  _xctool_ captures all build events and test results as structured JSON
objects.  If you're building a continous integration system, this means
you don't have to regex parse _xcodebuild_ output anymore.

  Try one of the [Reporters](#reporters) to customize the output or get
the full event stream with the `-reporter json-stream` option.

* **Human-friendly, ANSI-colored output.**

  _xcodebuild_ is incredibly verbose, printing the full compile command
and output for every source file.  By default, _xctool_ is only verbose
if something goes wrong, making it much easier to identify where the
problems are.

  Example:

	![pretty output](https://fpotter_public.s3.amazonaws.com/xctool-uicatalog.gif)

## Requirements

You'll need Xcode's Command Line Tools installed.  From Xcode, install
via _Xcode &rarr; Preferences &rarr; Downloads_.

## Usage

xctool's commands and options are mostly a superset of xcodebuild's.  In
most cases, you can just swap __xcodebuild__ with __xctool__ and things will
run as expected but with more attractive output.

You can always get help and a full list of options with:

```
path/to/xctool.sh -help
```

### Building

Building products with _xctool_ is the same as building them with
_xcodebuild_.

If you use workspaces and schemes:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  build
```

If you use projects and schemes:

```
path/to/xctool.sh \
  -project YourProject.xcodeproj \
  -scheme YourScheme \
  build
```

All of the common options like `-configuration`, `-sdk`, `-arch` work
just as they do with _xcodebuild_.

NOTE: _xctool_ doesn't support directly building targets using
`-target`; you must use schemes.

### Testing

_xctool_ has a __test__ action which knows how to build and run the
tests in your scheme.  You can optionally limit what tests are run
or change the SDK they're run against.

To build and run all tests in your scheme, you would use:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  test
```

To build and run just the tests in a specific target, use the `-only` option:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  test -only SomeTestTarget
```

You can go further and just run a specific test class:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  test -only SomeTestTarget:SomeTestClass
```

Or, even further and run just a single test method:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  test -only SomeTestTarget:SomeTestClass/testSomeMethod
```

You can also run tests against a different SDK:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  test -test-sdk iphonesimulator5.1
```

#### Building Tests

While __test__ will build and run your tests, sometimes you want to
build them without running them.  For that, use __build-tests__.

For example:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  build-tests
```

You can optionally just build a single test target with the `-only` option:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  build-tests -only SomeTestTarget
```

#### Running Tests

If you've already built tests with __build-tests__, you can use
__run-tests__ to run them.  This is helpful if you want to build tests
once but run them against multiple SDKs.

To run all tests, you would use:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  run-tests
```

Just as with the __test__ action, you can limit which tests are run with
the `-only`.  And, you can change which SDK they're run against
with the `-test-sdk`.

## Reporters

xctool has reporters that output build and test results in different
formats.  By default, _xctool_ always uses the `pretty` reporter.

You can change or add reporters with the `-reporter` option:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  -reporter plain \
  build
```

By default, reporters output to standard out, but you can also direct
the output to a file by adding `:OUTPUT_PATH` after the reporter name:

```
path/to/xctool.sh \
  -workspace YourWorkspace.xcworkspace \
  -scheme YourScheme \
  -reporter plain:/path/to/plain-output.txt \
  build
```

### Included Reporters

* __pretty__: a text-based reporter that uses ANSI colors and unicode
symbols for pretty output (the default).
* __plain__: like _pretty_, but with with no colors or unicode.
* __phabricator__: outputs a JSON array of build/test results which can
be fed into the [Phabricator](http://phabricator.org/) code-review tool.
* __json-stream__: a stream of build/test events as JSON dictionaries,
one per line.  The structure looks like
[this](https://gist.github.com/fpotter/e8a0de3d3c81eaf58d20).

You could also __add your own__ Reporter - see
[Reporter.h](https://github.com/facebook/xctool/blob/master/xctool/xctool/Reporter.h).

## Configuration (.xctool-args)

If you routinely need to pass many arguments to _xctool_ on the
command-line, you can use an __.xctool-args__ file to speed up your workflow.
If _xctool_ finds an __.xctool-args__ file in the current directory, it
will automatically pre-populate its arguments from there.

The format is just a JSON array of arguments:

```
[
  "-workspace", "YourWorkspace.xcworkspace",
  "-scheme", "YourScheme",
  "-configuration", "Debug",
  "-sdk", "iphonesimulator",
  "-arch", "i386"
]
```

Any extra arguments you pass on the command-line will take precendence
over those in the _.xctool-args_ file.

## Contributing

Bug fixes, improvements, and especially new
[Reporter](https://github.com/facebook/xctool/blob/master/xctool/xctool/Reporter.h)
implementations are welcome.  Submit a [pull
request](https://help.github.com/articles/using-pull-requests).

## License

Copyright 2013 Facebook

Licensed under the Apache License, Version 2.0 (the "License"); you may
not use this work except in compliance with the License. You may obtain
a copy of the License in the LICENSE file, or at:

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
