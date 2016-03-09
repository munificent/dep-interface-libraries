# Interface Libraries

This proposal lives at: https://github.com/munificent/dep-interface-libraries

Author:

- [Bob Nystrom][bob] (rnystrom@google.com)

Stakeholders:

- [Erik Ernst][erik] (eernst@google.com)
- [Florian Loitsch][florian] (floitsch@google.com)
- [Lasse R.H. Nielsen][lasse] (lrn@google.com)

Contributors:

- [John Messerly][john] (jmesserly@google.com)
- [Natalie Weizenbaum][natalie] (nweiz@google.com)

[bob]: https://github.com/munificent
[erik]: https://github.com/eernstg
[florian]: https://github.com/floitschG
[john]: https://github.com/jmesserly
[lasse]: https://github.com/lrhn
[natalie]: https://github.com/nex3

## Summary

Let an import or export resolve to one of several libraries, all of which are
compatible with an *interface library* describing their shared capabilities. A
Dart implementation selects which library to use at runtime or compile time. The
interface library is used for static analysis.

## Motivation

From its inception, Dart has run on multiple different "platforms": the native
VM running on the console, Dartium, compiled to JavaScript, etc. These
implementations vary in their capabilities, which is why core libraries like
"dart:io" and "dart:html" aren't available on all of them. As we move into
ever-more-diverse mobile platforms, this problem magnifies. We have "dart:sky"
now and may find ourselves with "dart:android", "dart:ios", etc.

Often, different platforms *do* have the same capability, just exposed through a
different API. You can do HTTP and WebSockets in the browser and in the
command-line VM, but you can't do them using the same API.

Users want to write cross-platform libraries that encapsulate these differences,
so that higher level code is platform independent. Today, this is impossible. If
your library imports "dart:html", it fails *at compile time* on the standalone
VM. You never get to `main()`.

For example, the [http][] package would like to make HTTP requests using the
[`HttpClient`][io client] class from "dart:io" when used on the command-line
while using an [`HttpRequest`][html request] on the browser.

[http]: https://pub.dartlang.org/packages/http
[io client]: https://api.dartlang.org/apidocs/channels/stable/dartdoc-viewer/dart:io.HttpClient
[html request]: https://api.dartlang.org/apidocs/channels/stable/dartdoc-viewer/dart:html.HttpRequest

## Examples

We'll walk through a more detailed example that goes through each phase of the
proposal below. For now, we'll do the tiniest possible sample so you can get a
flavor for the syntax and semantics.

Let's say we want to make a library for showing warnings to the user. On the
standalone VM, we want it to print to stderr. On the browser, it should show an
alert. The user should be able to write this program:

```dart
import 'warn.dart';

main() {
  warn("Configuration-specific code proposals are a tar pit!", severe: true);
}
```

They should be able to run (or compile and run) it on any Dart platform and have
it to do the right thing. The main library looks like:

```dart
// warn.dart
String warn(String message, {bool severe}) {
  if (severe) message = message.toUppercase();

  // <show message in config-specific way...>
}
```

This proposal is about filling in that hole. The first thing we have to do is
make a separate little library that defines the interface we will use to talk to
platform-specific behavior. In this case, it's simple:

```dart
// warn_interface.dart
void showMessage(String message) {}
```

We're leaving the body empty because this is only an interface. All we care
about is its signature. We could make the function `external` too. We could even
plug in the implementation for one of the platforms here and treat that platform
as the "default". But, to make this example more explicit, we'll split out each
platform separately.

Now we "implement" this library for each platform. By that, I just mean we
define a separate library for each platform that defines a public function with
the exact same name and signature:

```dart
// warn_io.dart
import 'dart:io' as io;

void showMessage(String message) {
  io.stderr.writeln(message);
}
```

```dart
// warn_html.dart
import 'dart:html' as html;

void showMessage(String message) {
  html.window.alert(message);
}
```

Finally, back in "warn.dart" we wire them in using this proposal:

```dart
// warn.dart
import 'warn_interface.dart'
    if (dart.library.io) 'warn_io.dart'
    if (dart.library.html) 'warn_html.dart';

String warn(String message, {bool severe}) {
  if (severe) message = message.toUppercase();

  showMessage(message);
}
```

The interesting part is the `import` and the `if (...)` clauses on it. We call
an import or export with these clauses a *configured directive* . You can
probably guess how it works. The first URI is always the *interface* or *default
library*. It's the library that things like IDEs and static analyzers see. It's
where you go to if you "Go to Definition" on a member declared in the configured
library.

