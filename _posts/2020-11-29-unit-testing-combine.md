---
layout: post
title:  "Unit Testing with Combine"
date:   2020-11-28 12:00:00 +0500
---

There's lots of excitment around reactive programming on iOS with last year's introduction of the Combine framework from Apple.  Since then there's been a wide variety of tutorials, explanations, and discussions around how to code using Combine.  Unfortunatley there's not been a ton of talk around how to properly unit test code that utilizes Combine.   This post aims to show why unit testing a reactive stream is different than imperative code and provide an example of how this could be better tested than using usual methods.
  <br>
  <br>
# (Not So) Great Expectations
When learning how to unit test Combine code the 1st thing that most people try to do is use expectations.  This is natural because most combine code is asynchronous and unit tests need some way of handling this asynchrony.  It's also something people are already familiar with so it's the 1st tool they will reach for.   While it can work, expectations present two major problems when testing reactive code:

1. They have a built-in wall clock timing which can be problematic.
2. They are usually only used to indicate that an expected reactive event was received.   They only indicate that something happened at one given point in time.

  <br>
### Expect Timing Issues
Using an expectation to track asynchronous work can be problematic because it waits according to actual wall clock time.  This means actual time must elapse during your test in order for the test to proceed.  If you have hundreds or thousands of unit tests even minute amounts of time can add up can cause your unit tests to take a long time to run.  Another problem with it is that you must specify a timeout.   How long is enough?  Maybe some slow code makes it take longer than you alloted for and your test fails because of it.  Or worse, you have introduced a bug that breaks a bunch of tests, each of which must now wait for the maximum alloted time before the tests can fail and move on.  This will certainly show up in how long it takes to run your tests!

  <br>
### Expect Multiple Events
A `Publisher` is different than most code in that it doesn't represent a value or an event.  It represents a *series of values or events over time*.  This is quite different than waiting for a single result, and it makes the usage of an expectation for testing quite confusing.   How do you know when you're done receiving events?  How do you know when you've waited long enough?  The guesswork for this can quickly become unintelligible.
  <br>
  <br>
  <br>
# Keen Observation
After a number of years of working with reactive code and specifically the wonderful testing support in RxSwift's [RxTest][rxtest] module, I've come to learn that unit testing a stream should capture all of its output so that the test can determine its complete shape.  This includes testing the values that come out of it, how many it emits, in what order, and possibly if the stream completed successfully or with an error.  Think about it, if you only test that you received the one event you were expecting, there could more events coming you didn't expect and subscribers and operators downstream are still going to react to them and do things you may not expect!

Let's assume that the `sut` (System Under Test) exposes a publisher with the following declaration:
```swift
let buttonIsEnabled: AnyPublisher<Bool, Never>
```
Assuming this button appears on a UI somewhere there's room for this stream to emit a number of values over time.  Perhaps it's disabled to start until the user enters valid data which would then enable the button.  Maybe further interactions from the user such as deleting the text she entered then disables the button again and further text entry then enables it once more.   Clearly this stream's behavior extends past a single event and instead tells a story of changing values over time.  A unit test for this could do something like the following:
```swift
var values = [Bool]()
var completedWithoutError = false
var completedWithAnError: Error?
_ = sut.buttonIsEnabled
    .sink { completion in
        if case let .failure(error) = completion {
            completedWithAnError = error
        } else {
            completedWithoutError = true
        }
    } receiveValue: { value in
        values.append(value)
    }

let expectedValues = [
    false,
    true,
    false,
    true
]
XCTAssertEqual(expectedValues, values)
XCTAssertTrue(completedWithoutError)
```

Note that we can now inspect each value that was emitted by the stream and compare the entire shape of these emissions to an expected sequence of values.  We can also verify that the stream completed, errored-out, or didn't complete at all.  This lets us test for the things we expect and also verify that things we didn't expect didn't happen.   

Another benefit here is that there are no timeouts or wall clock time so if something goes wrong with this test it will fail quickly and move on to the next test.  What a win!

