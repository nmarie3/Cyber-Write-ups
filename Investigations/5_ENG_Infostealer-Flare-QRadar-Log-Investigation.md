# Infostealer: Flare & QRadar Log Investigation - 20260807

Note: This investigation has been updated on 2026/08/09 below the original investigation. Spoiler: The original download source was found.

A little bit of background on this investigation: There was a device infected with infostealer malware. We became aware of this because their stolen data was found on the dark web (thanks to Flare we confirmed this). <br>
This investigation itself was long as I tried to work with what little I had, but I thought it was important enough to document my process as I was able to use Flare and dive deep into QRadar network logs.<br>

So let's start from the beginning.<br>
We were informed of a user's personal details leaked on the darkweb. Although the forensic's department had collected the infected devices and would be handling the case, the SOC team was asked to look into suspicious traffic revolving a certain URL on that network.<br>
Thanks to that request, this incident immediately grabbed my attention, and so I decided to dig deeper and do my own investigation.

Note: Because this investigation was done of my own accord, I was not shared any extra information from the forensics department and only had access to a our Flare account and QRadar network logs to work with.<br>
Those results are as follows.

## The Investigation

First lets take a look at what the Flare page looks like.<br>
From the Global Search bar, we specified `features.domain:` to look for the domain of the user's work email address.

![alt text](images/Infostealer-darkweb/flare.png)

In the above image, on the left side, we can see that this specific domain name was found in numerious stealer logs. We can even see there's Salesforce login credentials. Not good.<br>
On the right side, we're shown all the data that was stolen. For obvious reasons I have this redacted.<br>
But at the top there's the estimated infection date, along with the file path of the malware. Since these logs appeared to have been stolen in June, I focused my attention only on this month. Unfortunately for my solo investigation it was given the generic name of `Installer_x64.exe`, so it'll be difficult for me to pinpoint what file it is without a hash. But that's okay. Let's work with what we got. Since the right column is long, we can scroll down and see more info about our infected user.

![alt text](images/Infostealer-darkweb/dark-files.png)

After scrolling down a bit, I noticed there were files of everything that was stolen from the device. Two of these caught my attention. "Files" and "History".<br>
"Files" contained all the software and other files that were on the PC at the time of the infection (from what I could tell it only saved .png and .txt files though of the software listed). "History" on the otherhand, were logs of the user's internet search history. I did open both Chrome and Edge files. Chrome had very few logs, so I focused my attention on the Edge history.

At this point in time, I was more interested in finding out how the device got infected, what was the cause.<br>
So I did a search for .exe files. And I did find something VERY interesting.

![alt text](images/Infostealer-darkweb/user-search.png)

As you can see highlighted is a software called Tools Unlock v6.9.<br>
And right under that are microsoft links related to a trojan. Clearly this screams malicious enough.<br>
I've also highlighted `go.microsoft.com/fwlink`. This caught my attention because it's a redirect/forward link. And I kinda had a hunch it might be a "Learn More" Microsoft Defender popup.<br>
So I tried to see what I could find about that particular software in general. I searched on Google to see if the SHA256 hash was documented anywhere. I did find a hash, and then I searched it up on VirusTotal. That result is below.

![alt text](images/Infostealer-darkweb/toolsunlock.png)

50/71 security venders have flagged this file as malicious.

Scrolling down to the Contacted IP addresses section, I noticed one ip was in red. When I took a look at it, it was listed as a Telegram ip. Since Telegram is often used for C2 traffic, that was a huge red flag. So I went and looked at my own logs for the month of June for the same Telegram ip address. And lo and behold, there it was. It connected to two different local ips on two different days. At this point, I turned my attention to the earliest date and that local ip, June 16.

Note: I should mention that in the Flare logs, there was no date that matched the 16th. The earliest infection/post was for the 8th, and then later dates all appeared around mid-late July. At this point, I wasn't looking at network logs for the 8th since the Telegram ip only appeared on the 16th. It's absolutely plausible that there was another infection on the device, or Flare just made an incorrect estimation of the original date.

