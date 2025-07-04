---
title: Testing the Different Architectures for SwiftUI Projects
date: 2025-07-07 10:00:01 -0400
tags: taskmanager swiftui architecture tca mvvm testing
category: coding
---

Continuing with our discussion of the two common architectures from our last post **MVVM** and **TCA**, we're going to talk about testing with the different architectures, as well as some of the debugging lessons learned from coding up the sample code.

All of the code for this blog post is in this [sample code repo][post-project].

<!--more-->

# Testing

When testing your app, just check business logic. If you're testing how code that you haven't written works, you're doing things incorrectly. For example, just using some out of the box Swift functionality, if you're writing code to verify the way that, say, `UserDefaults` work, you're wasting your time.

If you're creating a library, by all means, verify that your library works as it is supposed to. But if you're using a library, just verify your business logic around the library calls. That's where your time is better spent.

## MVVM

One of the things that I really liked about working with MVVM is how easy it was to work with and to test. There wasn't anything that wasn't standard Swift involved.

We'll test the `TagView` code that we've been using in our previous examples:

### TagView Testing

Our initializer, defines the `textTag` property from the `tag` property, so that should be tested in our business logic.

{% highlight swift %}
init(_ tag: Tag, editMode: EditMode = .inactive) {
   self.editMode = editMode
   self.tag = tag
   self.textTag = tag.toString
}
{% endhighlight %}

So we add testing for that:

{% highlight swift %}
// MARK: - Initializer
func test_init_initializesThingsProperly() {
   let expectedPayload = "2025-07-06"
   let expectedTag = Tag(.due, payload: expectedPayload)
   sut = .init(Tag(.due, payload: expectedPayload),
               editMode: .inactive)

   XCTAssertEqual(sut.tag, expectedTag)
   XCTAssertEqual(sut.textTag, expectedTag.toString)
   XCTAssertEqual(sut.editMode, .inactive)
}
{% endhighlight %}

The bulk of our business logic is our view model's `convertTagIfValid(from:)` function.

{% highlight swift %}
func convertTagIfValid(from string: String) {
   guard let convertedTag = string.toTag() else {
      self.textTag = tag.toString
      return
   }
   self.tag = convertedTag
   self.editMode = .inactive
}
{% endhighlight %}

We'll need to test both paths through this code:

* Our *happy path*, where the string converts successfully into a `Tag` entity.
* And our *unhappy path*, where the string doesn't convert and the `textTag` property is reset.

{% highlight swift %}
// Our happy path test
func test_convertTag_whenSuccessful_setsEverythingProperly() {
   let tag = Tag("tag")
   let expectedTag = Tag(.due, payload: "2025-07-06")

   sut = .init(tag)
   sut.textTag = expectedTag.toString
   sut.editMode = .active

   sut.convertTagIfValid(from: expectedTag.toString)

   XCTAssertEqual(sut.tag, expectedTag)
   XCTAssertEqual(sut.textTag, expectedTag.toString)
   XCTAssertEqual(sut.editMode, .inactive)
   XCTAssertEqual(sut.isEditing, false)
}

// Our unhappy path test
func func_convertTag_whenUnsuccessful_resetsBackToPreviousTagText() {
   let tag = Tag("tag")
   sut = .init(tag)

   // Invalid conversion
   let invalidText = "invalid"
   sut.textTag = invalidText
   sut.editMode = .active
   XCTAssertEqual(sut.textTag, invalidText)
   sut.convertTagIfValid(from: invalidText)

   XCTAssertEqual(sut.tag, tag)
   XCTAssertEqual(sut.textTag, tag.toString)
   XCTAssertEqual(sut.editMode, .active)
   XCTAssertEqual(sut.isEditing, true)
}
{% endhighlight %}

This still leverages the older XCTest model, rather than the new Swift testing model. But the big thing is still the old standbys for testing:

* Set up your test environment's state.
* Optionally, verify the initial state.
* Call the function that you want to test.
* Verify the expectations of how it should change the state.