Following that are one or more *configurations*. Each is an `if ()` clause
followed by a URI. The condition in the `if` is a simple test against
environment declarations (`-D` command-line arguments). Some environment
declarations are pre-populated by a Dart implementation to let the user know
which "dart:" libraries are available.

At runtime (or at compile time in dart2js), these `if ()` clauses are tried one
at a time. Each condition is evaluated. If it comes out `true` then we use the
following URI instead of the default one. If none of the clauses match, it just
falls back to using the interface URI.

That's basically it. The actual user-visible behavior is very straightforward.
When figuring out what URI to use in an import or export, it picks from one of a
set of options based on some compile-time constants.

### Library compatibility

Imagine if "warn_io.dart" looked like:

```dart
// warn_io.dart
import 'dart:io' as io;

void showMassage(String message) {
  io.stderr.writeln(message);
}
```

We misspelled the function name. That means the call to `showMessage()` in
"warn.dart" is going to fail at runtime since this library doesn't define that.
If the static checker only looks at "warn_interface.dart", we won't get any
static warnings about this.

To avoid problems like this where a configuration-specific library can't
correctly replace the interface library, there is an additional set of static
warnings to determine *library compatibility*.

When the analyzer sees a configuration-specific import, it first analyzes all of
the referenced libraries as normal (since they are on their own perfectly valid
libraries). In addition, it goes through each configuration-specific library and
compares it to the interface library.

It looks at both of their public APIs and reports static warnings if they are
not *compatible* with each other. We'll define "compatible" more precisely
below, but the intent is to give you reasonable confidence during static
analysis that all of your different configuration-specific will plug-in
correctly at runtime.

## Proposal

The runtime (or compile time) behavior is simple, so we'll cover that first. The
static checking is more complex. In fact, its complexity lies on a continuum.
There are simple static rules we can support that aren't as expressive. Or we
can allow more constructs and statically check them, but the resulting rules are
more complex.

It isn't clear how much static checking is really needed here, so the proposal
is broken into phases. The first phases is the simplest but least statically
expressive. Later phases give the user more fine-grained, powerful static
validation. The idea is to implement the phases incrementally and stop once we
reach a point where users are happy with what can be known statically.

### Syntax

We insert an optional list of `if (...)` clauses in `import` and `export`
directives after the initial URI. More precisely, we change
**importSpecification** and **libraryExport** to:

**importSpecification:**<br>
&nbsp;&nbsp;`import` uri configuration\* (`as` identifier)? combinator\* `;` |<br>
&nbsp;&nbsp;`import` uri configuration\* `deferred` `as` identifier combinator\* `;`<br>
&nbsp;&nbsp;;

**libraryExport:**<br>
&nbsp;&nbsp;metadata `export` uri configuration\* combinator\* `;`<br>
&nbsp;&nbsp;;

And we add:

**configuration:**<br>
&nbsp;&nbsp;`if` `(` test `)` uri<br>
&nbsp;&nbsp;;

**test:**<br>
&nbsp;&nbsp;dottedName (`==` stringLiteral)?<br>
&nbsp;&nbsp;;

**dottedName:**<br>
&nbsp;&nbsp;identifier (`.` identifier)\*<br>
&nbsp;&nbsp;;

String literals are not allowed to contain interpolation.

### Runtime behavior

When a directive containing configurations is processed, each configuration is
tried in order. If its `test` evaluates to `true`, then the following `uri` is
used and the rest of the configurations are ignored. If all tests fail (or there
are none), the initial default `uri` is used.

Tests are string comparisons, where a `dottedName` is used to create a key, and
that key is used to look up a string value in the environment (using the
equivalent of `String.fromEnvironment()`). String literals are any normal Dart
string literals without interpolation.

To evaluate a test:

 1. If the `test` does not have a `==` clause, it implicitly is `== "true"`.
 2. The `dottedName` is converted to a string, `key`, by concatenating the
    characters of all `identifier` parts, a separated by `.`. (In other words,
    ignore any whitespace, the same as we do with library names.)
 3. Let `lookup` be the result of looking up `key` in the environment, as by
    the constant expression `const String.fromEnvironment(key)`.
 4. Convert `stringLiteral` to a string, `stringValue`, as if it was a
    compile-time constant expression.
 5. If `lookup` is equal to `stringValue` (contains the same code units), then
    return `true`.
 6. Otherwise (the key is not in the environment, or `lookup` has a different
    value) return `false`.

After a URI is chosen, the directive is processed as normally using that URI.

### Predefined constants

We often use configuration-specific code to detect which Dart implementation a
program is running in, or which "dart:" libraries are available. To expose that,
every Dart implementation exposes a set of corresponding constants as
part of the environment.

