# The MovilePay iOS Style Guide

If you are a MovilePay employee, you can find documentation that explains how the project iself works in the main repository's README.

## Creating new screens in Rapiddo

We use MVVM-C (Model View View Model with Coordinators) as our architecture with our own Coordinator implementation (which you can find documentation for in the main project's README)

There are no Storyboards in MPay projects - everything is done via code. While people have mixed opinions about this, we believe that view coding is the best option as it allows multiple people to code in the same screen at the same time, merge conflicts are simple and you gain more control over your screens, not to mention that you don't have to wait 10 minutes for Xcode to load the visual representation of your screen - everything is just regular code.

In addition, this allows use to use dependency injection in a straight forward manner. You'll see that there are no singletons in MovilePay - everything relevant to a class must be directly sent to it. This allows us to write better tests and make sure that classes can only do what they are supposed to. The only exception is some properties in the `Style` struct due to the limitations we have regarding dependencies.

This section details how to create a new screen called `MyScreen`. In normal conditions, it would consist of:

### `MyScreenCoordinator`

In MVVM-C, The Coordinator is the object responsible for handling screen transitions. It retains its inner `UIViewController` and delegates it in order to know when to transition to another Coordinator.

```swift
import UIKit

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

If your screen needs to retain or pass a `MovilePayDelegate` delegate forward, you should use `MovilePayCoordintor` instead as it holds an unowned reference to the delegate.

```
final class QRScannerCoordinator: MovilePayCoordinator {
    let client: HTTPClient

    init(client: HTTPClient, delegate: MovilePayDelegate) {
        self.client = client
        let viewModel = QRScannerViewModel(client: client)
        let viewController = QRScannerViewController(viewModel: viewModel)
        super.init(rootViewController: viewController, delegate: delegate)
        viewController.delegate = self
    }
}
```

### `MyScreenView`

The View is where everything regarding the visual aspect of the ViewController should be built and retained. Note that we are not in an `UIViewController`!. Instead of adding views to the ViewController, we use a separate view file and override the `UIViewController`'s default `view` property by overriding `loadView()`, as you will see below in the `MyScreenViewController` explanation.

If your view is able to represent several states (like loading, loaded and error), you should abstract this logic as a `State` enum.

Despite holding the references to all internal views, state-related delegates like `UITableViewDelegate` should be placed in the ViewController.

```swift
import UIKit

protocol MyScreenViewDelegate: AnyObject {
    func somethingHappened()
}

final class MyScreenView: UIView {

    enum State {
        case none
        case loading
        case updated
        case error(Error)
    }
    
    var state: State? = nil
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
    
    func render(state: State) {
        self.state = state
        switch state {
            //Update the view based on the state
        }
    }
}
````

### `MyScreenViewModel`

The ViewModel handles a ViewController's business logic. Ideally, this is where API calls happen and where the data source is retained. Changes are routed back to the ViewController in the shape of a `didSet(state:)` delegate.

```swift
import Foundation

protocol MyScreenViewModelDelegate: AnyObject {
    func didSet(state: MyScreenView.State)
}

final class MyScreenViewModel {

    let client: HTTPClient
    let persistence: Persistence
    
    var myData = [MyData]()
    
    weak var delegate: MyScreenViewModelDelegate?

    init(client: HTTPClient, persistence: Persistence) {
        self.client = client
        self.persistence = persistence
    }
    
    func getMyData() {
        self?.delegate?.didSet(state: .loading)
        //some request
        //myData = the result
        self?.delegate?.didSet(state: .updated)
    }
}

```

### `MyScreenViewController`

Finally, the ViewController wraps together the previous three classes. Usually, the ViewController doesn't do anything besides routing information between the parties (the exception being state-relevant delegates like `UITableView` ones).

Note that we use a protocol named `HasCustomView` in order to allow the ViewController to access the inner `MyScreenView` through a `customView` property. You can read more about `loadView()` [here.](https://swiftrocks.com/writing-cleaner-view-code-by-overriding-loadview.html)

```swift
import UIKit

protocol MyScreenViewControllerDelegate: AnyObject {
    func continue()
}

final class MyScreenViewController: CoordenableViewController {

    weak var delegate: MyScreenViewControllerDelegate?
    let viewModel: MyScreenViewModel

    init(viewModel: MyScreenViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
        viewModel.delegate = self
        title = L10n.myScreenTitle
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
    }
}

extension MyScreenViewController: MyScreenViewModelDelegate {
    func didSet(state: MyScreenView.State) {
        customView.render(state: state)
    }
}

extension MyScreenViewController: MyScreenViewDelegate {
    func someButtonTouched() {
        delegate?.continue()
    }
}

extension MyScreenViewController: HasCustomView {
    typealias CustomView = MyScreenView
}
```

# Style Guide

In order to maintain our status as a highly-scalable superapp, all MovilePay code must follow these guidelines. This is not a complete list by any means, so feel free to add your own suggestions!

## Regarding Frameworks

Please do not import frameworks "just because". Try to do things natively, only importing frameworks if it means a massive improvement for every provider (such as `Cartography` and `PromiseKit`). Remember that this is not a regular app - Because we're an SDK, we need to be lightweight. If you need a very specific interaction, you can make the host apps provide this info through `MovilePayDelegate`.

## Naming

Using descriptive names makes code easier to read and understand. Use the Swift naming conventions described in the [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/). Also refer to the Clean Code's chapter on naming for more examples. Remember what was said at the Comment's section and be aware that property/parameter/method names should be enough documentation. Make sure that their purpose can be fully understood purely by reading its name.

# Views

The views on the project should follow the following structure:

### Setup

The initial setup of a view should happen inside a `setup()` method called inside the `UIView`'s init.

```swift
override init(frame: CGRect) {}
    super.init(frame: frame)
    setup()
}
```

Every view should be built purely by code, with constraints handled by the using the `Cartography` framework. To setup the subviews of a view, additional `setup()` methods should be implemented and called on the main `setup()` method of the view.

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
    let button = Button(style: .primary)
    button.setTitle(L10n.confirm, for: .normal)
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

We posesses abstracted versions of common cell methods for both `UITableViews` and `UICollectionViews`.

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

### Button

All the buttons used on the project must be of type `Button` to make sure that it follows the designated button styles of the app. Creating a new button is just a matter of choosing a style: 

```swift
let button = Button(style: .primary)
```

### Label / Fonts

In order to make sure that the correct fonts are used througout the SDK, every label in the project must be a `Label`. Just like `Buttons`, creating a new label is just a matter of picking a style:

```swift
let label = Label(style: .title2)
```

`Label.Style` is an enum covering all font sizes and weights used in the SDK. If you need to use a font outside the context of an `UILabel`, use the `defaultFont` property of the `RapiddoLabel.Style` like shown below:

```swift
button.titleLabel?.font = Label.Style.title2.font
```

### EmptyStateView

MPay's `EmptyStateView` has two main uses in the project: displaying a friendly message on screens that exhibit data that doesn’t exist yet and displaying error messages related to failed requests. Alongside a message, the view may also have a button and/or an image.

The empty state view works with an `EmptyStateMode`. The mode defines all the visual information that will be displayed by the view. To create a new mode, extend `EmptyStateMode` and define a new static method. Here is an example:

```swift
static func noOrders() -> EmptyStateMode {
    return EmptyStateMode(image: nil, text: Localization.noOrdersEmptyState, hidesButton: true)
}
```

Handling the action of the button can be done both with closures or delegates. For the latter, make your ViewController conform to the `EmptyStateViewDelegate` protocol and implement the `emptyStateViewButtonTouched(for mode: EmptyStateMode)` method. If your view is capable of displaying several types of `EmptyStateModes`, you should use the `mode` property to tell them apart.

To display errors in general, you should use the global `EmptyStateMode.error(Error)` `EmptyStateMode`.

### Colors and Themes

To keep things organized and easy to maintain, the `Style` struct contains the standard for all visual matters of the app.
- Structs:
 - `Colors`: Contains all the colors used by the app. Syntax sugar: `tintColor = Color.main.darkBackground`
 - `Margins`: The horizontal and vertical margins used by the app. Syntax sugar: `tintColor = Margin.main.small`
- Components:
 - `successView()`: A Lottie animation view provided by the host app for a "success" operation.
 - `loadingView()`: A Lottie animation view provided by the host app for a "loading" operation.
 - `textField()`: A TextField-like view provided by the host app.
 - `segmentedControl()`: A UISegmentedControl-like view provided by the host app.

`Style` also contains the image loading/caching provided by the host app. You can use the UIImageView wrappers in this case:
`imageView.fetchImage(url:)`
`imageView.cancelImageDownload()`

If a color or margin is not available inside these structs, consider talking to the designer to see if it was an oversight. In some extreme cases, we can resort to hardcoded values.

## Assets & Strings

You should always use `SwiftGen` when referencing images and strings.

The SDK's main project already contain a Run Script phase to generate reference files, so most of the times all you have to do is merely compile the project in order to update the reference files.

### Images

After adding an image to the SDK's `.xcassets` and compiling, you can access it on the `Asset` struct.

```swift
imageView.image = Asset.emptyStatePlaceholder.image
```

### Strings

On a fast growing project like MPay, keeping track of all the text displayed on the app can be quite challenging. Especially for i18n, it’s easy to have some lost strings inside the project. For that reason all the strings in the project are kept in the `Localizable.strings` file. Even though we currently support only the Portuguese language doing this from the beginning make things much easier when we decide to support other languages.

Another benefit of this is that we can use `SwiftGen` to create a static references to those strings, making the code cleaner and easier to maintain. Every time a new text should be added to the project, create a new entry on the `Localizable` file, e.g. `"PLEASE_TRY_AGAIN" = "Por favor tente novamente.";` and, just like with images, compile the project in order to generate a static refenrece. The property will be inside the `L10n` type of that project. Then all you have to do to use it is:

```swift
let message = L10n.pleaseTryAgain
```

One thing to keep in mind is that not all of the text that is displayed on the app is kept inside the app. A lot of them are sent to the app by the server.

### Error Handling

The way errors are presented to users is a big deal to any mobile project. In Mpay, this is no different. There are two ways to display errors in Rapiddo:

- Empty States
- Toast provided by the host app

While the error state of an `EmptyStateView` should be rendered directly on its view, all other types of errors should be presented by calling `MovilePayDelegate`'s `showNonIntrusiveError` in the current Coordinato.

### EmptyStates

As mentioned in the `EmptyState` section, you can use an `EmptyStateView` in order to block access to a view, either to warn that there's nothing there or to display an error.

## Protocol Conformance

In particular, when adding protocol conformance to a model or view, prefer adding a separate extension for the protocol methods. This keeps the related methods grouped together with the protocol and can simplify instructions to add a protocol to a class.

```swift
extention MyModel: SomeProtocol {
    func someProtocolRequiredMethod() -> Int {
        return 10
    }
}
```

For class protocols, use `: AnyObject` instead of `: class`. The latter is popular, but it's not supposed to be used.

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

However, we do have an exception when it comes to documentation. In general, if the class you're building is supposed to be abstracted upon (which is the case of most `RapiddoCore` classes), then you should ignore these rules and document your code just like if you were building a framework (which is the case of `RapiddoCore`! :smile: ) by using Swift's documentation formats.

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