There are currently two functional tabs in our `TaskPaper` app: the `Tasks` and `Edit` functionality.

### TextView Testing

The main functionality of the `TextView` view model is similar to the `TagView`. The user enters text into `TextEditor` SwiftUI via the `updatedText(text:)` function. Similar to what we saw in the `TagView`, our testing will need to verify both the happy path and unhappy path. Here, we have an error message added to the UI to indicate to the user when there's a problem. 

{% highlight swift %}
// MARK: - Updated Text Conversion Tests
func test_updatedText_whenEmpty_setsExpectedError() {
   sut = .init(from: TMType.Mock.TopLevel.text)

   sut.updatedText(text: "")

   XCTAssertEqual(sut.type, TMType.Mock.TopLevel.text)
   XCTAssertEqual(sut.text, TMType.Mock.TopLevel.text.toString)
   XCTAssertEqual(sut.errorMessage, "Unable to convert <>")
}
{% endhighlight %}

An empty string is flagged as something that doesn't convert.

{% highlight swift %}
func test_updatedText_whenMultiple_setsExpectedError() {
   sut = .init(from: TMType.Mock.TopLevel.text)

   sut.updatedText(text: TMType.Mock.TopLevel.project.toString + "\n" +
                   TMType.Mock.Projects.projectWithTasks.toString)

   XCTAssertEqual(sut.type, TMType.Mock.TopLevel.project)
   XCTAssertEqual(sut.text, TMType.Mock.TopLevel.project.toString)
   XCTAssertEqual(sut.errorMessage, 
                  "Only the first converted type model is saved. You may need to " +
                  "change indentation to keep them under the proper project.")
}
{% endhighlight %}

Our logic is such that only a single `TMType` model is supported. When more than one is entered, only the first is used, but an error message indicates to the user where the problem might be and how to fix it.

{% highlight swift %}
func test_updatedText_whenConverted_setsTextProperly() {
   sut = .init(from: TMType.Mock.TopLevel.text)
   sut.errorMessage = "Invalid"

   sut.updatedText(text: TMType.Mock.Projects.projectWithTasks.toString)

   XCTAssertEqual(sut.type, TMType.Mock.Projects.projectWithTasks)
   XCTAssertEqual(sut.text, TMType.Mock.Projects.projectWithTasks.toString)
   XCTAssertNil(sut.errorMessage)
}
{% endhighlight %}

And then lastly, the happy path, where the `tag` converts properly, the `text` properties is properly set, and any lingering `errorMessage` is properly cleared.

The `TaskMasterAndDetailView.ViewModel` tests are a little more challenging, but the logic is straight-forward. We have the business logic wrapped in a handful of view model functions:

* `addItem(from:)` which adds an initial item with an optional `String?` value, which allows use to easily convert from `nil` or an empty string to a set value, such as `"New Item"` or the like.
* `select(item:)` which sets up the state of the view model when the user selects an item from the list.
* `process(_:)` which handles the business logic to verify and test the various responses from the `TaskDetailView` such as *cancel*, *delete*, or *save*.

As the view model also holds the state of the `Task` feature functionality, it makes it even easier to test. A function is called and then the state of the view model is verified against expectations.

As the `TaskDetailView` has their own local view model that triggers the changes the the `TaskMasterAndDetailView.ViewModel`'s `responseType` property, that nees to be tested as well.

## MVVM (Combine)

The MVVM (Combine) code is slightly different, due to leveraging the `StateBindingViewModel`.

---

Next article, we'll continue with the SwiftUI UI/UX for our `TaskManager` app with the TCA architecture.

[post-project]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/TestingArchitect-2025-07-06
[mvvm-samplecode]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/TestingArchitect-2025-07-06/MVVM/TaskManager
[mvvm-combine-samplecode]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/TestingArchitect-2025-07-06/MVVM-Combine/TaskManager
[tca-samplecode]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/TestingArchitect-2025-07-06/TCA/TaskManager
