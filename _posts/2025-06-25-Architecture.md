---
title: Architectures for SwiftUI Projects
date: 2025-06-25 10:00:01 -0000
tags: taskmanager swiftui architecture tca mvvm
category: coding
---

Three common architectures for modern iOS apps are: **MVVM**, **TCA**, and **VIPER**.

This post will talk about using MVVM and TCA for our spec `TaskManager` app.

All of the code for this blog post is in this [sample code repo][post-project].

<!--more-->

## App Functionality

We've ensured that the three versions of the app: *MVVM*, *MVVM with Combine*, and *TCA* all have the same functionality:

* Three tabs:
  * **Task**
  
    The Task tab has a list of tasks, the ability to add a new task, an ability to edit an existing task, and an ability to delete an existing task.
    
  * **Text**
  
    The Text tab shows a single task and its children, converting from text into the task.
    
  * **Setting**
  
    A placeholder view that will have settings in the future.

Let's walk through each of the architectures in more detail:

## MVVM

MVVM is defined as follows:

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
struct TagView: View {
   @Observable
   class ViewModel {
      // MARK: Properties
      /// The edit state of the view
      var editMode: EditMode
      /// The actual tag
      private(set) var tag: Tag {
         didSet {
            text = tag.toString
         }
      }
      /// The entered text to be converted into the tag
      var text: String
   
      // MARK: Computed Properties
      var isEditing: Bool {
         editMode == .active
      }
   
      // MARK: Initializer
      init(_ tag: Tag, editMode: EditMode = .inactive) {
         self.editMode = editMode
         self.tag = tag
         self.text = tag.toString
      }
   
      // MARK: Helper Function
      /// Converts into a tag, if valid
      /// - parameter from: the text to attempt to convert
      func convertTagIfValid(from string: String) {
         guard let convertedTag = string.toTag() else {
            self.text = tag.toString
            return
         }
         self.tag = convertedTag
         self.editMode = .inactive
      }
   }

   // ...
}
{% endhighlight %}

For the `TagView`, we'll want to be able to detect whether the app is in or not in edit mode, which will trigger whether or not the view should show a static `Tag` model view or whether it should show a text field that will trigger an attempt to convert the tag and save it, if appropriate.

Let's breakdown the elements:

#### Model properties

{% highlight swift %}
   // MARK: Properties
   /// The edit state of the view
   var editMode: EditMode
   /// The actual tag
   private(set) var tag: Tag {
    didSet {
      text = tag.toString
    }
   }
   /// The entered text to be converted into the tag
   var text: String
   
   // MARK: Computed Properties
   var isEditing: Bool {
    editMode == .active
   }
{% endhighlight %}

These are all the properties that the `ViewModel` will be able to support and the `View` will be able to read/interact with. The `editMode` can be set to `.inactive` or `.active` to indicate whether or not the `View` will be an editable or non-editable view and toggle between them. The `tag` conforms to the `Tag` model and the editable text field will leverage it. The `textTag` is a string value that is populated by the `tag` and this text value is used to convert into a `tag`. There's also a helper computed property to tell whether the view is in edit mode.

#### Initializer
{% highlight swift %}
  init(_ tag: Tag, editMode: EditMode = .inactive) {
    self.editMode = editMode
    self.tag = tag
    self.textTag = tag.toString
  }
{% endhighlight %}

This simplifies the initialization by ensuring that the textTag is initialized to the string value of the `Tag` model, when the `tag` is initialized.

#### Conversion Functionality
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

If the entered text converts properly to a `Tag` entity, the tag and text are updated. (This ensures that the text is normalized. And things like `@tag()` properly becomes `@tag`.) If it doesn't, the previously saved tag is used to reset the text. Edit mode only turns off, if the text converts properly to the `tag`.

#### Using the `ViewModel` in our SwiftUI `View`

Now, we see how the `View` uses the `ViewModel`, so that it can be used.

{% highlight swift %}
@State var viewModel: ViewModel

// How it will be initialized...
TagView(viewModel: .init(Constants.MockTag.test))
{% endhighlight %}

#### How it's used in the SwiftUI `View`
{% highlight swift %}
   var body: some View {
     if viewModel.isEditing {
       tagEditView
     } else {
       tagView
     }
   }
   // Accessing the property
   tagView(tag: viewModel.tag)
   // Binding the property
   TextField(Constants.Tag.placeholder,
             text: $viewModel.text,
             axis: .vertical)
   .onSubmit {
      viewModel.convertTagIfValid(from: viewModel.text)
   }

{% endhighlight %}

