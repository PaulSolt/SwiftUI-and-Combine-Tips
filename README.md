# SwiftUI and Combine Rules

1. `@Publisher` needs to be contained in a class. Publishers are reference types.

	```swift
	class SearchStore: ObservableObject {
	    @Published var query: String = “Swift”
	}
	
	```
	
	Error
	
	```swift
	// ERROR: 'wrappedValue' is unavailable: @Published is only available on properties of classes
	struct SearchStore {
	    @Published var query: String = "Swift"
	}
	```


2. Wrap a `UISearchBar` in `UIViewRepresentable`, it's not available with Xcode 11 Beta 7

3. When conforming to the protocol `UIViewRepresentable`, you should change the generic return types and parameter labels to provide context.

	The system uses type inference to set the UIViewType if you make one of the required methods using your type. So you don't need to explicitly define the `typealias`.


	```swift
	typealias UIViewType = UISearchBar
	```

	 1. Give it meaningful names `searchBar` instead of `uiView`
	 2. `UISearchBar` instead of `SearchBar.UIViewType`


4. Only update the content, not the sizing when conforming to `UIViewRepresentable`

	```swift
	func updateUIView(_ searchBar: UISearchBar, context: UIViewRepresentableContext<SearchBar>) {
	    searchBar.text = text
	}
	```

5. There's no first responder, you can try doing something like:

	```swift
	func updateUIView(_ searchBar: UISearchBar, context: UIViewRepresentableContext<SearchBar>) {
	    print("Update searchBar")
	    searchBar.text = text
	
	    if searchBar.window != nil, !searchBar.isFirstResponder {  // checking window prevents crash in a sheet before in view hierarchy
	        searchBar.becomeFirstResponder()
	    }
	}
	```

6. Use a `Coordinator` object to subscribe to events from UIKit controls, give it a name that's meaningful like `SearchBarCoordinator`.

	```swift
	class SearchBarCoordinator: NSObject, UISearchBarDelegate {
	    @Binding var text: String
	    var completion: SearchCompletion?
	    
	      init(text: Binding<String>, onChange completion: SearchCompletion? = nil) {
	        _text = text
	        self.completion = completion
	    }
	    
	    func searchBar(_ searchBar: UISearchBar, textDidChange searchText: String) {
	        text = searchText
	        completion?()
	    }
	}
	```

	In your `UIViewRepresentable` class
	
	```swift
	func makeCoordinator() -> SearchBarCoordinator {
	    return SearchBarCoordinator(text: $text, onChange: completion)
	}
	```


# TODO: More research Needed

1. Add sinks to variables, but hold onto the publishers (otherwise you may not get notifications?)

```swift
_ = searchBar.publisher(for: \.text)
.sink { (value) in
    print("Getting values from my publisher: \(value ?? "")")
    
}
```