For every `dart:` library that the embedder exposes, a corresponding constant
is implicitly added to the environment with value "true". The constant's name
starts with "dart.library." and is suffixed by the import-name of the library.
For example, for an embedder that supports `dart:io` the constant
`dart.library.io` is in the environment, and has the value "true".

All library constants start with "dart." and the Dart platform reserves the
use of constants starting with that prefix. In case of conflict with a
user-defined environment variable, the user-defined value shadows the
value that is provided by the embedder.

### Static checking

#### Phase 0: nothing

Since the runtime behavior does not rely on static checking, we can start with
no compatibility checking at all. It is up to users of configuration-specific
imports and exports to test carefully and ensure that all of the configurations
work.

#### Phase 1: functions

Given Dart's focus on static analysis, we expect users do want some checking for
compatibility. The simplest level is to allow configurable *behavior*, but not
configurable *types*.

In other words, we statically check top-level functions. As you'll see in later
phases, this is dramatically simpler. The process works like so:

**To tell if two libraries are compatible:**

1.  Produce the *visible namespace* of each library. This is the export
    namespace of the library after the `show` and `hide` combinators in the
    directive have been applied.
2.  If the visible namespace of either library contains a type declaration
    (class, enum, or typedef), fail.
3.  If the maps of names to members of the two visible namespaces are not
    compatible, fail.

**To tell if two maps of names to members A and B are compatible:**

1.  If the maps do not have the same sets of names, fail.
2.  For each corresponding pair of members A.m and B.m:
    1.  If A.m and B.m are not the same *kind*, fail. "Kind" is which kind of
        declaration the member is: method, getter, setter. Note that variables
        are not a kind because a variable is just a getter (final) or pair of
        getter and setter (non-final).
    2.  Else if A.m is a method:
        1.  If their return types are not compatible, fail.
        2.  If their parameter lists are not compatible, fail.
    3.  Else if A.m is a getter:
        1.  If their return types are not compatible, fail.
    4.  Else (A.m is a setter):
        1.  If their parameter types are not compatible, fail.

**To tell if two types A and B are compatible:**

1.  If A and B are both `void`, succeed.
2.  Else if A and B are both named types:
    1.  If their type argument lists are not compatible, fail.
    2.  If they do not refer to the same declaration, fail.
3.  Else (A and B are function types):
    1.  If the return types are not compatible, fail.
    2.  If their parameter lists are not compatible, fail.

**To tell if two parameter lists are compatible:**

1.  If the mandatory parameter lists are not compatible, fail.
2.  If the optional positional parameter lists are not compatible, fail.
3.  If one declares a named parameter that the other lacks, fail.
4.  For each corresponding pair of named parameters:
    1. If the types are not compatible, fail.
    2. If the default values are not identical, fail.
5.  Otherwise, succeed.

**To tell if two lists of types are compatible:**

1.  If the lists have different lengths, fail.
2.  If any corresponding pair of types in the lists aren't compatible, fail.
3.  Otherwise, succeed.

Note that this is *much* tighter than assignment compatibility, or even
subtyping.

#### Phase 2: Types

The first phase gets us pretty far. We can define and statically check
configuration-specific functions.

It even lets us define classes that *appear* to be configuration-specific by
using a factory constructor that calls a configuration-specific factory
function:

```dart
// warn.dart
import 'warn_interface.dart'
    if (dart.library.io) 'warn_io.dart'
    if (dart.library.html) 'warn_html.dart'
    as configured;

abstract class Warning {
  factory Warning(String message) => configured.createWarning(message);

  void show();
}
```

```dart
// warn_interface.dart
import 'warn.dart';

Warning createWarning(String message) {}
```

```dart
// warn_io.dart
import 'dart:io';
import 'warn.dart';

class IOWarning implements Warning {
  final String _message;

  IOWarning(this.message);

  void show() {
    stderr.writeln(message);
  }
}

Warning createWarning(String message) => new IOWarning(message);
```

You can guess what the browser one looks like. In most cases, this works fine.
What this doesn't let users do is *extend* the Warning class. Since it has a
factory constructor and no generative constructors, extending it is off the
table.

In some cases, we could work around that by having a single
configuration-independent class that forwards its *methods* to
configuration-specific function instead of doing the configuration at
construction time. But, if we want to have a lot of configuration-specific
state, that's annoying.

#### Implementing "dart:" types

Even more pernicious is if you want to actually construct an instance of a
platform-specific "dart:" type. For example, let's say we want to provide a
platform-independent way of working with "dart:html"'s [GamepadButton][] class.
(I'm using GamepadButton here because it's tiny. In practice, people want to do
this with important classes like [Node][] and [Element][].)

