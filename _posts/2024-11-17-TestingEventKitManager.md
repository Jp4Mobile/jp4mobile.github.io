---
title: Testing Event Kit Manager
date: 2024-11-17 10:00:01 -0400
tags: taskmanager eventkit testing
category: coding
---

Unit Testing a manager that wraps third-party functionality can be challenging. You don't want to waste your time and energy testing something that one assumes that the vendor has already tested and supports. If they've given you an API, you need to trust that API is accurate.

What you want to test is your business logic and your code. There are two ways to do that. One is to leverage a pseudo object that conforms to the API but you can control the outputs. Ie; if you should be thrown an error, you can trigger that; or if you should be given a response, you can control what the response is.

This lets your tests focus on your business logic, not someone else's.

All of the code for this blog post is in this [sample code repo][post-project].

<!--more-->

## Faking EKEventStore

As we learned in the [EventKit Manager blog post][ekmanager-blog], the main path into interacting with the `EventKit` is the `EKEventStore` object. So that's what we're going to fake and inject into our EventKitManager and test, so that we can verify our code.

This post might be easier, if you follow along in the sample code. In the `EKManagerTests` file, we create a fake event store class: `FakeEKEventStore`. This needs to conform to the functionality of the `EKEventStore`, but in each case it should allow the injection of a canned return or thrown response to the caller.

{% highlight swift %}
class FakeEKEventStore: EKEventStore {
   var errorToThrow: Error?

   // MARK: - Access
   var accessResult: Bool?

   override func requestFullAccessToEvents() async throws -> Bool {
      if let errorToThrow {
         throw errorToThrow
      }

      if let accessResult {
         return accessResult
      }
        
      return try await super.requestFullAccessToEvents()
   }

   override func requestFullAccessToReminders() async throws -> Bool {
      if let errorToThrow {
         throw errorToThrow
      }
        
      if let accessResult {
         return accessResult
      }
        
      return try await super.requestFullAccessToReminders()
   }
}
{% endhighlight %}

This allows us to inject `errorToThrow` and `accessResult` into our access request functions: `requestFullAccessToEvents` and `requestFullAccessToReminders`.

The logic is relatively simple to follow:

* _If there's an error, throw it._
* _Otherwise, if there's a result, return it._
* _And the fallback, if nothing was injected, make the actual call._

### Gotchas

Falling through and making the actual call may or may not meet your needs. My rule of thumb when I was faking the event store was whether or not the calls were changing things within the event store.

## Using the fake in tests

Different people may have different practices in terms of how they name and write their unit tests. Over time, I've learned that when I go back to my unit tests, after weeks or months of other work, clarity is much more useful than brevity, so that I know exactly what should be being tested, what the conditions are and what the expectations are. That's one of the reasons that I name my tests as I do.

Let's break down what this test is doing:

{% highlight swift %}
// MARK: - Access Wrapper Tests
// MARK: Events
func test_requestCalendarAccess_whenThrowingAnError_thenThrowsError() async {
  // Set up injection
  let fakeEventStore = FakeEKEventStore()
  fakeEventStore.errorToThrow = TestError.error
  sut = EKManager.shared
  sut.eventStore = fakeEventStore

  // Verify that the error was thrown
  do {
    try await sut.requestCalendarAccess()
    XCTFail("Should not have succeeded")
  } catch {
    XCTAssertEqual(error as? TestError, .error)
    XCTAssertFalse(sut.hasCalendarAccess)
  }
}
{% endhighlight %}

In my unit tests, I group tests by logic and then by area, which makes it much easier to read through the code and to find the tests at a later date. Then I name my unit tests by the function being tested, what the set up is and lastly what the expectations of the test is.

First we instantiate the fake event store, inject the error that our fake event store will throw, and then we inject the fake event store into the `EKManager` that will be testing.

Then we wrap the call and verify that it wasn't succeeding, that the error is as expected, and lastly the manager access values are as expected.

### Gotchas

Testing best practice involves making sure at the end of your test (or in the teardown), you've reset your test environment back to the pre-test state. Otherwise, you may find that you've introduced flakiness in your test environment and subsequent tests may no longer conform to your assumptions, or even worse, you've added this same sort of issue to your environment on devices that you're testing on.

