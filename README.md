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
