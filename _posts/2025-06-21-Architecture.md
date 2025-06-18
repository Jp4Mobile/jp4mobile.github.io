---
title: Architectures for SwiftUI Projects
date: 2025-06-21 10:00:01 -0400
tags: taskmanager swiftui architecture tca mvvm
category: coding
---

Three common architectures for modern iOS apps are: **MVVM**, **TCA**, and **VIPER**.

This post will talk about using MVVM and TCA for our spec `TaskManager` app.

All of the code for this blog post is in this [sample code repo][post-project].

<!--more-->

## MVVM

### Model - View - View Model 

`View` ↔️ `View Model` ↔️ `Model`

#### Model

As we've talked through multiple blog posts so far, we have a lot of models that make up the various elements of the Task Manager data.

#### View

These will be our UI/UX layer, as described above. It won't communicate directly with the model. Each view will have the appropriate *View Model*.

#### View Model

Each view model handles both the data segregation so that each *View* only sees the data that they're supposed to as well as the business logic to ensure that the *Model* can be updated.

### MVVM Sample Code

All of this will be based upon the [MVVM Sample Code from the repo][mvvm-samplecode].

Let's examine the `TagView` that we built previously, as we incorporate view models. I like Paul Hudson's idea of incorporating the definition of the view model within the SwiftUI view struct to make it clear which `ViewModel` is associated with which SwiftUI view. (He outlined this idea in his MVVM post: [Hacking With SwiftUI on MVVM][hws-viewmodel]).

{% highlight swift %}
@Observable
class ViewModel {
  // Model properties
  var editMode: EditMode
  private(set) var tag: Tag
  var textTag: String

  init(_ tag: Tag, editMode: EditMode = .inactive) {
    self.editMode = editMode
    self.tag = tag
    self.textTag = tag.toString
  }

  func convertTagIfValid(from string: String) {
    guard let convertedTag = string.toTag() else {
      self.textTag = tag.toString
      return
    }
    self.tag = convertedTag
    self.textTag = convertedTag.toString
  }
}
{% endhighlight %}

For the `TagView`, we'll want to be able to detect whether the app is in or not in edit mode, which will trigger whether or not the view should show a static `Tag` model view or whether it should show a text field that will trigger an attempt to convert the tag and save it, if appropriate.

Let's breakdown the elements:

#### Model properties

{% highlight swift %}
// Model properties
var editMode: EditMode
private(set) var tag: Tag
var textTag: String
{% endhighlight %}

These are all the properties that the `ViewModel` will be able to support and the `View` will be able to read/interact with. The `editMode` can be set to `.inactive` or `.active` to indicate whether or not the `View` will be an editable or non-editable view and toggle between them. The `tag` conforms to the `Tag` model and the editable text field will leverage it. the `textTag` is a string value that will be read after editing and if valid, convert into the `tag`.

#### Initializer
{% highlight swift %}
  init(_ tag: Tag, editMode: EditMode = .inactive) {
    self.editMode = editMode
    self.tag = tag
    self.textTag = tag.toString
  }
{% endhighlight %}

Simplify the initializer with a valid tag and whether or not the `TagView` starts in edit mode.

The textTag is always initialized by a valid `Tag` entity.

#### Conversion Functionality
{% highlight swift %}
  func convertTagIfValid(from string: String) {
    guard let convertedTag = string.toTag() else {
      self.textTag = tag.toString
      return
    }
    self.tag = convertedTag
    self.textTag = convertedTag.toString
  }
{% endhighlight %}

If the entered string successfully converts into a `Tag` entity, the `tag` changes and updates the string to ensure that it is normalized. Otherwise, we toss out the invalid text and revert back to the current tag as a string.

#### Using the `ViewModel` in our SwiftUI `View`

First, the `View` will need a `ViewModel` entity so that it can be interacted with. We just need to define the property and use that in our initializers:

{% highlight swift %}
@State var viewModel: ViewModel

// How it will be initialized...
TagView(viewModel: .init(Constants.MockTag.test))
{% endhighlight %}