In our case, we're injecting a fake event store into a singleton. That could clearly cause problems. So some extra changes needed to be made to our `EKManager` as well as to our test environment.

{% highlight swift %}
#if DEBUG

extension EKManager {
   func reset() {
      self.eventStore = EKEventStore()
      self.hasCalendarAccess = false
      self.hasReminderAccess = false
   }
}

#endif
{% endhighlight %}

I only needed the reset function for unit tests, so wrapping it to ensure that it would never end up in a production release seems like adequate protection.

Then all we have to do is call it after every unit test. Happily, Apple has given us a function that's called after every unit test called `tearDown()`. There's also a similar class based function that is called after each test suite file has finished running, but for our purposes resetting after each unit test is sufficient.

{% highlight swift %}
override func tearDown() {
   sut.reset()
   sut = nil

   super.tearDown()
}

{% endhighlight %}

## Types of tests to run

Looking at my access requesting methods, there's not that much to test:

* When the `EventKit` functions are called and they throw an error, throw an error from the `EKManager` wrapper function.
* When the user did not grant access, the `EKManager` does not think that the user has granted access.
* When the user did grant access, the `EKManager` thinks that the user has granted access.

Here are the other two cases:

{% highlight swift %}
func test_requestCalendarAccess_whenUnsuccessful_doesNotGrantAccess() async throws {
   // Set up injection
   let fakeEventStore = FakeEKEventStore()
   fakeEventStore.accessResult = false
   sut = EKManager.shared
   sut.eventStore = fakeEventStore

   try await sut.requestCalendarAccess()
   XCTAssertFalse(sut.hasCalendarAccess)
}

func test_requestCalendarAccess_whenSuccessful_grantsAccess() async throws {
   // Set up injection
   let fakeEventStore = FakeEKEventStore()
   fakeEventStore.accessResult = true
   sut = EKManager.shared
   sut.eventStore = fakeEventStore

   try await sut.requestCalendarAccess()
   XCTAssertTrue(sut.hasCalendarAccess)
}
{% endhighlight %}

From the lowest level wrapper function, now we can build up to calling functions and validate our business logic.

We have a verify access function that checks the current access and then handles the call to request access from the user.

Similar to request access functions, this will do the following:

* When the `EKManager` already has access, don't check.
* When the `EKManager` wrapping function returns `false`, the verify access function throws an error indicating that.
* When the `EKManager` wrapping function returns `true`, the verifying access function does nothing.

{% highlight swift %}
// MARK: Verify Access
func test_verifyAccess_forEvents_whenAccessHasBeenGranted_doesNotRequestAccess() async throws {
   let fakeEventStore = FakeEKEventStore()
   fakeEventStore.errorToThrow = TestError.error
   sut = EKManager.shared
   sut.eventStore = fakeEventStore
   sut.set(hasCalendarAccess: true)

   try await sut.verifyAccess(for: .event)
   XCTAssertTrue(sut.hasCalendarAccess)
}

func test_verifyAccess_forEvents_withoutAccess_throwsError() async {
   let fakeEventStore = FakeEKEventStore()
   fakeEventStore.accessResult = false
   sut = EKManager.shared
   sut.eventStore = fakeEventStore

   do {
      try await sut.verifyAccess(for: .event)
      XCTFail("Should not succeed")
   } catch {
      if let ekError = error as? EKManager.EKManagerError,
         case .calendarAccessDenied = ekError {
         return
      } else {
         XCTFail("Unknown error: \(error)")
      }
   }
}
{% endhighlight %}

We didn't need to test the happy path of the `verifyAccess(for:)` method directly, because they'll be tested in the retrieval tests.

## Retrieval tests

Let's walk through the logic of our retrieval function for the `getEvents` method.

{% highlight swift %}
func getEvents(from startDate: Date = Date()) async throws -> [EKEvent] {
   let calendar = try await getCalendarToUse(for: .event)

   let endDate = Calendar.current.date(byAdding: .month, value: 1, to: startDate) ?? startDate
   let predicate = eventStore.predicateForEvents(withStart: startDate,
                                      end: endDate,
                                      calendars: [calendar])
   
   let allEvents = eventStore.events(matching: predicate)

   return allEvents
}
{% endhighlight %}

Digging into the `getCalendarToUse(for:)` method, you'll dig down to the `retrieveEventCalendars()` method. Before it tries to retrieve the calendars, it'll make sure that the user has granted access to the calendar events.

