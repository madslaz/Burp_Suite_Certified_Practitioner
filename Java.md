### Method vs. Class
* **Class**: Think of class as a blueprint. For example, a Car class wouold be a blueprint that defines what a car is; it has properties (like color, numberOfDoors) and it has abilities (like startEngine(), accelerate(), brake()). The class itself is not a car; it's just the design for one.
* **Method**: Method is an action or a function that an object created from a class can perform. It's a block of code that runs when it's called. Using your Car blueprint, startEngine() would be a method.
* **Object**: Instance of a class and is the fundamental building block of object-oriented programming. Like a filled-out template (or "blueprint") with its own data and behaviors. 
* Example code: `Car myBlueCar = new Car();`
  * `Car` (the Type): This part declares the type of the variable. It's a promise to the compiler, saying, I am bout to create a variable that will hold an object made from the `Car` blueprint.
  * `myBlueCar` (the Name): This is the name of our variable. Once we create our specific car object, this is the name we'll use to refer to it.
  * `=` : This is the assignment operator. It means 'take whatever is on the right and put it into the variable on the left.'
  * `new Car()` (the Creation): This is the most magical part. The `new` keyword tells the Java Virtual Machine (JVM) to take the `Car` blueprint (the class) and actually create a new, unique object in memory. `Car()` is a special method called a constructor that runs to initialize the new object.
 
### Modifiers
* `public` is an access modifier that means the method can be called from any other class
* `final` means the behavior of this method cannot be changed (overriden) by any class that might inherit.
* `synchronized` is a keyword related to multithreading. It's like putting a "one person at a time" lock on the method, prventing potential errors if multiple parts of the program try to run it simultaneously.  
