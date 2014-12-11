# Language Features

## Primitives

Skew has a few primitive types: void, bool, int, float, double, and string. Primitives are immutable and non-nullable since that maps the cleanest to all target platforms. The range of the int type is left unspecified to improve code readability in generated code. There is no unsigned integer type since it leads to performance penalties for some targets. Strings define the + and += operators which will automatically call toString() on any non-string type.

```
bool b = false;
int i = 0;
float f = 0.0f;
double d = 0.0;
string s = "";
```

Skew also has list literals that use the built-in List type.

```
List<int> list = [1, 2, 3];
```

Variables can be implicitly typed using the var keyword for convenience, which takes the type from the assigned value. The above code could also be written like this:

```
var list = [1, 2, 3];
```

## Objects

Skew's object model is similar to Java. A class can inherit from at most one base class and can implement any number of interfaces. This is a lowest common denominator between desirable target platforms but is still very flexible.

```
interface Encoder {
  virtual void encodeInt(int value);
  virtual void encodeString(string value);
}

class SimpleEncoder : Encoder {
  string data = "";
  override void encodeInt(int value) { data += value + ","; }
  override void encodeString(string value) { data += value + ","; }
}

class User {
  int id;
  string name;

  void encode(Encoder encoder) {
    encoder.encodeInt(id);
    encoder.encodeString(name);
  }
}
```

## Constructors

A constructor is a function called new with no return type. C++-style initializer lists can be used to initialize the base class and member variables before the body of the constructor executes. Initializer lists are required for final member variables.

```
class Node {
  int weight;
  List<Node> children;

  new(List<Node> children) : weight = 0, children = children {}
}

class NamedNode : Node {
  final string name;

  new(List<Node> children, string name) : super(children), name = name {}
}
```

When not explicitly declared, constructors are automatically generated with one argument for each member variable without a default value in declaration order. This greatly simplifies defining objects in many situations. For example, the above code can be simplified to this since both constructors can be generated automatically:

```
class Node {
  int weight = 0;
  List<Node> children;
}

class NamedNode : Node {
  final string name;
}
```

Unlike Java, the new keyword is not required to construct an instance:

```
Node createTree() {
  return Node([
    NamedNode([], "B"),
    NamedNode([], "C"),
  ]);
}
```

## Methods

Like C++ and C#, methods are not overridable by default. Overridable methods in a base class must be explicitly declared virtual. Overriding methods must be declared using the override keyword, which both makes it more obvious which methods are overriding something and causes compile errors when the original method is removed from the base class. There is no explicit abstract keyword but methods can be made abstract by marking them virtual and omitting the function body. Classes with at least one abstract method are considered abstract and cannot be constructed.

```
class Base {
  virtual void foo();
  static void bar() {}
}

class Derived : Base {
  override void foo() {}
}
```

Functions don't have to be declared inside a class and can exist on their own like in C++.

## Enums

An enum is a compile-time integer constant. Enums automatically convert to ints but ints don't automatically convert to enums. Each enum type has a toString() method that is automatically generated when referenced, so using operator + with a string automatically substitutes the name of the enum. The special enum flags declaration generates values suitable for bitwise masks and includes type-safe bitwise operators.

```
enum EntryType {
  FILE,      // 0
  DIRECTORY, // 1
  SOFT_LINK, // 2
  HARD_LINK, // 3
}

enum flags EntryFlags {
  HIDDEN,  // 1
  READ,    // 2
  WRITE,   // 4
  EXECUTE, // 8
  PERMISSION_MASK = READ | WRITE | EXECUTE,
}

bool isReadWrite(EntryFlags flags) {
  return (flags & EntryFlags.PERMISSION_MASK) == (EntryFlags.READ | EntryFlags.WRITE);
}
```

Due to type inference, the name of the enum can usually be omitted for better readability. The enum names are still prevented from leaking into the global namespace, unlike in C/C++. The function above is better written like this:

```
bool isReadWrite(EntryFlags flags) {
  return (flags & .PERMISSION_MASK) == (.READ | .WRITE);
}
```

## Namespaces

Namespaces introduce a named scope for organization. Multiple namespace blocks with the same name will all be merged together. Using statements inject all symbols from that namespace into their local scope.

```
namespace ui.widgets {
  class Label {
    void setText(string text) {}
  }
}

namespace ui.helpers {
  using ui.widgets;

  Label labelWithText(string text) {
    var label = Label();
    label.setText(text);
    return label;
  }
}
```

## Extension Blocks

Extension blocks are a cross between partial classes and extension methods from C#.

They are like partial classes because their contents are copied into the body of the type they extend. In addition to providing a flexible way to organize instance methods independent from their types, they give code a way to implement an interface on a class defined elsewhere:

```
class View {
  virtual void scrollTo(int x, int y);
}

...

interface ScrollToUser {
  virtual void scrollToUser(User user);
}

in View : ScrollToUser {
  override void scrollToUser(User user) {
    var position = user.locationInView(this);
    scrollTo(position.x, position.y);
  }
}
```

They are like extension methods because they can wrap imported types and enum types. For example:

```
in int {
  int squared() {
    return this * this;
  }
}
```

## Generics

Generics are available for classes, interfaces, and functions. They are implemented using type erasure to ensure a compact and readable implementation. They are more than capable for simple type substitution but lack certain advanced features like covariant and contravariant conversions.

```
interface HashCode<K> {
  virtual int hashCode(K key);
  virtual bool checkEquality(K left, K right);
}

class HashMap<K, V, HC is HashCode<K>> {
  HC hashCode;
  virtual V get(K key);
  virtual void set(K key, V value);
}

void bulkSet<K, V, HC is HashCode<K>>(HashMap<K, V, HC> map, K key, List<V> values) {
  for (var i = 0; i < values.size(); i++) {
    map.set(key, values[i]);
  }
}
```

## Asserts

Asserts are currently implemented using a special assert statment. They are compiled away into nothing when turned off, so make sure the condition has no side effects.

```
void signup(DB db, User user) {
  assert !db.users.has(user.id);
  db.users.add(user);
  assert db.users.has(user.id);
}
```

## Operator Overloading

This is done using method annotations that tell the compiler to replace the operator with call to that method. Operator overloading can make code more readable in many cases, but can also dramatically reduce readability when used incorrectly. Use with good judgement.

```
enum Axis { X, Y }

class Vector {
  double x, y;

  @OperatorSubtract
  Vector subtract(Vector p) {
    return Vector(x - p.x, y - p.y);
  }

  @OperatorGet
  double get(Axis a) {
    return a == .X ? x : y;
  }

  @OperatorSet
  void set(Axis a, double v) {
    if (a == .X) x = v;
    else y = v;
  }
}

class Player {
  Vector position;

  Vector deltaTo(Player other) {
    return other.position - position;
  }

  void moveAlongAxis(Axis axis, double delta) {
    position[axis] = position[axis] + delta;
  }
}
```