![alt text](images/Infostealer-darkweb/turbo-ip.png)
![alt text](images/Infostealer-darkweb/telegram-log.png)

I reported this discovery to my superior, along with the file name I suspected to be the orgin of the malware.<br>
But to be honest, I still wasn't satisfied with this. I got curious about the file itself and downloaded it from VirusTotal. From there I reuploaded it as a private scan to see what would be revealed in the pcap.<br>
I saw a Telegram packet that had a GET request to a profile called `@turb00m`, and then I noticed a text/html packet. Inspecting that text/html packet further, I found a url for a Steam Community profile. I inspected the page, but it was set to private. The user name, `76561198689449626`, being just a bunch of numbers was suspicious, possibly an api call, but as for what was actually on the profile, I couldn't confirm. It's possible there was a link or payload in the profile details or comment section. But for now that's just a guess.<br>
The images below are both of the private profile and the public details I could find on the profile. The details of the profile show that it was created only recently (Jun. 12) and had last been public Jul. 11 (as of this screenshot). It had also changed its name on Jun. 24. The community URL matches that which was in the pcap.

If the malicious file was only supposed to be Tools Unlock that connects to the C2, why would there be Steam traffic?<br>
So I git curious if the public pcap on VirusTotal showed anything different. And it did.

[Private scan pcap]<br>
![alt text](images/Infostealer-darkweb/private-pcap.png)

[Public pcap]<br>
![alt text](images/Infostealer-darkweb/public-pcap.png)

![alt text](images/Infostealer-darkweb/telegram.png)
![alt text](images/Infostealer-darkweb/steam-profile.png)
![alt text](images/Infostealer-darkweb/steam-profile-details.png)

In the public pcap there was a strange URL that seemed to be communicating with the Steam profile. Then just seconds apart, all three connections suddenly cut.<br>
It's unclear what role the Steam profile played exactly (perhaps there was a link or payload in the comment section to connect to the strange site), but there was no doubt that this wasn't just a coincidence.<br>
Since I had confirmed the presence of the Telegram traffic (same ip address), I was curious if the same behavior could be found in the network logs I had. I edited my QRadar search to include the local ip of the device the Telegram traffic was found, set the time frame for 1:15~1:45, the timeframe the Telegram traffic occured, and searched for anything with the word "steam" in the payload. And BINGO! Steam community traffic was found!

[Note: The host name for this was indeed steamcommunity.com, I forgot to include the hostname catergory in the screenshot.]
![alt text](images/Infostealer-darkweb/steam-traffic.png)

If Steam traffic was confirmed, then next there had to be an unusual website hitting the firewall, right?<br>
So that was my next goal, find suspicious web traffic. This was the harder part. At first I focused on looking for suspicious ip locations, but that proved fruitless. Then I realized some ips didn't show any flags/locations. When I took a closer look into one, I noticed it was blocked.

![alt text](images/Infostealer-darkweb/blocked-traffic.png)

This blocked site made a number of attempts to get through the firewall. The domain name too was very suspicious, `sip.rzrrent[.]com`. I mean, even the category had a discription of 悪意のあるサイト (malicious site).<br>
But if this computer had its data stolen, and this site was blocked, that had to of meant that there was another site that did in fact get through.<br>
So I looked for more ips that didn't have a location/flag. And there I found a site. `en.taiwebs[.]com`. From the name of it I couldn't tell for sure if it was suspicious (turns out the site used to be a host for illegal software downloads). I mean, the category this time said 違法・非論理的 (illegal/illogical), buuut just in case, I put it through urlscan.io to see what came up. Aaand the server was down!

![alt text](images/Infostealer-darkweb/taiwebs-traffic.png)
![alt text](images/Infostealer-darkweb/taiwebs-urlscan.png)

