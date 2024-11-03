---
title: Event Kit Manager
date: 2024-11-10 10:00:01 -0400
tags: taskmanager eventkit
category: coding
---

Apple's ``EventKit`` can be a pain to work with. There's specific logic that you need to make sure that you follow, so your app handles the permissions properly.

The easier way to do this is to wrap the ``EventKit`` functionality in a manager that will manage the requests for permissions and ensure that the actual calls to the services are done properly.

All of the code for this blog post is in this [sample code repo][post-project].

<!--more-->

There are some challenges with using using the EventKit. Not the least of it, is that you need to follow Apple's guidelines to make sure that you request access properly for both events, reminders, and optionally their calendars, so that you can create the appropriate calendars, as well as read and write events and reminders.

## Requesting Access

There are a few facets of this. Working from the inside of your project out:

1. Add the appropriate privacy descriptions for the calendar and/or reminder access to your app's `Info.plist`.
2. Within your ``EventKit`` retrieval logic, make sure that you're _requesting access_ and if you don't receive access, _communicate that to your user_ and _end your retrieval_.
3. Once you have access, _retrieve_ the calendar and then with the calendar, you can retrieve ``Event`` or ``Reminder`` objects.

![Info.plist](/img/InfoPlist-2024-11-10.png)

For my sample code, I asked for full access, as you can see in both the screenshot or by perusing the sample code.

Apple uses that description as it presents the request for access to the user, as you can see below:
![Request Event/Calendar Access](/img/RequestEventAccess-2024-11-10.png)
![Request Reminder/Calendar Access](/img/RequestReminderAccess-2024-11-10.png)

Now, let's walk through the logic of my `EKManager` which will wrap the ``EventKit``.

### Access Wrappers

The access wrappers are simple to write:

```
func requestCalendarAccess() async throws {
  do {
    let access = try await eventStore.requestFullAccessToEvents()
    hasCalendarAccess = access
  } catch {
    hasCalendarAccess = false
    throw error
  }
}
```

We use the event store and then through that, we request access for either event/calendar access or reminder access. I illustrated the event/calendar access, but reminder access is equally straightforward.

### Access Verification Logic

Within the places that need to access your events or reminders, the logic also follows, though it may be repetitive in the various places you use it.

```
if !hasCalendarAccess {
  try await requestCalendarAccess()
}
guard hasCalendarAccess else {
  throw EKManagerError.calendarAccessDenied
}
// At this point you can continue to your business logic.
```

## Retrieving Data

Before you can retrieve calendar events or reminders, you'll need to retrieve the appropriate calendar for those ``EventKit`` entities.

### Retrieving Calendars

Once again, Apple makes that easy for developers by having an API that is easy to understand. We can retrieve all event or reminder calendars with the `eventStore.calendars(for: .event)` or `eventStore.calendars(for: .reminder)`, then the developer can pick the one that they want.

Or if the user would like to just go with whatever calendar that the user already prefers to use, you can just grab the defaults using:
`eventStore.defaultCalendarForNewEvents` or `eventStore.defaultCalendarForNewRemindres`.

### Creating Calendars

Or you could create your own calendars. Myself, I prefer to create my own calendar, so that any events will be branded in a `"TaskManager"` calendar for events and reminders, which will make it easier for myself in terms of testing or with an eye towards removing items in mass, if a user requests it.

#### Gotchas

A newly created calendar will need at least a title and a source.

```
let calendar = EKCalendar(for: entityType, eventStore: eventStore)
calendar.title = "TaskManager"
calendar.source = try await getEventSource(for: entityType)
try eventStore.saveCalendar(calendar, commit: true)
```

### Finding the Source

An `EKSourceType` can represent a variety of sources. Usually, it's a good idea to leverage what the user is already using, if possible.

The logic is basically, use the same source as the default calendar for that entity, if possible. Otherwise, try to see if iCloud is possible for the user. If that doesn't work, try to see if local access is available.

```
func getEventSource(for entityType: EKEntityType) async throws -> EKSource {
	let `default` = try await getDefaultCalendar(for: entityType).source
	let isICloudPresent: (EKSource) -> Bool = {
		$0.title.lowercased().contains("icloud")
	}
	let iCloud = eventStore.sources.first(where: isICloudPresent)
	let local = eventStore.sources.first(where: { $0.sourceType == .local })

	guard let source = `default` ?? iCloud ?? local else {
		throw EKManagerError.noSourceFound
	}

	return source
}
```

### Retrieving Events or Reminders

There are two ways to retrieve event or reminder items. Either those that match a specific predicate or by trying to retrieve a specific event or reminder item by their unique identifier.

In my initial code, I went with the more generic to retrieve all events and reminders for the TaskManager calendar that I created above.

The `getEvents()` function creates a predicate to search for calendar events within a known calendar for the next month. Start and end dates can be reconfigured to work best with individual requirements or for any calendar that the user has access to.

