---
layout: post
title: "inCTF No-Shake solution"
date: 2020-07-30
---

The challenge' info card gives us a link to the website.


![infocard](/assets/posts/inCTF-NoShake/images/p1.png)


#### Website's first page:


![firstPage](/assets/posts/inCTF-NoShake/images/p2.png)


Let's send some random input to the admin and intercept the request:

#### Our request:


![adminReq](/assets/posts/inCTF-NoShake/images/p3.png)


#### Response on browser:


![adminResp](/assets/posts/inCTF-NoShake/images/p4.png)


As you can see, our request contains some weird parameter calls `Cmd`. 

The `Cmd` parameter is the SMTP command `STARTTLS` which tells the server that the client wants to secure the current connection.

The numbers 250,220 on the admin' response are SMTP server response codes:

`220`- The server is ready.
`250`- Request mail action okay completed.

I tried to send the admin another SMTP command but he didn't like it.

Now I want to cause some error on the SMTP command validation, let's pass the `Cmd` variable as an array.

We got the following PHP error message:


![phpError](/assets/posts/inCTF-NoShake/images/p5.png)


We can assume from the error message that the admin validates our SMTP command by its length. 

After trying to put another 8 characters string (like the length of STARTTLS) the admin' message changed but nothing special.

Now let's add to our 8 characters string some weird character, for example ';'

The result was the key `sT4yH0M3`.

We received some key, cool.

Note: you can insert any special character to the 8 characters `Cmd` to receive the key. 


On each response from the admin, we could notice the HTML comment that says `go to final.php`, let's do it.

The first page on final.php asks for a key query parameter so let's pass the key that we just received from the admin.


The result after sending the key to final.php is a little piece of the source code:


![firstSource](/assets/posts/inCTF-NoShake/images/p6.png)


Now we want to cause the if condition to become `true` so we will receive some hidden data from the server.

For the comparison between our param `abc` and the hidden variable `flag1`, the server uses strcasecmp. 

We can send `abc` as an array so `strcasecmp` will return `NULL`, there is an exclamation mark before this function, so !NULL in PHP is equal to 1 which means `true`.

Next source code piece:


![secondSource](/assets/posts/inCTF-NoShake/images/p7.png)


The server asks for another query parameter - `def`. The server unserializes this parameter and overrides the value of the `flag` property. Then it compares between `flag` property and `xyz` property from our unserialized object.

Our goal is to cause the condition to become `true`, so another hidden data will be sent to us.

For that purpose, we should pass the `xyz` property as a reference to flag property, so when the value of `flag` will be changed, `xyz` will be a reference to the same value.

Payload:

`O:8:"stdClass":2:{s:4:"flag";s:6:"blabla";s:3:"xyz";R:2;}`

Now another piece of source code returned, but it was long and unreadable, so I copied it to my text editor and added indentations, the code:


![thirdSource](/assets/posts/inCTF-NoShake/images/p8.png)


The next required query parameter is `ghi`, if it's set, the server creates two classes - Dogs and Cats.

Dogs class has implemented the magic method `__wakeup()` so it will be called automatically when our parameter will be unserialized.

The Dogs' magic method calls to `close` method on the `obj` property. 

You can see that the `close()` method is occurring on Cats class and it executes the cmd property on the server system.

So, we want to pass to the server serialized object which is an instance of Dogs class with the property `obj` set to an instance of Cats class with the property cmd set to our command. Mine gonna be `cat final.php`.

#### RCE payload:


![rcePayload](/assets/posts/inCTF-NoShake/images/p9.png)


#### Receiving the flag:


![flag](/assets/posts/inCTF-NoShake/images/p10.png)