The conversion of the `TagView` basically extracted individual `@State` objects into an an `@Observable` *viewModel*. And those objects are interacted with through that `@Observable` *viewModel*.

#### Navigation

Let's examine the more complicated MVVM models for our the `TaskView` and the `TaskDetailView`, wrapped within a `TaskMasterAndDetailView`. These illustrate communication between multiple views, passing data between them--as the user adds a new task from the task view, which can be edited and verified in the detail view.

We do this by defining a viewModel in the master view and bind it for the children (`TaskView` and the `TaskDetailView`) to interact with.

{% highlight swift %}
@Observable
class ViewModel {
   var selectedItem: IdentifiedTMType?
   var responseType: DetailResponseType?
   var detailMode: DetailMode = .add
   var items: [IdentifiedTMType] = []

   init(...) {...}

   // MARK: - Helper Methods
   func addItem(from text: String?) {...}

   func select(item: IdentifiedTMType) {...}

   func process(_ response: DetailResponseType?) {...}
}
{% endhighlight %}

The `IdentifiedTMType` wraps our `TMType` in an `Identifiable` and `Hashable` wrapper. This allows us to easily edit or delete a specific TMType object.
The `DetailResponseType` is an enum that allows the detail view to pass back to the master view whether the item was canceled, saved, or deleted.
The `DetailMode` allows the `TaskDetailView` to toggle between the *add* mode or the *edit* mode.

Our main property is `items` array.

Then we have our helper methods to wrap the functionality needed throughout the app: `addItem` to add a new item, `select` a known `IdentifiedItem`, and `process` the response from the `TaskDetailView`.

Each of the children, `TaskView` and `TaskDetailView` just bind the master's view model and leverage that.

People can look over the `TextView` to see how a view model can interact directly with a `TextEditor`, handle the flow between entered text that may or may not successfully convert into a valid `TMType`, as well as presenting an error message for the user. This is relatively straightforward (similar to the `TagView`), as it is a single view/view model pair with no navigation: text is entered; upon submission, an attempt to parse the text into `TMType` models and normalize them into parents and children models, as we have discussed in previous blog posts. If there are problems, an error message will be presented to the user and the type and text cleaned up appropriately.

### MVVM Flow

Seeing the MVVM Flow in action:

![MVVM Flow](/img/MVVM-2025-06-24.gif)

## MVVM (with combine)

There's a variant of the MVVM with combine that I saw in [Eduardo Sanches Bocato's][bocato-medium] medium article on [Improving MVVM (with Combine)][medium-mvvm-combine].

To sum up, there is an `Equatable` state object encapsulating the model properties. The `StateBindingViewModel` wraps the state object in functionality to allow users to access the state's properties, bind the state's properties so that it be read/write from SwiftUI views, update the state's properties, as well as ensure notification on state changes.

### MVVM (Combine) Sample Code

We won't be discussing it, but I converted the MVVM Sample Code into [MVVM (Combine) Sample Code from the repo][mvvm-combine-samplecode].


### MVVM-Combine Flow

Seeing the MVVM-Combine Flow in action:

![MVVM-Combine Flow](/img/MVVM-Combine-2025-06-24.gif)

## TCA

**T**he **C**omposable **A**rchitecture is a Swift adaption of the Redux framework. It follows a *Unidirectional Data Flow* model to ensure state management and a single source of truth.

Point Free has some really useful tutorials, if readers would like to find out more than what I've done: [TCA tutorials][tca-tutorials].

Within TCA, the three components flow from one to another in a single direction:

### State - View - Action

`State` ➡️ `View` ➡️ `Action` ↩️

Each flows from one to the other in a single direction: `State` to `View` to `Action` and back to `State`.

#### State

The state of your app. This can contain the data/model to ensure that the state is contained. As state can also refer to a specific view, it could also refer to the state of the view itself.

### View

