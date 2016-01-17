# Derived Collection of Enum Cases

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-derived-collection-of-enum-cases.md)
* Author(s): [Jacob Bandes-Storch](https://github.com/jtbandes)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

It is a truth universally acknowledged, that a programmer in possession of an enum with many cases, must eventually be in want of dynamic enumeration over them.

This topic has come up three times on the swift-evolution mailing list so far:

- [List of all Enum values (for simple enums)](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151207/001233.html) (December 8, 2015)
- [Proposal: Enum 'count' functionality](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151221/003819.html) (December 21, 2015)
- [Draft Proposal: count property for enum types](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160111/006853.html) (January 17, 2016)

Enumerating enumerations in Swift is also a popular topic on Stack Overflow:

- [How to enumerate an enum with String type?](http://stackoverflow.com/questions/24007461/how-to-enumerate-an-enum-with-string-type) (June 3, 2014; question score 131)
- [How do I get the count of a Swift enum?](http://stackoverflow.com/questions/27094878/how-do-i-get-the-count-of-a-swift-enum) (November 23, 2014; question score 37)

## Motivation

Simple enums are finite, and their values are statically known to the compiler, yet working with them programmatically is challenging. It is often desirable to iterate over all possible cases of an enum, or to know the number of cases (or maximum valid rawValue).

Currently, however, there is no built-in reflection or enumeration support. Users must resort to manually listing out cases in order to iterate over them:

```swift
enum Attribute {
    case Date, Name, Author
}
func valueForAttribute(attr: Attribute) -> String { …from elsewhere… }

// Cases must be listed explicitly:
[Attribute.Date, .Name, .Author].map{ valueForAttribute($0) }.joinWithSeparator("\n")
```

For RawRepresentable enums, users have often relied on iterating over the known (or assumed) allowable raw values:

*Excerpt from Nate Cook's post, [Loopy, Random Ideas for Extending "enum"](http://natecook.com/blog/2014/10/loopy-random-enum-ideas/) (October 2014):*

```swift
enum Reindeer: Int {
    case Dasher, Dancer, Prancer, Vixen, Comet, Cupid, Donner, Blitzen, Rudolph  
}
extension Reindeer {
    static var allCases: [Reindeer] {
        var cur = 0
        return Array(
            GeneratorOf<Reindeer> {
                return Reindeer(rawValue: cur++)
            }
        )
    }
    static var caseCount: Int {
        var max: Int = 0
        while let _ = self(rawValue: ++max) {}
        return max
    }
    static func randomCase() -> Reindeer {
        // everybody do the Int/UInt32 shuffle!
        let randomValue = Int(arc4random_uniform(UInt32(caseCount)))
        return self(rawValue: randomValue)!
    }
}
```

Or creating the enums by `unsafeBitCast` from their hashValue, which is assumed to be unique:

*Excerpt from Erica Sadun's post, [Swift: Enumerations or how to annoy Tom](http://ericasadun.com/2015/07/12/swift-enumerations-or-how-to-annoy-tom/), with full implementation in [this gist](https://gist.github.com/erica/dd1f6616b4124c588cf7) (July 12, 2015):*

```swift
static func fromHash(hashValue index: Int) -> Self {
    let member = unsafeBitCast(UInt8(index), Self.self)
    return member
}

public init?(hashValue hash: Int) {
    if hash >= Self.countMembers() {return nil}
    self = Self.fromHash(hashValue: hash)
}
```



There are many problems with these existing techniques:

- They are ad-hoc and can't benefit every enum type without duplicated and code.
- They are not standardized across codebases, nor provided automatically by libraries such as Foundation and {App,UI}Kit.
- They are at worst dangerous, and at best prone to bugs (such as when enum cases are added, but the user forgets to update a hard-coded static collection of cases).

## Precedent in other languages

- Rust does not seem to have a solution for this problem.

- C#'s Enum has several [methods](https://msdn.microsoft.com/en-us/library/system.enum_methods.aspx) available for reflection, including `GetValues()` and `GetNames()`.

- Java [implicitly declares](http://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.9.3) a static `values()` function, returning an array of enum values, and `valueOf(String name)` which takes a String and returns the enum value with the corresponding name (or throws an exception). More examples [here](http://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.9.3).

- The Template Haskell extension to Haskell provides a function `reify` which extracts [info about types](http://hackage.haskell.org/package/template-haskell-2.10.0.0/docs/Language-Haskell-TH-Syntax.html#t:Info), including their constructors.

## Proposed solution

Introduce a `CaseEnumerable` protocol. Conforming to CaseEnumerable will automagically derive a `static var cases`, whose type is a CollectionType of all the enum's values.

Like `ErrorType`, the `CaseEnumerable` protocol will not have any user-visible requirements; merely adding the conformance is enough to enable case enumeration.

```swift
enum Ma { case 马, 吗, 妈, 码, 骂, 麻, 🐎, 🐴 }

extension Ma: CaseEnumerable {}

Ma.cases         // returns some CollectionType whose Generator.Element is Ma
Ma.cases.count   // returns 7
Array(Ma.cases)  // returns [Ma.马, .吗, .妈, .码, .骂, .麻, .🐎, .🐴]
```

Conformances can even be added for enums which are defined in other modules (and those imported from C headers):

```swift
extension NSTextAlignment: CaseEnumerable {}

Array(NSTextAlignment.cases)  // returns [NSTextAlignment.Left, .Right, .Center, .Justified, .Natural]
```

## Detailed design

Enum cases are enumerated in the order they appear in the source code.

The `cases` collection does not necessitate `Ω(number of cases)` static storage. For integer-backed enums, only the range(s) of valid rawValues need to be stored, and the enum construction can happen dynamically.

Attempting to derive CaseEnumerable for a non-`enum` type will result in a compiler error.

Attempting to derive CaseEnumerable for an enum with associated values will result in a compiler error.

## Possible variations

I'd like us to discuss these, but they should be folded into either *Proposed solution* or *Future directions* before the proposal is submitted for review.

- For enums with raw values, a static `rawValues` property (a collection of `RawValue` rather than the enum type itself) could also be synthesized.

- CaseEnumerable could have a user-visible declaration requiring `static var cases`, which would allow users to add conformances for custom non-`enum` types.
    - In this case, adding a conformance for a non-`enum` type would not be a compiler error, it would just require an explicit implementation of `static var cases`, since the compiler wouldn't synthesize it.
    - This would probably require `cases` to be `AnySequence<Self>`, or to introduce an `AnyCollection`, since we aren't able to say `associatedtype CaseCollection: CollectionType where CaseCollection.Generator.Element == Self`.

- It would be nice to have a way of supporting this for OptionSetType structs. I would recommend that `cases` for an OptionSetType should include only the already-declared static properties (not all possible combinations of them). However, I'm not sure it fits into this proposal.

## Impact on existing code

This proposal only adds functionality, so existing code will not be affected. (The identifier `CaseEnumerable` doesn't make any significant appearances in Google and GitHub searches.)

## Alternatives considered

The community has not raised any solutions that differ significantly from this proposal, except for solutions which provide strictly **more** functionality. These are covered in the next section, *Future directions*.

An alternative is to *not* implement this feature. The cons of this are discussed in the *Motivation* section above.

The functionality could also be provided entirely through the Mirror/reflection APIs, but this would result in much more obscure and confusing usage patterns.

## Future directions

Many people would be happy to see even more functionality than what's proposed here. I'm keeping this proposal intentionally limited, but I hope the community can continue discussing the topic to flesh out more features.

Here are some starting points, which are **not** part of this proposal:

- Support for enum case **names**. It would be useful to get case names even for enums which have integer rawValues. This could be part of the existing reflection APIs, or it could take the form of derived implementations of StringLiteralConvertible/CustomStringConvertible.
    
- Support for enums with associated values.
    - When all associated values are themselves CaseEnumerable, this could happen automatically:
        ```swift
        enum Suit: CaseEnumerable { case Spades, Hearts, Diamonds, Clubs }
        enum Rank: Int, CaseEnumerable {
            case Ace = 1, Two, Three, Four, Five, Six
            case Seven, Eight, Nine, Ten, Jack, Queen, King
        }
        enum Card {
            case Joker
            case Value(Rank, Suit)
        }

        // This now works, and generates all possible card types (Joker, Value(Ace, Spades), ...)
        extension Card: CaseEnumerable {}
        ```
    - If associated values aren't CaseEnumerable, but all cases are homogeneous, the `cases` collection could vend functions of `AssociatedValueType -> EnumType`:
        ```swift
        enum LogMessage { case Error(String), Warning(String), Info(String) }
        extension LogMessage: CaseEnumerable {}
        
        LogMessage.cases  // elements are (String) -> LogMessage
        ```
    - If Swift had anonymous sum types like `A | B | C`, then `E.cases` could vend elements of type `A->E | B->E | C->E`.
       ```swift
       enum Expr { case Apply(Expr, Expr), Tuple(Expr, Expr), Literal(Int) }
       extension Value: CaseEnumerable {}
       
       // This example is pretty contrived, but illustrates the functionality.
       let fortyTwos = Expr.cases.map {
           // $0 is of type `Int -> Expr | (Expr, Expr) -> Expr`
           switch $0 {
           case let lit as Int -> Expr:  // handles .Literal
               return lit(42)
           case let bin as (Expr, Expr) -> Expr:  // handles .Apply and .Tuple
               return bin(.Literal(42), .Literal(42))
           // all cases are covered
           }
       }
       ```
    
- Support for generic enums.
    - CaseEnumerable could be conditionally supported depending on the generic argument(s). A great example would be Optional:
        ```swift
        enum MyEnum: CaseEnumerable {}
        extension Optional: CaseEnumerable where Wrapped: CaseEnumerable {}
    
        // Optional<MyEnum>.cases effectively contains `MyEnum.cases + [.None]`
        ```
