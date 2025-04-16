# Concurrency-Receipes

Swift Concurrency introduced several powerful features that make concurrent programming easier, safer, and more efficient. These include async/await, actors, structured concurrency, task groups, and more. Here's a detailed explanation of these concepts, with examples and diagrams where applicable.

1. Async/Await
Explanation
async and await are keywords used to handle asynchronous operations in a more readable and maintainable way compared to traditional completion handlers or callbacks.

* async is used to mark functions that will run asynchronously.
* await is used within an async context to pause execution until an asynchronous operation completes.
Example
Imagine fetching a user's profile and then fetching their posts sequentially:
```
import Foundation

struct Profile {
    let id: Int
    let name: String
}

struct Post {
    let id: Int
    let content: String
}

// Mock async functions
func fetchProfile() async -> Profile {
    // Simulate network delay
    await Task.sleep(2 * 1_000_000_000) // 2 seconds
    return Profile(id: 1, name: "Alice")
}

func fetchPosts(profile: Profile) async -> [Post] {
    // Simulate network delay
    await Task.sleep(1 * 1_000_000_000) // 1 second
    return [Post(id: 1, content: "Hello, world!"), Post(id: 2, content: "Swift is awesome!")]
}

// Main async function
func loadData() async {
    let profile = await fetchProfile()
    print("Fetched profile: \(profile.name)")
    
    let posts = await fetchPosts(profile: profile)
    for post in posts {
        print("Post: \(post.content)")
    }
}

// Run
Task {
    await loadData()
}

Diagram
    Task {
        └── loadData() async
             ├── fetchProfile() async --- 2 seconds delay --->
             └── fetchPosts(profile) async --- 1 second delay ---
    }

```
2. Structured Concurrency
Explanation
Structured concurrency ensures that asynchronous tasks are started and managed in a well-organized manner, making it easier to handle errors and manage task lifecycles.

Example
```
import Foundation

func fetchAllData() async {
    async let profile = fetchProfile()
    async let posts = fetchPosts(profile: profile)
    
    do {
        let (profileResult, postsResult) = try await (profile, posts)
        print("Profile: \(profileResult.name)")
        for post in postsResult {
            print("Post: \(post.content)")
        }
    } catch {
        print("Error occurred: \(error)")
    }
}

Task {
    await fetchAllData()
}
Diagram
    fetchAllData() async
        ├── fetchProfile() async - run concurrently
        └── fetchPosts(profile) async - run concurrently

    Tasks await completion of both fetchProfile and fetchPosts
    and handle the results together.
```
3. Actors
Explanation
Actors are a new reference type that protect mutable state by isolating concurrent access. This means that only one task can access an actor's mutable state at a time.

Example
```
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }

    func getValue() -> Int {
        return value
    }
}

// Usage
let counter = Counter()
Task {
    await counter.increment()
    print("Current value: \(await counter.getValue())")
}
Diagram
    Actor: Counter
    ├── value
    ├── increment() async
    └── getValue() async

    Task -----> Counter.increment()
    Task -----> Counter.getValue()
```
4. Task Groups
Explanation
Task groups allow for the creation and management of a group of related tasks. This is useful when you need to perform several related asynchronous operations concurrently and then process their results collectively.

Example
```
import Foundation

func fetchUserData() async throws -> [String] {
    var results = [String]()
    
    try await withThrowingTaskGroup(of: String.self) { group in
        group.addTask {
            try await fetchProfile().name
        }

        group.addTask {
            try await fetchPosts(profile: fetchProfile()).map { $0.content }
        }

        for try await result in group {
            results.append(result)
        }
    }
    
    return results
}

Task {
    do {
        let data = try await fetchUserData()
        print("User Data: \(data)")
    } catch {
        print("Error occurred: \(error)")
    }
}
Diagram
    fetchUserData() async
        ├── Task Group
            ├── Task: fetchProfile().name
            └── Task: fetchPosts(profile: fetchProfile()).content

    Wait for all tasks in the group to complete and then collect the results.
```
