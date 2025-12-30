### Introduction to Serialization
* **Serialization** is the process of converting complex data structures, such as objects and their fileds, into a "flatter" format that can be sent and received as a sequential stream of bytes.
* Serializing data makes it much simpler to write complex data to inter-process memory, a file, or database, and send complex data, for example, over a network, between different components of an application, or in an API call.
* Crucially, when serializing an object, its state is also persisted. In other words, the object's attributes are preserved, along with their assigned values.
* **Deserialization** is the process of restoring this byte stream to a fully functional replica of the original object, in the exact state as when it was serialized. The website's logic can then interact with this deserialized object, just like it would with any other object.
* Many languages offer native support for serialization; exactly how objects are serialized depends on the language. Some languages serialize into binary formats, whereas others use different string formats, with varying degrees of human readability. 