That's... weird. But another thing caught my attention. On urlscan.io, the ip address that was below the website name was different from the ip addresss that was listed in the network logs. This time I searched the ip address and it revealed a long list of domain names being used with that ip just within the past 2 weeks. Deffinitly malicious. For the heck of it, I checked the blocked ip address as well. Same thing, a long list of recycled and very malicious looking domains.

![alt text](images/Infostealer-darkweb/ipcheck1.png)
![alt text](images/Infostealer-darkweb/ipcheck2.png)

So that confirmed it. Just like on the public pcap, our logs also had suspicious site traffic.<br>
Unfortunately since I didn't have the pcaps for the network I was investigating, I wasn't able to identify what was happening inside the packets. I don't know what the payload looked like or what was being sent/received. But just in case, I looked just a little further a few minutes on the logs for any last suspicious traffic. And BOOM! A Russian ip address.

![alt text](images/Infostealer-darkweb/russia-ip.png)

Looking at the bytes sent, it's hard to say for sure whether or not this was used to send the stolen data (the data that was stolen wasn't much to begin with, so it's not entirely impossible). However, there was no need for this user to be accessing this site to begin with, and it shows up around the same time as the incident, so I thought it was worth reporting on.

So, let's look at the timeline.<br>
<pre style="background:none; border:none; padding:0; font-family:monospace; line-height:1.2;">
6/16<br>
│   ├── 13:17:45 First communication with Telegram (@turb00m)<br>
│       ├── 13:17:46 First communication with Steam (76561198689449626)<br>
│           ├── 13:17:51 First communication with Blocked Site (sip.rzrent)<br>
│           └── 13:24:40 Last communication with Blocked Site<br>
│           ├── 13:29:17 First communication with Passed Site (en.taiwebs)<br>
│           └── 13:36:00 Last communication with Passed Site<br>
│       └── 13:38:09 Last communication with Steam<br>
│   └── 13:38:15 Last communication with Telegram<br>
│       ├── 13:41:26 First communication with Russian IP (yandex)<br>
│       └── 13:43:35 Last communication with Russian IP<br>
</pre>

In total the actual attack was about 20 minutes (excluding the russian ip). If the first blocked site had been successful, it would have been shorter. But this also means that the threat actor probably had to manually adjust their attack method to get through. I doubt it was automated because there was about a 5 minute gap between the block and success.

I reported my discovery and figured that was the last of it.<br>
But as I was just doing some final Goggling, I did find an X post that gave me one additional detail.

![alt text](images/Infostealer-darkweb/twitter.png)

The Steam Community profile is the same.<br>
The Telegram account is the same.<br>
And the original blocked site is exactly the same.<br>
The infostealer I looked into is apparently a `Vidar Stealer`.

Vidar Stealers are an older type of infostealer from 2018, but apparently it's been revamped as Vidar 2.0.<br>
I found this blog to be good at explaining what it does. It also mentions the same type of traffic that I experienced in this investigation, so I think we have a match.<br>
`https://www.picussecurity.com/resource/blog/vidar-malware-how-the-multithreaded-windows-stealer-works`

And so, that is how I went from simply having stolen data on the dark web to a full-blown network investigation. This investigation in particular was incredibly fun to do. It really felt like I was desperately collecting puzzle pieces to then figure out how it all logically fits together, and seeing the mess/history of the situation unfold in front of me.

## Update 2026/08/09

Good news, the forensics team shared a short report of their investigation on the infected device.<br>
They had also came to the conclusion that the root of the infection was a ToolsUnlock.exe file. During their investigation, they found a file called `Get_Link.txt` that was created in the user's Downloads folder on `6/16 13:16:15`.<br>
Inside this file was a Dropbox link to a ToolsUnlock.zip and a password to extract it. The following:
```
hxxp[:]//www.dropbox[.]com/scl/fl/hskmvo1wjh2sgkoagdcvq/ToolsUnlock.zip?rlkey=vz3y5ii3i8yh606rpsrbjc88d&st=v13baa5c&dl=1
Password: w8zt3a
```