[gamepadbutton]: https://api.dartlang.org/1.12.0/dart-html/GamepadButton-class.html
[node]: https://api.dartlang.org/1.12.1/dart-html/Node-class.html
[element]: https://api.dartlang.org/1.12.1/dart-html/Element-class.html

We define an interface that matches the real one:

```dart
// gamepad_button_interface.dart
abstract class GamepadButton {
  bool get isPressed;
  double get value;
}
```

On the standalone VM where "dart:html" isn't natively supported, we write our
simulated one:

```dart
// gamepad_button_io.dart
class GamepadButton {
  final bool isPressed;
  final double value;

  GamepadButton(this.isPressed, this.value);
}
```

On the browser, we want to provide an unwrapped instance of the real, native
GamepadButton class. This is important because we may be passing it to other
"dart:html" APIs that only work with the real deal backed-by-C++ class.

We can accomplish that like this:

```dart
// gamepad_button.dart
export 'gamepad_button_interface.dart';
    if (dart.library.io) 'gamepad_button_io.dart'
    if (dart.library.html) 'dart:html'
    show GamepadButton;
```

At runtime, this actually works. There's nothing saying we can't use a "dart:"
library as a configuration-specific one. Well, it *would* work, that is, if we'd
spelled `pressed` correctly. There's a typo in our class. The real GamepadButton
uses `pressed`, but our class calls it `isPressed`.

This is the kind of stuff we'd like to catch with static checking. Doing that
requires being able to check configuration-specific *types* for compatibility in
addition to functions.

We change the above checks to:

**To tell if two libraries are compatible:**

1.  Produce the *visible namespace* of each library. This is the export
    namespace of the library after the `show` and `hide` combinators in the
    directive have been applied.
2.  If the maps of names to members of the two visible namespaces are not
    compatible, fail.

**To tell if two maps of names to members A and B are compatible:**

1.  If the maps do not have the same sets of names, fail.
2.  For each corresponding pair of members A.m and B.m:
    1.  If A.m and B.m are not the same *kind*, fail. "Kind" is which kind of
        declaration the member is: typedef, class, enum, constructor, method,
        getter, setter. Note that variables are not a kind because a variable
        is just a getter (final) or pair of getter and setter (non-final).
    2.  If A.m is a typedef:
        1.  If their type parameter lists are not compatible, fail.
        2.  If their return types are not compatible, fail.
        3.  If their parameter lists are not compatible, fail.
    3.  Else if A.m is a class:
        1.  If A.m and B.m are not compatible (see below), fail.
    4.  Else if A.m is a enum:
        1.  If the list of enum values in A.m and B.m are not identical, fail.
    5.  Else if A.m is a constructor:
        1.  If A.m is generative and B.m is not, or vice versa, fail.
        2.  If A.m is const and B.m is not, or vice versa, fail.
        3.  If their parameter lists are not compatible, fail.
    6.  Else if A.m is a method:
        1.  If A.m is abstract and B.m is not, or vice versa, fail.
        2.  If their return types are not compatible, fail.
        3.  If their parameter lists are not compatible, fail.
    7.  Else if A.m is a getter:
        1.  If A.m is abstract and B.m is not, or vice versa, fail.
        2.  If their return types are not compatible, fail.
    8.  Else (A.m is a setter):
        1.  If A.m is abstract and B.m is not, or vice versa, fail.
        2.  If their value types are not compatible, fail.

**To tell if two types A and B are compatible:**

1.  If A and B are both `void`, succeed.
2.  Else if A and B are both named types:
    1.  If their type argument lists are not compatible, fail.
    2.  If they refer to the same declaration, succeed.
    3.  Else if A refers to a type in the visible namespace of one of the
        libraries being checked and B refers to a type with the same name in the
        other library, succeed. (These types will themselves also be checked for compatibility as we walk the members of the library above.)
    4.  Else fail.
3.  Else if A and B are function types:
    1.  If the return types are not compatible, fail.
    2.  If their parameter lists are not compatible, fail.
4.  Else fail (they are different kinds of types).

And we add:

**To tell if two lists of type parameters are compatible:**

1.  If the lists have different lengths, fail.
2.  For each lockstep pair of type parameters:
    1. If one has an upper bound and the other does not, fail.
    2. If the upper bounds are not compatible, fail.
3.  Otherwise, succeed.

**To tell if two classes are compatible:**

