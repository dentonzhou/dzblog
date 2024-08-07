---
title: Letting Good Bots In
date: 2024-08-06T15:45:00
weight: 50
subtitle: ""
subheading: 
author: denton
snippet: A few simple steps for identifying good bots so they can bypass rate limiters and other defenses.
description: Rate limiters help mitigate abuse and attacks. But we still need good bots to crawl our sites. Here's how we grant our good bots unlimited access to our servers.
sidebar: dev
category: ""
prevPost: ""
nextPost: ""
image: ""
imagePositionX: 50
imagePositionY: 50
showImageInHeader: true
isVisible: true
isArchived: false
isContentHidden: false
contentMessage: ""
relatedPost1: ""
relatedPost2: ""
relatedPost3: ""
relatedPost4: ""
relatedPost5: ""
relatedPost6: ""
relatedPost7: ""
relatedPost8: ""
relatedPost9: ""
---

I recently built a rate limiter into [OpenCourser](https://opencourser.com?utm_source=dzblog&utm_medium=devnote). It checks the number of requests coming from the same visitor over a period of time. 

It's relatively straightforward. If a visitor exceeds a set number of requests, then the server returns a `429` error. Otherwise, the view function renders its response and sends it back to the visitor.

One important feature of this rate limiter is its ability to exempt "good bots" from these checks. Regardless of how many requests they send, these bots should _not_ run into `429` errors.

## What’s a good bot anyway?

Bots are everywhere, tirelessly crawling and scraping the web.

Earlier versions of OpenCourser used Google Analytics to track visits. Google Analytics filtered out a great deal of this bot activity so that I'd see activity from mostly legitimate (i.e. non-bot) visitors.

It's only when I decided to roll my own analytics did I notice the bots, mostly from the rubbish requests they'd make to nonexistent endpoints.

Of course, you don't need to roll your own analytics to see the bots. If you're running your own server, you could just inspect your logs. Using nginx, I do this with this shell command:

```sh
sudo tail -f /var/log/nginx/access.log
```

In just a few short seconds, I can spot bots from Meta, Ahrefs, Bytedance, and Bing.

These requests don't bother me too much. They tend to obey `robots.txt` and they're good about spacing out their requests. Very polite. Not at all malicious.

But some bots are better than others, and it's the "good bots" that I _want_ to visit. My definition of a good bot is simple. If it helps drive traffic and doesn't DDoS my site, it's welcome to crawl away.

These good bots include the likes of Googlebot and Bingbot, which crawl with the intention of potentially indexing my pages on search engines. It's a win-win (with caveats that are out of scope for this post).

## Why distinguish the good bots anyway?

If we only had neutral or good bots, we wouldn't need to distinguish the goodness of a bot. But not all bots are nice and some are, unfortunately, harmful.

Many of these bots scour the web looking for vulnerabilities to take advantage of, poking and prodding away at security holes.

The easiest offenders to spot are the ones that try to access auth pages for popular content management systems. Wordpress endpoints like `/wp-login.php` appear frequently, for example.

These are easy to catch, limit, and even ban for my case because none of my sites use a well known CMS. The [little blogging platform](https://github.com/dentonzh/Eggspress) this post is hosted on doesn't even have an auth endpoint. But I digress.

In the past two weeks, nearly 2,200 requests were made to endpoints starting with `wp-`. This is a drop in the bucket, but it’s a lot for a relatively unknown site on the Internet. I can only imagine how many nonsense requests a better known site might receive.

Less often, “bad” bots hit the site a bit too hard, DDoSing it either intentionally or unintentionally in the process.

OpenCourser has thankfully experienced only minor downtime from this. Services bounced back on their own without manual intervention. Props to firewalls, WAFs, and Cloudflare for that.

So why bother putting together a rate limiter now?

Well, for one, prevention is the best medicine. But also, because the new and improved OpenCourser comes with a few new bells and whistles. Some of these are rather costly to run in that they require more CPU time and memory. There are also a calls made to APIs of LLM providers sprinkled in the mix.

To recap: rate limiting stops bots from making too many requests to OpenCourser. But some bots are good because they put us on the (search engine) map. We want to let these select few good bots to make as many requests as they need to to get the job done.

This brings us to the meat of this post: how do we actually identify a good bot?

## By User-Agent

Every time a person or a bot visits OpenCourser, they're creating an HTTP request. This request could be thought of as a letter in the mail system. When you create the request with your browser, you're telling it where to connect (`https://opencourser.com`).

Unbeknownst to most users, the browser will also enrich this request with other data, tucked neatly into a set of request headers. One of these is the `User-Agent`, which packs [a lot of information](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent) about the browser, device, and operating system from which the request originated. This header can sometimes include information about bots as well. 

Google’s bots, for example, are identified by the user agent token “googlebot”:

> `Mozilla/5.0 (Linux; Android 6.0.1; Nexus 5X Build/MMB29P) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/W.X.Y.Z Mobile Safari/537.36 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)`

Bing’s bots are similarly identified by a user agent token containing “bingbot”:

> Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko; compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm) Chrome/

It's easy enough to inspect the `User-Agent` of an incoming request:

```python

# Using Flask

from flask import request

user_agent = request.headers.get(‘User-Agent’)


# Using Django

def my_view(request):
    user_agent = request.META.get(‘HTTP_USER_AGENT’)


# Using FastAPI

@app.get(‘/‘)
async def read_root(request: Request):
    user_agent = request.headers.get(‘User-Agent’)

````

 If we find that a good bot's string is in the `User-Agent`, we might decide that that request really came from a good bot. One way to do this is to use regex to pattern match the known names of our good bots:

```python

def is_good_bot_user_agent():
	  # Returns True if User-Agent indicates a good bot
    user_agent = request.headers.get('User-Agent', '')
    good_bot_patterns = [
        r'Googlebot',
        r'bingbot',
        r'Baiduspider',
        r'YandexBot',
        r'DuckDuckBot'
    ]
    return any(
        re.search(pattern, user_agent, re.I) for pattern in good_bot_patterns)
```

This alone is a great start. Request headers can, however, be modified. This means a bad bot can easily bypass this check by spoofing its `User-Agent` to include strings like `Googlebot` or `bingbot` or `DuckDuckBot` or `YandexBot`, etc.

Still, this check is inexpensive to run and it's worth feeding every request through to see whether we need to perform additional verification steps.

## Verify by IP address

The easiest way to validate a good bot is to create a whitelist of known IP addresses. I’ve been using this [GitHub repository](https://github.com/AnTheMaker/GoodBots) from [Anton Röhm](https://github.com/AnTheMaker) that contains whitelists of known IPs belonging to good bots. It's updated daily.

Using Python, we can get the IP address of an incoming request like so:

```python

# Using Flask

from flask import request

ip_address = request.remote_addr


# Using Django

def my_view(request):
    ip_address = request.META.get(‘HTTP_X_FORWARDED_FOR’)


# Using FastAPI

@app.get(‘/‘)
async def read_root(request: Request):
    ip_address = request.client

```

We can then check if the IP address from the request matches with one found in our IP whitelist. Since DuckDuckBot can only be validated by IP address, I’ll use DuckDuckBot as our example:

```python

import requests

if ‘DuckDuckBot’ in user_agent:
    duckduckbot_ips_url = ‘https://raw.githubusercontent.com/AnTheMaker/GoodBots/main/iplists/duckduckbot.ips’
    duckduckbot_ips = requests.get(duckduckbot_ips_url).text.split(‘\n’)
    return ip_address in duckduckbot_ips

```

This snippet returns `True` if an incoming request has a `User-Agent` that contains the string `DuckDuckBot` and _also_ has an IP address found in the IP whitelist for DuckDuckBot.

## Verify by reverse DNS lookup

A reverse DNS lookup uses the Domain Name System (DNS) to find the domain name associated with an IP address. 

It works similarly to our IP address lookup. Instead of looking up one IP against a list of valid IP addresses, however, we instead use the DNS to see their registered domain. We can use this domain to verify a good bot:

```python

import socket
import request

# Using Flask as an example to check Googlebot
# See code snippets above for getting ip with Django or FastAPI

def is_google_bot_verified():
	ip = request.remote_addr
	hostname = socket.gethostbyaddr(ip)[0]
	
	if hostname.endswith('.googlebot.com'):
		return True
		
	if hostname.endswith('.google.com')
		return True

	return False

```

The `socket` library gives us a straightforward way to perform a reverse DNS lookup. It returns the domain associated with an IP address.

It's not enough just to perform the reverse DNS lookup, however.

To do any real verification, we also need to know which domains are valid. For this, we rely on the bots' operators to tell us which domains their bots use. Companies that maintain good bots like [Google](https://developers.google.com/search/docs/crawling-indexing/verifying-googlebot) will typically provide the valid domain(s) to which their bots and crawlers are registered.

As of this writing, the IP address `66.249.90.77` belongs to one of Google's many crawlers.

When we run `socket.gethostbyaddr('66.249.90.77')[0]`, we see this output:

`'rate-limited-proxy-66-249-90-77.google.com'`

According to Google's documentation, any IP with reverse DNS mask ending in `.google.com`  or `.googlebot.com` is a valid. We can therefore conclude that the IP above belongs to a good bot and allow it to crawl OpenCourser without being subjected to our rate limiter.

## Reverse DNS lookups are preferable to IP lookups

Reverse DNS lookups are superior to IP lookups in almost every way. The reasons for this boil down to ease of maintenance and accuracy.

We're fortunate to have Anton's IP whitelist. It fits our use case perfectly and it's updated daily, which is likely more than sufficient.

But in our snippets above, we've hardcoded our IP lookup functions to reference this repository's files.

What if the repository stopped updating one day? Sure, we could write additional code that alerts us to our whitelists going stale. We could even add some fallback whitelists to look up. But that doesn't rule out the need for us to return one day to kick the tires on our IP lookup functions and possibly change a few lines of code.

Then there's the question of accuracy. IP addresses change.

It's unlikely, but not impossible, for a malicious actor to get their hands on an IP address that once belonged to a good bot. If our whitelist didn't account for this handover, we could potentially grant a bad bot unfettered access to our resources.

Less uncommon is a search engine adding new bots and crawlers with IP addresses unaccounted for by the whitelist. This could cause us to unwittingly restrict a good bot's access to our site.

Reverse DNS lookups help us dispense with the maintainability and accuracy issues. There's no ambiguity when we run a DNS query to find a domain name given an IP address.

Until DuckDuckBot and bingbot offer a (reliable) means to check their bots using reverse DNS lookups, we'll likely need to use at least some IP lookups to validate our good bots.