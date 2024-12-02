---
title: Text Models
date: 2024-12-01 10:00:01 -0400
tags: taskmanager text-conversion regex
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

Let's put that all together in a text file with a certain level of complexity, so that we can be sure that we've caught some of our edge cases:

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

Other Project:
 - Item for the other project

{% endhighlight %}

For this blog post, the code is a little more complex than it needs to be so that I can illustrate the steps a little better.

## Our New Models

First, we have our tags. At their simplest, there are either `@tag` or `@tag(payload)`.

{% highlight swift %}
enum Tag {
  case tag(String)
  case payloadTag(String, String)
}
{% endhighlight %}

Then we have our `TaskPaper` types:

{% highlight swift %}
enum TPType {
  case project(String)
  case task(String)
  case text(String)
}
{% endhighlight %}

And then lastly, we can create a model with both of our new enums:

{% highlight swift %}
struct TMType {
  let tabLevel: Int
  let type: TPType
  let tags: [Tag]
  let children: [TMType]
}
{% endhighlight %}

### Gotchas

One of the first gotchas comes from how we might be parsing things.

If we'll be updating our models, as we're parsing it, `tags` and `children` may not all be known as the record is being parsed, so some mutating functions might be a little easier to work with:

{% highlight swift %}
struct TMType {
  let tabLevel: Int
  let type: TPType
  private(set) var tags: [Tag]
  private(set) var children: [TMType]
  
  // MARK: Update Functionality
  mutating func append(tag: Tag) {
    tags.append(tag)
  }
  
  mutating func append(child: TMType) {
    children.append(child)
  }
}
{% endhighlight %}

## Regex Builder

Regex is a powerful tool for parsing strings with known formats.
 
You may be familiar with something like this `/\d{3}\D?\d{3}\D?\d{4}/` in terms of a phone number validation regex.

In Swift 5.7, RegexBuilder was added, which is a bit different. There are plenty of blog posts about how RegexBuilder works, so I'll just talk about my use of it.

{% highlight swift %}
// Set up references
let indentRef = Reference(Substring.self)
let hyphenRef = Reference(Substring.self)
let colonRef = Reference(Substring.self)
let textRef = Reference(Substring.self)

// Set up the regex
let tmTypeSearch = Regex {
	// Initial Indent
	Capture(as: indentRef) {
		ZeroOrMore(.whitespace)
	}
	// Whether or not it's a task
	Capture(as: hyphenRef) {
		Optionally {
			"-"
		}
	}
	ZeroOrMore(.whitespace)
	// Text of the element (including tags)
	Capture(as: textRef) {
		OneOrMore(
			ChoiceOf {
				// All of the valid characters for the text area.
				CharacterClass(.anyNonNewline)
			}, .reluctant
		)
	}
	// Whether it's a project or not
	Capture(as: colonRef) {
		Optionally {
			":"
		}
	}
}
{% endhighlight %}

This breaks a string into several elements that the regex can be captured and then parsed.

If there's white space at the front, that can be parsed to figure out the indentation level.
If there's a hyphen, that can tell us that this should become a `task`.
If there's a colon, that can tell us that this should become a `project`.
Whatever text is there, becomes the basis of the text for the model.

### Gotchas

We'll have to do a bit of additional processing to handle the tags. Otherwise `- Task`, `- Task @tag`, `- Task @tag(payload)` would not have the identical `TPType`.

Happily, regex can help with that, too.

{% highlight swift %}
// Set up references
let textRef = Reference(Substring.self)
let payloadRef = Reference(Substring.self)

// Set up the regexes
// ie; @<tag>(<payload>)
let payloadSearch = Regex {
	"@"
	Capture(as: textRef) {
		OneOrMore(.word)
	}
	"("
	Capture(as: payloadRef) {
		OneOrMore(
			ChoiceOf {
				CharacterClass(.word)
				CharacterClass(.whitespace)
				CharacterClass(.digit)
				"-"
				":"
			}
		)
	}
	")"
	ZeroOrMore(.whitespace)
}
// ie; @<tag>
let tagSearch = Regex {
	"@"
	Capture(as: textRef) {
		OneOrMore(.word)
	}
	ZeroOrMore(.whitespace)
}
{% endhighlight %}

