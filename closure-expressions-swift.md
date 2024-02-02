# Closure Expressions in Swift
original documentation: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/
- Group code that executes together, without creating a named function.
- Similar to callback functions in Javascript: used to pass a block of executable code as an argument to a function.
- first-class citizen: can be passed around and used as values.
- Base syntax:
```
{ (<parameters>) -> <return type> in
	<statements>
}
```
- unlike a named function, the *parameters* and *return type* are written *inside* the curly braces, not outside.
- `in` signifies the start of the closure's *body*. It indicates the parameters and return type have finished being declared. 
### Implicit Types and Returns
- Parameter and return types can be inferred. For example, this code for the method `sorted`:
```
reversedNames = names.sorted(by: { (s1: String, s2: String) -> Bool in return s1 > s2 } )
```
- can be rewritten as:
  `reversedNames = names.sorted(by: { s1, s2 in return s1 > s2 } )`
		  *note: .sorted(by: ) is a method which allows you to specify the condition for sorting an array by returning a Bool value to say whether any given value should appear before or after any other given value in the final sorted array.*
  - **Single-expression closures** can also implicitly return the result by omitting the return keyword:
  `reversedNames = names.sorted(by: { s1, s2 in s1 > s2 } )`
### Shorthand Argument Names
  - Argument names can be referred using a shorthand notation, with `$0, $1 `referring to the 1st, 2nd arguments, etc.
	  - The argument list can be omitted if you use these, because the type of the argument is inferred from the function type which the closure is being used on, and the numbers represent the names and imply the number of arguments.
`reversedNames = names.sorted(by: { $0 > $1 } )`
- In this particular case of the sorted() method, you can use Swift's string-specific implementation of the greater-than operator `>` (note: look up **Operator Methods**) to make the expression even shorter:
  `reversedNames = names.sorted(by: >)`
  ### Trailing Closures
  - You can also write the closure outside of the method as a **trailing** closure instead, if it makes it easier to read:
`reversedNames = names.sorted() { $0 > $1 }`
- If a closure expression is provided as the function’s or method’s only argument and you provide that expression as a trailing closure, you don’t need to write a pair of parentheses `()` after the function or method’s name when you call the function:
`reversedNames = names.sorted { $0 > $1 }`
- Trailing closures are most useful when the closure is sufficiently long that it isn’t possible to write it inline on a single line.
### Capturing Values
  - A closure can *capture* constants and variables from it's surrounding context, and it holds on to them in memory even after original scope that defined them no longer exists. These variables/constants are now captured, and their memory reference is preserved and tied to the closure. Swift handles memory management involved in disposing of captured variables when they're no longer needed.
