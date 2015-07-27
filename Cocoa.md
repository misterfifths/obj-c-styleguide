# A Curmudgeonly Foreword

In some ways this is not your typical style guide. I find the stylistic micromanaging in most guides ultimately fruitless. There's no use fighting against years of muscle memory and personal aesthetics to ensure `if ()` over `if()`. Consistency is important, but more at the practical level than the typographic.

In that spirit, this guide aims to ensure clarity, predictability, and de facto avoidance of common gotchas. Except for cases where it overlaps with those goals, I'm not going to dictate elements of personal style.

I leave you with three rules of thumb:

1. Be consistent.
2. Try to fit in with the code around you
3. [Leave the code in better condition than you found it.](https://github.com/RobotsAndPencils/objective-c-style-guide#boyscout--girl-guide)


# Contents

- [Indisputable Truths *(tl;dr)*](#indisputable-truths-tldr)
- [Shameless Plug](#shameless-plug)
- [Solved Problems](#solved-problems)
- [Specific Guidance](#specific-guidance)
  - [Dot Notation](#dot-notation)
  - [Whitespace](#whitespace)
  - [Namespacing](#namespacing)
  - [Properties and ivars](#properties-and-ivars)
  - [Property declarations, `@synthesize`, etc.](#property-declarations-synthesize-etc)
  - [Visibility](#visibility)
  - [Project Organization](#project-organization)
  - [Naming Things](#naming-things)
  - [Singletons](#singletons)
  - [+initialize and +load](#initialize-and-load)
  - [Protocols and Delegates](#protocols-and-delegates)
  - [Interface Stuff](#interface-stuff)
  - [Literals](#literals)
  - [C Stuff](#c-stuff)
  - [Constants](#constants)
  - [Enumerations and bitmasks](#enumerations-and-bitmasks)
  - [Numeric Types and 64-bit Considerations](#numeric-types-and-64-bit-considerations)
  - [NSNumber and NSValue](#nsnumber-and-nsvalue)
  - [Code Shape and "The Golden Path"](#code-shape-and-the-golden-path)
- [Inspirations](#inspirations)


# Indisputable Truths *(tl;dr)*

* Use ARC. If you can't use ARC, kindly inform the client that their requirements are absurd.
* Don't make everything public. Conceptually private properties belong in the class extension (the anonymous category) in the implementation file. ([More...](#visibility))
* Use `dispatch_once` for singletons. And for the love of God, don't override `+alloc` or anything equally crazy to prevent new instances of your class. ([More...](#singletons))
* Prefix *everything*. Preferably with three characters. Yes, even category methods. And absolutely do not co-opt any of the Apple prefixes for your own code. ([More...](#namespacing))
* Give [Apple's guidelines on naming methods](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html) a thorough reading. If you ever think "what should I name this method?", it has the answers. ([More...](#naming-things))
* Four spaces for indents. No tabs. This is not the Xcode default; you should change it.
* Use [CocoaPods](http://cocoapods.org/) for external modules. If the module you want doesn't have a pod, use Git submodules (or, better, write a [podspec](http://guides.cocoapods.org/syntax/podspec.html) for it). Only commit the full external library as a last resort.


# Shameless Plug
Strongly consider using [Amaro](https://github.com/crushlovely/Amaro), our iOS bootstrapper. It sets you up with a ready-to-build project incorporating many of the principles here.


# Solved Problems
Don't reinvent the wheel, a'ight?

* Managing external libraries: [CocoaPods](http://cocoapods.org/)
* Common string formatting (times and dates, addresses, file size, etc.): [FormatterKit](https://github.com/mattt/FormatterKit)
* Logging that is faster and more flexible than `NSLog`: [CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack)
* Sensible, version-controllable build settings for new projects: [xcconfigs](https://github.com/jspahrsummers/xcconfigs)
* Avoiding common causes of [stringly typed code](http://c2.com/cgi/wiki?StringlyTyped): `@keypath` from [libextobjc](https://github.com/jspahrsummers/libextobjc), and `NSStringFromClass`.
* Avoiding strongly capturing `self` in a block (without writing tons of boilerplate): `@weakify` and `@strongify` from [libextobjc](https://github.com/jspahrsummers/libextobjc)
* Safer, smaller model (de)serialization: [Mantle](https://github.com/MantleFramework/Mantle) or [JSONModel](https://github.com/icanzilb/JSONModel).


# Specific Guidance

## Dot Notation

Always use dot notation when dealing with properties. Use bracket notation in all other instances, especially when invoking class methods.

*Prefer*

```objective-c
self.view.backgroundColor = [UIColor redColor];
[UIApplication sharedApplication].delegate
myArray.firstObject
```

*Disprefer*

```objective-c
[[self view] setBackgroundColor:[UIColor redColor]];
UIApplication.sharedApplication.delegate
[myArray firstObject]
```

**Rationale**
There is an assumption that property access will be idempotent and cheap, which is often not true of methods. That said, Apple has been moving towards property-ifying a lot of methods, regardless of their cost. Consider `-[NSDictionary allValues]`, which generates a new array on each call — it became a property in iOS 8, presumably for better Swift support.


## Whitespace
Use four spaces for indenting code. Do not use tabs. This is not the Xcode default; you should change it.

Make liberal use of vertical whitespace to separate logical sections of a method. Use [`#pragma mark`](http://nshipster.com/pragma/) to group related methods — for instance, delegate implementations. (Though [some would argue](http://bendyworks.com/geekville/articles/2014/2/single-responsibility-principle-ios) that `#pragma mark` is a sign that your class is trying to do too much. Consider that possibility as well.)

**Rationale**
Multiple developers using their own indenting preferences leads to ugly code, and no one wants to have to think about what their freaking tab button should do. Readability, my friends. Readability.


## Namespacing

Historically, Apple has recommended a 3-character prefix for all classes, functions, protocols, categories names, category methods/properties, constants, and so on. There is anecdotal evidence that, with the advent of Swift, this may be [becoming passé](http://inessential.com/2014/07/24/prefixes_considered_passe).

So, my guidance is nuanced. Use a prefix in your application code if you prefer it. But you **must** use a prefix in code that is part of a library, to avoid conflicts with others.

If you decide to (or must) use a prefix, use a *project*-specific one, preferably 3 characters, but no less than 2. Use it **everywhere**. Under *no* circumstances should you co-opt a well-established existing prefix (e.g. `NS`, `AF`, `CL`).

*Prefer*

```objective-c
@interface CRLMapView : MKMapView  { ... }

@interface NSString (MR5Regexes) {
    -(BOOL)mr5_doesMatchRegularExpressionFromString:(NSString *)regexString;
}

CGPoint CRLPointMakeIntegral(CGPoint pt);

static NSString * const CRLServiceErrorDomain = @"CRLServiceErrorDomain";
```

*Disprefer*

```objective-c
@interface MKMyMapView : MKMapView { ... }

@interface MapView : MKMapView { ... }

@interface NSString (Regexes)  {
    -(BOOL)doesMatchRegularExpressionFromString:(NSString *)regexString;
}

CGPoint CGPointMakeIntegral(CGPoint pt);

static NSString * const ServiceErrorDomain = ...
```

**Rationale**
This is to avoid name collisions against other third-party projects, private APIs, and future APIs. For instance, many folks added `-[NSArray firstObject]` through categories, and when iOS 4 exposed that method, their category implementations overrode the first-party one. In this particular case it wasn't a big problem, but you can imagine that it would be Bad if your implementation of a method differed significantly.

Also, from a readability perspective, it's useful to be able to easily distinguish third-party code.

More reading on the topic of namespacing can be found in [this NSHipster article](http://nshipster.com/namespacing/).


## Properties and ivars

Use properties over ivars, almost always. There are two exceptions where ivars are acceptable:

* The property is *only* accessed through pointers (e.g. integers used with the `OSAtomic` API, or a few things from Foundation).

* You've profiled and found that the `objc_msgSend` overhead is actually significant in your use case. This is an *exceedingly* rare case.

Public and protected ivars are never acceptable. Use a separate header with a subclass-only category for 'protected' properties. See [Visibility](#visibility).

**NB** Accessing properties through their getter/setter in initializers and `-dealloc` [can be dangerous](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html#//apple_ref/doc/uid/TP40004447-SW6). The instance is not entirely initialized at those points, and the side-effects of the getter/setter (including any external KVO) may make bad assumptions about the object. So, **there are exactly 3 places where you should access the synthesized ivar for a property: `-init`, `-dealloc`, and overridden getter/setter implementations.**

*Prefer*

```objective-c
@interface CRLDummy ()
@property (nonatomic, assign) BOOL internalState;
@end

-(id)init {
    // (omitting init boilerplate...)
    _internalState = 2;
}

// But, in every other method except -dealloc and overridden accessors,
self.internalState = 2;
```

*Disprefer*

```objective-c
@interface CRLDummy () {
    BOOL internalState;
}

-(id)init {
    // (omitting init boilerplate...)
    self.someProperty = 2;
}

// And elsewhere,
internalState = 2;  // or self->internalState
```

**Rationale**
The traditional argument for ivars is that they avoid the overhead of `objc_msgSend` incurred when accessing properties. However, [that concern is unwarranted](http://blog.bignerdranch.com/4005-should-i-use-a-property-or-an-instance-variable/) in 99% of cases.

Standardizing around properties provides a unified syntax for accessing member data, and adds override points for setters and getters. It also sidesteps unintentional issues that may be caused by accessing ivars directly. For instance:

* Properties declared `copy` will only actually copy their values if assigned through the setter. (And the fact that this isn't the case with `strong` and `weak` is certain to be confusing to newcomers.)
* Direct ivar access isn't KVO-compliant — observers will not be able to detect changes to properties made via the ivar.
* Even though you may not explicitly reference `self`, accessing ivars in a block still implicitly captures a strong reference to `self`. Surprise!


## Property declarations, `@synthesize`, etc.

In keeping with the guidance in [Dot Notation](#dot-notation), make liberal use of `readonly` properties with custom getters to expose cheap computed data. For instance, consider the property `userLocationVisible` on `MKMapView`.

**Rationale** Values like this could be implemented as a method, but the property syntax is more semantically meaningful. Also it's convenient to use dot notation.

&nbsp; 

Use `nonatomic` on all properties.

**Rationale** `atomic` has [performance issues on ARM and is generally not all that helpful anyway, despite being the default](http://www.crashlytics.com/blog/what-clang-taught-us-about-objective-c-properties).

&nbsp;

Use `copy` for any property whose type could potentially be mutable (`NSString`, `NSArray`, `NSSet`, etc.).

**Rationale** Take the example of `NSString`. Since it's a subclass, you could easily be passed an `NSMutableString`, only to have its value change without your class noticing.

&nbsp;

Use the newer `strong` instead of `retain`.

**Rationale** They are functionally identical, but `strong` more accurately describes the semantics under ARC.

&nbsp;

Always specify the storage semantics for a writable property. That is, a writable property must have one of `strong`, `weak`, `assign`, or `copy`.

**Rationale** No one can remember what the default is.

&nbsp;

If your property is an adjectival boolean, Apple recommends that you override the getter to give it the prefix 'is': `@property (nonatomic, assign, getter=isOpaque) BOOL opaque`

&nbsp;

In service of [DRY](http://c2.com/cgi/wiki?DontRepeatYourself), let Xcode auto-synthesize ivars for you unless:

* You are implementing an `optional` property from a protocol (these must be explicitly synthesized). For consistency's sake, give the synthesized ivar an underscore prefix, just as if it was auto-synthesized: `@synthesize state = _state`.

* Your code (or your superclass — *\*ahem\** `NSManagedObject`) is doing some runtime voodoo which calls for `@dynamic`.


## Visibility

Objective-C lacks the visibility specifiers common in other object-oriented languages. That is not an excuse to make everything public.

Everything that is an implementation detail belongs in the `.m` file. Basically, if there is no reason for a consumer of your class to see it, or if it may change dramatically, don't put it in the `.h`.

Similarly, don't expose mutable datatypes through public properties. You don't want users to be able to change data out from under your nose. Expose an immutable copy of your internal data.

For properties that should only be available to subclasses ("protected"), use a category in a separate header. For an example from Apple, see `UIGestureRecognizerSubclass.h`.

*Prefer*

```objective-c
// CRLDummy.h
@interface CRLDummy : NSObject
@property (nonatomic, readonly) NSArray *data;
@end

// CRLDummy.m
@interface CRLDummy ()
@property (nonatomic, assign) BOOL internalState;
@property (nonatomic, strong) NSMutableArray *mutableData;
@property (nonatomic, weak) IBOutlet UILabel *dataLabel;
@end

@implementation CRLDummy
-(NSArray *)data {
    return [NSArray arrayWithArray:self.mutableData];
}
@end
```

*Disprefer*

```objective-c
// CRLDummy.h
@interface CRLDummy : NSObject
@property (nonatomic, strong) NSMutableArray *data;
@property (nonatomic, assign) BOOL internalState;
@property (nonatomic, weak) IBOutlet UILabel *dataLabel;
@end
```

**Rationale** Encapsulation. Ease of refactoring. Etc. etc.

&nbsp;

A specific case of the above principle is that of `-[UIViewController initWithNibName:bundle:]`. It is the default initializer for view controllers, but in 99.9% of cases, the name of the nib for your VC is an implementation detail. Don't make users of your class supply it! Supply a custom `-init` method that does the right thing. See ["initWithNibName:bundle: Breaks Encapsulation"](http://oleb.net/blog/2012/01/initWithNibName-bundle-breaks-encapsulation/).

&nbsp;

Minimize the number of imports in your header files. Use `@class` and `@protocol` forward-declarations in headers whenever possible, and do the actual import in your `.m`. See `UIView.h` for inspiration.

**Rationale** Avoids polluting the namespace of consumers of your class with implementation details.


## Project Organization

Keep your Xcode project in sync with the filesystem. Xcode groups should have corresponding physical folders.

**Rationale** Makes it far easier and less surprising to deal with the project in source code management and on the command line.


## Naming Things

Objective-C has notoriously verbose names for ... most everything. Apple has [a very good document](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html) describing best practices for naming things. Read it. It has the answers.

Some highlights from that document:

&nbsp;

Do not use `and` in method names, unless it communicates that the method performs multiple actions.

*Prefer*

```objective-c
-(id)initWithCenter:(CGPoint)center radius:(CGFloat)radius;
-(void)enqueueJob:(id)job andUpdateView:(UIView *)view;
```

*Disprefer*

```objective-c
-(id)initWithCenter:(CGPoint)center andRadius:(CGFloat)radius;
-(void)enqueueJob:(id)job updateView:(UIView *)view;
```

&nbsp;

Only use `get` in accessor methods when they return results indirectly (via a pointer or through a callback, for instance).

*Prefer*

```objective-c
-(id)viewForJob:(id)job;
-(void)getHue:(CGFloat *)h saturation:(CGFloat *)s brilliance:(CGFloat *)b;
```

*Disprefer*

```objective-c
-(id)getViewForJob:(id)job;
-(void)hue:(CGFloat *)h saturation:(CGFloat *)s brilliance:(CGFloat *)b;
```


## Singletons

The [One True Way](http://cocoasamurai.blogspot.com/2011/04/singletons-your-doing-them-wrong.html) to make singletons is with `dispatch_once` and a static instance:

```objective-c
+(CRLSingletonClass *)sharedInstance {
    static dispatch_once_t onceToken;
    static CRLSingletonClass *singletonInstance;
    
    dispatch_once(&onceToken, ^{
        singletonInstance = [[CRLSingletonClass alloc] init]
    });
    
    return singletonInstance;
}
```

I recommend that that be the end of it — don't try to enforce that the shared instance is the only one, and please don't do anything unholy like overriding `+alloc`. If you're feeling particularly overprotective, consider marking the `init`s with [`__attribute__((unavailable))`](https://blog.twitter.com/2014/attribute-directives-in-objective-c). But ultimately, there's very little use in writing code to babysit consumers of your class.

Apple gives little guidance on naming the accessor for a singleton. Generally `sharedWhatever` is appropriate when there is intended to be only one instance of the class (e.g. `+[UIApplication sharedApplication]`). If multiple instances could conceivably exist, something like `standardWhatever` or `defaultWhatever` is more appropriate (e.g. `+[NSUserDefaults standardUserDefaults]` or `+[NSFileManager defaultManager]`).


## +initialize and +load

`+initialize` can be a useful place to set up data that will be shared by all instances of your class. However, there is a big caveat. If your class is subclassed, and the subclass doesn't implement `+initialize`, your implementation will be called more than once. So, **always guard the body of `+initialize` with a check that `self` is the class you think it is**:

```objective-c
+(void)initialize {
    if(self != [CRLExpectedClass class]) return;
    // ... real initialization code ... 
}
```

As Mike Ash [points out](https://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html), you should do this on *every class*, even if you can't imagine it will be subclassed. Under the covers, KVO creates dynamic subclasses that will trickle back to your `+initialize`.

Relatedly, from that Mike Ash article, avoid `+load`. It's a scary place.


## Protocols and Delegates

Declare protocols in the same header as the related class.

Always annotate the methods of your protocol with `@optional` and `@required`. No one can remember the default.

Remember to check if an adopter of your protocol responds to the selector of optional methods before invoking them.

Delegate properties should be strongly typed (`id<CRLDelegate>`) and `weak`, to avoid retain cycles.

Strict adherence to [Apple's guidelines for delegate method names](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CodingGuidelines/Articles/NamingMethods.html#//apple_ref/doc/uid/20001282-1001803-BCIDAIJE) is important. You should be able to look at a method signature and know with 99% certainty if it is part of a delegate protocol.

[Informal protocols](https://developer.apple.com/library/ios/documentation/general/conceptual/devpedia-cocoacore/Protocol.html) are a relic. Don't.


## Interface Stuff

`IBOutlet` properties should almost always be `weak`. There are a few rare exceptions:

* If the outlet is a top-level object in your NIB, you need to store a strong reference to it.

* If you will be removing the outlet from the view hierarchy (but want to reuse it later), you will need a strong reference to it in order to avoid it being released.

**Rationale** The view hierarchy owns most things that you would create an outlet for. Having weak outlets avoids surprising retain cycles.

&nbsp;

With almost no exceptions, `IBOutlets` (and view properties in general) should be private.

**Rationale** See [Visibility](#visibility).

&nbsp;

Keep in mind that `-viewWillAppear` and `-viewDidAppear` will likely be called multiple times (e.g. if a modal comes and goes). You may find it valuable to set a flag on the first call to `-viewWillAppear` so you know if this is a new appearance or just the disappearance of a modal or popping of a navigation stack.

&nbsp;

As of iOS 6, `-viewWillUnload` and `-viewDidUnload` are dead. Your view will never be unloaded; in a low memory situation, only its backing store will be cleared. Rejoice!

&nbsp;

Do not leave `-initWithNibName:bundle:` as the preferred initializer for consumers of your view controllers. The location of the NIB for your view is an implementation detail! See the discussion in [Visibility](#visibility)

&nbsp;

When subclassing `UIView`, be sure to implement both `-initWithCoder:` and `-initWithFrame:`, so that your view can be instnatiated from Interface Builder and code.


## Literals

Use them.

*Prefer*

```objective-c
@[obj1, obj2]
@{ @"first": obj1, @"second": obj2 }
@1
@YES
@(anInteger + 3)
```

*Disprefer*

```objective-c
[NSArray arrayWithObjects:obj1, obj2, nil]
[NSDictionary dictionaryWithObjectsAndKeys:obj1, @"first", obj2, @"second", nil]
[NSNumber numberWithInt:1]
[NSNumber numberWithBool:YES]
[NSNumber numberWithInt:anInteger + 3]
```

**Rationale** Just ... look.


## C Stuff

Use `NS_INLINE` (compiler-agnostic `static inline`) functions instead of function-like macros, whenever possible.

Note that the bodies of inline functions must appear in a header, so that the compiler has the code in order to inline them.

*Prefer*

```objective-c
NS_INLINE void CRLMakeTheMagic(NSInteger a, NSInteger b) { ... }
```

*Disprefer*

```objective-c
#define CRLMakeTheMagic(a, b) do { ... } while(0)
```

**Rationale** Functions are typesafe. Also they don't require the gross syntax needed to make macros safe in all contexts (the `do... while` wrap or other constructs).

## Constants

Be aware that the meaning of the `const` keyword is context-sensitive. Read declarations in reverse. Consider these two variables:

```objective-c
const NSString *var1 = @"hi"   // Nope.
NSString * const var2 = @"hi"  // Yup!
```

Reading in reverse, `var1` is "a pointer to an `NSString` that is constant". That is redundant, however, as `NSString` is constant by nature (`NSMutableString` being the non-constant version). This leaves `var1` assignable by anyone; exactly what you were hoping to prevent. `var2`, however, reads as "a constant pointer to an `NSString`", which is what was wanted.

&nbsp;

Declare public constants the way Apple does — put an `extern` reference to the variable name in a header, and the actual values in the implementation file. Note that constant definitions like these should be *outside* of any `@interface` or `@implementation` blocks, since they are not an Objective-C feature.

Use constant variables over `#define`d macros whenever possible. Define macros only for things that will actually be used by the preprocessor (e.g. in an `#if`).

*Prefer*

```objective-c
// CRLAPIClient.h
extern NSString * const CRLAPIClientSecret;
extern NSString * const CRLAPIClientID;

// CRLAPIClient.m
NSString * const CRLAPIClientSecret = @"my-secret";
NSString * const CRLAPIClientID = @"my-client-id";
```

*Disprefer*

```objective-c
// CRLAPIClient.h
NSString * const CRLAPIClientSecret = @"my-secret";

// or ...
#define CRLAPIClientSecret @"my-secret"
```

**Rationale** Typesafety, Swift accessibility, avoidance of linker issues.


## Enumerations and bitmasks

Use the `NS_ENUM` and `NS_OPTIONS` macros [detailed here](http://nshipster.com/ns_enum-ns_options/) for enumerations. Generally use `NSInteger` as the underlying type for `NS_ENUM`s, and `NSUInteger` for `NS_OPTIONS`. Options enumerations should be named in the plural or with a gerund. For instance, `UIViewAnimationOptions` and `UIViewAutoResizing`. Enums that represent independent options should be named with a singular: `UIViewAnimationTransition`.

*Prefer*

```objective-c
typedef NS_ENUM(NSInteger, CRLAnimationCurve) { ... }
typedef NS_OPTIONS(NSUInteger, CRLAnimationOptions) { ... }
```

*Disprefer*

```objective-c
typedef enum _CRLAnimationCurve { ... } CRLAnimationCurve
```

**Rationale** These macros allow the compiler to do much better checking about your use of enumerations, improve completion in Xcode, and import properly into Swift. Also the syntax is a little more sane.

&nbsp;

Name the individual elements of an enumeration or bitmask with a prefix of the enum's name (singularized if need be).

*Prefer*

```objective-c
typedef NS_ENUM(NSInteger, CRLAnimationCurve) {
    CRLAnimationCurveCubic,
    CRLAnimationCurveSinusoidal
}

typedef NS_OPTIONS(NSUInteger, CRLAnimationOptions) {
    CRLAnimationOptionAutoReverse
}
```

*Disprefer*

```objective-c
typedef NS_ENUM(NSInteger, CRLAnimationCurve) {
    // anything not beginning with "CRLAnimationCurve"...
    Cubic,
    CRLCurveTypeSinusoidal
}
```

**Rationale** Consistency, predictability, and Swift compatibility.


## Numeric Types and 64-bit Considerations

The iPhone 5s introduced the ARM64 platform and changed the underlying type of several Cocoa typedefs. `CGFloat` became a `double`, and `NSInteger` and `NSUInteger` became `long`. So, to ensure backward- and forward-compatibility:

&nbsp;

Stick to the Cocoa typedefs wherever possible: the `NS`-prefixed integer types, `CGFloat`, and so on.

&nbsp;

Consider importing `tgmath.h` in your prefix header. This is a [C99 feature](http://libreprogramming.org/books/c/tgmath/) that provides type-generic wrappers around the standard `math.h` functions. After including this, you can write code like `cos(someCGPoint.x)` without warnings on all architectures. Avoid the type-specific versions like `cosf` and `cosl`.

&nbsp;

Untrain yourself from typing the `f` suffix on floating-point literals. `CGFloat` is a double on ARM64, so you're really just wasting keystrokes by typing it. Moreover, any concerns over performance are unwarranted; the downcast on 32-bit architectures will happen at compile time.

*Prefer*

```objective-c
CGPoint pt = CGPointMake(5.0, 34.2);
CGFloat newWidth = 0.25 * CGRectGetWidth(self.view.frame);
```

*Disprefer*

```objective-c
CGPoint pt = CGPointMake(5.0f, 34.2f);
CGFloat newWidth = 0.25f * CGRectGetWidth(self.view.frame);
```

&nbsp;

Use the most semantically meaningful primitive type possible. For example, if a number could never conceivably be negative (e.g., an array index), use an `NSUInteger`.

Also, prefer Apple's specialized typedefs when they make sense. For instance, use `NSTimeInterval` instead of `double` for durations expressed in seconds.

**Rationale** Interoperability with Apple's APIs, type genericity, and expressiveness.


## NSNumber and NSValue

`NSNumber` has one and **only** one use: to put primitives into Objective-C containers (`NSArray`, `NSDictionary`, etc.). Do not use it for properties or local variables; always prefer a primitive type when possible.

**Rationale** `NSNumber` is annoying since you can't do math on it. Also an `NSNumber` loses all the semantic meaning in an actual primitive type — since it can store floats, integers, unsigned integers, and booleans, it leaves no indication of what type the variable is actually intended to be.

&nbsp;

The same goes for `NSValue`, which is responsible for boxing structs. Don't use it for anything other than putting structs in containers.


## Code Shape and "The Golden Path"

Keep the "golden path" (i.e. the main logic flow) through a function at the lowest possible level of nesting. Multiple return statements are fine. See [Robots and Pencils' style guide](https://github.com/RobotsAndPencils/objective-c-style-guide#golden-path).

This probably combats your muscle memory for `-init` functions, but give it a shot.

*Prefer*

```objective-c
-(void)someMethod {
    if(![someOther boolValue]) return; // ... or otherwise handle the error
    //Do something important
}
```

*Disprefer*

```objective-c
-(void)someMethod {
    if([someOther boolValue]) {
        //Do something important
    }
}
```

**Rationale** Readability. Writability (nicer to not have to worry about matching deeply nested brackets). And, semantically, this makes the main purpose of a block of code more obvious.


# Inspirations

This guide stands on the shoulders of many other such guides.

* [Apple](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html#//apple_ref/doc/uid/10000146-SW1)
* [Robots and Pencils](https://github.com/RobotsAndPencils/objective-c-style-guide)
* [GitHub](https://github.com/github/objective-c-conventions)
* [The New York Times](https://github.com/NYTimes/objective-c-style-guide)
