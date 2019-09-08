# SwiftUI and Combine Tip

## Demo Repositories

* [Combine Timer (Count up)](https://github.com/PaulSolt/CombineTimer)


## Tips


1. `@Published` needs to be contained in a class. Publishers are reference types.

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

11. Use the `.continuous` style of `RoundedRectangle` for super smooth corners (like Apple's corners on your MacBook or Apple TV).

	```swift
	RoundedRectangle(cornerRadius: 5, style: .continuous)
		.foregroundColor(Color(white:0.95))
	```

12. Use `@EnvironmentObject` for dependency injection of your model into different view controllers. Use this when you don't want to keep passing an @ObservableObject from parent `View` to child `View` when using initializer dependency injection.

13. Play with Combine or imperative code in the onAppear callback.

	NOTE: There is no `viewDidLoad()`, but you can get similar behavior in a SwiftUI project by using `onAppear` publisher.

	```swift
	struct ContentView: View {
	    @State private var text: String = "Swift"
	    
	    var body: some View {
	        Text(text)
	            .onAppear(perform: viewDidAppear)
	    }
	    
	    func viewDidAppear() {
	        print("Hello World!")
	        // Experiment here
	        
	        _ = ["Hello", "World"]
	            .publisher
	            .sink(receiveCompletion: { (status) in
	                print("status: \(status)")  // called when finished "iterating" array
	            }) { (word) in
	                print(word) // called for each element in the array
	        }
	    }
	}
	
	struct ContentView_Previews: PreviewProvider {
	    static var previews: some View {
	        ContentView()
	    }
	}
	```

14. You can't mutate state in `body` or helper functions unless the variables are marked as such `@State`

Error 1

```swift
struct ContentView: View {
    @State private var text: String = "Swift"
    
    var timer: AnyCancellable? = nil
    
    var body: some View {
        VStack {
            Text(text)
            Button(action: {
	       // ERROR: Cannot assign to property [i.e. timer]: 'self' is immutable
                timer = Timer.publish(every: 0.2, on: .main, in: .default)
                    .sink {
                        print($0)
                }
```


15. Make sure you call `autoconnect()` or `connect()` on your Timer publisher. 

	```swift
	timer = Timer.publish(every: 0.2, on: .main, in: .default)
	    .autoconnect()
	    .sink { date in
	        print(date)
	    }
	```

16. 


## Nice and Safe

1. No more bugs from forgetting to call `super.methodName()` when using struct value types.

	```swift
	override func viewDidLoad() {
	    super.viewDidLoad()
	    
	    print("Don't forget to call super!")
	}
	```
	
	
	```swift
	Text("Hello World")
	    .onAppear {
	    print("Only my code here")
	}
	```


2. If you create a new timer, the old timer will be cleaned up automatically (no more duplicate timers and juggling timer creation logic!)


# BUGS: 

A List of bugs I've found, but haven't reported yet. Feel free to send Apple Feedback reports. I will attach Feedback #'s when I (or you) file them.

1. BUG: Playgrounds with Combine deadlock Xcode 11 Beta 7
	TODO: REPORT THIS BUG
	
	 If I run, then I try to double click text to select, Xcode is deadlocked

	Workaround: use a Mac Terminal app or iPhone app to prototype code

2. BUG: `Text("Hello ... long text").lineLimit()` doesn't work in Xcode 11 Beta 7

	TODO: REPORT THIS BUG
	
	```swift
	Text(self.review.body)
	    .font(.body)
	    .lineLimit(Int.max) // 10)
	```

3. BUG: Error message is not helpful about why the Combine publisher cannot assign to the property. It should talk about the "Character" != String type mismatch?

	```swift
	struct ContentView: View {
	    @State private var text: String = "Swift"
	    
	    var body: some View {
	        Text(text)
	            .onAppear(perform: viewDidAppear)
	    }
	    
	    func viewDidAppear() {
	        print("Hello World!")
	        // Experiment here
	        
	        _ = ["Hello", "World"]
	            .publisher
	            .sink(receiveCompletion: { (status) in
	                print("status: \(status)")  // called when finished "iterating" array
	            }) { (word) in
	                print(word) // called for each element in the array
	        }
	        
	        // Does not match type! (Character != String)
	        "Hello world!"
	            .publisher
	//            .map({ String($0) })	// FIX: Map to String
	            .assign(to: \.text, on: self) // ERROR: Type of expression is ambiguous without more context
	    }
	}
	
	struct ContentView_Previews: PreviewProvider {
	    static var previews: some View {
	        ContentView()
	    }
	}
	```

4. 


# TODO: More research Needed

1. Add sinks to variables, but hold onto the publishers (otherwise you may not get notifications?)

```swift
_ = searchBar.publisher(for: \.text)
.sink { (value) in
    print("Getting values from my publisher: \(value ?? "")")
    
}
```

2. You can't add Publishers from `UIViewRepresentable` views (or at least I can't figure out how). The interface doesn't allow you to store state outside the `context`. 

	QUESTION: How do you make this a publisher?


## Resources

* [Learn & Master the Basics of Combine in 5 Minutes](https://medium.com/ios-os-x-development/learn-master-%EF%B8%8F-the-basics-of-combine-in-5-minutes-639421268219)
* [URLSession and the Combine Framework](https://theswiftdev.com/2019/08/15/urlsession-and-the-combine-framework/)
* [Fetching values from API and Integrating Mapkit in SwiftUI](https://medium.com/a-developer-in-making/fetching-values-from-api-and-integrating-mapkit-in-swiftui-3277806d9090)
