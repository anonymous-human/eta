# Basics

## Prerequisites

You must have a basic understanding of sequenceables/monads to understand the rest of the section. Please check out the chapter on **Sequenceable** in [Tour of Eta](https://tour.eta-lang.org) for better understanding.

## Quick Start

When interfacing with Java, you should import the `Java` module from the standard library:



```eta
import Java
```

This will import the `Java` sequenceable and related helper functions for working inside the sequenceable.



Consider the following Java code:



```java
package eta.example;

public class Counter {

  private int counter;

  public int publicCounter;

  public static final int COUNTER_MAX = 1000;

  public static int numCounters;

  public Counter() {}

  public Counter(int initial) {
    this.counter = initial;
  }

  public void increment() {
    this.counter = Math.min(this.counter + 1, COUNTER_MAX);
    this.publicCounter = counter;
  }

  public int get() {
    return counter;
  }

  public void set(int value) {
    this.counter = Math.min(value, COUNTER_MAX);
    this.publicCounter = counter;
  }
}
```

A Java method is simply a function that takes an object as an implicit argument bound to the `this` variable. Implicit contexts such as these can be represented as a monad, a state monad to be specific. A state monad threads state through each monadic action so the state is being passed around internally even though it’s not visible in the code.



This correspondence is the basis for the built-in `Java` sequenceable in Eta.



The above example can be imported as follows:



```eta
data Counter = Counter @eta.example.Counter
  deriving Class

foreign import java unsafe "@new" newCounter :: Java a Counter
foreign import java unsafe "@new" newCounterWith :: Int -> Java a Counter
foreign import java unsafe increment :: Java Counter ()
foreign import java unsafe get :: Java Counter Int
foreign import java unsafe set :: Int -> Java Counter ()
foreign import java unsafe "@static @field eta.example.Counter.COUNTER_MAX"
  cOUNTER_MAX :: Java a Int
foreign import java unsafe "@field publicCounter" getPublicCounter
  :: Java Counter Int
foreign import java unsafe "@field publicCounter" setPublicCounter
  :: Int -> Java Counter ()
foreign import java unsafe "@static @field numCounters" getNumCounters
  :: Java a Int
foreign import java unsafe "@static @field numCounters" setNumCounters
  :: Int -> Java a ()
```

## Defining a Java Wrapper Type

When working with the FFI, you need a way to refer to a given Java class inside of Eta. This is done through Java Wrapper Types (JWTs).



### General Syntax

```eta
data X = X @[class-name]
  deriving Class
```

- `[class-name]` should be the fully qualified Java class name and `X` should be the Eta name you would use to refer to the corresponding Java class in foreign imports. Note that `[class-name]` can also be converted to an array type by appending `[]`.

- The `Class` typeclass is a built-in typeclass that is a marker for a JWT. **Make sure all your JWTs derive a Class instance**.


## The Java Sequenceable

As mentioned before, the `Java` monad is used to contain the implicit `this` context. It can be effectively thought of as a state monad with a given Java object as the state.



```eta
newtype Java c a = Java {- Internal definition -}
```



As can be seen from the above definition, the `Java` sequenceable has two type parameters `c` and `a`. The `c` parameter should be some JWT and the `a` parameter is the return type of the sequenceable.

## Java Foreign Import Declarations

Foreign import declarations are used to import a Java method as an Eta monadic action, typically in the Java monad.



### General Syntax

```eta
foreign import java [safety] "[import-string]" [eta-identifier]
  :: [arg-type-1] -> [arg-type-2] -> .. -> [return-type]
```

1. `[safety]` - `safe` or `unsafe`.
    - `unsafe` is the option you would typically select. In this case, the java method identified in the `[import-string]` will be run directly. This can be dangerous if the function can block in which case it will block the Eta RTS and reduce efficiency.
    - `safe` is the option you would select for functions that you would expect to block for some time, so they will be safely run in another thread to prevent the call from blocking the Eta RTS. This option must also be used when importing a Java method that eventually calls an exported Eta function.

2. `[import-string]` can take the following forms:
    - `[java-method-name]`: Binds to an instance method. `[java-method-name]` should be an unqualified Java instance method name.
    - `@static [java-method-name]`: Binds to a static method. `[java-method-name]` should be a fully qualified Java static method name.
    - `@new`: Binds to a constructor. The class to construct will be determined by the return type of the declaration.
    - `@field [java-field-name]`: Binds to a getter or setter of an instance field, determined by the type signature. `[java-field-name]` should be an unqualified Java instance field name.
    - `@static @field [java-field-name]`: Binds to a getter or setter of a field, determined by the type signature. `[java-field-name]` should be a fully qualified Java static field name.
    - `@interface [java-interface-method]`: Binds to an interface method, determined by the type signature. `[java-interface-name]` should be a unqualified Java interface method name.
    - `@wrapper [java-interface-method]`: Used for generating an Eta function that will generate an interface implementation, determined by the type signature. `[java-interface-name]` should be a unqualified Java interface method name. See [Working With Java Interfaces](/docs/eta-concepts/java-interop/java-generics#working-with-java-interfaces) for more information.
    - `@wrapper @abstract [java-abstract-method]`: Used for generating an Eta function that will generate an abstract class implementation, determined by the type signature. `[java-method]` should be a unqualified Java abstract method name. See [Working With Java Interfaces](/docs/eta-concepts/java-interop/java-generics#working-with-java-interfaces) for more information.
    - Not present: If you do not specify an import string, it will be taken as an instance method import and the `[java-method-name]` is taken to be the same as `[eta-identifier]`.

3. `[eta-identifier]` should be a valid Eta identifier that will be used for calling the corresponding Java method inside of Eta code.

4. `[argTypeN]` should be a marshallable Eta type. See [Marshalling Between Java and Eta Types](/docs/eta-concepts/java-interop/jwts#marshalling-between-java-and-eta-types).
5. `[returnType]` can be of three forms:
    - `Java [jwt] [return-type]`: This is the form that is used typically and is always safe to use. `[jwt]` should be the JWT for the class which the declaration pertains. If the declaration has a `@static` annotation, this can be left free with a type variable instead of a concrete type. `[return-type]` should be a marshallable Eta type.
    - `IO [return-type]`: This form is also safe and can be used for convenience. Note that if the import string does not have a `@static` annotation, you must supply the relevant JWT as the first argument (`[argType1]`). `[return-type]` should be a marshallable Eta type.
    - `[return-type]`: This form has no monadic context and should only be used for immutable Java objects whose methods do not perform any side effects. Note that if the declaration does not have a `@static` annotation, you must supply the relevant JWT as the first argument (`[argType1]`). `[return-type]` should be a marshallable Eta type.

## Exporting Eta Methods

Just as you can import Java methods into Eta, you can also export Eta functions into Java.



### General Syntax


```eta
foreign export java "[export-string]" [eta-identifier]
  :: [arg-type-1] -> [arg-type-2] -> .. -> Java [export-jwt] [return-type]
```

1. `[export-string]` should consist of `@static` followed by a fully qualified Java class name with the name of the static method appended to it with a dot. This is the method name that the exported function should be referred to in the Java world. (e.g. `"@static com.org.SomeClass.someMethodName"`)

2. `[eta-identifier]` should be a valid Eta identifier for an existing Eta function that is the target of the export.

3. `[arg-type-n]` should be a marshallable Eta type.

4. `[export-jwt]` should be a JWT that refers to the class name of the exported class.

5. `[return-type]` should be a marshallable Eta type which is the result of the Eta function.

## Next Section

We will now proceed with Java Wrapper Types in detail.