A keen-eyed reader may notice that somehow these values are being emitted in our test without the test manipulating the `sut` to cause these events to occur during the test.  Lets expand on this test to account for that.
  <br>
  <br>
  <br>
# Manipulating the SUT to Drive Event Collection

The example above showed a button enabling/disabling over time due to some until-now-unexplained input.  Let's expand our `sut` which happens to be a view model for a screen that allows the user to go through the standard "I forgot my password" screen.   The user will enter an email address and press a button to ask the server to send them a link to reset their password.   The inputs for this screen are just a text field and a button for the user to press.   The button should be enabled when the user has entered something that looks like an email address and be disabled otherwise.   Here's what a simplified version of the view model's interface may look like:

```swift
struct ForgotPasswordViewModel {
    struct UIInputs {
        let emailAddressTextChanged: AnyPublisher<String, Never>
        let submitButtonTapped: AnyPublisher<String, Never>
    }

    let buttonIsEnabled: AnyPublisher<Bool, Never>
    init(uiInputs: UIInputs) {
        // ....
    }
}
```
This view model takes 2 publishers as inputs that represent the user changing the text in a text field and the user tapping the submit button, respectively.  The view model exposes an output publisher called `buttonIsEnabled` which we discussed in the previous section.

What we want to do in our unit test for `buttonIsEnabled` is to actually manipulate the text that the user has entered so we can test that `buttonIsEnabled` emits the proper values as it changes.  How do we provide this input in a test?  This is one of the few great uses for a `Subject`.

Subjects are special objects that can be both a publisher and a subscriber.  They should be used [sparingly][avoid-subjects] as they are imperative in nature and often lead people that are new to reactive programming to learn bad habits.  Unit tests, however, are a perfectly normal place to utilize them to serve as input streams for your tests.

Here's an enhanced version of our unit test that shows how to use this subject to fill in the missing events that were going to drive the test:

~~~swift
let emailAddressTextChangedTrigger = CurrentValueSubject<String, Never>("")
let uiInputs = ForgotPasswordViewModel.UIInputs(
    emailAddressTextChanged: emailAddressTextChangedTrigger.eraseToAnyPublisher(),
    submitButtonPressed: Empty().eraseToAnyPublisher()
)
let sut = ForgotPasswordViewModel(uiInputs: uiInputs)

var values = [Bool]()
var completedWithoutError = false
var completedWithAnError: Error?
_ = sut.buttonIsEnabled
    .sink { completion in
        if case let .failure(error) = completion {
            completedWithAnError = error
        } else {
            completedWithoutError = true
        }
    } receiveValue: { value in
        values.append(value)
    }

emailAddressTextChangedTrigger.send("user@company.com")
emailAddressTextChangedTrigger.send("notanemailaddress")
emailAddressTextChangedTrigger.send("a@b.c.d.com")

let expectedValues = [
    false,
    true,
    false,
    true
]
XCTAssertEqual(expectedValues, values)
XCTAssertNil(completedWithAnError)
XCTAssertFalse(completedWithoutError)
~~~

Lines 1-6 show the use of a `CurrentValueSubject` being used as input to the `sut`.   We use a `CurrentValueSubject` because they have a starting value and 
since we're modeling the input from a `UITextField` that makse sense because `UITextField` always has a value (`nil` or `""` being the default).  We pass
the subject as an input to our view model and then the test flows as it did before.  The only other difference is at line 22 where we manipulate the subject to
emulate the user changing the text in the text field and drive the `sut`.  These are the events that were "missing" in the previous version of the test.   Now this test is complete
in that it generates the needed input events and captures the output events so we can be sure that the `sut` operates correctly.
  <br>
  <br>
  <br>
# What about asynchrony?
So far our unit tests have not shown what to do with asynchronous behavior that our `Publisher` may exhibit.  This can come about in 2 different ways: using 
`subscribe:on:` or `receive:on:` to move event processing to a different thread/queue is one way and the other is when the publisher itself or code you're calling 
will move the work to another thread/queue in its own implementation.  We want to unit test such publishers but we don't want to pay the cost of waiting in 
real time for the result.  The answer here is `Scheduler`.   

