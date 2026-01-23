## Java
### Method vs. Class
* **Class**: Think of class as a blueprint. For example, a Car class wouold be a blueprint that defines what a car is; it has properties (like color, numberOfDoors) and it has abilities (like startEngine(), accelerate(), brake()). The class itself is not a car; it's just the design for one.
* **Method**: Method is an action or a function that an object created from a class can perform. It's a block of code that runs when it's called. Using your Car blueprint, startEngine() would be a method.
* **Object**: Instance of a class and is the fundamental building block of object-oriented programming. Like a filled-out template (or "blueprint") with its own data and behaviors. 
* Example code: `Car myBlueCar = new Car();`
  * `Car` (the Type): This part declares the type of the variable. It's a promise to the compiler, saying, I am bout to create a variable that will hold an object made from the `Car` blueprint.
  * `myBlueCar` (the Name): This is the name of our variable. Once we create our specific car object, this is the name we'll use to refer to it.
  * `=` : This is the assignment operator. It means 'take whatever is on the right and put it into the variable on the left.'
  * `new Car()` (the Creation): This is the most magical part. The `new` keyword tells the Java Virtual Machine (JVM) to take the `Car` blueprint (the class) and actually create a new, unique object in memory. `Car()` is a special method called a constructor that runs to initialize the new object.
* A method, such as `L()` being called on a class name means it must be a **static method** - regular methods can be called on objects but only static methods can be called on the class name directly. 

### Inheritance 
* The `super` keyword in Java is a special reference used within a child class to refer to its immediate parent class. Thinking back to our discussion about inheritances with the class MountainBike and the class Bicycle, the super keyword has two main uses:
 * Calling the parent's constructor. If a child class has its own constructor, it often needs to call the parent class's constructor to properly initialize the inherited parts. You do this with `super()`. For example, in a MountainBike constructor, you might have super`()` to call the Bicycle constructor.
 * If a child class has overriden a method from its parent (meaning it provided its own version of that method), you can still explicitly call the parent's version of that method using super.methodName(). Example shown below:
```
class Bicycle {
  public void brake() { System.out.println("Bicycle slowing.");}
}
class MountainBike extends Bicycle {
  @Override
  public void brake() {
    super.brake(); // Call's Bicycle brake method
    System.out.println("Mountain bike disc brakes engage.");
  }
}
```
 
### Modifiers
* `public` is an access modifier that means the method can be called from any other class
* `final` means the behavior of this method cannot be changed (overriden) by any class that might inherit.
* `synchronized` is a keyword related to multithreading. It's like putting a "one person at a time" lock on the method, prventing potential errors if multiple parts of the program try to run it simultaneously.  

### The Dot Operator
* The dot (.) operator is used to access members (variables or methods) of an object or a class.
 * When you see a method called on a variable that holds an object, like `getClass().getName()`, it means that `getName()` is a method that belongs to an object returned by `getClass()`.
 * When you see a method called directly on a ClassName, it means the method is a static method (for example `ClassName.Method()`). Static methods belong to the class itself, not to a specific object created from that class, You don't need to create an object (`new ClassName()`) to call a static method. Think of it like a shared utility function that anyone can use directly through the class's name.
 * Let's take a look at a heavily obfuscated method chain: `C7V1.LB(getClass().getName());`. `C7V1` is an obfuscated class name, while `.LB(...)` is a static method that belongs to the `C7V1` class. `.getClass().getName()` is the argument being passed to `C7V1.LB()`. Let's break it down: `getClass()` is a method that every Java object inherits from the universal `Object` class. When you call `getClass() ` (from inside `onCreate()`, it returns a `Class` object that represents the class of the current object. `.getName()` is a method called on the `Class` object (the one returned by `getClass()`). It returns the fully qualified name of that class as a String.
    * So overall, it's calling a static method `LB` on the `C7V1` class, and it's passing the result of calling `getName()` on the object returned by `getClass()` as an argument. This is a common pattern for logging or tracking which class's lifecycle method is currently executing.
 * You can only use the (.) dot operator to call a method on an object.
  
### Catch-all
* **Singleton**: Singleton is a design pattern in programming that ensures a class has only one instance (one object) throughout the entire application's lifetime. It provides a global point of access to that single instance.
 * Often used for things like loggers, configuration managers, and database connection pools. 