1.  If one is abstract and the other is not, fail.
1.  If their type parameter lists are not compatible, fail.
2.  If their superclasses are not compatible, fail.
3.  If their list of mixins are not compatible, fail.
4.  If their list of implemented interfaces are not compatible, fail.
5.  If their maps of names to public constructors are not compatible, fail.
6.  If their maps of names to public instance members (including inherited
    ones) are not compatible, fail.
7.  If their maps of names to public static members are not compatible, fail.

*TODO: Can we ignore private classes in superclass, superinterfaces, and mixins?*

## Alternatives

Since we've been talking about "configuration-specific" code for literally
years, there are a large number of other proposals and variations. This proposal
is a (hopefully harmonious) synthesis of the smart ideas of others.

### "Configured Imports"

Its most direct inspiration is Lasse's [Configured Imports][] proposal. This
proposal has almost identical runtime semantics. It differs in a few ways:

 1. The main change, suggested by Florian, is that this proposal *requires* a
    default "unconfigured" library, the "interface library". This simplifies
    static analysis because there is always a single source of truth when
    looking at the import or export statically.

 2. The syntax for this one is different and, I hope, more easily understood.

 3. This proposal allows both `import` and `export` directives to be configured.

[configured imports]: https://github.com/lrhn/dep-configured-imports

### Function-only "Configured Imports"

Erik was one of the first to think deeply about the complexity in statically
analyzing configured public types to be configured. His [variation of Configured
Imports][variation] solves that problem neatly and is much simpler. This
proposal's "Phase 1" is based directly off that, though with different syntax.

[variation]: https://github.com/eernstg/dep-configured-imports

### "Import When"

Natalie's [Import When][] proposal was one of the first to introduce the idea of
a pure interface view of a library. Her proposal has Dart create those
implicitly for any library, allowing a library whose *implementation* uses a
platform-specific feature to still have its public types used as interfaces even
on other platforms.

[import when]: https://github.com/nex3/dep-import-when

Supporting this for all libraries, even "dart:io" and "dart:html" is very
powerful and would let, for example, existing code that consumes "dart:html"
objects able to be run on the standalone VM provided you give it objects whose
implementations don't require "dart:html". In other words, it lets you use
"dart:html" *type annotations*, even on the standalone VM.

However, supporting these interface implicitly, for all "dart:" libraries adds a
size burden on Dart implementations that may be resource-constrained. It also
isn't clear how this would handle new "dart:" libraries provided by custom
embedders like "dart:sky".

### "External libraries"

The [External Libraries][] proposal is a riff on the unspecified "patch files"
system that our core libraries currently use. It's structurally very similar to
this proposal: you have a library whose public signature is an "interface" and
one or more configuration-specific libraries that implement that.

The main difference is that with that proposal, the "library signature" is
declared in place in the library that references the configuration-specific
libraries instead of in a separate file. This is a bit more terse, but makes the
runtime semantics more confusing to specify.

Is the configuration-specific library *exported* from the canonical library or
*textually included*, or something else? All answers lead to confusing corner
cases. Realizing this is what led me to abandon external libraries in favor of
the proposal here.

[external libraries]: https://github.com/munificent/dep-external-libraries

## Implications and limitations

**TODO**

## Deliverables

**TODO**

### Language specification changes

**TODO**

### Prototype implementation

The [prototype][] directory contains patches for the [Dart SDK repository][sdk].

Currently, they implement the parsing and runtime behavior for the standalone VM
and dartj2. However, instead of allowing full condition expressions, they just
support a single `vm` or `js` identifier. The former is true in the standalone
VM, and the latter is true in dart2js.

In addition, there is a patch to the analyzer that lets it parse and ignore the
syntax. Fuller support for static analysis is coming.

[prototype]: https://github.com/munificent/dep-interface-libraries
[sdk]: https://github.com/dart-lang/sdk

### Tests

**TODO**

## Patents rights

TC52, the Ecma technical committee working on evolving the open [Dart standard][], operates under a royalty-free patent policy, [RFPP][] (PDF). This means if the proposal graduates to being sent to TC52, you will have to sign the Ecma TC52 [external contributer form][] and submit it to Ecma.

[tex]: http://www.latex-project.org/
[language spec]: https://www.dartlang.org/docs/spec/
[dart standard]: http://www.ecma-international.org/publications/standards/Ecma-408.htm
[rfpp]: http://www.ecma-international.org/memento/TC52%20policy/Ecma%20Experimental%20TC52%20Royalty-Free%20Patent%20Policy.pdf
[external contributer form]: http://www.ecma-international.org/memento/TC52%20policy/Contribution%20form%20to%20TC52%20Royalty%20Free%20Task%20Group%20as%20a%20non-member.pdf