What is seen on the screen by the user based upon the **State**, and their manipulations of the elements on the screen become an **Action**. (The initial **State** and the **Reducer**--which we'll discuss more in a moment--are in the **Store**, which we discuss a bit below.)

#### Action

All possible supported actions that the app can perform. As an action can also refer to a specific view, it could also refer to all of the available actions that the view can perform. This can be a button being tapped, a text field having text entered, a toggle toggling, etc...

### Diving a little deeper into TCA

But how does that work, you may wonder.

#### Reducer

The magic of **TCA** is that all of the business logic is in the reducer. These take an **Action**, update the **State** and return an **Effect**. At this point, I've probably lost you, so I'll put in an example:

{% highlight swift %}
struct State: Equatable {
  var lastSavedState: String = ""
  var text: String = ""
}

enum Action {
  case setText(String)
  case cancelButtonTapped
  case saveButtonTapped
}

// Extracted reducer
Reduce { state, action } in
   switch action {
      case let .setText(text):
        state.text = text
        return .none
      case .cancelButtonTapped:
        state.text = state.lastSavedState
        return .none
      case .saveButtonTapped:
        state.lastSavedState = state.text
        return .none
   }
{% endhighlight %}

In this case, we have a state that includes two properties: `lastSavedState` and `text`. We have a simple set of actions, the user can enter text, and tap on either a cancel or save button.

All of the business logic is in the reducer.

* Entering text updates the text property in the **State**.
* Tapping the cancel button, sets the text back to the last saved state.
* Tapping the save button, sets the last saved state.

#### Effect

An effect is a wrapper around any piece of work or a task such a network call or an asynchronous task. This could result in a new *Action* that can be fed back into the *Reducer*.

In our example above, each case returns `.none` ending the flow, but in a more complicated model, we may have an error state such as this:

{% highlight swift %}
struct State: Equatable {
  var errorMessage: String?
  var lastSavedState: String = ""
  var text: String = ""
}

enum Action {
  case setText(String)
  case cancelButtonTapped
  case saveButtonTapped
  case error(String)
}

// Extracted reducer
Reduce { state, action } in
   switch action {
      case let .setText(text):
        state.text = text
        return .none
      case .cancelButtonTapped:
        state.text = state.lastSavedState
        return .none
      case .saveButtonTapped:
        guard let validState = validate(state.text) else {
          return .send(.error(Constant.InvalidState))
        }
        
        state.lastSavedState = validState
        return .none
      case let .error(errorMessage):
        state.errorMessage = errorMessage
        return .none
   }
{% endhighlight %}

This illustrates the way a reducer can return a different action so that the reducer will then process that new action.

#### Environment

This layer is where network, persistent storage, OS service functionality goes. We're not doing much with that, but we will as we expand are sample code.

#### Store

This wraps everything together, including the initial state and the reducer, as well as the initializer. This is bound into the **View**.

### TCA Sample Code

All of this will be based upon the [TCA Sample Code from the repo][tca-samplecode].

Let's examine our TCA version of the `TagView` that we built previously, as we incorporate the `TCA` methodology. (There are three different versions of it: `TCASimpleBindingTagFeature`, `TCATagFeatureFirstPass`, and `TCATagFeatureFinalPass`.)

In `TCASimpleBindingTagFeature`, there aren't specific `Action` elements. Instead, we just bind the state. This makes it so that the `View` directly changes the `State`. This feels like it breaks the spirit of TCA, so I tried to do better.

{% highlight swift %}
// MARK: Action
enum Action: BindableAction, Sendable {
   case binding(BindingAction<State>)
}

// MARK: Body (Reducer)
var body: some Reducer<State, Action> {
   BindingReducer()
}

// And in our view directly affecting the State:
   TextField(Constants.Tag.placeholder,
             text: $store.text,
             axis: .vertical)
   .onSubmit {
     onTextFieldSubmission()
   }

func onTextFieldSubmission() {
   defer {
     store.editMode = .inactive
   }

   guard let updatedTag = store.text.toTag() else {
     // TODO: Present an error
     store.text = store.tag.toString
     return
   }
   store.tag = updatedTag
}
{% endhighlight %}

In `TCATagFeatureFirstPass`, we now have an `Action` enum defined with actual things that a user can do:

{% highlight swift %}
enum Action {
   case tapped
   case entered(String)
   case submitted(String)
   case saved(Tag)
   case error(String)
}

// MARK: Body (Reducer)
var body: some ReducerOf<Self> {
   Reduce { state, action in
     switch action {
     case .tapped:
       state.editMode = .active
       return .none

     case let .entered(text):
       state.text = text
       return .none

     case let .submitted(text):
       guard let newTag = text.toTag() else {
         let errorMessage = "Unable to convert <\(text)> into a tag."
         return .send(.error(errorMessage))
       }

       return .send(.saved(newTag))

     case let .error(errorMessage):
       state.errorMessage = errorMessage
       return .none

     case let .saved(newTag):
       state.tag = newTag
       state.errorMessage = nil
       return .none
     }
   }
}
{% endhighlight %}

We can clean that up a bit more. And our state having `didSet` functionality ensures that the business logic is in two places, the reducer and the state. Let's clean that up a bit more in our final version of the `TCATagView`:

#### Reducer

We define our state and the actions available. You'll note that I've taken the liberty of leveraging our previous work with the `TextView` code from the *MVVM* and *MVVM-Combine* examples.

In the **State**, we see the properties of the feature:

{% highlight swift %}
@Reducer
struct TagConverter {
   // MARK: State
   @ObservableState
   struct State: Equatable {
      // MARK: Properties
      /// An optional error message to present to the user, if present
      var errorMessage: String?
      /// The edit state of the view
      var editMode: EditMode = .inactive
      /// The actual tag
      var tag: Tag
      /// The entered text to be converted into the tag
      var text: String

      // MARK: Computed Properties
      var isEditing: Bool {
         editMode == .active
      }
      var shouldPresentError: Bool {
         errorMessage != nil
      }

      // MARK: Initializer
      init(tag: Tag) {
         self.tag = tag
         self.text = tag.toString
      }
   }
{% endhighlight %}

In the **Action**, we see the behaviors/functionality supported in the feature:

{% highlight swift %}
   // MARK: Action
   enum Action {
      /// When the user taps on the tag to initiate editing
      case tapped
      /// When the user changes the text in the text field
      case entered(String)
      /// When the user taps return to submit the answer
      case submitted
   }
{% endhighlight %}

In the **Body** of the reducer, we see the flow from action to action supported in the feature:

A future developer can easily see the flow of the feature:

* A user taps a view, which toggles the editMode into an active state.
* When editable, a user can enter text into the text field, which can be submitted.
* When submitted, an attempt is made to convert the text into a valid `Tag` object.
* If not successful, an error is presented.
* If successful, the tag is updated, the text normalized, and any presented errors are cleared away.

(You'll note that the `State` object, leverages `didSet` functionality to ensure that the `text` and `editMode` properties are updated appropriately on changes to the `task` and `errorMessage` state properties.)

{% highlight swift %}
   // MARK: Body (Reducer)
   var body: some ReducerOf<Self> {
     Reduce { state, action in
       switch action {
       case .tapped:
         state.editMode = .active
         return .none
   
       case let .entered(text):
         state.text = text
         return .none
   
       case .submitted:
         // As you can see, all the logic is now in the reducer.
         // Extraneous actions are removed so this is pared down to just what
         // is needed.
         guard let newTag = state.text.toTag() else {
            // In the error case, the error is created and state set up to
            // present it all from within the submission.
            let errorMessage = "Unable to convert <\(state.text)> into a tag."
            state.errorMessage = errorMessage
            return .none
         }
   
         // In the success case, we set the updated tag.
         state.tag = newTag
         state.errorMessage = nil
         // Clear the state
         state.editMode = .inactive
         state.text = newTag.toString
         return .none
       }
     }
   }
{% endhighlight %}

#### View

The magic happens in the the `View`. This is where we bind our store and access it.

{% highlight swift %}
struct TCATagView: View {
   @Bindable
   var store: StoreOf<TagConverter>
   @ScaledMetric(relativeTo: .caption) private var scaledPadding = Spacing.default

    var body: some View {
        VStack {
            if let errorMessage = store.errorMessage {
                Text(errorMessage)
                    .font(.caption)
                    .foregroundStyle(.red)
            }
            if store.isEditing {
                editTagView
            } else {
                tagView
            }
        }
    }
}
   // ...
   // Accessing a store property and sending in an action:
   // Accessing a store property
   textTag(tag: store.tag)
      .onLongPressGesture {
         // Sending in an action
         store.send(.tapped)
      }
   // Binding a store property
   TextField(Constants.Tag.placeholder,
             text: $store.text.sending(\.entered),
             axis: .vertical)
      .onSubmit {
         store.send(.submitted)
      }
{% endhighlight %}

#### Navigation

Unlike in the MVVM examples, the flow between features within TCA architecture is done differently. The TCA Tutorials explain much better than I can (with ample examples that build upon the information learned), but I will try to walk readers through how I've used it in the Task flows within my sample code.

**TCATaskFeature**

With our state, we define our identified array of tasks, so that we can can present them, add to our array or remove from the array. We also have two different types of destinations: an alert to confirm deletion and our add/edit feature where an item (new or existing) can be edited.

{% highlight swift %}
@Reducer
struct TCATaskFeature {
   // MARK: State
   @ObservableState
   struct State: Equatable {
      var tasks: IdentifiedArrayOf<TCATask> = []

      @Presents var alert: AlertState<Action.Alert>?
      @Presents var destination: Destination.State?
   }
{% endhighlight %}

The actions supported within our feature: a new task can be added, an alert can communicate back, a destination can communicate back, a deletion has been requested (and an alert is needed to confirm it), and an edit task has been initiated.

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

When we walk through the reducer itself, we can see how the actions flow from one to another:

* When the add button is tapped, the add/edit feature becomes a destination for a newly created task in *add* mode.
* When the add/edit feature sends back a save response, the newly created (and now properly edited) task is added into the store.
* An add cancelation, ensures nothing happens.
* When an item has been selected, the add/edit feature becomes a destination for the selected task in *edit* mode.
* When the add/edit feature sends back a save response, the updated task is replaced in the store.
* When the add/edit feature sends back a delete response, a delete sent action is sent.
* An edit cancelation, ensures nothing happens.
* When a delete sent action is sent, an alert is presented to confirm the deletion.
* An alert that isn't confirmed, does nothing.
* An alert once confirmed, ensures that the task is deleted.

{% highlight swift %}
   // MARK: Body
   @Dependency(\.uuid) var uuid
   var body: some ReducerOf<Self> {
      Reduce { state, action in

         switch action {
         case .addButtonTapped:
            state.destination = .addTask(
               TCAAddEditTaskFeature.State(
                  mode: .add,
                  task: TCATask(id: self.uuid(),
                                task: .init(type: .text("")))
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
               TCAAddEditTaskFeature.State(
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
}
{% endhighlight %}

When the destinations (alert or add/edit feature) are triggered, this is handled by the destination states and actions that allow the reducer to handle those.

{% highlight swift %}
// MARK: - Destination
extension TCATaskFeature {
   // MARK: Reducer
   @Reducer
   enum Destination {
      case addTask(TCAAddEditTaskFeature)
      case editTask(TCAAddEditTaskFeature)
   }
}
// MARK: State
extension TCATaskFeature.Destination.State: Equatable {}
{% endhighlight %}

**TCATaskFeatureView**

As we did with the `TCATagView`, the store is bound. And the view accesses the store properties, sends actions via the store, or binds elements of the store such as the scope.

{% highlight swift %}
struct TCATaskFeatureView: View {
   @Bindable var store: StoreOf<TCATaskFeature>

   var body: some View {
      NavigationStack {
         List {
            ForEach (store.tasks) { task in

               HStack {
                  Text(task.task.type.toString)
                     .bodyMode() // A view modifier for `Text` view.
                  Spacer()
                  Button {
                     store.send(.editTask(task))
                  } label: {
                     Image.Task.Icon.edit
                        .tint(Color.green.opacity(0.8))
                  }
                  .buttonStyle(.borderless)
               }

            }
         }
         .navigationTitle(Constants.TaskView.title)
         .toolbar {
            ToolbarItem {
               Button {
                  store.send(.addButtonTapped)
               } label: {
                  Image(systemName: "plus")
               }
            }
         }
      }
       // Defines the add task version of the `TCAAddEditTaskView`.
       .sheet(
         item: $store.scope(state: \.destination?.addTask,
                            action: \.destination.addTask)
      ) { addTaskStore in

         NavigationStack {
            TCAAddEditTaskView(store: addTaskStore)
         }
      }
       // Defines the edit task version of the `TCAAddEditTaskView`.
       .sheet(
         item: $store.scope(state: \.destination?.editTask,
                            action: \.destination.editTask)
      ) { editTaskStore in

         NavigationStack {
            TCAAddEditTaskView(store: editTaskStore)
         }
      }
       // Defines the alert presentation.
       .alert($store.scope(state: \.alert, action: \.alert))
   }
}
{% endhighlight %}

### TCA Flow

Seeing the TCA Flow in action:

![TCA Flow](/img/TCA-2025-06-24.gif)

---

Next article, we'll delve further into testing MVVM and TCA architectures, as I decide which better fits what I want to do with the `TaskManager` app.

[post-project]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Architect-2025-05-25
[mvvm-samplecode]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Architect-2025-05-25/MVVM/TaskManager
[mvvm-combine-samplecode]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Architect-2025-05-25/MVVM-Combine/TaskManager
[tca-samplecode]: https://github.com/Jp4Mobile/SampleCode/tree/main/posts/projects/Architect-2025-05-25/TCA/TaskManager
[hws-viewmodel]: https://www.hackingwithswift.com/books/ios-swiftui/introducing-mvvm-into-your-swiftui-project
[medium-mvvm-combine]: https://bocato.medium.com/improving-mvvm-forms-in-swiftui-14b032065095
[bocato-medium]: https://bocato.medium.com
[tca-tutorials]: https://pointfreeco.github.io/swift-composable-architecture/main/tutorials/meetcomposablearchitecture/