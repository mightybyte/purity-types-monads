% Coding and Reasoning with Purity, Strong Types, and Monads
% Doug Beardsley (mightybyte)
% January 29, 2012

# Programmers will ALWAYS make mistakes

- ![oops](http://i.imgur.com/Uko3wuw.png)
- Progress dictates that complexity will be near the limits of our
  capabilities.
- What do we do about this?
    - Make the computer detect as many of our mistakes as possible
- How does Haskell accomplish this?
    - Strong static type system
    - Purity
    - Monads are one of the nicer ways to do I/O in pure, strongly
      typed languages.

# Monad press missing the point

- Lots of talk about monads these days
- Notably, Douglas Crockford's YUIConf keynote "Monads & Gonads"
- "Anything that returns 'this' is kind of a monad." --Douglas Crockford
- "JQuery is a monad." --Joe Javascript Programmer
- But that's not what Haskellers are so zealous about.

# Monads in static and dynamic languages

- Haskell monad definition

```haskell
return :: a   -> m a
(>>=)  :: m a -> (a -> m b) -> m b
```

- Haskell separates the context from the return value.
- The m has to stay the same--you can't suddenly switch contexts.

# Monads in static and dynamic languages

- Haskell monad definition

```haskell
return :: a   -> m a
(>>=)  :: m a -> (a -> m b) -> m b
```

- Haskell separates the context from the return value.
- The m has to stay the same--you can't suddenly switch contexts.
- "A dynamically typed language is...a static language with a single
  type" --Roman Cheplyaka
- Dynamic language monad definition

```haskell
return :: U   -> U
(>>=)  :: U   -> (U -> U)   -> U
```

- (also from Roman Cheplyaka)

# Monads in static and dynamic languages

- Haskell monad definition

```haskell
return :: a   -> m a
(>>=)  :: m a -> (a -> m b) -> m b
```

- Haskell separates the context from the return value.
- The m has to stay the same--you can't suddenly switch contexts.
- "A dynamically typed language is...a static language with a single
  type" --Roman Cheplyaka
- Dynamic language monad definition

```haskell
return :: U   -> U
(>>=)  :: U   -> (U -> U)   -> U
```

- (also from Roman Cheplyaka)
- No concept of a "return value"
- In that context, monads are just a pattern for concise "chaining" syntax
- The main point is the interaction between monads, strong static types, and purity.

# Case Study 1: Data Fusion

- Large scale, real time, multi-threaded system
- Large shared data structure
- Ambitious performance requirements
- Fairly mature system
    - ~10 man-years of development effort
    - ~120k lines of Java code

# Speculative refactoring to improve performance

- Existing approach: locks

```java
synchronized ( entity ) {
  updateEntity(entity, newInput);
}
```

- Considered changing to use a read-copy-update approach

```java
newEntity = updateEntity(entity, newInput);
entities.insert(newEntity); // uses CAS instruction for non-blocking insert
```

- `updateEntity()` is a long and complicated algorithm
- Want to ensure that I'm not modifying the entity during updateEntity()
    - `entity.setXYZ(blah)` should be impossible
- I don't trust myself to not screw this up somewhere in a large code base.
- Can't use `const` -- it only applies to the reference, not the data inside
- Because Java is impure, this is difficult/unnatural.

# How it might be done in Java

```java
public class ImmutableEntity {
  public ImmutableEntity(Entity e);
  // only define getter methods
}
```

- Slightly inconvenient
- But...

# How it might be done in Java

```java
public class ImmutableEntity {
  public ImmutableEntity(Entity e);
  // only define getter methods
}
```

- Slightly inconvenient
- But...

```java
Position p = immutableEntity.getPosition();
p.setX(5); // Wait a minute...
```

# How it might be done in Java

```java
public class ImmutableEntity {
  public ImmutableEntity(Entity e);
  // only define getter methods
}
```

- Slightly inconvenient
- But...

```java
Position p = immutableEntity.getPosition();
p.setX(5); // Wait a minute...
```

- So...

```java
public class ImmutablePosition {
  public ImmutablePosition(Position e);
  // only define getter methods
}
```

# How it might be done in Java

```java
public class ImmutableEntity {
  public ImmutableEntity(Entity e);
  // only define getter methods
}
```

- Slightly inconvenient
- But...

```java
Position p = immutableEntity.getPosition();
p.setX(5); // Wait a minute...
```

- So...

```java
public class ImmutablePosition {
  public ImmutablePosition(Position e);
  // only define getter methods
}
```

- You have to do this all the way down to the primitives
- Adds a lot of extra code in an already verbose language


# Haskell solves this easily

- Existing approach

```haskell
updateEntity :: Entity -> Input -> m ()
```

- Read-copy-update

```haskell
updateEntity :: Entity -> Input -> Entity
swapEntity :: Entity -> m ()
```

- Because Haskell is pure and strongly typed, we know that `updateEntity`
  cannot modify the global entity repository.
- The distinction between "`a`" and "`m a`" is crucial here.

```haskell
return :: a   -> m a
(>>=)  :: m a -> (a -> m b) -> m b
```

# Case Study 2: Heist

- An HTML5 template system
- Designed around the DOM
- Uses data model from the xmlhtml package

```haskell
data Node = TextNode Text
          | Comment Text
          | Element
            { name     :: Text
            , attrs    :: [(Text, Text)]
            , children :: [Node]
            }
```

- Allows you to populate HTML pages with dynamic content

# DOM transformations with splices

- Markup is all valid HTML5
- No special syntax

```html
<h1><user:login/></h1>
<ul>
  <li>Age: <user:age/></li>
</ul>
```

- Associates Haskell functions (called splices) with tags
- Mental model

```haskell
type Splice m = Node -> m [Node]
```

- `m` is the run time monad
- Need to read from databases etc, so can't be `Node -> [Node]`
- Actual implementation

```haskell
type Splice m = HeistT m [Node]
```


# Powerful but slow

- Heist was essentially an interpreted language
- Every time the template is rendered
    - Traversing DOM
    - Transforming it
    - Rendering the transformed DOM to markup
- Compared to concatenative templating...very slow

# Speed up by changing to a compiled language

- Change splices to run as much as possible at load time
- But we still have to allow run time data to be inserted

```haskell
HeistT m [Node]
```

...becomes

```haskell
HeistT n IO (DList (Chunk n))
```

- Chunk is our intermediate representation
- It encapsulates two possibilities
    - Pure `ByteString` known at load time
    - A run time computation (the monad `n`) generating a `ByteString`

# Chunks compiled into a run time render function

- Adjacent pure items collapsed
- End up with

```haskell
[ Pure "<h1>", Runtime getUserLogin, Pure "</h1><ul><li>Age: "
, Runtime getUserAge, Pure "</li></ul>"]
```

- This "object code" finally converted into `(n Builder)`

# Implementation

- Proof of concept coded from scratch by Gregory Collins and Jasper Van der
  Jeugt at UHac.
- My job: figuring out how the new and old paradigms should coexist.
- Very open-ended problem
- Should I completely eliminate old interpreted method?
    - ...no
    - Significant knowledge and experience contained in the existing Heist code and test suite
- Ship as one or two packages?
    - ...one
    - Avoid code duplication
- Separate interpreted and compiled or allow them both to be used?
    - ...allow both
    - Allow users to transition more incrementally

# Benefits of doing this in Haskell

- I originally wanted to take you step-by-step through the evolution.
- Unfortunately that doesn't show up in commit logs.
- Points that stick out in my mind
    - The distinction between load time and run time is crucial
        - Exactly what Haskell's strongly typed purity gives us
    - The context (run time or load time) is whatever monad we're in
    - Benefits to me
        - Reuse the HeistT abstraction and reduce code
        - Less potential for implementation error
    - Benefits to the users
        - Can't confuse a compiled splice for an interpreted splice
    - Type system was invaluable in helping me explore the design space

# How might it look in Java?

- What does it mean to be running in a particular run time monad?
- Program life cycle
    - Process starts up, do a bunch of initialization.
    - Listen on a socket and wait for HTTP requests.
    - Parse the HTTP request into an HttpRequest object.
    - Call the appropriate user-defined handler function and make the
      HttpRequest data available to it.
- Being in the run time monad means you have an HttpRequest.
- What if you're not in the run time monad?

# How might it look in Java?

- What does it mean to be running in a particular run time monad?
- Program life cycle
    - Process starts up, do a bunch of initialization.
    - Listen on a socket and wait for HTTP requests.
    - Parse the HTTP request into an HttpRequest type.
    - Call the appropriate user-defined handler function and make the
      HttpRequest data available to it.
- Being in the run time monad means you have an HttpRequest.
- What if you're not in the run time monad?
    - `httpRequest == NULL`

# How might it look in Java?

- What does it mean to be running in a particular run time monad?
- Program life cycle
    - Process starts up, do a bunch of initialization.
    - Listen on a socket and wait for HTTP requests.
    - Parse the HTTP request into an HttpRequest type.
    - Call the appropriate user-defined handler function and make the
      HttpRequest data available to it.
- Being in the run time monad means you have an HttpRequest.
- What if you're not in the run time monad?
    - `httpRequest == NULL`
- Java doesn't give you concepts for run time and load time.  It gives you
  null pointer checks (a run time concept).
- Even if you assume the null pointer check is already done, run time == the
  existence of an HttpRequest.  This is much less modular and useful than
  Haskell's typed monads.

# A Deeper Dive

```haskell
newtype HeistT n m a = HeistT {
    runHeistT :: Node
              -> HeistState n
              -> m (a, HeistState n)
}
data HeistState n = HeistState
    { _spliceMap           :: HashMap Text (HeistT n n [Node])
    , _compiledSpliceMap   :: HashMap Text (HeistT n IO (DList (Chunk n)))
    , ...
    }
```

- A lot of information encoded here
- `n` represents the run time monad
- `m` represents the "currently running" monad
- The load time monad is IO, so compiled splices have to run in IO (type
  level guarantee)
- We could be even more strict and define our own load time monad
    - `newtype Loadtime a = Loadtime { runLoadtime :: IO a }`
- For run time "interpreted" splices, the run time monad is the same as the monad they're
  running in.

# Types are both stringent and flexible

```haskell
HeistT n n [Node]             -- Interpreted (run time) splice
HeistT n IO (DList (Chunk n)) -- Compiled (load time) splice
```

- What if we had something like this

```haskell
data PersonType = Grunt
                | Manager
                | Executive deriving (Show, Enum, Bounded)
enumSelectSplice :: Show a
                 => Text     -- ^ Name attribute
                 -> [a]
                 -> HeistT n n [Node]
personSelect = enumSelectSplice "person-type" [minBound..maxBound]
```

- No dynamic data involved here, we should be able to reuse this code for both
  compiled and interpreted splices.
- For compiled splices just call a simple render function that converts
  `[Node]` into a ByteString.
- Simple type change...

# Types are both stringent and flexible

```haskell
HeistT n n [Node]             -- Interpreted (run time) splice
HeistT n IO (DList (Chunk n)) -- Compiled (load time) splice
```

- What if we had something like this

```haskell
data PersonType = Grunt
                | Manager
                | Executive deriving (Show, Enum, Bounded)
enumSelectSplice :: Show a
                 => Text     -- ^ Name attribute
                 -> [a]
                 -> HeistT n n [Node]
personSelect = enumSelectSplice "person-type" [minBound..maxBound]
```

- No dynamic data involved here, we should be able to reuse this code for both
  compiled and interpreted splices.
- For compiled splices just call a simple render function that converts
  `[Node]` into a ByteString.
- Simple type change...

```haskell
HeistT n m [Node]             -- Generic splice
```

# Restricting Times of Execution

- Binding a compiled splice at runtime has no effect because compiled splices
  are only processed once when the application loads.
- So we to make it impossible to bind new compile time splices at run time.
- We **do** want to be able to bind new compile time splices at compile time
- So we restrict the ability to modify compiledSpliceMap to the following
function

```haskell
withLocalSplices :: [(Text, Splice n)]
                 -> HeistT n IO a
                 -> HeistT n IO a
```

- This prevents the user from using this function in the wrong (run time)
context.
- Again, might be able to do it in Java with wrappers, but then you lose the
single HeistT abstraction.

# A Compiled Splice Example

```haskell
fooSplice :: MonadIO n => HeistT n IO (DList (Chunk n))
fooSplice = do
    -- :: HeistT n IO
    lift $ putStrLn "This executed at load time"
    C.yieldRuntimeText $ do
        -- :: RuntimeSplice n a
        lift $ putStrLn "This executed at run/render time"
        val <- lift getValueFromDatabase
        return $ pack $ show (val+1)
```

- Very clear distinction between load time and run time.
- If the distinction was not typed, the difference would be much less clear
  and easy to get wrong.

# Parting thoughts

- Later on, I discovered that I had to keep interpreted splices around because
  I needed their recursive properties.
    - Therefore, the unified HeistT became an even bigger win.
- We've seen that purity, strong types, and monads can:
    - Help you prevent data corruption in concurrent applications in pursuit
      of shorter critical sections and reduced lock contenion.
    - Prevent bugs that might arise from confusion between diferent phases of
      execution.
    - Enforce that compiled splices can only be bound at load time,
      eliminating the chance that you'll think they are bound when they're not.
    - Guide exploration of complicated problems
    - Expand your dictionary of concepts and ways of thinking about a problem.
- Languages influence the way we think about problems.
- "The thing you ought to be optimizing is *your* time." -- Douglas Crockford

# In case you're wondering about performance...

![](http://snapframework.com/media/img/heist-perf.png)

# Appendix

# Compiled splice formulations with run time data

> - "`a`" represents some piece of dynamic (runtime) data
> - Interpreted: `a -> [(Text, HeistT n n Template)]`
> - Compiled: `a -> [(Text, HeistT n IO (DList (Chunk n)))]`
> - `[(Text, a -> Builder)]`
>     - Useful, but not powerful enough for some cases
> - `[(Text, a -> HeistT n IO Builder)]`
>     - Useless
> - `[(Text, n a -> HeistT n IO Builder)]`
>     - Can't be generalized to work with [a]
> - `[(Text, IORef a -> HeistT n IO Builder)]`
>     - Not thread safe
> - `[(Text, Promise a -> HeistT n IO Builder)]`
>     - Final solution
> - Types and monads help us think about these things


