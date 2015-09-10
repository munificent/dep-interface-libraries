The patches here are a very rough prototype implementation of the proposed
syntax. They patch the Dart VM, dart2js compiler, and analyzer.

All three implementations only allow a single identifier inside the `if()`. On
the VM, it will look for a configuration whose identifier is `vm` and use that
over the default URL. On dart2js, it will do the same but for a `js`
configuration. All other configurations are ignored, and environment constants
are not used.

On the analyzer, it will parse the syntax but ignore all of the configurations
and use the default URI. It doesn't do any static validation of the
configuration-specific libraries.

Apply the patches to the [Dart SDK repository][sdk] at commit
645b809e636b7a396d805b2f8cc707ec3549c3e3.

[sdk]: https://github.com/dart-lang/sdk
