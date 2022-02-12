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

Next we attempted some reconnaissance using Wireshark, cURL and nmap because the name of the challenge could refer to port knocking.

We used Wireshark to find the IP address of the target, then cURL it which returned 0.Finally an nmap scan reveals that port 443 is open.This told us that that the solution did not involve hidden ports and that further investigation of the provided javascript file was required. 

Review of the source code that was provided revealed that the flag was the first post after the instantiation of the database:

```
const db = new Database();
db.createNote({ data: process.env.FLAG });
```

This means that the flag was in the post with id=0. As a result, the goal became to discover a way to leak the token which is generated with a secretary key variable:

```
this.secret = `secret-${crypto.randomUUID}`
```

Using replit.com to dynamically modify and run the source code, we were able to observe odd behavior where the above line of code would output “secret-undefined”. At this point, the CTF ended but we continued to work on the problem.

### Solution: 

After further work, it turns out that the version of Node.Js dictates the behavior of crypto.randomUUID. Running it on on replit.com produced undefined output but running it on a newer version of Node.JS produces the following output:

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

We realized that this was the flaw of the code. The secret creation did not call crypto.randomUUID properly so the output was the help text. So all we really had to do was create our token with that secret and with the id=0 attribute and include the result as a token to the CTF’s URL:

```
> crypto.createHmac('sha256', secret).update('0').digest('hex')
'7bd881fe5b4dcc6cdafc3e86b4a70e07cfd12b821e09a81b976d451282f6e264'
```

Submit the URL and retrieve the flag:

https://knock-knock.mc.ax/note?id=0&token=7bd881fe5b4dcc6cdafc3e86b4a70e07cfd12b821e09a81b976d451282f6e264 

![](images/img8.png?raw=true)
