---
layout: post
title: "ViewModel in Flutter"
date: 2025-04-14 09:48:55 +0200
tags: [Flutter]
style: fill
color: secondary
comments: false
---
The term ViewModel is rather vague when applied to Flutter. Although it’s frequently
mentioned in documentation and technical interviews, there’s no actual ViewModel
class or a class that can clearly be identified as one. Typically, state management
frameworks try to play that role — but it feels forced or artificial.

During my recent work on a few Flutter projects, I feel like I’ve arrived at an extremely
lightweight but powerful code snippet that generally offers the same capabilities I was used
to in Android Compose UI projects.

So, without further ado, here we go.

## View model as abstraction layer between UI and business logic

```dart
class Feature extends ViewModelWidget<FeatureViewModel> {
  const Authorization({super.key, factory = FeatureViewModel.factory});
      : super(factory: factory);

  @override
  Widget build(BuildContext context, viewModel) {...}
```

This is how the view model assumed to be instantiated and used.
`ViewModelWidget` extends `StatelessWidget` in order to get access to the
life cycle of the widgets and this is a real core of the whole approach.

The `build` method passing the `viewModel` as a parameter is a convenient way to
get the reference to the created view model instance.

The `factory` argument of the `ViewModelWidget` constructor obviously is supposed to
be called on demand to create the instance. Being learned горький урок from Android's
view models experience, I intentionally avoided attempts to deduce the factories method
and decided to make it explicit. 

The expected use of the view model pattern here is we can gather the
methods and properties that are needed by a widget in the `viewModel` class and access then as easy
as

```dart
viewModel.someProperty
```
and
```dart
viewModel.someMethod()
```

And here we get the first benefit: abstraction of business logic from the UI code. The ViewModel’s API
is the only thing a UI developer needs to work with. Moreover, we can modify the business logic without
affecting the UI code — as long as the API stays the same.

## Defining a Feature ViewModel class

Defining a ViewModel is as simple as defining any class, with the addition of a single static
factory method(which is just a convenience, not a requirement). This factory is to be passed
to the `ViewModelWidget` constructor and called to create the instance of the ViewModel.

```dart
class FeatureViewModel {
  // some properties and methods
  // ...
  
  // The context passed to the factory method
  // can be used to access the dependencies
  // and create the instance of the other resources
  static FeatureViewModel factory(BuildContext context) {
    // Create the dependencies and resources
    // ...
    return FeatureViewModel(context, /*dependencies, resources...*/);
  }
}
```

Now the time to get more use of the suggested approach

## Init, Dispose

As you should remember the `ViewModelWidget` inherited from `StatefulWidget` and got access to the
life cycle methods. We can signal our view model is interested in the life cycle events as well by
adding the interfaces `ViewModelWithInit`, `ViewModelWithDispose` and `ViewModelWithProviders` in
any combination.

```dart
class FeatureViewModel implements ViewModelWithInit, ViewModelWithDispose {
  
  @override
  void init(BuildContext context) {...}
  
  @override
  void dispose() { ... };
}
```

These `Init` and `Dispose` are trivial methods what do not need any explanation.
With them, it becomes possible to subscript to the streams, websockets, queues, etc. on demand 
and unsubscribe as soon as the widget goes out of the scope.

But there's the 3rd `providers` getter which is a bit more interesting.

##  Provide

```dart
class FeatureViewModel implements ViewModelWithProviders {
  
  @override
  List<SingleChildWidget> get providers => [...];
}
```

First, I assume the use of the `provider` package, as it's the only
DI/state management solution I find truly worth using in Flutter. If someone prefers a different frameworks,
they can easily modify `ViewModelWidget` by redefining and implementing their own `providers` getter
— it’s not a big deal.

Second it is nothing but just a wrapper around the `MultiProvider.providers` and its purpose is to
add the services to the context down to the hierarchy of the widgets started from the `ViewModelWidget`
instance.

Here the 3rd use: locally scoped services - no app lifetime services anymore.

## Access the view model instance

In the beginning I already mentioned the `ViewModelWidger.build` method that passes the
`viewModel` instance as its argument and nothing prevents anybody just to inherit their
widgets from `ViewModelWidget` and use its `build` method as many as needed. Due to reference
counting the same instance will be passed and extra cost will be paid only for the `StatefullWidget`
creation.

Meanwhile, it seemed too expensive and unnatural to me, so I decided to add the instance
of `FeatureViewModel` to the build context. This way, it can be accessed as
easily as `Provider.of<FeatureViewModel>(context)` in any widget down the hierarchy,
starting from the `ViewModelWidget` instance.

and there's syntax sugar for that:

```dart
extension ViewModelExtension on BuildContext {
  T viewModel<T>() {
    return Provider.of<T>(this, listen: false);
  }
}
```

That closely resembles the `viewModel` method from Android Compose UI and makes retrieving the
view model instance even a little bit cleaner:

```dart
final vm = context.viewModel<FeatureViewModel>();
```

... and even more beautiful:

```dart
extension FeatureViewModelExtension on BuildContext {
  FeaturenViewModel get authorizationViewModel =>
      viewModel<AuthorizationViewModel>();
}
```

## Conclusion

