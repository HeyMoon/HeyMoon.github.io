---
title: introduce_cdn
date: 2021-11-10 21:29:04
tags: 
    - CDN
categories: 
    - tech
---

CDN(Content Delivery Network) 是一个全球性的分布式网络服务，它从靠近用户的位置提供内容。通常，HTML/CSS/JS，图片和视频等静态内容由 CDN 提供。CDN 的 DNS 解析会告知客户端连接哪台服务器。

将内容存储在 CDN 上，可以从两个方面来提高性能：
+ 从靠近用户的数据中心提供资源
+ 你的服务器不必为CDN所满足的请求提供服务

#### push CDNs
只要你的服务器发生变化，Push CDN 就会收到新的内容。你负责提供内容、直接上传到CDN和重写URL以指向CDN。你可以配置内容何时过期和何时更新。内容只有在新增或改变的时候才会被推送，最大限度地减少流量，但最大限度地提高存储。

#### pull CDNs
Pull CDN 在第一个用户请求内容时从服务器上抓取新内容。这将会导致第一个请求变慢，但是当第二次访问该资源时，CDN 上已经缓存了该资源，就可以直接返回。

TTL(Time To Live) 决定一个指定资源的存活时间，单位是秒。

#### Pull CDNs 的优点
一般来说，Pull CDN 比 Push CDN 更容易配置。一旦最初配置好，Pull CDN就会根据要求在其服务器上无缝存储和更新内容。如果CDN没有检测到文件被修改，数据通常会在那里停留24小时或更长时间。对于低流量的网站或那些通过缓存、良好的代码等进行了充分优化的网站，Pull CDN 提供了加速，而不会对你的服务器提出太多的要求。一旦你的内容被pull 了，所需的维护是低的。

#### Push CDNs 的缺点
Push CDN 会增加服务器的负担（因为需要推送内容到 CDN）。或者你在一天内有很多变化的内容，Push CDN 会给你的服务器带来额外的压力。如果你的服务器有很大的负载，或者每天有几次新的内容，所有这些内容在你的服务器和CDN之间同步可能是弊大于利。

#### 如何选择
决定使用哪种CDN类型，在很大程度上是围绕着流量和下载量。

托管视频和播客（又称大型下载）的旅游博客会发现，从长远来看，Push CDN 更便宜、更有效，因为CDN不会重新下载内容，直到你主动将其推到CDN。

Pull CDN 可以帮助高流量的小型下载网站，在CDN服务器上保留最受欢迎的内容。内容的后续更新（或 "Pull"）并不频繁，不足以使成本上升到 Push CDN 的水平。换而言之，高流量站点使用 Pull CDN 效果不错，因为只有最近请求的内容保存在 CDN 中，流量才能更平衡地分散。

参考文献：
[The Differences Between Push And Pull CDNs](http://www.travelblogadvice.com/technical/the-differences-between-push-and-pull-cdns/)