```
func getEvents() async throws -> [EKEvent] {
	let calendar = try await getCalendarToUse(for: .event)

	let startDate = Date()
	let endDate = Calendar.current.date(byAdding: .month, value: 1, to: startDate) ?? startDate
	let predicate = eventStore.predicateForEvents(withStart: startDate,
												  end: endDate,
												  calendars: [calendar])
	
	let allEvents = eventStore.events(matching: predicate)

	return allEvents
}
```

The `getReminders()` function is a bit easier, as reminders are not required to have dates, so the predicate only checks against a specific reminder calendar.

#### Gotchas

Most of the ``EventKit`` functionality has shifted from completion handlers to the more current async/await model. Unfortunately, `fetchReminders(matching:, completion:)` is not one of them.

However Apple has an easy to use way to convert completion handlers to async calls leveraging the ``CheckedContinuation`` interface.

```
func getReminders() async throws -> [EKReminder] {
	let calendar = try await getCalendarToUse(for: .reminder)

	let predicate = eventStore.predicateForReminders(in: [calendar])

	return await withCheckedContinuation { continuation in

		eventStore.fetchReminders(matching: predicate) { reminders in
			continuation.resume(returning: reminders ?? [])
		}
	}
}
```

By using `withCheckedContinuation`, the completion handler for the call you're making can be converted to an `async` call that can return a value, throw an error, or just run asynchronously.

#### Optimizing by Retrieving Individual Items

This certainly works, but retrieving all of them to then search the full list is going to cause problems at scale.

It would be better to retrieve events and items by an ID number, so that the first time it's been retrieved, it can be manipulated by an individual item.

There are simple retrieval methods for that:

You can use `eventStore.event(withIdentifier: id)` for calendar events.

```
guard let calendarItem = eventStore.calendarItem(withIdentifier: id),
      let reminder = calendarItem as? EKReminder else {...}
```
### Removing Events or Reminders

The `eventStore.remove(_ event:, span:, commit:)` and `eventStore.remove(_ reminder:, commit:)` functions take the `EKEvent` and `EKReminder` models and remove them.

The `commit:` parameter with both functions indicates whether the removal happens now or later (in the case of a batched removal) when a subsequent commit is called.

The remove function wrapper and some proposed usages could help explain this a bit better:

```
func remove(event: EKEvent, shouldBatch: Bool = false) async throws {
	if !hasCalendarAccess {
		try await requestCalendarAccess()
	}
	guard hasCalendarAccess else {
		throw EKManagerError.calendarAccessDenied
	}

	// If we should batch, we shouldn't commit.
	try eventStore.remove(event, span:.thisEvent,  commit: !shouldBatch)
}
```

This could be used in this manner:
```
let titledEvents = try await getEvents()
   .filter { $0.title == specificTitle }

// Delete one
if let specificEvent = titledEvents.first {
  // Removes and commits the removal
  try ekManager.remove(specificEvent)
}

// Delete many
for titledEvent in titledEvents {
  // Don't commit these changes until ALL are successful removed.
  try await ekManager.remove(titledEvent, shouldBatch: true)
}
// After all the events were removed, we commit.
try ekManager.eventStore.commit()
```

### Create Event or Reminder

Similar to creating a calendar, creating a calendar event or a reminder uses the same logic after making sure they have the appropriate access for calendars and reminders:

```
let event = EKEvent(eventStore: eventStore)
// set the values: title, startDate, optional endDate, optional notes, and the calendar to create this event into.
try eventStore.save(event, span: .thisEvent, commit: shouldCommit)
```
And
```
let reminder = EKReminder(eventStore: eventStore)
// set the values: title, optional dueDateComponents (which may either be a date or a date with time), notes, and the calendar to create this reminder into.
try eventStore.save(reminder, commit: shouldCommit)
```

### Updating Events or Reminders

Now that we can retrieve a known event or reminder from, updating it's fairly straightforward:

```
func update(_ event: EKEvent, shouldBatch: Bool = false) async throws {
	try eventStore.save(event, span: .thisEvent)

	guard !shouldBatch else { return }

	try eventStore.commit()
}

func update(_ reminder: EKReminder, shouldBatch: Bool = false) async throws {
	try eventStore.save(reminder, commit: !shouldBatch)
}
```

### Manual Testing

For this week, the only testing that was done was manual testing. Inside the sample code's `ContentView`, you may see the `EKManager` being called directly to create reminders and events (as well as the requesting of permissions, creation of calendar, etc... that goes on behind the scenes for that to happen).

```
try await EKManager.shared.create(.reminder, model: testData.reminder)
try await EKManager.shared.create(.event, model: testData.event)
```

---

Next stop, testing the ``EventKit`` manager without needing to test Apple's ``EventKit`` itself by using APIs to test our logic.

[post-project]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/EventKitWrapper-2024-11-10