#### How it's used in the SwiftUI `View`
{% highlight swift %}
   // TextField binding...
   TextField(Constants.Tag.placeholder,
             text: $viewModel.textTag,
             axis: .vertical)
   // Property access
    textTag(tag: viewModel.tag)
{% endhighlight %}

So basically, we just converted from the previous use where the `editMode`, `textTag`, and `tag` were `@State` properties that could be accessed direction and moved into an `@Observable` _view model_, where they're interacted with from within that. In this use case, the view model isn't doing much, just reading, writing, stateful-binding and that's about it.

#### Navigation

A typical iOS app will have navigation, as such, our sample app supports tabs and push navigation.

Some of the ways that I've seen this illustrated on blogs involve convoluted functionality with generated `NavigationLink` entities, etc... I'll be honest, I wasn't happy with that. I tend to prefer simpler and easier to understand; this way other developers can look at my code and understand things quickly.

Based on that, I went with a simpler model of generating the appropriate `ViewModel` entities for detail views.

{% highlight swift %}
ForEach (viewModel.items) { item in
   /*
    I like solution: generate a subsequent ViewModel from the current ViewModel 
    and then inject that into the new view. I feel that this better fits the 
    MVVM paradigm. An example of this would be a list of movies that can 
    subsequently present a movie detail page upon movie card click.
    */
   NavigationLink {
      let viewModel: TaskDetailView.ViewModel = .init(item: item)
      return TaskDetailView(viewModel: viewModel)
   } label: {
      Text(item.type.toString)
   }
}
{% endhighlight %}

## MVVM (with combine)

There's a variant of the MVVM with combine that I saw in [Eduardo Sanches Bocato's][bocato-medium] medium article on [Improving MVVM (with Combine)][medium-mvvm-combine].

The minor changes are ensuring that there's an `Equatable` state object encapsulating the model properties. And then binding the elements leveraging `WritableKeyPath` functionality.

### MVVM (Combine) Sample Code

All of this will be based upon the [MVVM (Combine) Sample Code from the repo][mvvm-combine-samplecode].

Let's examine the changes to `TagView` from the previous example.

#### The _state_ model.
{% highlight swift %}
struct TagState: Equatable {
   var editMode: EditMode
   var tag: Tag {
      // Ensure that the textTag is always appropriately set.
      didSet {
         self.textTag = tag.toString
      }
   }
   var textTag: String

   init(_ tag: Tag, editMode: EditMode = .inactive) {
      self.editMode = editMode
      self.tag = tag
      self.textTag = tag.toString
   }
}
{% endhighlight %}

This wraps the appropriate model properties and encapsulates the logic to ensure that the editable string is always set to the string representation of the tag model.

#### ViewModel Leverages Bocato's Generic `StateBindingViewModel`

As you can see in Bocato's [Medium article][medium-mvvm-combine], as well as the code within the our sample code. The updated view model leverages both the binding by key path functionality and the update a key path to a given value functionality within.

{% highlight swift %}
final class ViewModel: StateBindingViewModel<TagState> {
   init(_ tag: Tag, editMode: EditMode = .inactive) {
      super.init(initialState: .init(tag, editMode: editMode))
   }

   var isEditing: Bool {
      state.editMode == .active
   }

   func convertTagIfValid(from string: String) {
      guard let convertedTag = string.toTag() else {
         update(\.textTag, to: state.tag.toString)
         update(\.editMode, to: .inactive)
         return
      }
      update(\.tag, to: convertedTag)
      update(\.editMode, to: .inactive)
   }
}
{% endhighlight %}

Walking through the various elements of the `ViewModel`, we have an initializer that takes our `Tag` entity and sets up both the internal `TagState` and the `ViewModel`. Then our dynamic `isEditing` property, so that we can simplify our checks against `viewModel.editMode == .active`. And then we've updated our `convertTagIfValid` functionality to leverage the `state` and the `update` functionality.

#### Changes to the `View` based on our changes.

{% highlight swift %}
   // TextField binding...
   TextField(Constants.Tag.placeholder,
             text: viewModel.binding(\.textTag),
             axis: .vertical)
   // Property access
   textTag(tag: viewModel.state.tag)
{% endhighlight %}


