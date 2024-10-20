---
title: Models and Extensions
date: 2024-11-03 10:00:01 -0400
tags: taskmanager models extensions date-utilities
category: coding
---

This is where we start getting into the nuts and bolts of the infrastructure for the project by building models, extensions to make the use of these models easier as I put together date utilities to make my life easier.

All of the code for this blog post is in this [sample code repo][post-project].

<!--more-->

For the `TaskManager` project, we need to convert through a small group of chosen date strings. They'll need to be able to handle the following:

* A reminder that has a date due (ie; a date without a time)
* A reminder that has a due date and time (ie; a date with a time)
* A appointment that has a due date and time, as well as an end time (ie; a date with a time and a second time for that same day)
* A recurring appointment that has a start date and time, as well as an end date and time (ie; two date and time models to indicate start and end)

The code for this blog post are in three main areas:

* Models
* Extensions
* Unit Tests

## Models

As I mentioned in the post about enums, an associated enum with a struct model for the parameters can make your life easier. In the `DateParameters` file, this is illustrated further.

By using models and protocols, I was able to normalize functionality across models, which make things a little easier.

The `FormattedDateRepresentable` protocol is simple enough. Anything conforming to it must have a `formattedDate` `String` property. In other words, each of these parameters will become a formatted date string.

This will ensure that I'll be able to easily convert back and forth between `Date` objects and `String` objects in known formats, but let's walk through how that is done.



## Extensions

## Unit Tests

[post-project]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Infrastructure-2024-11-03