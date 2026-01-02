### Introduction to Serialization
* **Serialization** is the process of converting complex data structures, such as objects and their fileds, into a "flatter" format that can be sent and received as a sequential stream of bytes.
* Serializing data makes it much simpler to write complex data to inter-process memory, a file, or database, and send complex data, for example, over a network, between different components of an application, or in an API call.
* Crucially, when serializing an object, its state is also persisted. In other words, the object's attributes are preserved, along with their assigned values.
* **Deserialization** is the process of restoring this byte stream to a fully functional replica of the original object, in the exact state as when it was serialized. The website's logic can then interact with this deserialized object, just like it would with any other object.
* Many languages offer native support for serialization; exactly how objects are serialized depends on the language. Some languages serialize into binary formats, whereas others use different string formats, with varying degrees of human readability.
  * Note that all of the original object's attributes are stored in the serialized data stream, including any private fields. To prevent a field from being serialized, it must be explicitly marked as "transient" in the class declaration.
  * Be aware that when working with different programming languages, serialization may be referred to as marshalling (Ruby) or pickling (Python). These terms are synonymous with "serialization" in this context.

#### Insecure Deserialization
* Insecure deserialization is when user-controllable data is deserialized by a website. This potentially enables an attacker to manipulate serialized objects in order to pass harmful data into the application code.
* It is even possible to replace a serialized object with an object of an entirely different class. Alarmingly, objects of any class that is available to the website will be deserialized and instantiated, regardless of which class was expected. For this readon, insecure deserialization is sometimes known as an "object injection" vulnerability.
* An object of an unexpected class might cause an exception. By this time, however, the damage may already be done. Many deserialization-based attacks are completed **before** deserialization is finished. This means that the deserialization process itself can initiate an attack, even if the website's own functionality does not directly interact with the malicious object. For this reason, websites whose logic is based on strongly typed languages can also be vulnerable to these techniques.
  * Explaining this more: Deserialization is like taking the flat-pack box of an IKEA chair and assembling it into a chair. Strong typing makes sure that once the chair is built, it is actually a "chair" and not a "bear". When an attacker sends you a box labeled "chair", you open it, and screw leg A into Base B, unbeknownst to you that you are triggering the bomb hidden in leg A. The bomb explodes during assembly - you never finish building the chair, so you never get to the final check that the final product is a valid "chair" - the type checking.
* Insecure deserialization arises because there is a general lack of understanding of how dangerous deserializing user-controllable data can be. Ideally, user input should never be deserialized at all.
* However, sometimes website owners think they are safe because they implement some form of additional check on the deserialized data. This approach is often ineffective because it is virtually impossible to implement validation or sanitization to account for every eventuality. These checks are fundamentally flawed as they rely on checking the data after it has been deserialized, which in many cases will be too late to prevent the attack.
* Vulnerabilities may also arise because deserialized objects are often assumed to be trustworthy. Especially when using languages with a binary serialization format, developers might think that users cannot read or manipulate the data effectively. However, while it may require more effort, it is just as possible for an attacker to exploit binary serialized objects as it is to exploit string-based formats. 
* Deserialization-based attacks are also made possible due to the number of dependencies that exist in modern websites. A typical site might implement many different libraries, which each have their own dependencies as well. This creates a massive pool of classes and methods that is difficult to manage securely. As an attacker can create instances of any of these classes, it is hard to predict which methods can be invoked on the malicious data. This is especially truth if an attacker is able to chain together a long series of unexpected method invocations, passing data into a sink that is completely unrelated to the initial source. It is, therefore, almost impossible to anticipate the flow of malicious data and plug every potential hole.
  * We are talking about **gadget chains** - when attackers targets your website with deserialization, they aren't usually uploading their own malicious code. They are using your code (and the code of the libraries you installed) against you. An attacker chains together a long series of classes (think of them like parts!) to do something small but useful. For example, Gadget A (in Library X) `When I am created, I write data to a file`, Gadget B (in Library Y) `When I am run, I execute whatever is in this file`; the attacker creates a serialized object that contains Gadget A inside of Gadget B, the website deserializes the object, Gadget A runs (because of a magic method) and writes a script to disk, Gadget B runs (triggered by Gadget A) and executes that script.
    * Magic method examples: `readObject()` for Java, `__wakeup()` and `__destruct()` for PHP (wakeup runs when `unserialize()` is called and destruct runs when the object is no longer needed and deleted from memory), `__reduce__` for Python (tells pickle library how to recreate the object)
* In short, it can be argued that it is not possible to securely deserialize untrusted input.

### How to Identify Insecure Deserialization 
#### PHP Serialization Format
* PHP uses a mostly human-readable string format, with letters representing the data type and numbers representing the length of each entry. For example, consider a `User` object with the attributes:
```
$user->name = "carlos";
$user->isLoggedIn = true;
```
* When serialized, this object may look something like this: `0:4:"User":2:{s:4:"name":s:6:"carlos";s:10:"isLoggedIn":b:1;}`
  * `0:4:"User"` - an object with the 4-character class name `"User"`
  * `2` - the object has 2 attributes
  * `s:4:"name"` - the key of the first attribute is the 4-chracter string `"name"`
  * `s:6:"carlos"` - the value of the first attribute is the 6-character string "carlos"
  * `s:10:"isLoggedIn"` - the key of the second attribute is the 10-character string "isLoggedIn"
  * `b:1` - the value of the second attribute is the boolean value `true`
* The native methods for PHP serialization are `serialize()` and `unserialize()`. If you have source code access, you should start by looking for `unserialize()` anywhere in the code and investigating further. 

