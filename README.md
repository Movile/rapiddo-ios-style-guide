# The Rapiddo iOS Style Guide

Welcome to the first Brazilian Superapp! In a project as big and sensitive as Rapiddo, things can get out of hand pretty quickly. Features are very different and have little correlation to each other, requiring major coordination between our team members. Because of that, and due to frequent team changes caused by the birth of projects like Rapiddo Pague and the Marmotex SDK, we created this guide to be sure that the iOS team members could easily hop between projects (and Marketplace modules) without major problems. There are minor differences between the projects/modules, but naming conventions and the overall feel of the project tends to be the same.

# Rapiddo Marketplaces's Module System

How could you possibly create an app with a full Taxi funcionality, two different Food ordering funcionatlities, a News reader, a Coupons feature, a full Music player with a monthly subscription, Phone topups and still have something that is barely maintainable at the end? How could you possibly have 6 screens called `HomeViewController` at the same app and still have things work correctly?

This issue was solved by using the power of Swift modules. You'll find that the main Rapiddo project doesn't actually contain any of these features - all the major feature of Rapiddo (called Providers) are instead internal CocoaPods projects completely independent from the main Rapiddo project. This not only allows us to develop new features without having to build the entire project, we can also empower namespacing to have several screens with the same name. You can find these projects at the `/Dependencies` folder of the main Rapiddo repository.

