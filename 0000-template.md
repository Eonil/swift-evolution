# Allow `guard let self = self else { … }` for weakly captured self in a closure.

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Author(s): [Hoon H.](https://github.com/Eonil)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

Weakly captured `self` in closures is an optional type and makes harder to referencing `self`.
And we have to use another temporary variable to access `self` in non-optional strong reference manner.
It would be nicer if we can access strong non-optional name `self` in this case with no extra variable.

Also we can elide `self` qualification after rebinding `self` because it becames strong non-optional 
reference again.

Swift-evolution thread: [Allowing `guard let self = self else { … }` for weakly captured self in a closure.](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160104/005439.html)

## Motivation

Weakly captured `self` is an optional type, and that makes harder to use return value of instance method.
Consider this example.

    class Foo {
        func produce() -> Int {
            return 1234
        }
        func consume(value: Int) {
            print(value)
        }
        func test1() {
            test2 { [weak self] in

            // Doesn't work because it's not unwrapped. Compile error.
            self?.consume(self?.produce()) 

            // Works, but considered harmful due to `!` on `self`.
            self!.consume(self!.produce()) 

            // OK, but harmful. Because `self` can be deallocated at any time. 
            // Even if `self` doesn't' deallocates, this looks and feels harmful.
            guard self != nil else { return }
            self!.consume(self!.produce()) 

            // OK, but feels ugly due to unnecessary new name `self1`.
            guard let self1 = self else { return }
            self1.consume(self1.produce()) 
		}
	}
}

## Proposed solution

I propose rebinding weakly captured `self` to strong non-optional `self` using `guard let ...` statement.

    class Foo {
        func produce() -> Int {
            return 1234
        }
        func consume(value: Int) {
            print(value)
        }
        func test1() {
            test2 { [weak self] in

            guard let self = self else { return }
            self.consume(self.produce()) 
		}
	}
}

This eliminates need for temporary variable.

`guard let` naturally makes optional weak reference into a strong non-optional reference.
Clean in semantics, easy to understand, and helps users to write safer code.
This is also efficient because users don't have to figure out a new name or set a convention for it.

This also can be applied to `if let` or same sort of constructs, but I think `guard let` would be more apropriate.

Even further, we can consider removing required reference to `self` after `guard let …`.

    guard let self = self else { return } 
    consume(produce()) // Referencing to `self` would not be required anymore.

I think this is almost fine because users have to express their intention explicitly with `guard` statement. 
If someone erases the `guard` later, compiler will require explicit self again, and that will prevent mistakes. 
But still, I am not sure this removal would be perfectly fine.

I am not sure whether this is already supported or planned. But lacked at least in Swift 2.1.1.



## Detailed design

This does not need any new synyax or API. Just adding a new feature in semantics.

## Impact on existing code

None.

## Alternatives considered

Alternatives are described in "Motivation" section. And none of them is clean, efficient and safe enough.

We can consider using of escaped keyword identifier using back-ticks (suggested by Christopher Rogers).

    guard let `self` = self else { return }

But I think this is more verbose, and confusing. 

There was a buggy behavior in Swift 2.1 that allows accessing `self` without back-ticks in this case. Anyway, it's been 
clarified as a bug and very likely to be removed by core team.

































is disallowed and emits a compiler error.


Currently, weakly captured `self` cannot be bound to `guard let …` with same name, and emits a compiler error.

	class Foo {
		func test2(f: ()->()) {
			// … 
		}
		func test1() {
			test2 { [weak self] in
				guard let self = self else { return } // Error.
				print(self)
			}
		}
	}

Do we have any reason to disallow making `self` back to strong reference? It’d be nice if I can do it. Please consider this case.

	class Foo {
		func getValue1() -> Int {
			return 1234
		}
		func test3(value: Int) {
			print(value)
		}
		func test2(f: ()->()) {
			// … 
		}
		func test1() {
			test2 { [weak self] in
				self?.test3(self?.getValue1()) // Doesn't work because it's not unwrapped.

				self!.test3(self!.getValue1()) // Considered harmful due to `!`.

				guard self != nil else { return }
				self!.test3(self!.getValue1()) // OK, but still looks and feels harmful.

				guard let self1 = self else { return }
				self1.test3(self1.getValue1()) // OK, but feels ugly due to unnecessary new name `self1`.

				guard let self = self else { return }
				self.test3(self.getValue1()) // OK.

			}
		}
	}

This also can be applied to `if let` or same sort of constructs.

Even further, we can consider removing required reference to `self` after `guard let …` if appropriate.

	guard let self = self else { return } 
	test3(getValue1()) // Referencing to `self` would not be required anymore. Seems arguable.

I think this is almost fine because users have to express their intention explicitly with `guard` statement. If someone erases the `guard` later, compiler will require explicit self again, and that will prevent mistakes. But still, I am not sure this removal would be perfectly fine.

I am not sure whether this is already supported or planned. But lacked at least in Swift 2.1.1.