#### Java Serialization Format
- Some languages, such as Java, use binary serialization formats. This is more difficult to read, but you can still identify serialized data if you know how to recognize a few tell-tale signs. For example, serialized Java objects always begins with the same bytes, which are encoded as `ac ed` in hexadecimal and `rO0` in Base64.
- Any class that implements the interface `java.io.Serializable` can be serialized and deserialized. If you have source code access, take note of any code that uses the `readObject()` method, which is used to read and deserialize data from an `InputStream`.

### Manipulating Serialized Objects
- Exploiting some deserialization vulnerabilities can be easy as changing an attribute in a serialized object. As the object state is persisted, you can study the serialized data to identify and edit interesting attribute values. You can then pass the malicious object into the website via its deserialization process. This is the initial step for a basic deserialization exploit.
- Broadly speaking, there are two approaches you can take when manipulating serialized objects. You can either edit the object directly in its byte stream form, or you can write a short script in the corresponding language to create and serialize the new object yourself. The latter approach is often easier when working with binary serialization formats. 

#### Modifying Object Attributes
- When tampering with the data, as long as the attacker preserves a valid serialized object, the deserialization process will create a server-side object with the modified attribute values. As a simple example, consider a website that uses a serialized `User` object to store data about a user's session in a cookie. If an attacker spotted the serialized object in an HTTP request, they might decode it to find the following byte stream: `0:4:"User":2:{s:8:"username";s:6:"carlos";s:7:"isAdmin";b:0;}`.
- The `isAdmin` attribute is an obvious point of interest. An attacker could simply change the boolean value of the attribute to `1` (true), re-encode the object, and overwrite their current cookie with this modified value. In isolation this has no effect; however, let's say the website uses this cookie to check whether the current user has access to certain admin functionality:
```
$user = unserialize($_COOKIE);
if ($user->isAdmin === true) {
// allow access to admin interface
}
```
- The vulnerable code would instantiate a `User` object based on the data from the cookie, including the attacker-modified `isAdmin` attribute. At no point is the authenticity of the serialized object checked. This data is then passed into the conditional statement and, in this case, would allow for an easy privilege escalation. This simple scenario is not common in the wild; however, editing an attribute value in this way demonstrates the first step towards accessing the massive amount of attack-surface exposed by insecure deserialization.

#### Modifying Data Types
* We've reviewed how we can modify attribute values in serialized objects, but it's also possible to supply unexpected data types. PHP-based logic is particularly vulnerable to this kind of manipulation due to the behavior of its loose comparison operator (`==`) when comparing different data types.
  * For example, if you perform a loose comparison between an integer and a string, PHP will attempt to convert the string to an integer, meaning that `5=="5"` evaluates to `true`. Likewise, on PHP 7.x and earlier, the comparison `0=="Example string"` evaluates to `true` because PHP treats the entire string as the integer `0`.
  * Consider a case where this loose comparison operator is used in conjunction with user-controllable data from a deserialized object. This could potentially result in dangerous logic flaws.
```
$login = unserialized($_COOKIE)
if ($login['password'] == $password) {
// log in successfully
}
```
 * Let's say an attacker modified the password attribute so that it contained the integer `0` instead of the expected string. As long as the stored password does not start with a number, the condition would always return `true`, enabling an authentication bypass. Note that this is only possible because deserialization preserves the data type. If the code fetched the password from the request directly, the `0` would be converted to a string and the condition would evaluate to `false`.
   * Note: in PHP 8 and later, the `0=="Example string"` comparison evaluates to `false` because strings are no longer implicitly converted to 0 during comparisons. As a result, this exploit is not possible on these versions of PHP. The behavior when comparing an alphanumeric string that starts with a number remains the same in PHP 8. As such, `5=="5 of something"` is still treats as `5==5`.
* Be aware that when modifying data types in any serialized object format, it is important to remember to update any labels and length indicators in the serialized data too. Otherwise, the serialized object will be corrupted and will not be deserialized.
* When working with binary formats, it is recommended to use `Hackvertor` extension, available in BApp. You can modify the serialized data as a string, and it will automatically update the binary data, adjusting the offsets accordingly. This can save you a lot of manual effort. 

#### Lab: Modifying Serialized Objects
* Decoding the session token as Base64 results in `O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:0;}`. Flipping 0 to 1 and then attaching the session token to the request to `GET /admin/delete?username=carlos` to delete Carlos.
* I was able to identify the admin panel by refreshing my account page with the altered session token, and the option appeared. I could use match and replace to make this easier.

#### Lab: Modifying Serialized Data Types
- Identified session-based serialization: `O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"hfamki0prqs03xdfocg7x81tbg93o2ac";}` as a cookie. I also noticed that when you get /my-account, the parameter accepts the parameter `wiener` - this ended up being a rabbit hole. Let's get back to the basics, and by that, I mean the comparison vulnerabilities we just chatted about in earlier versions of PHP.
- I managed to cause a fatal PHP error by tooling around with integers, let's take a look. Reminder, -> in PHP is the object operator, it allows you to access properties that belong to a specific object (in this case, $user is the object, and username is what we want). 
```
PHP Fatal error:  Uncaught Exception: (DEBUG: $access_tokens[$user-&gt;username] = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx, $user->;access_token = 1, $access_tokens = [xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx,xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx,xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx]) Invalid access token for user carlos in /var/www/index.php:8
```
- My first attempt at exploiting this failed because I attempted an integer substitution of the access_token `O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:1;}` but this wasn't working - why? Well, it's probably because the access token begins with an integer! So `i:1` failed for me because 1 == DOES NOT whatever integer the access token starts with.
- Okay, well since we don't want to play guess the integer, how about we use booleans? Remember `true` equals any string that is not empty so `true == "abc123"` or `true == "123abc"`. Let's do `O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}`.