So basically, we just converted from the previous use where the `editMode`, `textTag`, and `tag` were `@State` properties that could be accessed direction and moved into an `@Observable` _view model_, where they're interacted with from within that. In this use case, the view model isn't doing much, just reading, writing, stateful-binding and that's about it.

![MVVM (Combine) Flow](/img/MVVMCombine-Flow-2025-06-21.gif)

## TCA

**T**he **C**omposable **A**rchitecture is a Swift adaption of the Redux framework. It follows a *Unidirectional Data Flow* model to ensure state management and a single source of truth.

The three components flow from one to another in a single direction:

### State - View - Action

`State` ➡️ `View` ➡️ `Action` ↩️

Each flows from one to the other in a single direction: `State` to `View` to `Action` and back to `State`.

#### State

The state of your app. This can contain the data/model to ensure that the state is contained. As state can also refer to a specific view, it could also refer to the state of the view itself.

#### Action

All possible supported actions that the app can perform. As action can also refer to a specific view, it could also refer to all of the available actions that the view can perform.

#### Reducer

Encapsulates the business logic, taking an *Action* and returns an *Effect*.

#### Effect

An effect is a wrapper around any piece of work or a task such a network call or an asynchronous task. This could result in a new *Action* that can be fed back into the *Reducer*.

#### Environment

This layer is where network, persistant storage, OS service functionality goes.

#### Store

This wraps everything together, including the initial state and the reducer, as well as the initializer.

### TCA Sample Code

All of this will be based upon the [TCA Sample Code from the repo][tca-samplecode].

Let's examine the `TagView` that we built previously, as we incorporate the `TCA` methodology.

#### Reducer

We define our state and the actions available.

{% highlight swift %}
@Reducer
struct TagConverter {
    // MARK: State
    @ObservableState
    struct State: Equatable {
        var editMode: EditMode = .inactive
        var initialTag: Tag
        var tag: Tag {
            didSet {
                text = tag.toString
            }
        }
        var text = ""

        init(tag: Tag) {
            self.initialTag = tag
            self.tag = tag
            self.text = tag.toString
        }
    }

    enum Action: BindableAction, Sendable {
        case binding(BindingAction<State>)
    }

    var body: some Reducer<State, Action> {
        BindingReducer()
    }
}
{% endhighlight %}

#### View

The magic happens in the the `View`. We bind our store.

{% highlight swift %}
struct TCATagView: View {
    @Bindable var store: StoreOf<TagConverter>
    @ScaledMetric(relativeTo: .caption) private var scaledPadding = Spacing.default

    var body: some View {
        Group {
            if store.editMode == .active {
                textFieldTag
            } else {
                textTag
            }
        }
    }
}
{% endhighlight %}

Then we shift the text field binding and the view property access.

{% highlight swift %}
// TextField binding...
TextField(Constants.Tag.placeholder,
          text: $store.text,
          axis: .vertical)
// Property access
textTag(tag: store.tag)

{% endhighlight %}

Without a view model, we handle the conversion within a different function.

{% highlight swift %}
func onTextFieldSubmission() {
   defer {
      store.editMode = .inactive
   }

   guard let updatedTag = store.text.toTag() else {
      print("*Jp* \(self)::\(#function)[\(#line)] unable to convert")
      store.text = store.tag.toString
      return
   }
   store.tag = updatedTag
}
{% endhighlight %}

### Navigation and Alert Presentation

Within `TCA`, presenting alerts and other views and the communication done between them depend on the definition of state, actions, and the reducer. I added some more complications from the [TCA tutorials][tca-tutorials].

#### TCA Task Feature Walk Through

Let's break things down element by element:

##### Model

{% highlight swift %}
struct TCATask: Equatable, Identifiable {
   let id: UUID
   var text: String
}
{% endhighlight %}

##### Reducer (State)

{% highlight swift %}
@ObservableState
struct State: Equatable {
   var tasks: IdentifiedArrayOf<TCATask> = []