In Combine, schedulers are an abstract way to indicate which queue/thread/operation should be the one processing events in a Combine chain.  When we utilize 
`subscribe:on:` or `receive:on:` in code we're testing we are making explicit use of a scheduler.  In tests we want to utilize a scheduler that is meant for 
testing so that we can control time!

I mentioned earlier that the RxTest portion of RxSwift has an amazing tool called [TestScheduler][rxtest-testscheduler], and unfortunately no such corresponding 
scheduler exsists in Apple's offering of Combine.  Luckily the folks at [pointfree.co][pointfree] have put together a [repo][combine-schedulers] that gives us two missing tools that
give us everything we need.  It containts a `TestScheduler` and it contains a type-erased abstraction of `Scheduler` called `AnyScheduler`.  For some reason Apple failed to 
provide this type-erasure like they did with `AnyPublisher` but we'll want that as well.

In order to make a module unit-testable we need to utilize dependency injection so that the tests can provide mocks or simple dependencies in order to test the code.  
`Scheduler` is no different in this regard.  Imagine that our view model wants to push work to a background queue to keep the UI thread responsive.  It will probably
utilize `subscribe:on:` and/or `receive:on:` passing them a scheduler.  If we've hardcoded the scheduler to something like this:

~~~swift
uiInputs.submitButtonPressed
    .receive(on: DispatchQueue.global())
    .flatMap { _ in
        // Do a bunch of background work
    }
~~~

then testing becomes difficult because we are forcing the work to operate on a different queue than the test is, forcing the test to wait like it had to using expectations.
The scheduler should be passed into the view model just like any other dependency.
  <br>
  <br>
  <br>
# Putting Tests on the Schedule
So lets expand our view model's definition so when the user taps the submit button to initiate the "forgot my password" flow that the view model has inputs and outputs for this process:

~~~swift
struct ForgotPasswordViewModel {
    struct UIInputs {
        let emailAddressTextChanged: AnyPublisher<String, Never>
        let submitButtonPressed: AnyPublisher<Void, Never>
    }

    let buttonIsEnabled: AnyPublisher<Bool, Never>
    let resetRequestAccepted: AnyPublisher<Void, Never>

    init(uiInputs: UIInputs,
         backgroundScheduler: AnySchedulerOf<DispatchQueue>,
         uiScheduler: AnySchedulerOf<DispatchQueue>,
         api: ServerApi
    ) {
        resetRequestAccepted = uiInputs.submitButtonPressed
            .receive(on: backgroundScheduler)
            .flatMap { _ in
                api.resetPassword(....)  // Call the API
            }
            .receive(on: uiScheduler)
            .eraseToAnyPublisher()
    }
}
~~~
On line 8 we have a new output publisher that will emit when the reset password request has been successfully received by our server api.  This can indicate to the coordinator that it's time to change to a different screen, or whatever makes sense for the successful flow.  We are also injecting the server API and a scheduler for the background work and one for the ui work (lines 11-13).  Note the use of the type-erased `AnyScheduler` type for the
schedulers.   This will allow production code that creates a view model to pass in schedulers based on the actual queues that are needed:

~~~swift
let viewModel = ForgotPasswordViewModel(uiInputs: uiInputs,
                                        backgroundScheduler: DispatchQueue.global().eraseToAnyScheduler(),
                                        uiScheduler: DispatchQueue.main.eraseToAnyScheduler(),
                                        api: serverApi)
~~~
This takes care of the production case, now what about the unit test?
  <br>
  <br>
  <br>
# TestScheduler
I mentioned before that in the unit test we want to "control time".  What I meant by that is the `TestScheduler` is a special type of scheduler that only executes work on its
queue when it's told to.  What this means in a test is that we can set up our publishers inputs and subscriptions and then tell the scheduler when and how far ahead
to move its "clock".   This allows us control when scheduled work occurs and gives us 2 major benefits:

1. We can schedule input events at particular timestamps during the test.
2. We can advance the clock on the scheduler manually to whatever timestamp we want and inspect the results of any queued operations.

  <br>