So they identified where the ToolsUnlock file was downloaded from, BUT they also said they couldn't figure out where this "Get_Link.txt" file came from. And they just gave up there.<br>
Well, if it's in the Downloads folder, it was downloaded somewhere. I got curious. So I wanted to go have another look on Flare at the user's search history.... but then I found out the forensics team (who had the admin account) revoked the SOC team's guest account priveleges to view those details. Oh great, a dead end.

Or so I thought.

I was determined to find this mysterious file. So I stared down the network logs again. First I searched for any Dropbox traffic. Bingo, found. And the bytes recieved clearly screamed that something was being downloaded. This had to be ToolsUnlock.exe.

![alt text](images/Infostealer-darkweb/dropbox.png)

Okay, so I confirmed Dropbox. But where did the user get this link?<br>
I moved my attention to the time the forensics team said the file was created. And..... I found a possible lead.<br>

![alt text](images/Infostealer-darkweb/gdrive.png)

There was a log for `drive.usercontent.google.com`. This meant there was something downloaded from a Google Drive. And considering the time lined up just perfectly with the forensics team's report, I felt confident that this had to be the smoking gun. The only issue was.... this log only had the hostname. The log payload didn't have the full download link. All I knew was that something was downloaded from Drive and I have no idea where this Drive download came from. Another dead end.

Except I'm stubbon.<br>
I was determined to find that Drive link. After trying all sorts of attempts, I eventually decided to search for the Dropbox folder name, `hskmvo1wjh2sgkoagdcvq`, in Google. And in the results I saw a Malware Bazaar link. I decided to click it. I scholled down to the bottom and noticed a comment...

![alt text](images/Infostealer-darkweb/malbazaar.png)

BINGO!<br>
Same folder name, also calls Telegram, and has a malicious website (one I recongized from the public pcap)!<br>
And there at the very beginning was a GitHub link for a Dr.Fone Repair.<br>
So I gave the GitHub a visit.

![alt text](images/Infostealer-darkweb/drfone.png)

From appearances it looks like a normal git repository. I clicked around. It was posted in 2025.<br>
And then.... I hovered over the "Download" link.

![alt text](images/Infostealer-darkweb/drive-link.png)

THERE IT WAS!!!<br>
It was a `drive.usercontent` link! I was excited. I clicked download.<br>
The file that was downloaded was a .txt file called "Resource_Link.txt". And then I opened it......<br>
And it was another Dropbox link to ToolsUnlock.exe and under it a password!!!<br>
Basically the same exact file the forensics team had discovered!!!

![alt text](images/Infostealer-darkweb/download-txt.png)
![alt text](images/Infostealer-darkweb/link-txt.png)

AND THERE YOU HAVE IT!<br>
The user was originally trying to download a cracked version of Dr.Fone Repair, but the GitHub repo they downloaded from was a fake to lure visitors to download their ToolsUnlock malware!<br>
Mystery solved!!


## Bonus Digging (Before Update)

Just for fun (and desperate hope maybe I'd learn something new), I tried examining the malicious file (ToolsUnlock_v6.9).<br>
In the first image, I ran the .exe file through Detect It Easy.<br>

![alt text](images/Infostealer-darkweb/dte1.png)
![alt text](images/Infostealer-darkweb/dte2.png)

The file seems to be a 64-bit executable with a size of 3.3 bytes written in Go.<br>
It's got some entropy. 85% packed with Section 2 (.rdata) beeing packed at an entropy of nearly 6.9. No info about what kind of packer was used sadly. The damning malware code is most likely packed and obfuscated in that section.

After that I ran the file in Ghidra to see what kind of functions/DLLs were being called and for any specific strings that stood out.<br>
Right away I noticed many main functions with garbled text.

![alt text](images/Infostealer-darkweb/ghidra-main.png)

And that's about it!<br>
Thanks for reading!