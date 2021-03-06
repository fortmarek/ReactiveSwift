# Framework Overview

This document contains a high-level description of the different components
within the ReactiveSwift framework, and an attempt to explain how they work
together and divide responsibilities. This is meant to be a starting point for
learning about new modules and finding more specific documentation.

For examples and help understanding how to use RAC, see the [README][] or
the [Design Guidelines][].

## Events

An **event**, represented by the [`Event`][Event] type, is the formalized representation
of the fact that _something has happened_. In ReactiveSwift, events are the centerpiece
of communication. An event might represent the press of a button, a piece
of information received from an API, the occurrence of an error, or the completion
of a long-running operation. In any case, something generates the events and sends them over a
[signal](#signals) to any number of [observers](#observers).

`Event` is an enumerated type representing either a value or one of three
terminal events:

 * The `value` event provides a new value from the source.
 * The `failed` event indicates that an error occurred before the signal could
   finish. Events are parameterized by an `ErrorType`, which determines the kind
   of failure that’s permitted to appear in the event. If a failure is not
   permitted, the event can use type `Never` to prevent any from being
   provided.
 * The `completed` event indicates that the signal finished successfully, and
   that no more values will be sent by the source.
 * The `interrupted` event indicates that the signal has terminated due to
   cancellation, meaning that the operation was neither successful nor
   unsuccessful.

## Signals

A **signal**, represented by the [`Signal`][Signal] type, is any series of [events](#events)
over time that can be observed.

Signals are generally used to represent event streams that are already “in progress”,
like notifications, user input, etc. As work is performed or data is received,
events are _sent_ on the signal, which pushes them out to any observers.
All observers see the events at the same time.

Users must [observe](#observers) a signal in order to access its events.
Observing a signal does not trigger any side effects. In other words,
signals are entirely producer-driven and push-based, and consumers (observers)
cannot have any effect on their lifetime. While observing a signal, the user
can only evaluate the events in the same order as they are sent on the signal. There
is no random access to values of a signal.

Signals can be manipulated by applying [primitives][BasicOperators] to them.
Typical primitives to manipulate a single signal like `filter`, `map` and
`reduce` are available, as well as primitives to manipulate multiple signals
at once (`zip`). Primitives operate only on the `value` events of a signal.

The lifetime of a signal consists of any number of `value` events, followed by
one terminating event, which may be any one of `failed`, `completed`, or
`interrupted` (but not a combination).
Terminating events are not included in the signal’s values—they must be
handled specially.

### Pipes

A **pipe**, created by `Signal.pipe()`, is a [signal](#signals)
that can be manually controlled.

The method returns a [signal](#signals) and an [observer](#observers).
The signal can be controlled by sending events to the observer. This
can be extremely useful for bridging non-RAC code into the world of signals.

For example, instead of handling application logic in block callbacks, the
blocks can simply send events to the observer. Meanwhile, the signal
can be returned, hiding the implementation detail of the callbacks.

## Signal Producers

A **signal producer**, represented by the [`SignalProducer`][SignalProducer] type, creates
[signals](#signals) and performs side effects.

They can be used to represent operations or tasks, like network
requests, where each invocation of `start()` will create a new underlying
operation, and allow the caller to observe the result(s). The
`startWithSignal()` variant gives access to the produced signal, allowing it to
be observed multiple times if desired.

Because of the behavior of `start()`, each signal created from the same
producer may see a different ordering or version of events, or the stream might
even be completely different! Unlike a plain signal, no work is started (and
thus no events are generated) until an observer is attached, and the work is
restarted anew for each additional observer.

Starting a signal producer returns a [disposable](#disposables) that can be used to
interrupt/cancel the work associated with the produced signal.

Just like signals, signal producers can also be manipulated via primitives
like `map`, `filter`, etc.
Every signal primitive can be “lifted” to operate upon signal producers instead,
using the `lift` method.
Furthermore, there are additional primitives that control _when_ and _how_ work
is started—for example, `times`.

## Observers

An **observer** is anything that is waiting or capable of waiting for [events](#events)
from a [signal](#signals). Within RAC, an observer is represented as an [`Observer`][Observer] that accepts [`Event`][Event] values.

Observers can be implicitly created by using the callback-based versions of the
`Signal.observe` or `SignalProducer.start` methods.

## Lifetimes

When observing a signal or starting a signal producer, it is important to consider how long the observation should last. For example, when observing a signal in order to update a UI component, it makes sense to stop observing it once the component is no longer on screen. This idea is expressed in ReactiveSwift by the `Lifetime` type.

```swift
import Foundation

final class SettingsController {
  // Define a lifetime for instances of this class. When an instance is
  // deinitialized, the lifetime ends.
  private let (lifetime, token) = Lifetime.make()

  func observeDefaultsChanged(_ defaults: UserDefaults = .standard) {
    // `take(during: lifetime)` ensures the observation ends when this object
    // is deinitialized
    NotificationCenter.default.reactive
      .notifications(forName: UserDefaults.didChangeNotification, object: defaults)
      .take(during: lifetime)
      .observeValues { [weak self] _ in self?.defaultsChanged(defaults) }
  }

  private func defaultsChanged(_ defaults: UserDefaults) {
    // perform some updates
  }
}
```

The `token` is a `Lifetime.Token`, which we need to keep a strong reference to in order for the `Lifetime` to work.
(Note: It's crucial that there is only a single strong reference to `token`, so that it is deinitialized at the same time as `self`.)

`Lifetime` is useful any time that an observation might outlive the observer:

- In the `NotificationCenter` example above, without a `Lifetime` the observation would never complete (leaking memory and wasting CPU cycles).
- Consider a signal producer that fires a network request — incorporating a `Lifetime` might allow the request to be automatically cancelled if the observer is deinitialized.

## Actions

An **action**, represented by the [`Action`][Action] type, will do some work when
executed with an input. While executing, zero or more output values and/or a
failure may be generated.

Actions are useful for performing side-effecting work upon user interaction, like when a button is
clicked. Actions can also be automatically disabled based on a [property](#properties), and this
disabled state can be represented in a UI by disabling any controls associated
with the action.

## Properties

A **property**, represented by the [`PropertyProtocol`][Property],
stores a value and notifies observers about future changes to that value.

The current value of a property can be obtained from the `value` getter. The
`producer` getter returns a [signal producer](#signal-producers) that will send
the property’s current value, followed by all changes over time. The `signal` getter returns a [signal](#signals) that will send all changes over time, but not the initial value.

The `<~` operator can be used to bind properties in different ways. Note that in
all cases, the target has to be a binding target, represented by the [`BindingTargetProvider`][BindingTarget]. All mutable property types, represented by the  [`MutablePropertyProtocol`][MutableProperty], are inherently binding targets.

* `property <~ signal` binds a [signal](#signals) to the property, updating the
  property’s value to the latest value sent by the signal.
* `property <~ producer` starts the given [signal producer](#signal-producers),
  and binds the property’s value to the latest value sent on the started signal.
* `property <~ otherProperty` binds one property to another, so that the destination
  property’s value is updated whenever the source property is updated.

Properties provide a number of transformations like `map`, `combineLatest` or `zip` for manipulation similar to [signal](#signals) and [signal producer](#signal-producers)

## Disposables

A **disposable**, represented by the [`Disposable`][Disposable] protocol, is a mechanism
for memory management and cancellation.

When starting a [signal producer](#signal-producers), a disposable will be returned.
This disposable can be used by the caller to cancel the work that has been started
(e.g. background processing, network requests, etc.), clean up all temporary
resources, then send a final `interrupted` event upon the particular
[signal](#signals) that was created.

Observing a [signal](#signals) may also return a disposable. Disposing it will
prevent the observer from receiving any future events from that signal, but it
will not have any effect on the signal itself.

For more information about cancellation, see the RAC [Design Guidelines][].

## Schedulers

A **scheduler**, represented by the [`Scheduler`][Scheduler] protocol, is a
serial execution queue to perform work or deliver results upon.

[Signals](#signals) and [signal producers](#signal-producers) can be ordered to
deliver events on a specific scheduler. [Signal producers](#signal-producers)
can additionally be ordered to start their work on a specific scheduler.

Schedulers are similar to Grand Central Dispatch queues, but schedulers support
cancellation (via [disposables](#disposables)), and always execute serially.
With the exception of the [`ImmediateScheduler`][Scheduler], schedulers do not
offer synchronous execution. This helps avoid deadlocks, and encourages the use
of [signal and signal producer primitives][BasicOperators] instead of blocking work.

Schedulers are also somewhat similar to `NSOperationQueue`, but schedulers
do not allow tasks to be reordered or depend on one another.


[Design Guidelines]: APIContracts.md
[BasicOperators]: BasicOperators.md
[README]: ../README.md
[ReactiveCocoa]: https://github.com/ReactiveCocoa/
[Signal]: ../Sources/Signal.swift
[SignalProducer]: ../Sources/SignalProducer.swift
[Action]: ../Sources/Action.swift
[CocoaAction]: https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/ReactiveCocoa/CocoaAction.swift
[Disposable]: ../Sources/Disposable.swift
[Scheduler]: ../Sources/Scheduler.swift
[Property]: ../Sources/Property.swift
[MutableProperty]: ../Sources/Property.swift#L28
[Event]: ../Sources/Event.swift
[Observer]: ../Sources/Observer.swift
[BindingTarget]: ../Sources/UnidirectionalBinding.swift