![Module Graph](https://i.imgur.com/W8dZBjY.png)

[Movile has a public video where we speak about Rapiddo's architecture. (Portuguese)](https://www.youtube.com/watch?v=mowj7aYf1x4)

If you are a Rapiddo employee, you  can find documentation that explains RapiddoCore in depth at the main repository's README.

## Creating new screens in Rapiddo

We use MVVM-C (Model View View Model with Coordinators) as our architecture with our own Coordinator implementation (which you can find documentation for in the main project's README)

There are no Storyboards in Rapiddo projects - everything is done via code. While people have mixed opinions about this, we believe that view coding is the best option as it allows multiple people to code in the same screen at the same time, merge conflicts are simple and you gain more control over your screens, not to mention that you don't have to wait 10 minutes for Xcode to load the visual representation of your screen - everything is just regular code.

In addition, this allows use to use dependency injection in a straight forward manner. You'll see that there are no singletons in Rapiddo - everything relevant to a class must be directly sent to it. This allows us to write better tests and make sure that classes can only do what they are supposed to.

This section details how to create a new screen called `MyScreen`. Please note that we have a Sourcery template in the `/ClassGen` folder that you can use to generate all the files needed to create a new screen using our architecture.

In normal conditions, `MyScreen` would consist of:

### `MyScreenCoordinator`

In MVVM-C, The Coordinator is the object responsible for handling screen transitions. It retains it's inner `UIViewController` and delegates it in order to know when to transition to another Coordinator.

```swift
import UIKit
import RapiddoCore

final class MyScreenCoordinator: Coordinator {
    init(client: HTTPClient, persistence: Persistence) {
        let viewModel = MyScreenViewModel(client: client, persistence: persistence)
        let viewController = MyScreenViewController(viewModel: viewModel)
        super.init(rootViewController: viewController)
        viewController.delegate = self
    }
}

extension MyScreenCoordinator: MyScreenViewControllerDelegate {
    func continue() {
        let coordinator = NextCoordinator()
        push(coordinator, animated: true)
    }
}
```

### `MyScreenViewModel`

The ViewModel handles a ViewController's business logic. Ideally, this is where API calls happen and where the data source is retained.

```swift
import RapiddoCore

final class MyScreenViewModel {

    let client: HTTPClient
    let persistence: Persistence
    
    var myData = [MyData]()

    init(client: HTTPClient, persistence: Persistence) {
        self.client = client
        self.persistence = persistence
    }
    
    func getMyData() {
        //some request
        //myData = the result
    }
}

```

### `MyScreenView`

The View is where everything regarding the visual aspect of the ViewController should be built and retained. Note that we are not in an `UIViewController`!. Instead of adding views to the ViewController, we use a separate view file and override the `UIViewController`'s default `view` property by overriding `loadView()`, as you will see below in the `MyScreenViewController` explanation.

```swift
import UIKit
import Cartography
import RapiddoCore

protocol MyScreenViewDelegate: class {
    func somethingHappened()
}

final class MyScreenView: UIView {

    weak var delegate: MyScreenViewDelegate?

    private let aView: UIView = {
        //View setup
    }()

    private let anotherView: UIView = {
        //View setup
    }()

    override init(frame: CGRect) {
        super.init(frame: frame)
        setup()
    }

    required init?(coder aDecoder: NSCoder) {
        fatalError()
    }

    private func setup() {
        setupAView()
        setupAnotherView()
    }

    private func setupAView() {
        addSubview(aView)
        //Constraints
    }

    private func setupAnotherView() {
        addSubview(anotherView)
        //Constraints
    }
    
    func render() {
        //Update the view
    }
}
````

### `MyScreenViewController`

Finally, the ViewController wraps together the previous three classes. Usually, the ViewController doesn't do anything besides routing information between the parties (the exception being `UITableView` delegates).

Note that we use a protocol named `SmartViewController` in order to allow the ViewController to access the inner `MyScreenView` through a `smartView` property.

```swift
import UIKit
import RapiddoCore

protocol MyScreenViewControllerDelegate: class {
    func continue()
}

final class MyScreenViewController: CoordenableViewController {
    weak var delegate: MyScreenViewControllerDelegate?

    let viewModel: MyScreenViewModel

    init(viewModel: MyScreenViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
        title = Localization.myScreenTitle
    }

    required init?(coder aDecoder: NSCoder) {
        fatalError()
    }

    override func loadView() {
        let myScreenView = MyScreenView()
        view = myScreenView
        myScreenView.delegate = self
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        viewModel.getSomething()
        smartView.render()
    }
}

extension MyScreenViewController: MyScreenViewDelegate {
    func someButtonTouched() {
        delegate?.continue()
    }
}

extension MyScreenViewController: SmartViewController {
    typealias SmartView = MyScreenView
}
```

# Style Guide

In order to maintain our status as a highly-scalable superapp, all Rapiddo code must follow these guidelines. This is not a complete list by any means, so feel free to make your own suggestions!

## Naming

Using descriptive names makes code easier to read and understand. Use the Swift naming conventions described in the [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/). Also refer to the Clean Code's chapter on naming for more examples. Remember what was said at the Comment's section and be aware that property/parameter/method names should be enough documentation. Make sure that their purpose can be fully understood purely by reading its name.

# Views

The views on the project follow a very simple structure.

### Setup

The initial setup of a view should happen inside a `setup()` method called inside the `UIView`'s init.

```swift
override init(frame: CGRect) {}
    super.init(frame: frame)
    setup()
}
```

Every view should be built purely by code, with constraints handled by the using the `Cartography` framework. To setup the subviews of a view, additional  `setup()` methods should be implemented and called on the main `setup()` method of the view.

```swift
private func setupTableView() {
    addSubview(tableView)
    constrain(tableView, self) { view, superview in
        view.edges == superview.edges
    }
}
```

The constraints of each subview should be added inside the setup of that subview. It’s important to always remember to add the subview to it’s superview before adding any constraints and pay attention to the order on which the setup methods are called.

```swift
private func setup() {
    setupTableView()
    setupEmptyState()
    setupLoadingView()
    setupSegmentedControl()
}
```
Actions of buttons and other views are configured in the setup method of that view.

### Subview Creation

All subviews should be created and configured using closures. Any setup that doesn’t depend on state or dynamic information must be done inside the closure.

```swift
let confirmButton: RapiddoButton = {
    let button = RapiddoButton(style: .positive)
    button.setTitle(Localization.confirm, for: .normal)
    button.isEnabled = false
    return button
}()
```

### Rendering content on Views

To display updated information, a view should implement a `render(_:)` method that will be called by that view’s superview (observe that this could be, and often will be, a `ViewController`). Views that displays dynamic information (e.g. loading states related to server requests)  should have a nested `State` enum.

```swift
final class MyView: UIView {
    enum State {
        case loading
        case updated
        case error(Error)
    }
}
```

The render method should then handle these states. Here is an example:

```swift
func render(state: State) {
    switch state {
    case .loading:
        renderLoading()
    case .updated:
        renderUpdated()
    case let .error(error):
        renderError(error)
    }
}
```
# TableView & CollectionView

### Registering and Dequeueing

Rapiddo posesses abstracted versions of common cell methods for both `UITableViews` and `UICollectionViews`.

To register a cell type, all you have to do is call the `register(_:)` method:

```swift
tableView.register(MyTableViewCell.self)
// or
collectionView.register(MyCollectionView.self)
```
Registering of cells must also be done on the closure used to create the view.

In a similar fashion, dequeueing of cells is also just a matter of calling the respective methods:
```swift
tableView.dequeue(type: MyTableViewCell.self)
// or
collectionView.dequeue(type: MyCollectionView.self, for: indexPath)
```

### TableView/CollectionView Delegates

The delegate methods of both Collection and Table views must be implemented on the `ViewController` that has that view as a subview.

## Delegation Patterns

One of the most important and frequent patterns used in the the project is the `delegate` pattern. The communication between the different layers of the project (Views, ViewControllers, ViewModels and Coordinators) is made mostly by them. When naming delegate methods, keep in mind the following recommendations:
- If the method represent an action of the user on a component make this explicit on the name of the method, e.g.`userDidSelect(name: String)`.
- If the delegate belongs to a view that might be used with more than one instance of it at the same superview, then it’s ok to add a parameter identifying the view. E.g. `emptyStateView(_ view: EmptyStateView didSelectButton button: RapiddoButton)`. Otherwise, prefer not to.
- Use `touched` instead of `tapped` or `clicked` when referring to touch events.

## Components & Styles

Some specific components are used throughout the project. Let’s see them.

### RapiddoButton

All the buttons used on the project must be of type `RapiddoButton` in order to make sure that it follows the designated button styles of the app. Creating a new button is just a matter of choosing a style: 

```swift
let button = RapiddoButton(style: .positive)
```

At the moment, `RapiddoButton.Style` can be either `positve` or `neutral`. You should use `positive`  style when you want to draw the attention of the user to the action performed by the button and `neutral` when that is not the case. In extreme cases where the button does not match any of the current styles, you can pass a `nil` style and configure the button manually.

### RapiddoLabel / Fonts

In order to make sure that the correct fonts are used througout Rapiddo, every label in the project must be a `RapiddoLabel`. Just like `RapiddoButtons`, creating a new label is just a matter of picking a style:

```swift
let label = RapiddoLabel(style: .title2)
```

`RapiddoLabel.Style` is an enum covering all font sizes and weights used in Rapiddo. If you need to use a font outside the context of an `UILabel`, use the `defaultFont` property of the `RapiddoLabel.Style` like shown below:

```swift
button.titleLabel?.font = RapiddoLabel.Style.title2.defaultFont
```

In the extreme case where you are required to use a font that is not part of our pre-determined styles, you can use the `custom` family of styles.

```swift
RapiddoLabel.Style.customBold(17).defaultFont
```

However, try to first talk to the designer to see if it's not possible to adapt such font to one of our pre-determined ones.

### EmptyStateView

Rapiddo's `EmptyStateView`  has two main uses in the project. Display a friendly message on screens that exhibit data that doesn’t exist yet and display error messages related to failed requests. Alongside a message, the view may also have a button and/or an image.

The empty state view works with an `EmptyStateMode`. The mode defines all the visual information that will be displayed by the view. To create a new mode, extend `EmptyStateMode` and define a new static method. Here is an example:

```swift
static func noOrders() -> EmptyStateMode {
    return EmptyStateMode(image: nil, text: Localization.noOrdersEmptyState, hidesButton: true)
}
```

To handle the action of the button, conform to the `EmptyStateViewDelegate` protocol and implement the `emptyStateViewButtonTouched(for mode: EmptyStateMode)` method. If your view is capable of displaying several types of `EmptyStateModes`, you should use the `mode` property to tell them apart.

To display errors in general, you should use the global `EmptyStateMode.error(Error)` `EmptyStateMode`.

### Colors and Themes

To keep things organized and easy to maintain, we keep all colors, margins and some key dimensions on the `Style` struct.
`Style` has four nested structs:
- `Colors`: Contains all the colors used by the app.
- `Theme`: The theme struct works just like the colors one, but it tends to define concepts like `tintColor` and `darkBackground` instead. This is mostly used by the Providers that are also used outside of the Rapiddo app.
- `Margins`: The horizontal and vertical margins used by the app.
- `Dimensions`: Key dimensions, such as button heights.

All those components are part of the `RapiddoCore` framework. Always use them if possible. If a color or margin is not available inside these structs, consider talking to the designer to see if it was an oversight. In some extreme cases, we can resort to hardcoded values.

## Assets & Strings

You should always use `SwiftGen` when referencing images and strings.

Rapiddo and most Provider's subprojects already contain a Run Script phase to generate reference files, so most of the times all you have to do is merely compile the project in order to update the reference files.

### Images

After adding an image to it's Provider's `.xcassets` and compiling, you can access it on the `Asset` struct.

```swift
imageView.image = Asset.emptyStatePlaceholder.image
```

### Strings

On a fast growing project like Rapiddo, keeping track of all the text displayed on the app can be quite challenging. Especially for i18n, it’s easy to have some lost strings inside the project. For that reason all the strings in the project are kept in the `Localizable.strings` file. Even though we currently support only the Portuguese language doing this from the beginning make things much easier when we decide to support other languages.

Another benefit of this is that we can use `SwiftGen` to create a static references to those strings, making the code cleaner and easier to maintain. Every time a new text should be added to the project, create a new entry on the `Localizable` file, e.g. `"PLEASE_TRY_AGAIN" = "Por favor tente novamente.";` and, just like with images, compile the project in order to generate a static refenrece. The property will be inside the `Localization` type of that project. Then all you have to do to use it is:

```swift
let message = Localization.pleaseTryAgain
```

One thing to keep in mind is that not all of the text that is displayed on the app is kept inside the app. A lot of them are sent to the app by the server.

Another thing is that the `SwiftGen` scripts are defined by each Provider! If you're trying to add a string to Marmotex, you have to run Marmotex's example project.

### Error Handling

The way errors are presented to users is a big deal to any mobile project. In Rapiddo, this is no different. There are three ways to display errors in Rapiddo:

- Empty States
- Toast
- SmartMessages

While the error state of an `EmptyStateView` should be rendered directly on its view, all other types of errors should be presented by calling the current Coordinator's `presentError()` method.

### EmptyStates

As mentioned in the `EmptyState` section, you can use an `EmptyStateView` in order to block access to a view, either to warn that there's nothing there or to display an error.

### Toasts

If the error you recieved is not an `APIError` with an underlying `SmartMessage`, calling `presentError()` at your Coordinator will briefly display a small message view at the top of the screen. This is good when you need to display an error that doesn't need to block the user's screen, but note that if you're already rendering an `EmptyStateView` alongside this error, then displaying a toast in unnecessary. In these cases, you should call `presentError(onlyDisplaySmartMessages: true)` instead.

### SmartMessages

Application errors that require an action by the user or a explicit acknowledgment are displayed on special popups as `SmartMessages`. These errors get mapped as `APIErrors` and are returned by the server as a response of a request. Normally smart messages carry at least one action represented by a deeplink that should be handled by conforming your Coordinator to the `HandlesPopupAction` protocol. See the `DeepLink` documentation for more details.

## Protocol Conformance

In particular, when adding protocol conformance to a model or view, prefer adding a separate extension for the protocol methods. This keeps the related methods grouped together with the protocol and can simplify instructions to add a protocol to a class.

```swift
extention MyModel: SomeProtocol {
    func someProtocolRequiredMethod() -> Int {
        return 10
    }
}
```

## Unused Code

Unused (dead) code should be removed. Don't worry about losing stuff, that's what Git is for :smile:

## Comments and Documentation

Use comments only to explain intent. Don't use comments to explain things that are already obvious, such as a protocol conformance. Code should be as self-documenting as possible, so if you feel the need to use comments to explain what the method itself is doing, consider refactoring it into something more clearer.

Bad comment:
```swift
//The tableView delegate
extension MyViewController: UITableViewDelegate {}
```

Good comment:
```swift
func loadData() {
    //We need to add a test header
    //because of a backend limitation.
    //They will fix this in the next release.
    client.add(header: "test", value: "true")
}
```

However, we do have an expection when it comes to documentation. In general, if the class you're building is supposed to be abstracted upon (which is the case of most `RapiddoCore` classes), then you should ignore these rules and document your code just like if you were building a framework (which is the case of `RapiddoCore`! :smile: ) by using Swift's documentation formats.

If the class is not supposed to be abstracted upon, we think that using clear names is enough. There are exceptions, so talk to your team and see what they think about it.

## Closure Expressions

Use trailing closure syntax only if there's a single closure expression parameter at the end of the argument list. Give the closure parameters descriptive names.

When defining a closure that captures `self`, the unwrapped property should be named `strongSelf`.

```swift
foo.bar { [weak self] in
    guard let strongSelf = self else {
        return
    }
}
```

## Final

All classes or members of a class that are not meant to be overriden should be marked as `final`.
