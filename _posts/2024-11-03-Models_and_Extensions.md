---
title: Models and Extensions
date: 2024-11-03 10:00:01 -0400
tags: taskmanager models extensions date-utilities
category: coding
---

This is where we start getting into the nuts and bolts of the infrastructure for the project by building models, extensions to make the use of these models easier as I put together date utilities to make my life easier.

Luckily, `TaskManager` makes things a little easier in terms of dates. We only have two major date formats to convert to and from a `Date` foundation class: `"yyyy-MM-dd"` and `"yyyy-MM-dd HH:mm"` (ie; `"2024-10-31"` and `"2024-10-31 10:31"`).

All of the code for this blog post is in this [sample code repo][post-project].

<!--more-->

We want to make this as simple for users of these functions as possible, so the API we'd like is this:
{% highlight swift %}
let halloween = Date(format: .date, "2024-10-31")
let rembembranceDay = Date(format: .dateTime, "2024-11-11 11:00")
{% endhighlight %}

And the converse:
{% highlight swift %}
let apptDay = Date().string(format: .date)
let apptDayTime = Date().string(format: .dateTime)
{% endhighlight %}

## Gotchas

An optimistic idea of how to handle this might be to use convert via `DateComponents`. We can split the formatted strings into integer components and then convert to a date.

