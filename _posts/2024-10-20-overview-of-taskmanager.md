---
layout: post
title:  "TaskManager Overview"
date:   2024-10-19 13:30:01 -0400
tags: taskmanager taskpaper productivity
category: coding

---

Breaking down a project into milestones and figuring out your roadmap can be a struggle. Let's walk through an example for the project that I'll be working on in a series of blog posts.

<!--more-->

I am a big fan of [TaskPaper][task-paper] by [Hogs Bay Software][hogs-bay] and I use it extensively to keep track of my life across multiple devices and platforms. It has a simple yet rich interface.

{% highlight TaskPaper %}
Project:
  Notes on a project.
  TaskPaper uses a hierarchical structure to keep track of parents and
  ownership.
  ie; all of the below items are within this project.

  - Item for that project.
    - Child item with a note.
      Note about that item.
  - Tagged Item @tag
  - Tagged Item where the tag has a payload @tag(payload)
{% endhighlight %}

But, and with developers there is almost always a but, I wish it integrated a bit better with some of the built ins for my Apple ecosystem, so that I would only have to check one place to see what I've got going on.

That's the origin of my `TaskManager` project, I hope that I can keep the same simplicity of [TaskPaper][task-paper], but integrate it with my `Calendar` and `Reminder` apps built into iOS. (Yes, [TaskPaper][task-paper] supports exporting and importing into the reminders app, but unfortunately, not in the way I was imagining.)

I have a nodding acquaintance with Apple's [EventKit][event-kit] which gives an integration into the calendar(s), appointments, and reminders.

This seems perfect for my use of [TaskPaper][task-paper], where I have a number of items with due date tags with the following formats:

* Open-ended or no due date:
  > `@due`
* Due date:
  > `@due(2024-10-31)`
* Due date and specific time:
  > `@due(2024-10-31 12:30)`
* Appointments:
  > `@due(2024-10-31 12:30-13:00)`
  >
  > `@due(2024-10-31 12:30 thru 13:00)`
* Spanning appointments:
  > `@due(2024-10-31 13:00 thru 2024-11-06 15:00)`
  
It's easy to imagine these converting to a calendar event or a reminder in specific lists/calendars.

```
Groceries:
  - Get candy @home
Fitness:
  - Trainer @gym @due(2024-10-31 07:00-08:00)
    Studio 617
Jp4Mobile:
  - Blog post on enums in Swift @work @due(2024-10-31)
  - Check blog engagement @work @due(2024-11-07)
```

Such as an event: ![Event from example](/img/Event-2024-10-20.png)

Or reminders: ![Reminder from example](/img/Reminders-2024-10-20.png)

## Project Breakdown

As with any project, much of the work is breaking it down to smaller deliverables, so that you can both see incremental improvements, but also understand some of the concepts.

For this, I see the following milestones:

- Infrastructure
  > This is going to be the behind the scenes things to make everything possible.
  >
  > It will include things like:
  - Models
    > The data representation around the date structures above.
  - Extensions
    > These extensions to the Apple [Date][date] foundation structure to make life easier for the `TaskManager` app.
  - [EventKit][event-kit] Wrapper
    > From a cursory reading of the Apple documentation, there's certain logic and repetitive boilerplate checking of permissions that make a manager or some other form of wrapper a necessity, so that callers will have a cleaner interface.
- UI
  > What will be put on the screen and allow a user to interact with it.
  - Main Text View
    > To present the various elements in a [TaskPaper][task-paper] file(s)
    >
    > These could be `Project`, `Note`, `Item`, `Tag` on any `Item` and include their nested structure.
  - `Project` Detail View
    > To present a single `Project`
  - `Note` Detail View
    > To present a single `Note`
  - `Item` Detail View
    > To present a single item
  - `Tags` Search Results View
    > To present `Item`s that conform to the `Tag`(s) searched for.
  - Settings View
- Refinement
  > Nothing is ever perfect out of the gate.
  >
  > Interacting with things will help me iterate and make it better (or at least more useful to me).

## Blog Posts

As I work through my deliverables, I'll be posting about some of the lessons learned, useful bits that I've learned, and outlining some of the pain points to try to help others.

Posts will include [sample code][sample-code] in the [Jp4Mobile repo][jp4mobile-repo]. Feel free to reach out if you have questions.

[task-paper]: https://www.TaskPaper.com
[hogs-bay]: https://www.HogsBaySoftware.com
[event-kit]: https://developer.apple.com/documentation/eventkit
[date]: https://developer.apple.com/documentation/foundation/date
[sample-code]: https://github.com/Jp4Mobile/SampleCode
[jp4mobile-repo]: https://github.com/Jp4Mobile