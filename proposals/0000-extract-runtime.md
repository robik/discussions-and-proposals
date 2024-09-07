---
title: Extraction of runtime layer
author: Robert Pasiński <robert.pasinski@callstack.com>
date: 07-09-2024
---

# RFC0000: Extraction of Runtime layer

## Summary

Main goal of this RFC is to create a new layer in RN stack, that lives between
Hermes (jsi) and React Native.

Shortly, the goals are:

 - Extract the React Native non-ui logic into separate package
 - Keep compatibility with existing code as much as possible
 - Make long term maintenance, versioning and updating easier
 - Make React Native more inclusive towards OOT platforms

The scope of this is pretty big, because of that old architecture is not considered.

## Motivation

React Native is a very big and complex project. It still continues to grow in size 
and complexity and even more things are planned to be added, or are expected to be.

By splitting the current project into smaller pieces, with explicit API boundaries,
we should be able to simplify the maintenance and testing in the long run. This, of course would require
us to specify a "stable" interface to work with. This mosty probably won't be achieved easily and promptly,
but is a good long term goal to try to achieve.

Understanding the current dependencies and boundaries is also quite problematic and raises the entry barrier to contributing
to the React Native project.

Right now, updating the React Native projects is a long and troubling process.
This is (non exclusively) caused by the fact that the scope of changes between version goes across many API surfaces.
Having the (limited) possibility to update the runtime version in a project independently of UI would make it
more approachable by reducing the all-or-nothing upgrading pattern.

Additionally, there are quite a few missing pieces in the current ecosystem, that from what I now, are hard to add 
for different reasons, such as `i18n`, `subtleCrypto` and others. There is also 
an ongoing effort to increase the compatibility with web environments by bringing 
the WebAPIs to the React Native. 
It should be possible to alter some parts of the React Native without creating and maintaining
a fork or a long list of internal patches.

Finally, working on bringing new APIs to React Native from Web requires us to create UI to even be able to test them, 
or otherwise is very hard to do.

### Goals

**The main goal is to make it possible for end-users, by altering build and runtime configuration,
to create perfectly fit runtime variant they need for the application.**

> __NOTE__: We are still talking about headless APIs, not related to platform UIs.

To do this, we should have an extensible layer, that can be built upon and configured, just like LEGO blocks. 

For example, if the project is in need for the I18n API (and therefore ICU), and they are ready to accept the 
drawbacks of increased app binary, it should be easily doable, without forking the React Native.
The same can be said about other APIs, such as `crypto`, where they might want to use our
provided solution, not use it at all, or use in-house closed source solution.

The same can be said about changing the JS engine. The proposed layer would take care of allowing user
to use another engine by using the already existing `jsi` interface. JS engine should be treated
like a building block similar to others that can be changed easily by build time configuration.

Right now, we have something similar already: the C++-only TurboModules, which are kind of tightly coupled 
with the rest of the React Native and the distinction can be quite confusing for newcomers.

Such blocks would have access to Event Loop and JS engine, so new globals can be added in as-needed
basis.

__Secondary goal is to be able to develop and test those headless modules easily, on the CI pipeline without visual environment__.

By having the possibility to run the "blocks" (C++ only turbo modules) in a headless JS environment we
are able to test the implementations easily on many CI platforms and on different JS engines (ideally).

__Tertiary goal is to bring OOT platforms closer to first-class citizens__

If a Turbo Module does not need access to UI in any way to provide its features, having it work with `react-native-runtime`
rather than `react-native` would make it easier to maintain for currently OOT platforms (such as rn-windows and rn-mac) where
the current `react-native` code cannot be re-used in 100% (needs further clarification). 
Right now we are biased towards mobile platforms only.

## Detailed design

> **NOTE**: Because we are talking about headless code, TurboModules refer to C++ only TurboModules unless
> otherwise noted.

### Overview

Before proceeding to the description of the suggested changes, let's consider an example of a React Native application 
with additional OOT platform can be represented in current state (more or less):

![Current State Diagram](./assets/0000-rn-diagram-current-oot.png)

The proposed `react-native-runtime` layer would be everything from React Native that is
independent of the UI layer (so essentially everything headless) with enough extension points and mount
points to make it elastic enough to be extended by UI layer or platform layer, if needed.

