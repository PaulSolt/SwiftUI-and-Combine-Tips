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

8. For Optional values, you can test against `nil`  to present placeholder values with placeholder colors.

	`description` is an optional field from Github's repository API.
	
	```swift
	struct Repository: Decodable, Identifiable {
	    let id: Int
	    let name: String
	    let description: String?
	}
	```

	RepositoryRow.swift

	```swift
	import SwiftUI
	
	struct RepositoryRow: View {
	    let repository: Repository
	    
	    var body: some View {
	        VStack(alignment: .leading) {
	            Text(repository.name)
	                .font(.headline)
	            Text(repository.description ?? "No Description")
	                .font(.subheadline)
	                .foregroundColor((repository.description != nil) ? .black : Color(UIColor.systemGray))
	        }
	    }
	}
	
	let repositoryData = Repository(id: 1, name: "Things", description: "A Todo app")
	
	struct ContentView_Previews: PreviewProvider {
	    static var previews: some View {
	        RepositoryRow(repository: repositoryData)
	    }
	}
	```
	
	NOTE: Don't use `@State`  for values that will not change. If you pass in a model value type to a SwiftUI view, keep the variable as a normal Swift constant.

	Your `struct` will get an automatic initializer, which you can populate with test data for your previews.

9. Hide the keyboard after a commit (Enter key) on a `TextField`

```swift
TextField("Type something...", text: $query, onCommit: {
    self.fetch()
    // hide keyboard
    UIApplication.shared.sendAction(#selector(UIResponder.resignFirstResponder), to:nil, from:nil, for:nil)
})
```

10. Customize a `TextField` to look distinct

	```swift
	TextField("Type something...", text: $query, onCommit: {
	    self.fetch()
	})
	.padding()
	.background(
	    RoundedRectangle(cornerRadius: 5, style: .continuous)
	        .foregroundColor(Color(white:0.95))
	)
	.padding()
	```

11. Use the Continuous Corner Radius of RoundedRectangle

	```swift
	RoundedRectangle(cornerRadius: 5, style: .continuous)
		.foregroundColor(Color(white:0.95))
	```



# TODO: More research Needed

1. Add sinks to variables, but hold onto the publishers (otherwise you may not get notifications?)

```swift
_ = searchBar.publisher(for: \.text)
.sink { (value) in
    print("Getting values from my publisher: \(value ?? "")")
    
}
```