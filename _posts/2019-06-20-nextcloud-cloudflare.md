---
layout: post
title: "Nextcloud & Cloudflare: Investigation into excessive data usage"
---
Or, how 3 well designed components behave stupidly together.

I host a medium-sized [Nextcloud](https://nextcloud.com) installation that is quite actively used.

Recently, I was notified by my users that performance is extremely bad. Upon investigation, it was clear something bad was going on.
Data transfer was multiplied by 40 compared to the previous period.

![Data transfer in june-july 2018]({{ site.baseurl }}/assets/nextcloud-cloudflare/bandwidth-2018.png)
![Data transfer in june-july 2019]({{ site.baseurl }}/assets/nextcloud-cloudflare/bandwidth-2019.png)

This increase in data transfer usage resulted in exhausting the allowed transfer per month after only 10 days.
With the provider this server is hosted with, going over the allowed datatransfer does not result in extra charges.
Instead, they throttle the bandwidth to 10Mbps.


# The components

## Nextcloud

The VPS also runs some less-bandwidth intensive services other than Nextcloud.
For isolation and accounting purposes, a separate PHP-fpm pool is used for running Nextcloud.

To avoid it eating all RAM, the pool is limited to a maximum of 6 children.
This means at most 6 requests can be handled in parallel.
Under normal circumstances this limited amount of parallellism is not a problem.
The maximum is hit occasionally, and in that case the new request has to wait until a previous request has finished.

When a file is being downloaded, this uses a PHP-fpm process for the full duration it takes to download the file.
Because uploading a file is normally not limited by the networkspeed (the response is sent to cloudflare, not the end-user directly), this does not pose a problem and processes are freed up quickly again.

However, when bandwidth becomes the limiting factor, uploading the file becomes slower. This means the PHP-fpm processes will spend a longer time to serve a single request, quickly exhausting the available pool.

Here you can see that the active PHP-fpm processes almost never fall below the maximum anymore after the bandwidth cap is applied.

![Outgoing traffic from the server]({{ site.baseurl }}/assets/nextcloud-cloudflare/machine-traffic.png)
![Active PHP-fpm processes]({{ site.baseurl }}/assets/nextcloud-cloudflare/fpm-active.png)

**Key behavior**: On all files served from the repository, cache control headers are set.

```
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
```

## PDF.js

PDF.js is embedded in Nextcloud and used to display PDF documents from the filesystem.

**Key behavior**: Uses HTTP Byte-Range requests when the remote server supports them to fetch only the parts of the file that it needs.

## Cloudflare

Cloudflare is put in front of the Nextcloud instance, acting as a barrier to block bad bots and to cache static files to reduce data usage and increase speed.

The cache settings of Cloudflare are set to its defaults, which means static files are cached and HTML is not cached.

**Key behavior**: Tries to serve Byte-Range requests from cache. If the file is not in the Cloudflare cache, it is requested in full to cache it. Cloudflare also respects Cache-Control headers.

# The result: 20-30 times more data transfered than necessary

What happens when a user opens a PDF file?

1. PDF.js sends a byte-range request for the metadata of the PDF.
2. Cloudflare sends a request for the full PDF file. The request looks like a static file, so they probably want to cache it.
3. Nextcloud sends a no-cache header and sends back the full PDF.
4. Cloudflare receives the PDF, but can't cache it. It responds to PDF.js with the partial content it requested and then discards the file.
5. PDF.js recognizes that the server supports byte-range requests, and keeps using them to load pages on demand.
6. User scrolls through the PDF. Every new page that needs to be loaded fires off an other byte-range request.
7. Requests are handled by Cloudflare, the full file is requested from the origin server and a partial response is sent back to the client.

Repeat this for multiple pages of a PDF when a user is scrolling through them. Most of the PDFs are scanned content, so they have reasonably large pages.

![Byte-range requests sent by PDF.js]({{ site.baseurl }}/assets/nextcloud-cloudflare/pdfjs-partial.png)
[![Full responses by the server]({{ site.baseurl }}/assets/nextcloud-cloudflare/pdfjs-requests.png)]({{ site.baseurl }}/assets/nextcloud-cloudflare/pdfjs-requests.png)

# How to fix

You have 2 options if you want to fix this issue.

The first and most simple option is to simply disable Cloudflare for the domain you run Nextcloud from. Of course, this solution also has the downside that static assets of Nextcloud are no longer cached by Cloudflare and you no longer have the protection either.

The second option, which seems to work for now, is to put in a page rule in Cloudflare to bypass the cache for all Nextcloud WebDAV requests.
![Bypass cache for Nextcloud WebDAV]({{ site.baseurl }}/assets/nextcloud-cloudflare/cloudflare-page-rule.png)

## Results

After bypassing the cache for WebDAV requests, the amount of bytes transmitted through the Nextcloud WebDAV endpoint immediately dropped to a fraction of what it previously was.

[![WebDAV traffic before and after]({{ site.baseurl }}/assets/nextcloud-cloudflare/webdav-traffic.png)]({{ site.baseurl }}/assets/nextcloud-cloudflare/webdav-traffic.png)

A more detail view after applying the fix. Transfer rates went from 5 MB/s and over to mostly under 200 kB/s, which is a reduction of 25 times.

[![WebDAV traffic after, zoomed in]({{ site.baseurl }}/assets/nextcloud-cloudflare/webdav-traffic-after.png)]({{ site.baseurl }}/assets/nextcloud-cloudflare/webdav-traffic-after.png)
