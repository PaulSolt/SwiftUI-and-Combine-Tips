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

	Here is a simple implementation that wraps up a `UISearchBar` and provides a completion handler for text events from the `UISearchBarDelegate` protocol.

	```swift
	typealias SearchCompletion = () -> Void
	
	struct SearchBar: UIViewRepresentable {
	
	    @Binding var text: String
	    var completion: SearchCompletion? = nil
	    
	    init(text: Binding<String>, onChange completion: SearchCompletion? = nil) {
	        _text = text
	        self.completion = completion
	    }
	    
	    // The coordinator allows us to listen for updates to our searchbar
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
	        
	        func searchBarSearchButtonClicked(_ searchBar: UISearchBar) {
	            completion?()
	        }
	    }
	    
	    func makeCoordinator() -> SearchBarCoordinator {
	        return SearchBarCoordinator(text: $text, onChange: completion)
	    }
	    
	    func makeUIView(context: UIViewRepresentableContext<SearchBar>) -> UISearchBar {
	        let searchBar = UISearchBar()
	        searchBar.delegate = context.coordinator
	        return searchBar
	    }
	    
	    func updateUIView(_ searchBar: UISearchBar, context: UIViewRepresentableContext<SearchBar>) {
	        searchBar.text = text
	    }
	}
	```

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


	 A coordinator is used to listen to updates from the delegate protocol. You can bind the results from the callback delegate methods to a `@Binding`, which then gets bound to the `UIViewRepresentable`'s binding.
	

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
	@Binding var text: String

	func makeCoordinator() -> SearchBarCoordinator {
	    return SearchBarCoordinator(text: $text), onChange: completion)
	}
	```


7. `@State` is for internal variables that you want to use to trigger UI updates when modified. Keep them private. You don't use it to expose values externally, use `@Binding` or `@ObservedObject` for those situations.

8. 


# TODO: More research Needed

1. Add sinks to variables, but hold onto the publishers (otherwise you may not get notifications?)

```swift
_ = searchBar.publisher(for: \.text)
.sink { (value) in
    print("Getting values from my publisher: \(value ?? "")")
    
}
```