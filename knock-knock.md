# Knock Knock

Knock Knock is a JavaScript challenge where we are given a website that functions as a pastebin and the underlying js code. 

### Problem:

The challenge website presented as a simple pastebin. Users can give input that is then available at a link with the parameters ```id``` and ```token```.

![](images/img1.png?raw=true)

If the ```id``` or ```token``` parameters are changed, then the system tells you the token is invalid and it does not paste the input.

![](images/img2.png?raw=true)

### Attempts:

Initially, we approached this as a XSS problem. The system is vulnerable to script injection and does output a response to ```<script>alert(1)</script>```. 

![](images/img3.png?raw=true)

We attempted to inject script that would return the file ```/flag```.

```<script>function httpGet(theUrl)   { 	var xmlHttp = null;  	xmlHttp = new XMLHttpRequest(); 	xmlHttp.open( "GET", theUrl, false ); 	xmlHttp.send( null ); 	return xmlHttp.responseText;   }   alert(httpGet("https://knock-knock.mc.ax/flag"));</script>```

![](images/img4.png?raw=true)

Next we attempted some reconnaissance using Wireshark, cURL and nmap.

We use Wireshark to find the IP address of the target, then cURL it which returns 0.Finally an nmap scan reveals that port 443 is open.

This told us that that the solution did not require XSS alone and that further investigation of the provided js was required. 

Review revealed that the flag was likely at ```id=0```. Desired method is injection of javascript code by something like eval() in the input. We set up a request catcher at https://myreq.requestcatcher.com/. 

Can we leak ```this.secret``` with XSS so that we can generate the token for ```id=0```?

First, we found the IP address of the target to enable sniffing.

![](images/img5.png?raw=true)

Curling 72.21.91.29 returns 0. 

![](images/img6.png?raw=true)

Scanned with nmap to find hidden port: 443. 80 is the one we use to browse properly, as the Wireshark interception result shows.

![](images/img7.png?raw=true)

Goal should be leaking ```this.secret```. In the source code above, we see ```this.secret = secret-${crypto.randomUUID}```

Maybe we can reveal ```this.secret``` by posting it to a server by setting up a request catcher as:https://myreq.requestcatcher.com/ - no luck :(

At this point, the CTF ended and we were unable to find the flag in time.

### Solution: 

It turns out that:

```
> const secret = `secret-${crypto.randomUUID}`;
undefined
> secret
'secret-function randomUUID(options) {\n' +
  '  if (options !== undefined)\n' +
  "    validateObject(options, 'options');\n" +
  '  const {\n' +
  '    disableEntropyCache = false,\n' +
  '  } = options || {};\n' +
  '\n' +
  "  validateBoolean(disableEntropyCache, 'options.disableEntropyCache');\n" +
  '\n' +
  '  return disableEntropyCache ? getUnbufferedUUID() : getBufferedUUID();\n' +
  '}'
```

So all we really had to do was create our own “secret” with the id=0 attribute and include the result as a token to the CTF’s URL:

```
> crypto.createHmac('sha256', secret).update('0').digest('hex')
'7bd881fe5b4dcc6cdafc3e86b4a70e07cfd12b821e09a81b976d451282f6e264'
```

Submit the URL and retrieve the flag:

https://knock-knock.mc.ax/note?id=0&token=7bd881fe5b4dcc6cdafc3e86b4a70e07cfd12b821e09a81b976d451282f6e264 

![](images/img8.png?raw=true)