```
func makeIncrementer(forIncrement amount: Int) -> () -> Int { 
	var runningTotal = 0 
	func incrementer() -> Int { 
		runningTotal += amount 
		return runningTotal 
	} 
	return incrementer 
}  

let incrementByTen = makeIncrementer(forIncrement: 10)

incrementByTen() // returns a value of 10 
incrementByTen() // returns a value of 20 
incrementByTen() // returns a value of 30

let incrementBySeven = makeIncrementer(forIncrement: 7) 

incrementBySeven() // returns a value of 7
incrementByTen() // returns a value of 40
```
- In the above example, focus on the incrementer() function within makeIncrementer. This function is what gets returned.
- 1st: it doesn't take in any arguments, yet it has access to runningTotal and amount. 
- 2nd: both `amount` and `runningTotal` remain tied to the closure even after the call to makeIncrementer, where they were originally defined, goes out of scope. 
- For each instance of `makeIncrementer()`, they have their own distinct copies of `runningTotal` and `amount`, as you can see by calling incrementBySeven and incrementByTen concurrently. These variables were *captured* and are now contained within the data structure of their instance.
- Also note that incrementByTen and incrementBySeven are declared as constants, but their contents are seemingly changing each call.
- This works because closures are actually *reference* types - assigning them to an instance stores a reference to that closure, not the closure itself. This reference doesn't change. 
### Escaping Closures
- An escaping closure is a closure which is passed as an argument to a function, then called after that function returns and is out of scope. 
- A closure 'escapes' the scope of the function it's passed to, remaining in memory and executing at a later time. 
- The @escaping attribute is used in the parameter definition of the closure to indicate that it needs to remain in memory or is allowed to escape outside of scope. 
```
func asyncFunction(completion: @escaping () -> Void) { 
	// Simulate asynchronous work with a delay 
	DispatchQueue.main.asyncAfter(deadline: .now() + 1) { 
		completion() 
	} 
} 

asyncFunction { 
	print("This is an escaping closure, called after the function returns") 
}
```
- In this example, asyncFunction() places the completion() closure into the DispatchQueue for 1 second later. By the time completion() is called, asyncFunction() is out of scope and memory.
- Another example:
```
func fetchData(completion: @escaping (Data?) -> Void) {
    let url = URL(string: "https://example.com/data")!
    let task = URLSession.shared.dataTask(with: url) { data, response, error in
        completion(data)
    }
    task.resume()
}

fetchData { data in
    if let data = data {
        print("Received data: \(data)")
    }
}
```
- In this example, fetchData takes in an escaping closure. 
- This closure determines what to do with data returned from a network request. The closure could change depending on the need/context, which provides flexibility.
- fetchData sets up the network request:
	- creates a url object.
	- defines a dataTask which retrieves the contents of the URL.
		- data, response, and error are returned and passed into a closure which executes when the network request completes.
	- calls the dataTask with task.resume().
### Escaping Closures in Classes and Structs - Capturing `self`
- Capturing `self`, the reference to a class, in an escaping closure can cause a strong reference cycle to occur.
- A closure which is escaped from a class must refer to self explicitly to avoid strong reference cycles. 
	- This is shown in the call to `someFunctionWithEscapingClosure` within `SomeClass` below.
- `self` can also be captured by including it in the closure's capture list, allowing it to be referenced implicitly. This is shown in `SomeOtherClass`.
- You can't capture `self` in an escaping closure if the instance is a struct or enum, as they don't allow shared mutability (these are *value* types, not reference). 
```
var completionHandlers: [() -> Void] = [] // global array to store completion handlers, to be executed freely once they are ready

func someFunctionWithEscapingClosure(completionHandler: @escaping () -> Void) {
	completionHandlers.append(completionHandler)
}

func someFunctionWithNonescapingClosure(closure: () -> Void) {
	closure()
}

class SomeClass {
	var x = 10
	func doSomething() {
		someFunctionWithEscapingClosure { self.x = 100 }
		someFunctionWithNonescapingClosure { x = 200 }
	}
}

let instance = SomeClass()
instance.doSomething()
print(instance.x)
// Prints "200"

completionHandlers.first?()
print(instance.x)
// Prints "100" 

class SomeOtherClass {
	var x = 10
	func doSomething() {
		someFunctionWithEscapingClosure { [self] in x = 100 }
		someFunctionWithNonescapingClosure { x = 200 }
	}
}

struct SomeStruct {
	var x = 10
	mutating func doSomething() {
		someFunctionWithNonescapingClosure { x = 200 }
		// someFunctionWithEscapingClosure { x = 100 } // Error
	}
}

var instance2: SomeStruct = SomeStruct()
instance2.doSomething()
print(instance2.x)
```
### Autoclosures
- a way to make syntax a little more convenient - lets you omit braces around an expression that is being passed to a function. 
- Not very common to implement yourself - more used in existing library functions.
	- i.e. you probably shouldn't do it - it makes code confusing.
- e.g.:
```
func serve(customer customerProvider: () -> String) {
	print("Now serving \(customerProvider())!")
}
serve(customer: { customersInLine.remove(at: 0) } )
```
is the same as this (with @autoclosure):
```
func serve(customer customerProvider: @autoclosure () -> String) {
	print("Now serving \(customerProvider())!")
}
serve(customer: customersInLine.remove(at: 0) )

```