With this view model approach, I can access a bunch of properties and methods needed for a given widget,
create resources with lifetimes scoped to the widget (like `TextEditingController` instances, which
are the most common example), and—most importantly for me, as someone addicted to reactive
programming—create streams and provide their data to widgets just as easily as I could with React
hooks or Android Compose UI states.

 > And that's all about the approach itself, the further description is about implementation
 > details not everyone might be interested in.

## Tech internals

## Reference counting

Like Android Compose UI the suggested implementation of `ViewModelWidget` tracks the use of
the view models of the same class and creates the new instances only one reusing them on the
sequential factory method calls.

## The reference implementation

Due to its simplicity, I prefer to keep the entire source code snippet in a single file that I
can copy from one project to another, rather than creating a full library or package. Since the
idea is still fresh (at least to me, as a Flutter newcomer), this approach offers flexibility—for
me and others—to add tweaks and additional functionality that can make it even more useful.

```dart
library;

import 'package:flutter/widgets.dart';
import 'package:logging/logging.dart';
import 'package:provider/provider.dart';
import 'package:provider/single_child_widget.dart';

/// An interface view model might implement
/// if it has to run some code on initialization
abstract interface class ViewModelWithInit {
  void init(BuildContext context) {}
}

/// An interface view model might implement
/// if it has to run some code on dispose
abstract interface class ViewModelWithDispose {
  void dispose() {}
}

/// An interface view model might implement
/// if it has to run some code on dispose
abstract interface class ViewModelWithProviders {
  List<SingleChildWidget> get providers;
}

abstract class ViewModelWidget<VM> extends StatefulWidget {
  final VM Function(BuildContext) _factory;

  const ViewModelWidget({super.key, required factory}) : _factory = factory;

  Widget build(BuildContext context, VM viewModel);

  @override
  State<ViewModelWidget<VM>> createState() => _ViewModelState<VM>();
}

class _ViewModelState<VM> extends State<ViewModelWidget<VM>> {
  late final VM vm;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    vm = _registry.get<VM>(context, widget._factory);
  }

  @override
  void dispose() {
    if (vm is ViewModelWithDispose) {
      (vm as ViewModelWithDispose).dispose();
    }
    _registry.release<VM>();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (Provider.of<VM?>(context, listen: false) == null) {
      if (vm is ViewModelWithProviders) {
        final List<SingleChildWidget> providers = [];
        providers.addAll((vm as ViewModelWithProviders).providers);
        providers.add(Provider(create: (_) => vm));
        return MultiProvider(
          providers: providers,
          child: Builder(
            builder: (context) {
              return widget.build(context, vm);
            },
          ),
        );
      } else {
        return Provider<VM>(
          create: (_) => vm,
          child: Builder(
            builder: (context) {
              return widget.build(context, vm);
            },
          ),
        );
      }
    }
    return Builder(
      builder: (context) {
        return widget.build(context, vm);
      },
    );
  }

  static final _ModelViewRegistry _registry = _ModelViewRegistry();
}

class _ModelViewRegistry {
  final Map<Type, _ModelViewReference> _instances = {};

  VM get<VM>(BuildContext context, VM Function(BuildContext) factory) {
    final existing = _instances[VM];
    if (existing == null) {
      final vm = factory(context);
      if (vm is ViewModelWithInit) {
        (vm as ViewModelWithInit).init(context);
      }
      // NOTE: it won't be added if init has thrown
      final ref = _ModelViewReference(vm, context);
      log.finest("instantiated $ref");
      _instances[VM] = ref;
      return vm;
    }
    log.finest("reuse $existing");
    if (existing.counter == 0) {
      log.severe(
          "The $VM instance is referenced 0 times, this should never happen during referencing");
    }
    if (!isDescendant(existing.context, context)) {
      // TODO: it is assumed all the models of the same class are
      // created within the same hierarchy, but it can be extended
      // if the map key contains both ViewModel class and context
      log.severe(
          'The ViewModel\'s context is not a descendant of the context it was instantiated first.');
    }
    final vm = existing.vm as VM;
    existing.counter += 1;
    return vm;
  }

  void release<VM>() {
    final existing = _instances[VM];
    if (existing == null) {
      log.severe("There's no previously initialized view model $VM");
      return;
    }
    log.finest("release $existing");
    if (existing.counter <= 0) {
      log.warning(
          "The $VM instance is referenced ${existing.counter} times, this should never happen during release");
      // try to remove it anyway
      existing.counter = 1;
    }
    existing.counter -= 1;
    if (existing.counter == 0) {
      _instances.remove(VM);
      if (existing is ViewModelWithDispose) {
        (existing as ViewModelWithDispose).dispose();
      }
    }
  }

  static bool isDescendant(BuildContext descendant, BuildContext ancestor) {
    BuildContext? current = descendant;
    while (current != null) {
      if (current == ancestor) {
        return true;
      }
      current = current.findAncestorStateOfType<State>()?.context;
    }
    return false;
  }
}

class _ModelViewReference<VM> {
  final VM vm;
  final BuildContext context;
  int counter = 1;

  _ModelViewReference(this.vm, this.context);

  @override
  String toString() {
    // TODO: implement toString
    return "$VM (reference counter=$counter)";
  }
}

final log = Logger("ViewModel");

extension ViewModelExtension on BuildContext {
  T viewModel<T>() {
    return Provider.of<T>(this, listen: false);
  }
}
```