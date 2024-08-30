+++
title = 'How I set up my server'
date = 2024-08-23T02:12:24Z
draft = false
+++

## What will this server be used for?
In order to determine the appropriate host and domain for my server, I needed to clearly define what I would use the server for.

I knew, at minimum, I wanted to:
- Host a small blog
- Host an RSS reader (as many hosted RSS readers are rather pricey)

But I also was thinking of hosting additional services in the future, such as:
- activitypub server / client
- a media server
- personal web projects

To keep things simple and to avoid my common trend of "aiming too big" with projects, to the point where I never finish them, I decided to focus primarily on the blog and RSS reader projects as a driving force for the server.


## Choosing a Host
When choosing a host for my server, I came across two options:
1. Host out of a machine you own and run in your home
2. Host out of a virtual private server (VPS) you rent from a provider

Whilst the idea of owning my own hardware seems to offer the most independence, due to the unreliable nature of my own power and internet networks, I wasn't sure if I wanted to host the machine myself. 

I wasn't super fond of the idea of renting either, as that is another monthly cost on top of  other costs such as a domain. However, services like [Hetnezer] offered VPS's for as low as 4 dollars a month, something I could upgrade in the long run


## Choosing a Domain
The hardest part of all - a domain.

I decided to register a domain with Cloudflare because they are pretty cheap! I didn't go for the cheapest domain; however, gilly.garden was still significantly cheaper than competitors. 

What is really nice about Cloudflare is that the price they show you is the price you will renew at. A lot of other domain providers will show you a really low price for the first year. However, once you are locked in and go to renew, the price may be significantly higher. With Cloudflare, I felt like I was avoiding pricing surprises.

## What's Next?
If you are reading this, it means I got the blog setup! Right now, it is pretty simple. Moving forward, I want to work on making my own custom theme. For now, I am using Hugo with the [Typo Theme](https://themes.gohugo.io/themes/typo/)

