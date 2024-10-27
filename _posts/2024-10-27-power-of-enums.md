---
title: "The Power of Enums in Swift"
date: 2024-10-27 10:00:01 -0400
tags: taskmanager enums
category: coding

---

Enums are one of my favorite parts of the Swift language. They're one of the most powerful value-based objects.

In this post, we'll be discussing increasingly complex uses of enums.

All of the code from this blog post is in a [playground][post-playground].

<!--more-->

## Simple Enums

At it's simplest enums are just different known options of something.

```
enum ColorSimple {
   case red, orange, yellow, green, blue, purple
}
```

This can be nice, but if the programmer wants to allow the user to select  from the known color choices, this will need display text.

```
func name(from color: ColorSimple) -> String {
   switch color {
    case .red:
        return "Red"
    case .orange:
        return "Orange"
    case .yellow:
        return "Yellow"
    case .green:
        return "Green"
    case .blue:
        return "Blue"
    case .purple:
        return "Purple"
   }
}
```

This certainly works, but it's repetitive and could lead to a confusing code base with functionality having to do with the colors possibly spread over a number of files.

Luckily, enums can have raw values associated with them, as well as functions and dynamic variables.

## Enums With Raw Values

When dealing with network calls and models, there could be numeric codes for a REST call's arguments.

```
enum UserType: Int {
  case unenrolled = 1
  case subscription // This would have a raw value of 2
  case lifetime = 99
}
```

These could be used in a hypothetical query.

```
let users = try await allUsers(of: .unenrolled)
```

Back to the colors example, cleaned up a bit.
```
enum ColorRaw: String {
  case red
  case orange
  case yellow
  case green
  case blue
  case purple
  
  /// Name to display
  var displayName: String {
    self.rawValue.capitalized
  }
  
  /// SwiftUI Context Color
  func swiftUIColor() -> Color {
	switch self {
	case .red:
		return .red
	case .orange:
		return .orange
	case .yellow:
		return .yellow
	case .green:
		return .green
	case .blue:
		return .blue
	case .purple:
		return .purple
	}
  }
}
```

Unsurprisingly, this means that enums can be protocol conformant, making them even more powerful.

```
enum LoginError: Error {
  case invalidEmail
  case invalidPassword
  case networkError
}
```

And allow developers to use them in increasingly varied ways.

## Enums With Associated Values

Enums can also have associated values, which may simplify APIs.

```
enum DateFormat {
  /// YMD
  /// ie; (yyyy-MM-dd)
  case date(Int, Int, Int)
```
While this works, it's not hard to see that this could grow to be a problem and more burdensome as the number of arguments grow. Instead it might be easier to handle with specific model structures.
```
  /// YMD HM YMD HM
  case dateTimeDateTime(DateTimeDateTimeParameters)
}
```

Not only is this easier to read and use, but it allows the ability to use models that may have logic that would allow the normalization and validation within the models.

## Enums as arguments

They can also be used as arguments to clarify APIs.

ie; `print(today.string(format: .date))` Which is simple and easily understood as, _Print the variable of today as a formatted string using the date format._

That can be done rather easily.

```
extension Date {
  /// TaskMaster Specific date formats
  enum TMDateFormat {
    /// Date in our YMD format
    /// ie; yyyy-MM-dd
    case date
    /// Date in our YMD H:S format
    /// ie; yyyy-MM-dd HH:mm
    case dateTime
    
    var format: String {
      switch self {
        case .date:
          return "yyyy-MM-dd"
        case .dateTime:
          return "yyyy-MM-dd HH:mm"
      }
    }
  }
  
  /// Create a formatted string with our known formats.
  func string(format: TMDateFormat) -> String {
    let dateFormatter = DateFormatter()
    dateFormatter.dateFormat = format.format
    return dateFormatter.string(from: self)
  }
}
```

## Incidental Use of Enums

Another useful element of enums is that they cannot be instantiated in other ways, too.

```
enum Constant {
    enum Feature {
        enum Welcome: FeatureDescribable {
            static let title: String = "Welcome"
        }
        enum Settings: FeatureDescribable {
            static let title: String = "Settings"

            enum Section: String, CaseIterable {
                case license = "Open Source Licenses"
                case privacy = "Privacy Policy"
                case support = "Support"
            }
        }
    }
}
```

[post-playground]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/playgrounds