   @Presents var alert: AlertState<Action.Alert>?
   @Presents var destination: Destination.State?
}
{% endhighlight %}

###### Supported Actions

{% highlight swift %}
enum Action {
   case addButtonTapped
   case alert(PresentationAction<Alert>)
   case deleteSent(TCATask)
   case destination(PresentationAction<Destination.Action>)
   case editTask(TCATask)

   @CasePathable
   enum Alert: Equatable {
      case confirmDeletion(id: TCATask.ID)
   }
}
{% endhighlight %}

This illustrates our feature functionality.

Users can add a new task, edit an existing task, present an alert to confirm deleting an existing task, or delete an existing task, if it has been confirmed.

##### Reducer

{% highlight swift %}
@Dependency(\.uuid) var uuid
var body: some ReducerOf<Self> {
   Reduce { state, action in

      switch action {
      case .addButtonTapped:
         state.destination = .addTask(
            AddEditTCATaskFeature.State(
               mode: .add,
               task: TCATask(id: self.uuid(),
                            text: "")
            )
         )

         return .none

      case let .alert(.presented(.confirmDeletion(id: id))):
         state.tasks.remove(id: id)
         return .none

      case .alert:
         return .none

      case let .deleteSent(task):
         state.alert = AlertState {
            TextState(Constants.Alert.message)
         } actions: {
            ButtonState(role: .destructive,
                     action: .confirmDeletion(id: task.id)) {
               TextState(Constants.Alert.deleteTitle)
            }
         }
         return .none

      case let .destination(.presented(.addTask(.delegate(.saveTask(task))))):
         state.tasks.append(task)

         return .none

      case let .destination(.presented(.editTask(.delegate(.saveTask(task))))):

         guard let index = state.tasks.firstIndex(where: { $0.id == task.id }) else {
            return .none
         }

         state.tasks[index] = task

         return .none

      case let .destination(.presented(.editTask(.delegate(.deleteTask(task))))):

         return .send(.deleteSent(task))

      case .destination:
         return .none

      case let .editTask(task):

         state.destination = .editTask(
            AddEditTCATaskFeature.State(
               mode: .edit,
               task: task
            )
         )

         return .none
      }

   }
   .ifLet(\.$destination,
         action: \.destination) {
      Destination.body
   }
   .ifLet(\.$alert, action: \.alert)
}
{% endhighlight %}

Our reducer illustrates how each state flows into one another:

* When the add button is tapped, a new empty task is created and the feature flows into the add/edit feature. The view is in add mode, where the user can create the new task and either save or cancel the new task.
* If canceled, nothing happens; if the task is saved, the feature flows back to the list of existing tasks.
* If the edit button on a specific task is tapped, then the feature flows to the add/edit feature. The view is in edit mode, where the user can edit the task, or delete, save, or cancel the task.
* If canceled, nothing happens; if the task is saved, the feature flows back to the list of existing tasks.
* If deleted, the feature flows back to the list of existing tasks, and an alert is being presented to confirm the deletion.
* If the deletion is not confirmed, the alert is dismissed and nothing happens; if the deletion is confirmed, the alert is dismissed and the existing task is deleted.

![TCA Flow](/img/TCA-Flow-2025-06-21.gif)

---

Next article, we'll delve further into testing MVVM and TCA.

[post-project]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Architect-2025-05-25
[mvvm-samplecode]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Architect-2025-05-25/MVVM/TaskManager
[mvvm-combine-samplecode]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Architect-2025-05-25/MVVM-Combine/TaskManager
[tca-samplecode]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Architect-2025-05-25/TCA/TaskManager
[hws-viewmodel]: https://www.hackingwithswift.com/books/ios-swiftui/introducing-mvvm-into-your-swiftui-project
[medium-mvvm-combine]: https://bocato.medium.com/improving-mvvm-forms-in-swiftui-14b032065095
[bocato-medium]: https://bocato.medium.com
[tca-tutorials]: https://pointfreeco.github.io/swift-composable-architecture/main/tutorials/meetcomposablearchitecture/