The parsing of this is a little more complicated, as we have two different regex to check against. (I tried to make a unified regex that would handle both `@tag` and `@tag(payload)`, but I couldn't figure out how to manage it and capture the payload properly, so I decided to just go with separate regex parsers, parse with each, and then replace `Tag.tag` models with the proper `Tag.payloadTag` models, if the exist.

## Using our regex parsers

We expect whole matches for the `tmTypeSearch`. If the whole record doesn't match, it's not a valid model.

The `tagSearch` and `payloadSearch` parses, we want to get all the matches within the string and turn them into arrays of our `Tag` model.

We also needed another pass for text with tags so that we can get the text up until the first `@` to normalize the text.

{% highlight swift %}
let normalizeText = Regex {
	Capture(as: textRef) {
		OneOrMore(.anyNonNewline, .reluctant)
	}
	ZeroOrMore(.whitespace)
	// We know there's a tag.
	"@"
	// And anything after to fill out the line.
	ZeroOrMore(.anyNonNewline)
}
{% endhighlight %}

## It can't be this easy, right?

Parsing a `String` into a `TMType` model was a necessary first step, but we also need to be able to turn an array of `String` objects into a normalized `TMType` array with populated `children` within the `TMType` models.

When the tabLevel is higher, it should become the child of the last child of the current `TMType` model. If the tabLevel is within the current `TMType` model, it should become a child at the appropriate level. Otherwise, we should append it to the list of `TMType` models, this becomes the new current `TMType` model, and continue parsing.

### Gotchas

Some useful recursive helper functions made this a lot easier:

* `func lastChild() -> TMType?`

	A method that will walk the model to find the last child or child's child.
* `func lastChild(with tabLevel: Int) -> TMType?`

	A method that will walk the model to find the last child or child's child with a specific indentation level.
* `func parent(of child: TMType) -> TMType?`

	A method that will walk the models to find the parent of a specific model.

* @discardableResult `mutating func replace(child: TMType, with updatedChild: TMType) -> Bool`

	A method to find a child within the `children` array and replace that object with the updated version.

With these, we can now dig through the various children to find the appropriate places to insert models at the proper level and then replace things up the chain.

## From Models Back to Strings

Converting from a `TMType` model to a `String` was much easier.

I made life easier for myself by setting up a simple protocol that each level of the `TMType` model could conform to.

{% highlight swift %}
protocol FormattedTMType {
    var toString: String { get }
}
{% endhighlight %}

Then I could stitch the pieces together appropriately:

### Tag models
{% highlight swift %}
var toString: String {
	switch self {
	case .tag(let tag):
		return "@\(tag)"
	case .payloadTag(let tag, let payload):
		return "@\(tag)(\(payload))"
	}
}
{% endhighlight %}

### TPType models
{% highlight swift %}
var toString: String {
	switch self {
	case .project(let string):
		return "\(string):"
	case .task(let string):
		return "- \(string)"
	case .text(let string):
		return string
	}
}
{% endhighlight %}

### TMType models
{% highlight swift %}
var toString: String {
    // Indicates the indentation level
	var result = String(repeating: "\t", count: tabLevel)
    // The `TPType` from above
	result += type.toString
	// The `Tag` from above, if present.
	tags.forEach { tag in
		if !result.isEmpty {
			result += " "
		}
		result += tag.toString
	}
	// The children, if present.
	guard !children.isEmpty else { return result }

	children.forEach { child in
		if !result.isEmpty {
			result += "\n"
		}
		result += child.toString
	}
	return result
}
{% endhighlight %}

## Testing

Testing here was especially important, so that the parsers could be verified both for the expected simple path and some of the more complicated edge cases.

We also needed to make sure that the tag payloads for our date models were parsing properly and we're able to convert from a `Tag.payloadTag` to the `TMDateType` models properly, so that we can better use these with our `EKManager` `EventKit` wrapper.

## Payload Parsing

Parsing the different date formats can be looks like it might difficult. Here's a list of the formats, from our previous blog entries:

* `2024-12-01`
* `2024-12-01 12:01`
* `2024-12-01 12:01-13:00` which could also be `2024-12-01 12:01 thru 13:00`
* `2024-12-01 12:01-2024-12-31 12:31` which could also be `2024-12-01 12:01 thru 2024-12-31 12:31`

### Gotchas

One solution might be to try to build a unified regex to handle all possible cases and capture the appropriate variables.

Another solution may be a cleaner more brute force solution that I went with:

I built a series of regex matches for each of the formats above. Then I just needed to figure out the possible match to check based upon the string length.

{% highlight swift %}
/// Helper Enum for Formatting...
enum DateParameterFormat: Equatable {
	case date
	case dateTime
	case dateTimeEndTime
	case dateTimeDateTime

	init?(from string: String) {
		switch string.count {
		case 10:
			// YYYY-mm-dd
			self = .date
		case 16:
			// YYYY-mm-dd HH:mm
			self = .dateTime
		case 22...32:
			// YYYY-mm-dd HH:mm-HH:mm
			// ...
			// YYYY-mm-dd HH:mm through HH:mm
			self = .dateTimeEndTime
		case 33...41:
			// YYYY-mm-dd HH:mm-YYYY-mm-dd HH:mm
			// ...
			// YYYY-mm-dd HH:mm through YYYY-mm-dd HH:mm
			self = .dateTimeDateTime
		default:
			return nil
		}
	}
}
{% endhighlight %}

Once that was done, the logic to compare the appropriate regex was equally clear. Check against the appropriate regex pattern, extract the captive values, plug them into our previously created date parameter structures. We can then extract the dates into our more generic `TMDateType`. My choice of doing that is to ensure that our validation logic is being called and the dates are valid.

{% highlight swift %}
func toTMDateType() throws -> TMDateType? {
	guard let format = TMDateType.DateParameterFormat(from: self) else { return nil }
...
	// Check the possible format against our regex patterns...
	switch format {
	case .date:
		// yyyy-MM-dd
		guard let match = self.wholeMatch(of: dateMatch) else {
			return nil
		}
		let year = match[yearRef]
		let month = match[monthRef]
		let day = match[dayRef]

		return try DateParameters(year: year, month: month, day: day).toDateType()

	case .dateTime:
		// yyyy-MM-dd HH:mm
		guard let match = self.wholeMatch(of: dateTimeMatch) else {
			return nil
		}
		let year = match[yearRef]
		let month = match[monthRef]
		let day = match[dayRef]
		let hour = match[hourRef]
		let minute = match[minuteRef]

		return try DateTimeParameters(date: .init(year: year, month: month, day: day),
									  time: .init(hour: hour, minute: minute)).toDateType()

	case .dateTimeEndTime:
		// yyyy-MM-dd HH:mm-HH:mm
		guard let match = self.wholeMatch(of: dateTimeEndTimeMatch) else {
			return nil
		}
		let year = match[yearRef]
		let month = match[monthRef]
		let day = match[dayRef]
		let hour = match[hourRef]
		let minute = match[minuteRef]
		let secondHour = match[secondHourRef]
		let secondMinute = match[secondMinuteRef]

		return try DateTimeEndTimeParameters(date: .init(year: year, month: month, day: day),
											 time: .init(hour: hour, minute: minute),
											 endTime: .init(hour: secondHour, minute: secondMinute)).toDateType()

	case .dateTimeDateTime:
		// yyyy-MM-dd HH:mm thru yyyy-MM-dd HH:mm
		guard let match = self.wholeMatch(of: dateTimeDateTimeMatch) else {
			return nil
		}
		let year = match[yearRef]
		let month = match[monthRef]
		let day = match[dayRef]
		let hour = match[hourRef]
		let minute = match[minuteRef]
		let secondYear = match[secondYearRef]
		let secondMonth = match[secondMonthRef]
		let secondDay = match[secondDayRef]
		let secondHour = match[secondHourRef]
		let secondMinute = match[secondMinuteRef]

		return try DateTimeDateTimeParameters(start: .init(date: .init(year: year, month: month, day: day),
														   time: .init(hour: hour, minute: minute)),
											  end: .init(date: .init(year: secondYear, month: secondMonth, day: secondDay),
														 time: .init(hour: secondHour, minute: secondMinute))).toDateType()
	}
}
{% endhighlight %}

Converting from formatted date strings to `TMDateType` will give us start and end dates, if appropriate.

Then, we just need to be able to convert from `TMDateType` back to formatted date strings. So I added a separator type enum and we can leverage the date parameter format we used above for this:

{% highlight swift %}
func toFormattedDateString(format: DateParameterFormat,
						   separator: DateFormatSeparatorType = .compact,
						   withSpaces: Bool = true) -> String? {
...
}
{% endhighlight %}

This allows us to ensure that the formats are valid by using business logic to verify things such as the `.date` or `.dateTime` are only valid if there's no `endDate` property of the `TMDateType`. Likewise, `.dateTimeEndTime` is only valid if the `endDate` is within the same calendar day as the `startDate` property.

## Testing

Unit testing allowed me to verify edge cases in my format parsing and helped me make sure that all of the variants worked properly.

The unit tests are repetitive but thorough.

---

Next week, we start working on the UI layer and getting into the SwiftUI of it all.

[post-project]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/TextModels-2024-11-24