In this article, we're mostly going to be concentrating on calendar events.

{% highlight swift %}
// MARK: - Retrieve Tests
// MARK: Events
func test_getEvents_withoutAccess_throwsError() async {
   let fakeEventStore = FakeEKEventStore()
   fakeEventStore.accessResult = false
   fakeEventStore.eventsResult = []
   sut = EKManager.shared
   sut.eventStore = fakeEventStore
   
   do {
      _ = try await sut.getEvents()
      XCTFail("Should not succeed")
   } catch {
      if let ekError = error as? EKManager.EKManagerError,
         case .calendarAccessDenied = ekError {
         return
      } else {
         XCTFail("Unknown error: \(error)")
      }
   }
}
{% endhighlight %}

In some ways, the error cases are the easier ones to write. When there's a problem, just catch and verify the error thrown.

{% highlight swift %}
func test_getEvents_withAccess_returnsEvents() async throws {
   let fakeEventStore = FakeEKEventStore()
   let event = EKEvent(eventStore: fakeEventStore)
   fakeEventStore.eventsResult = [event]
   sut = EKManager.shared
   sut.eventStore = fakeEventStore
   sut.set(hasCalendarAccess: true)

   let events = try await sut.getEvents()
   XCTAssertEqual(events, [event])
}

func test_getEvent_withAccess_returnsEvent() async throws {
   let fakeEventStore = FakeEKEventStore()
   let event = EKEvent(eventStore: fakeEventStore)
   fakeEventStore.eventResult = event
   sut = EKManager.shared
   sut.eventStore = fakeEventStore
   sut.set(hasCalendarAccess: true)
   let eventToTest = try await sut.getEvent(id: "ignored")
   XCTAssertEqual(eventToTest, event)
}
{% endhighlight %}

The happy path is similarly straight forward. We verify that the faked response is being propagated up.

{% highlight swift %}
func test_getEvent_withNoResult_throwsError() async {
   let fakeEventStore = FakeEKEventStore()
   sut = EKManager.shared
   sut.eventStore = fakeEventStore
   sut.set(hasCalendarAccess: true)
   do {
      _ = try await sut.getEvent(id: "ignored")
      XCTFail("Should not succeed")
   } catch {
      if let ekError = error as? EKManager.EKManagerError,
         case .eventNotFound = ekError {
         return
      } else {
         XCTFail("Unknown error: \(error)")
      }
   }
}
{% endhighlight %}

Our function to retrieve an event by identifier has one other bit of business logic. If there's no returned event, we throw a specific error.

### Gotchas

With *Test Driven Development*, you write the tests, before you develop the code. (An oversimplification, but bear with me.) This could lead to tests that will not pass. Or, perhaps, there's logic that needs to be verified, but your testing infrastructure doesn't support it yet. Or, maybe, you might have a flakey test due to timing problems in your code or cascading dependencies that are causing you issues.

There may be times where you may need to skip a test until you can resolve the issue. Apple makes that very easy:

{% highlight swift linenos %}
func test_getEvents_whenThrowingError_throwsError() async throws {
   throw XCTSkip("Unfortunately, though it should throw an error, " +
                 "faking EventStore doesn't allow us to overload " +
                 "this method.")

   let fakeEventStore = FakeEKEventStore()
   fakeEventStore.errorToThrow = TestError.error
   sut = EKManager.shared
   sut.eventStore = fakeEventStore
   sut.set(hasCalendarAccess: true)
   
   do {
      _ = try await sut.getEvents()
      XCTFail("Should not succeed")
   } catch {
      print("*Jp* \(self)::\(#function)[\(#line)] <\(error)>")
      if let ekError = error as? EKManager.EKManagerError,
         case .calendarAccessDenied = ekError {
         return
      } else {
         XCTFail("Unknown error: \(error)")
      }
   }
}
{% endhighlight %}

Line `#2` shows how to skip one of a test. It's usually good practice to fill in the message, often with a ticket number where you'll be fixing the issue or the reason why you're skipping your test.

---

Next week, one last infrastructure model before we can get to the UI layer.

[ekmanager-blog]: https://www.jp4mobile.com/coding/2024/11/10/EventKitManager.html
[post-project]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/TestingEKWrapper-2024-11-17