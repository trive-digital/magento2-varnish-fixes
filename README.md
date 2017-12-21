# Trive Varnish #

## Description ##
Magento 2 module which fixes issues with cache tags and 503 errors when using Varnish as caching application.


## Installation ##
Install via composer:
```
composer require trive/varnish
```

Modify your VCL file with the following content:
 ```
 sub vcl_backend_response {
     ...(redacted)
     
     # Trive - add these two lines below - start
     # collect all x-magento-tags headers
     std.collect(beresp.http.X-Magento-Tags);
     # Trive - add these two lines above - end"
     
     return (deliver);
 }
 ```
 
Enable the module:
```
bin/magento module:enable Trive_Varnish
bin/magento setup:upgrade
```

## The issue ##
When using Varnish as caching application in Magento 2, each page is tagged by cache tags of the content displayed on it. These tags are then used for cache invalidation and purges, once content is updated.

For example, if you open a home page on default installation, it will be tagged with 
`store, cms_page and cms_page_1` cache tags. 

Categories are tagged with cache tags of all products they have assigned. This means that, if you have a large catalog, with a single category containing about 2000 products, you're likely to have about 20kB worth of cache tags when first loading that category.

Cache tags are sent to Varnish as `X-Magento-Tags` header on initial, non-cached request to Varnish, so it can tag the URL with displayed content, and be able to clear all pages where a object appears, when requested.

Once `X-Magento-Tags` header gets longer than 8kB, default `http_resp_hdr_len` Varnish limit is hit and Varnish returns a 503 error to customers (backend fetch failed). 

Even after implementing workaround from [Magento Documentation](http://devdocs.magento.com/guides/v2.2/config-guide/varnish/tshoot-varnish-503.html), the issue is not solved, because another limit on Apache/nginx HTTP server might be hit, as they [both](http://httpd.apache.org/docs/2.2/mod/core.html#limitrequestfieldsize) [have](http://nginx.org/en/docs/http/ngx_http_core_module.html#large_client_header_buffers) 8kB max header value by default. 
 
## Affected stores ##
Your store is affected if you are:
* using Magento 2 (any version)
* having a large number of products on store, with at least one anchor category containing at least 2000 products
* using Varnish as Magento 2 caching application, with default `http_resp_hdr_len` limit

Your store might also be affected if you are:
* using Apache/nginx with default `LimitRequestFieldSize`, `large_client_header_buffers`, or some other limit that might be hit by HTTP headers 
* using nginx/HAProxy with default header length or request size limits
* [using Apache mod_proxy_fcgi](https://maxchadwick.xyz/blog/http-response-header-size-limit-with-mod-proxy-fcgi)
 
## Our solution ##
This module addresses the above issue by separating `X-Magento-Tags` into 8kB header blocks.
Sending multiple `X-Magento-Tags` headers don't hit most of the limits, and doesn't triggers 503, nor `FetchError http read error: overflow` errors during Varnish backend fetch.

Note that default Magento 2 VCL file should be updated after implementing this module, to be able to collect multiple `X-Magento-Tags` headers into one, during Varnish procesing.



