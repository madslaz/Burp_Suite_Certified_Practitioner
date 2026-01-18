#### Lab: Reflected XSS into a JavaScript String with Single Quote and Backslash Escape
- I input 'test' in the Search function, and I noticed it was reflected: "0 search results for 'test'" - so let's take a look at the source code. 
```javascript
var searchTerms = 'test';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
                    
```

- I was overthinking it in the end, too focused on the backslash and quote escaping ... the payload that worked was literally just `</script><script>alert(1)</script>`. Noticed when I could break out with </script>. 

#### Lab: Reflected XSS in Canonical Link Tag
- Reflects user input, escapes angle brackets. Solve the lab, perform XSS via alert function. Assume the simulated user will perform the folowing:
    - ALT+SHIFT+X
    - CTRL+ALT+X
    - ALT+X

* Only possible in Chrome. Canonical link element is an HTML element that helps webmasters prevent duplicate content issues in search engine optimization by specifying "canonical" or "preferred" version of a web page. 
- Originally, I was thinking something like `GET /post?postId=4&><script>alert(1)</script>`, but I forgot the hint mentions it escapes angle brackets. Next payload ... what about that mention of keyboard use?
- `<xss onkeypress="alert(1)" contenteditable style=display:block>test</xss>`
