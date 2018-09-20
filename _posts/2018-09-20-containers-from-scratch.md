---
layout: post
title: Containers from scratch
---

I've owned a Docker t-shirt since 2016. Occasionally, when I wear it, people ask me about Docker.

Until recently, I haven't had much to say; the company I worked for in 2016 had hosted a Docker meetup and gave the leftover t-shirts to employees.

Since then, I haven't used Docker in a significant way at work or in personal projects (though, that is changing).

Although the container hype has died down a bit and [it's now criminal to include stock photos of shipping containers in blog posts about Docker](https://twitter.com/johnmark/status/1037050824554373120), I decided to learn about the technology behind them by writing my own!

I'd previously explored [connecting to the Internet from a network namespace]({% post_url 2018-05-16-net-namespaces-veth-nat %}), but this project covers more ground. 
It also led to a [few]({% post_url 2018-09-13-strace-mount-rprivate %}) [other]({% post_url 2018-09-19-how-does-the-docker-run-p-option-work %}) [posts]({% post_url 2018-09-19-ps-proc-and-the-pid-namespace %}).

Its README is fairly comprehensive and includes asciicast demos, so I won't write much about it here.

[https://github.com/jmuia/go-container](https://github.com/jmuia/go-container)