### Input
Using our earlier example test of enabling/disabling the submit button, we can use the `TestScheduler` to schedule at what point an input event occurs if we want:

~~~swift
let testScheduler = DispatchQueue.testScheduler
testScheduler.schedule(after: testScheduler.now.advanced(by: 1)) {
    emailAddressTextChangedTrigger.send("user@company.com")
}
~~~

Here we have set this input event to occur 1 tick in the future.  Later on when we're ready to process events we can tell the test scheduler to advance its clock:

~~~swift
testScheduler.advance(by: 1)
~~~

This moves the test scheduler's clock forward by 1 and causes it to process any events that have been scheduled to run at that point.  We should have received
a value in our `values` array that we had set up in that example.

~~~swift
XCTAssertEqual("user@company.com", values.first)
~~~

  <br>
### Scheduled work
The other major benefit of the `TestScheduler` is that any work that gets scheduled on it via `receive:on:` or `subscribe:on:` will be executed when the clock advances
after being queued.  In our unit test we would create our view model by passing in the `TestScheduler` for both the background and ui schedulers like so:

~~~swift
let sut = ForgotPasswordViewModel(uiInputs: uiInputs,
                                  backgroundScheduler: testScheduler.eraseToAnyScheduler(),
                                  uiScheduler: testScheduler.eraseToAnyScheduler(),
                                  api: apiMock)
~~~
Now when we run our unit test we can simply advance the clock 2 ticks:

~~~swift
testScheduler.advance(by: 2)
~~~

which causes the "background" work scheduled by the `recevie:on` to be executed in the 1st tick advance and the work scheduled by the change to the ui scheduler in
the 2nd `receive:on:` to be executed during the 2nd tick advance.

Let's bring it all together for a full view of the test....

  <br>
### All Together Now
So now that we know what the scheduling story sort of looks like lets see how a full unit test of the `resetRequestAccepted` publisher on our view model would look like.
We'll use a similar setup as we did for the `buttonIsEnabled` test we did earlier and we'll be testing the success case where the user taps the submit
button and the request is made to the server correctly which should result in the view model's `resetRequestAccepted` publisher emitting one event.

~~~swift
let userTappedSubmitdTrigger = PassthroughSubject<Void, Never>()
let testScheduler = DispatchQueue.testScheduler
testScheduler.schedule(after: testScheduler.now.advanced(by: 1)) {
    userTappedSubmitdTrigger.send(())  // Simulate user tapping submit
}

let uiInputs = ForgotPasswordViewModel.UIInputs(
    emailAddressTextChanged: Empty().eraseToAnyPublisher(),
    submitButtonPressed: userTappedSubmitdTrigger.eraseToAnyPublisher()
)


let sut = ForgotPasswordViewModel(uiInputs: uiInputs,
                                  backgroundScheduler: testScheduler.eraseToAnyScheduler(),
                                  uiScheduler: testScheduler.eraseToAnyScheduler(),
                                  api: apiMock)

var values: [Void] = []
var completedWithoutError = false
var completedWithAnError: Error?
_ = sut.resetRequestAccepted
    .sink { completion in
        if case let .failure(error) = completion {
            completedWithAnError = error
        } else {
            completedWithoutError = true
        }
    } receiveValue: { value in
        values.append(value)
    }

testScheduler.advance(by: 3)

XCTAssertEqual(1, values.count)
XCTAssertNil(completedWithAnError)
XCTAssertFalse(completedWithoutError)
~~~

First we create a `PassthroughSubject` to simulate the user tapping the submit button.  Note that this is not a `CurrentValueSubject` because button taps don't have a
starting value they just happen so our subject mirrors this behavior.  Next we create the test scheduler and we schedule the button tap to happen at 1 tick in the future.

Next we construct our view model passing in the ui inputs, the test scheduler for both the background and ui scheduler and an api mock.   The mock would be elaborated on
more if we were handling error conditions, but that isn't the point of this discussion so we'll ignore it for now.

The rest of test follows in much the same way as the `buttonIsEnabled` test from earlier.  We set up event collection variables, subscribe to the publisher we're
testing (`resetRequestAccepted`), and collect any output that occurs.

