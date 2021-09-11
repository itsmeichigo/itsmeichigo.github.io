---
title: "Working with observable objects in SwiftUI"
date: 2021-09-11T10:00:02+07:01
draft: false
toc: false
tags:
  - swift
  - swiftui
  - observedobject
  - stateobject
---

Earlier this week I learned about `@StateObject` [initializer](https://developer.apple.com/documentation/swiftui/stateobject/init(wrappedvalue:)) when reviewing a pull request. The idea was to create a view model for a SwiftUI view using some injected parameter and keep the view model as a `@StateObject` so that the its state is persisted when the view is redrawn. The view creates the `@StateObject` like so:

```swift
class ItemViewModel: ObservableObject {
    @Published var itemName: String
    init(item: Item) {
        self.itemName = item.name
    }
}

struct ItemView: View {
    @StateObject private var viewModel: ItemViewModel
    init(item: Item) {
        _viewModel = StateObject(wrappedValue: ItemViewModel(item: item))
    }

    var body: some View {
        TextField("Item Name", text: viewModel.$itemName)
    }
}
```

The problem with this initializer is that it's not meant to be used directly. As Apple documentation says it:

> You don’t call this initializer directly. Instead, declare a property with the `@StateObject` attribute in a [`View`](https://developer.apple.com/documentation/swiftui/view), [`App`](https://developer.apple.com/documentation/swiftui/app), or [`Scene`](https://developer.apple.com/documentation/swiftui/scene), and provide an initial value

More interestingly, [an answer](https://swiftui-lab.com/random-lessons/#data-10) on the WWDC21's SwiftUI Lab confirms that this is an acceptable use. I'm not quite happy with this answer though, because it's not cool to go against the recommended use. So we can fix this by injecting the view model from outside, right?

```swift
struct ItemView: View {
    @StateObject var viewModel: ItemViewModel

    var body: some View {
        Text("Test: \(viewModel.id)")
    }
}

struct ItemListView: View {
    @ObservedObject var viewModel: ItemListViewModel

    var body: some View {
        ForEach(viewModel.items) { item in
            ItemView(viewModel: ItemViewModel(item: item))
        }
    }
}
```

But this doesn't seem right!

I've read several articles about the difference between `@StateObject` and `@ObservedObject`, and the general idea is simple: You should use `@StateObject` if the view you're using creates the instance of the `ObservableObject` itself. If it does not, use `@ObservedObject` instead.

Consider `@StateObject` something similar to `@State` but to use with `ObservableObject`. They both are created and owned by the SwiftUI view. Their values are controlled internally and persisted by SwiftUI internally throughout re-renders of the view. `@ObservedObject`, however, is not persisted by SwiftUI.

Back to our initial code problem. Using the injected `@StateObject` is incorrect, so we should go ahead and replace it with `@ObservedObject`. Normally we're done here, since we usually want fresh instances of items every time a list redraws itself.

But what if we want to keep the text field value when rotating the device? When device orientation changes, the item list is redrawn, which creates new views for the child items with new instances of their view models. This will reset the content of a `ItemView`'s text field to its initial value.

The solution is to hold on the the item view models to persist the state of the item views:

```swift
class ItemListViewModel: ObservableObject {
    private(set) var itemViewModels: [ItemViewModel]
    @Published var items: [Item]
    
    init(items: Item) {
        self.items = items
        self.itemViewModel = items.map { ItemViewModel(item: $0) }
    }
}

struct ItemListView: View {
    @ObservedObject var viewModel: ItemListViewModel

    var body: some View {
        ForEach(viewModel.items) { item in
            viewModel.itemViewModels.first(where: { $0.item == item }).map { viewModel in
                ItemView(viewModel: viewModel)
            }
        }
    }
}

struct ItemView: View {
    @ObservedObject var viewModel: ItemViewModel

    var body: some View {
        Text("Test: \(viewModel.id)")
    }
}
```

Since our `ItemListViewModel` now keeps references of all the items' view models, the text field of each item view is kept intact when the view is redrawn! 🎉

So whenever you find yourself needing an `ObservableObject` in a SwiftUI view, ask yourself two questions:

- Do I need **external (injected) data** to create the object? If yes, go for `@ObservedObject`, otherwise `@StateObject`.
- I need to use an `@ObservedObject`, but do I need to **persist its states**? If yes, keep a reference of the object somewhere, otherwise, feel free to create a new instance for every view.

I hope these tips are helpful for other folks working with SwiftUI in their projects!


More readings if you're interested:

- [SwiftUI Property Wrappers](https://swiftuipropertywrappers.com/)
- [What's the difference between @StateObject and @ObservedObject?](https://www.donnywals.com/whats-the-difference-between-stateobject-and-observedobject/)
- [How to initialize @StateObject with parameters in SwiftUI](https://sarunw.com/posts/how-to-initialize-stateobject-with-parameters-in-swiftui/)