{% highlight swift %}
// Optimistic, but problematic conversion:
let date = Calendar.current.date(from: DateComponents(year: 2024, month: 10, day: 31)
// This will result in a date for "2024-10-31 00:00"
{% endhighlight %}

Looks good, right?

Any programmer that has worked with dates will tell you about the perils of dealing with `Date` conversions. _Leap Day_ and the _Daylight Savings Time_ are edge cases that need to be dealt with.

{% highlight swift %}
// Illustrating the problem.
let leapDay2024 = Calendar.current.date(from: DateComponents(year: 2024, month: 2, day: 29)
// This will result in a date for "2024-02-29 00:00"
let leapDay2023 = Calendar.current.date(from: DateComponents(year: 2023, month: 2, day: 29)
// This will result in a date for "2023-03-01 00:00", as there was no leap day in 2023.
{% endhighlight %}

Apple's `DateFormatter` handles both the string parsing and the validation.

## Date Extension

[Sample code][date-extensions-swift]

{% highlight swift %}
extension Date {
    // MARK: - TMDateFormat
    /// Standardized TM Date Formats
    enum TMDateFormat {
        /// Date only
        case date
        /// Date Time
        case dateTime

        /// Input/Output format string
        var string: String {
            switch self {
            case .date:
                return Constants.Date.YMD
            case .dateTime:
                return Constants.Date.YMDHM
            }
        }
    }

    // MARK: - Properties
    private var dateFormatter: DateFormatter {
        DateFormatter()
    }

    // MARK: - Standardized Input/Output Transformations
    // MARK: Output strings
    func string(format: TMDateFormat) -> String {
        let dateFormatter = DateFormatter()
        dateFormatter.dateFormat = format.string

        return dateFormatter.string(from: self)
    }

    // MARK: Initializers
    init?(format: TMDateFormat, _ input: String) {
        let dateFormatter = DateFormatter()
        dateFormatter.dateFormat = format.string
        
        guard let date = dateFormatter.date(from: input) else {
            return nil
        }
        
        self = date
    }
}
{% endhighlight %}

By setting up the internal enum in a `Date` extension, we are able to leverage this model to keep track of our known format strings and ensure that using the `TMDateFormat` in the initializer and string conversion functions for a nice and clean API usage.

## Reminder Needs

With an eye on what we want to do next, we know that we'll need to convert dates into appropriate `Set<Calendar.Components>` for creating reminder entities within `EventKit` (this is how it will handle the difference between a day of reminder and a day with time reminder) similar to our `.date` and `.dateTime` formats.

Adding the `components` property to the `TMDateFormat` ensures the API will continue to be clear and easy to use.

{% highlight swift %}
/// Date Components
var components: Set<Calendar.Component> {
	switch self {
	case .date:
		return [.year, .month, .day]
	case .dateTime:
		return [.year, .month, .day, .hour, .minute]
	}
}
{% endhighlight %}

Then in out `Date` extension, we can add a function to do the conversion:
{% highlight swift %}
private var currentCalendar: Calendar {
   Calendar.current
}

// MARK: Component Conversions
func toDateComponents(format: TMDateFormat) -> DateComponents? {
	currentCalendar.dateComponents(format.components, from: self)
}
{% endhighlight %}

## Other Conversions

The universe of `TaskManager` date formats is small and while our `.date` and `.dateTime` formats are the heart of it. There are some other formats:

* A date and time and an end time on the same day:
	* `yyyy-MM-dd HH:mm-HH:mm` (ie; 2024-10-31 10:31-12:30)
* A date and time with an end date and time:
	* `yyyy-MM-dd HH:mm`thru`yyyy-MM-dd HH:mm` (ie; 2024-10-31 10:31thru2024-11-04 12:30)

### TMDateType Model

These formats can easily be converted into a date type model:

{% highlight swift %}
enum TMDateType {
   case date(Date)
   case beginEndDate(Date, Date)
}
{% endhighlight %}

We can add some dynamic properties to extract out the start and optional end date to make usage easier:

{% highlight swift %}
var startDate: Date {
  switch self {
    case .date(let date):
       return date
    case .beginEndDate(let start, _):
       return start
  }
}

var endDate: Date? {
  guard case .beginEndDate(_, let end) = self else {
     return nil
  }
  return end
}
{% endhighlight %}

All of the sample code is here: [`TMDateType`][date-parameters-swift]

### Other Parameter Models

This is relatively straightforward, if a bit repetitive.

{% highlight swift %}
struct DateParameters {
  let year: Int
  let month: Int
  let day: Int
}

struct TimeParameters {
  let hour: Int
  let minute: Int
}

struct DateTimeParameters {
  let date: DateParameters
  let time: TimeParameters
}

struct DateTimeEndTimeParameters {
  let date: DateParameters
  let time: TimeParameters
  let endTime: TimeParameters
}

struct DateTimeDateTimeParameters {
  let start: DateTimeParameters
  let end: DateTimeParameters
}
{% endhighlight %}

But there's some infrastructure work that will need to convert our various parameter models into formatted strings and our `TMDateType` model.

By leveraging protocols and ensuring that each of the parameter models conform to them, we can easily create a uniform API for our parameter models.

{% highlight swift %}
protocol FormattedDateRepresentable {
  var formattedDate: String { get }
}

protocol ParameterConvertible {
  func toDateType() throws -> TMDateType
}
{% endhighlight %}

Then we can convert to formatted strings with a useful extension to `Int` for string formatting them with fixed digits:

{% highlight swift %}
private extension Int {
    func leadingZeroString(digits: Int = 2) -> String {
        let format = "%0\(digits)d"
        return String(format: format, self)
    }
}
{% endhighlight %}

Looking at our `DateParameters` and `TimeParameters` models:
{% highlight swift %}
struct DateParameters: FormattedDateRepresentable {
    ...
    var formattedDate: String {
        // yyyy-MM-dd
        "\(year.leadingZeroString(digits: 4))-\(month.leadingZeroString())-\(day.leadingZeroString())"
    }
}

struct TimeParameters: FormattedDateRepresentable {
	...
    var formattedDate: String {
        "\(hour.leadingZeroString()):\(minute.leadingZeroString())"
    }
}
{% endhighlight %}

Converting to `TMDateType` will use this formatted strings and our `Date` extensions. By throwing errors for the failure cases, we can alert callers to incorrect data in our parameter models, when we try to convert them into `Date` objects.

{% highlight swift %}
struct DateParameters: FormattedDateRepresentable, ParameterConvertible {
	...
    func toDateType() throws -> TMDateType {
       guard let date = Date(format: .date, formattedDate) else {
          throw TMError.invalidFormattedString(formattedDate)
       }
       
       return .date(date)
    }
}
{% endhighlight %}

That's the bulk of the logic. The rest is just stitching parameter models together with some edge case logic to ensure that end dates are always after begin dates.

{% highlight swift %}
// DateTimeParameters conversions
var formattedDate: String {
	"\(date.formattedDate) \(time.formattedDate)"
}
func toDateType() throws -> TMDateType {
	guard let date = Date(format: .dateTime, formattedDate) else {
		throw TMError.invalidFormattedString(formattedDate)
	}

	return .date(date)
}

// DateTimeEndTimeParameters conversion
func toDateType() throws -> TMDateType {
	let startDateModel = DateTimeParameters(date: date, time: time)
	let endDateModel = DateTimeParameters(date: date, time: endTime)
	guard let startDate = Date(format: .dateTime, startDateModel.formattedDate),
		  let endDate = Date(format: .dateTime, endDateModel.formattedDate) else {
		throw TMError.invalidFormattedString(formattedDate)
	}
	guard startDate < endDate else {
		throw TMError.endDateNotAfterStartDate(startDate, endDate)
	}

	return .beginEndDate(startDate, endDate)
}

// DateTimeDateTimeParameters conversion
func toDateType() throws -> TMDateType {
	guard let startDate = Date(format: .dateTime, start.formattedDate),
		  let endDate = Date(format: .dateTime, end.formattedDate) else {
		throw TMError.invalidFormattedString(formattedDate)
	}
	guard startDate < endDate else {
		throw TMError.endDateNotAfterStartDate(startDate, endDate)
	}
	
	return .beginEndDate(startDate, endDate)
}
{% endhighlight %}

With that under our belts, we can add to `Date` extension, so that we can convert to and from our parameter models.

{% highlight swift %}
// MARK: Parameter Model Conversions
func toDateParameters() -> DateParameters? {
   let components = currentCalendar.dateComponents(TMDateFormat.date.components, from: self)

   guard let year = components.year,
         let month = components.month,
         let day = components.day else {
      return nil
   }

   return .init(year: year, month: month, day: day)
}

func toDateTimeParameters() -> DateTimeParameters? {
   let components = currentCalendar.dateComponents(TMDateFormat.dateTime.components, from: self)

   guard let year = components.year,
         let month = components.month,
         let day = components.day,
         let hour = components.hour,
         let minutes = components.minute else {
      return nil
   }

   return .init(date: .init(year: year,
                            month: month,
                            day: day),
                time: .init(hour: hour,
                            minute: minutes))
}

// MARK: Initializers
init?(date parameters: DateParameters) {
   self.init(format: .date, parameters.formattedDate)
}

init?(dateTime parameters: DateTimeParameters) {
   self.init(format: .dateTime, parameters.formattedDate)
}
{% endhighlight %}

## Unit Tests

One way to ensure that your code is at the level that you want it to be and reduce your bug count is to ensure that you test your code.

Unit tests will also set your code apart if perspective employers are looking over your code sample repos, because they're an integral part of the day to day business of coding.

Let's look at some of the unit tests in this sample project:

### DateParameters Tests
[`DataParametersTests.swift`][date-parameters-tests-swift]

I'd recommend against trying to test everything. Just going with the main logic and verify the likely unhappy paths.

For example, in `DateParameters` and the `DateParametersTests`, I can do the end to end tests to verify that parameter models convert to and from dates properly and that tests the whole flow.

{% highlight swift %}
var halloween: Date {
	Date(format: .dateTime, "2024-10-31 10:31") ?? Date()
}
// MARK: - Parameter Convertible Tests
func test_dateModel_convertsProperly() throws {
	let model = try XCTUnwrap(halloween.toDateParameters())
	sut = try model.toDateType()

	let halloweenDate = try XCTUnwrap(Date(format: .date, model.formattedDate))
	XCTAssertEqual(sut, .date(halloweenDate))
}

func test_dateTimeModel_convertsProperly() throws {
	sut = try halloween.toDateTimeParameters()?.toDateType()
	XCTAssertEqual(sut, .date(halloween))
}
{% endhighlight %}

Flipping through the sample code, you'll see we also test our formatted data to ensure that functionality as well.

### DateExtensions Tests
[`DataExtensionsTests.swift`][date-extensions-tests-swift]

In `DateExtensionsTests`, unfortunately, we have to test more of the invalid paths to ensure that no likely conversion paths were forgotten, such as:

* Invalid year
* Invalid month
* Invalid day
* Invalid hour
* Invalid minute

The conversions back and forth from formatted `String` to `Date` and back was relatively simple to code up.

{% highlight swift %}
// Happy Path testing...
func test_init_fromDate_withValidInput_returnsDate() {
	let dateFromYMD = Date(format: .date, "2024-10-31")
	let dateFromParameters = Date(date: .init(year: 2024, month: 10, day: 31))
	XCTAssertEqual(dateFromYMD, dateFromParameters)
}
// Unhappy Path testing...
func test_init_fromDate_withInvalidInput_returnsNil() {
	XCTAssertNil(Date(format: .date, "0-10-31"))
	XCTAssertNil(Date(format: .date, "2024-13-31"))
	XCTAssertNil(Date(format: .date, "2024-10-32"))
	XCTAssertNil(Date(format: .date, "2024-10-31 12:30"))
	XCTAssertNil(Date(format: .date, "unknown with valid 2024-10-31 in it"))
	XCTAssertNil(Date(date: .init(year: 0, month: 10, day: 31)))
	XCTAssertNil(Date(date: .init(year: 2024, month: 13, day: 31)))
	XCTAssertNil(Date(date: .init(year: 2024, month: 10, day: 32)))
}
{% endhighlight %}

The rest of the unit tests are much of the same, as we continue to test the other conversions.

---

Next stop, the ``EventKit`` manager to ensure that the required logic wrapping the various code is as easy to use as possible.

[post-project]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Infrastructure-2024-11-03
[date-parameters-swift]:https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Infrastructure-2024-11-03/TaskManager/TaskManager/Models/DateParameters.swift
[date-extensions-swift]:https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Infrastructure-2024-11-03/TaskManager/TaskManager/Utilities/Extensions/DateExtensions.swift
[date-parameters-tests-swift]:https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Infrastructure-2024-11-03/TaskManager/TaskManagerTests/DateParametersTests.swift
[date-extensions-tests-swift]:https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Infrastructure-2024-11-03/TaskManager/TaskManagerTests/DateExtensionsTests.swift
