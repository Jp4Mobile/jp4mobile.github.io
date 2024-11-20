---
title: Text Models
date: 2024-11-24 10:00:01 -0400
tags: taskmanager text-conversion
category: coding
---

One of the biggest factors into why I love `TaskPaper` is the way that it effortlessly converts to and from text files. For my `TaskManager` app, I consider that to be must have functionality as well.

We'll be talking about `String` parsing as we break down converting back and forth from a hypothetical `TaskPaper` project with multiple tasks. Some of these tasks may have a variety of tags. As well as the possibility of notes.

All of the code for this blog post is in this [sample code repo][post-project].

<!--more-->

## Our sample TaskPaper file

Again the three major components to a `TaskPaper` are:

* Project

A `Project Title:` (Any text that ends with a colon).

* Task

A `- Task Name` (Any text that begins with a dash).

* Text

Any other `text`.

* Tags

Optionally, there can be tags. `@tag` or `@tag(with content)`. Typically, these are usually used with tasks, but obviously, they can be on any of these items.

Let's put that all together in a text file:

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
  
  Let's illustrate some of the time formats that we've worked so hard on.
  - Something completed @test @due(2024-11-23) @done(2024-11-23)
  - Something with a set due date and time @test @due(2024-11-24 10:00)
  - An appointment @test @due(2024-11-24 10:00-10:30)
  - A spanning appointment @test @due(2024-11-23 11:00 thru 2024-11-24 10:00)
{% endhighlight %}

---

Next week, we start working on the UI layer and getting into the SwiftUI of it all.

[post-project]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/TextModels-2024-11-24