This includes, but is not limited to:

 - Event Loop
 - JS engine management
 - C++-only Turbo Module hosting

And ideally (out of scope for this RFC):
 - DevTools interface and debugging host (long term, after interface gets stabilized)

Having a runtime layer extract some of the pieces to shared layer:

![Diagram with Runtime](./assets/0000-diagram-with-runtime.png)

Now, with React Native layer focusing solely on UI layer, having an additional OOT platform would make the former diagram 
look more like the following (with additional APIs the user decided to add):

![Diagram with Runtime and OOT](./assets/0000-diagram-runtime-multiplatform.png)

The new layer would take care of `jsi` interface surface and let React Native (core/UI)
focus on working with UI and executing the user code, no matter the engine is currently selected.

At the heart of `react-native-runtime` would be the event-loop and JS interface, on which other building blocks 
would be based on.

This would, for example, allow community to implement custom storage solution using `localStorage` API,
ideally matching the DOM spec. By having the DevTools Interface as part of this package, they can also
hook up into `Storage` tab from `Dev tools` to provide excellent Developer Experience.

The same can be done with e.g. networking, where C++-modules would be able to hook up to `Networking`
tab to report and display network requests to provide the best DX possible.

### Runtime Composability

The goal of runtime composability is to essentially provide the features you need without paying for
ones you do not use.

This can be achieved by updating the process of how TurboModules are discovered and handled at build time level.
Considering these TurboModules aim to expose new APIs to the JS world, we could make them conditionally linked
(either statically or dynamically).

There are two approaches we can further discover:

 - Leveraging dynamic library constructor and destructor functions (`attribute(constructor)`/`attribute(destructor)`)
   
   This can be hard to do right, but ideally would be build-time only without any changes required in code or at runtime.
  
 - Exposing a Well known function that registers itself in React Native runtime 
   
   Similar to what we are doing now with TurboModules, we expect the library to export a function that we call in order to 
   augment to runtime. This is probably more compatible solution but also requires having a collection of expected 
   modules in code in order to load them.

Application develop should be able to decide which of the libraries should be linked. React Native would then
depend on the built runtime library.

### Per-Platform implementation

Suggested approach still allows developers to have runtime level TurboModules use platform specific code
by linking different libraries depending on the target platform.

### Codegen



## Drawbacks

 - Extracting API surface layer is a big task cost and time-wise

## Alternatives

None considered that would accomplish the same goals.

## Adoption strategy

If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries?

## How we teach this

Firstly, we should update the documentation to highlight the strict separation of layers
and the concept of headless, UI-independent runtime.
New documentation section can be added that shows what layer are APIs part of.

We should also document the fact that APIs which are part of the runtime are 
common to all platforms, whenever they are OOT platforms or not.

Perhaps we can use different terminology for C++ TurboModules which would be executed on runtime
and UI TurboModules which would be part of UI layer (`react-native`/`react-native-core`).

What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?

As for teaching new React Native developers, we can present our runtime as an equivalent to node-js, which
well known and documented runtime,

## Unresolved questions

 - What should be exactly the ideal API surface for `react-native-runtime` package?
 - What are the requirements and constraints to make proposed solution and improvement for OOT platforms.
 - How should it be versioned?
 - Should be strongly separate TurboModules (C++ only) from UI TurboModules? After extraction,
   they would work on different layers.

## Future opportunities

 - Possible NAPI (Node native API) integration with `react-native-package`
 - Because TurboModules would be UI agnostic, we could expose the C API for Turbo Modules, which will make
   us able to write turbo modules using other native languages such Rust, Zig, Go, Swift and any other language that 
 - is compatible with C API, which most of them are.
 - We can implement support for `package.json` `engines` field to ensure correct version
   of runtime is used.

### Impacted RFCs/ideas

This RFC alters (hopefully in a good way) how can be approach future ideas, such as:

 - Lean Core (2.0)
 - WebAPI implementation
 - Better OOT support
 - React Native Frameworks

## Changelog

| Date       | Author                                      | Change                    |
|------------|---------------------------------------------|---------------------------|
| 2024-09-07 | [Robert Pasiński](https://github.com/robik) | Initial version published |