Next is the key part: We advance the test scheduler's clock by 3 ticks.  This causes a few things to take place:
1. The input event we scheduled at 1 tick in the future gets fired.  This causes the viewModel to receive that tap event and start processing it.  Eventually it hits the
statement to move to the background scheduler via `receive:on:`.  This work is put on the test scheduler to run at the next clock tick.
2. When the clock tick moves to 2 ticks into the future that background work that was scheduled now executes and invokes the API mock.  Presumably the mock immediately
returns a result publisher.  The `flatMap` operator immediately subscribes to this publisher and the result is sent up out of the `flatMap`.  Next our viewmodel code schedules the resulting
event to go back to the UI scheduler via `receive:on:` again which in turn is scheduled on the test scheduler for the next clock tick.
3. The clock advances once more to the 3rd tick in the future where the work scheduled for the UI thread is executed, returning an event to the sink and into our collection variables.

Lastly our test continues to our asserts which inspect our collected output and determine if the test succeeded or not.  Note that because the `resetRequestAccepted` publisher
emits `Void` events we cannot compare the values themselves so we just compare the count.  That's good enough for `Void`!
  <br>
  <br>
  <br>
# A Better Expectation
Let's analyze what happened and why this is a vast improvement over expectations.   

First, we have abstracted scheduling such that production code can use real dispatch queues and UI threads but 
test code can use a virtual scheduler.  During the unit test we advance time in a serial way moving at whaever speed the CPU is capable of **instead of wall clock time**.  That means our
tests are running as fast as they can, and regardless of whether they pass or fail the code gets results quickly and isn't stuck waiting on a timeout.  This means your tests run super fast
even if they are failing.

Second, we now have the entire shape of the emissions of a publisher SUT so we can be sure that only the expected behavior is observed.  We can even ensure further unexpected events do not
occur by advancing the test scheudler way past the ticks we expect to have events on.   We could advance it by 100 or 1000 and our test would still be fast because it's virtual time
and not wall clock time!   Trying to be that exhaustive with expectations would be very complex and very error prone.
  <br>
  <br>
  <br>
# Refactoring away boilerplate
After going through this one might think that the code for properly testing a publisher is fairly extensive but most of it is easy D.R.Y.'d away.   Using our full example in the previous
section we could easily do the creation and setup of the view model and its input subjects during the `setup()` phase of unit tests and all tests on the view model would probably need them.

We can also refactor away the collection of output.  I have a sample class to do this called [PublisherObserver][publisher-observer] which can manage the subscription of the publisher
being tested, collect the output, including errors and completion events.  It has convience methods on it to make assertions easy to construct and even allows for testing that
a specfic error was emitted.  

Refactoring away the view model setup and event collection would leave our last test looking like this:
~~~swift
testScheduler.schedule(after: testScheduler.now.advanced(by: 1)) {
    userTappedSubmitdTrigger.send(())  // Simulate user tapping submit
}

let observer = PublisherObserver(sut.resetRequestAccepted)
testScheduler.advance(by: 3)

XCTAssertEqual(1, observer.values.count)
XCTAssertFalse(observer.witnessedCompletion)
XCTAssertFalse(observer.witnessedAnError)
~~~

This unit test is in great shape as it focuses on what's being tested and what the expectations are.  
<!--- One downside is that this quickly becomers boilerplate in most of your tests but this is easily abstracted away to keep tests clean and terse.  More on that later... -->




[rxtest]: https://github.com/ReactiveX/RxSwift/blob/main/Documentation/UnitTests.md
[avoid-subjects]: https://www.davesexton.com/blog/post/To-Use-Subject-Or-Not-To-Use-Subject.aspx
[rxtest-testscheduler]: https://github.com/ReactiveX/RxSwift/blob/main/RxTest/Schedulers/TestScheduler.swift
[pointfree]: https://www.pointfree.co/
[combine-schedulers]: https://github.com/pointfreeco/combine-schedulers
[publisher-observer]: https://gist.github.com/SlashDevSlashGnoll/f58abcc65f6798867d7b79e2d6f42fd1
