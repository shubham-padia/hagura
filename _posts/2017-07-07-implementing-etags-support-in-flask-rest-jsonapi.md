---
layout: post
title:  "Implementing ETags Support In flask-rest-jsonapi"
date:   2017-07-07 18:34:56 +0530
categories: gsoc
---
![Etag]({{site.baseurl}}/images/tag.png)

### What is an ETag ?
> An entity tag (ETag) is an HTTP header used for Web cache validation and conditional requests from browsers for resources.

### What is the need for an ETag ?
- Clients can make use of this and request complete data if and only if the data has changed else use their local cache.
- This can be used to ensure concurrency in the case of multiple clients trying to modify the same data at the same time. 

### How to implement ETags in the API framework ?

To implement ETags in the API framework, changes need to be done in the `dispatch_request` function of `Resource` class located at `resource.py` at the root of the framework.

A config variable will also be added in order to turn ETags on and off. You can name anything you want, but we went ahead with just `ETAG`. Now the first thing we should do is calculate the ETag hash from the original response. The response variable can be grabbed in `dispatch_request` and hashing can be performed on it.

```python
etag = hashlib.sha1(resp.get_data()).hexdigest()
resp.headers['ETag'] = etag # return ETag in response headers
```

# Why did we use SHA-1 ?
In the above mentioned lines of code, you will notice that we are using `SHA-1` for hashing purposes. `SHA-1` is known to have collisions, so why use it ? In ETags we are not storing the hashes anywhere but are returning the ETag in the response header directly. So there is a very less probability of collision even if we used `MD5`, so using `SHA-1` won't hurt much ;)

Till now, the above code enables to return an ETag but that is of no use if we do not support request headers `If-Match` and `If-None-Match`. Both of these headers can be obtained from the request as follows:
```python
if_match = request.headers.get('If-Match')
if_none_match = request.headers.get('If-None-Match')
```

For both `If-Match` and `If-None-Match` request headers, the system will accept a comma separated list of Etags. This can be accomplished as follows:
```python
etag_list = [tag.strip() for tag in if_match.split(',')]
```

For `If-Match`, the response is returned only if the ETag of the current response matches any of the comma-separeted ETags in the If-Match header. If none of the given ETags match, a 412 Precondition Failed status code will be returned. This can be implemented with a check as follows:
```python
if if_match:
    etag_list = [tag.strip() for tag in if_match.split(',')]
    if etag not in etag_list and '*' not in etag_list:
        exc = PreconditionFailed({'pointer': ''}, 'Precondition failed')
        return make_response(json.dumps(jsonapi_errors([exc.to_dict()])),
                             exc.status,
                             headers)
```

For `If-None-Match`, the response is returned only if the ETag of the current response does not match any of the comma-seperated ETags in the If-Match header. If none of the given ETags match, a 304 Not Modified status code will be returned.
```python
elif if_none_match:
    etag_list = [tag.strip() for tag in if_none_match.split(',')]
    if etag in etag_list or '*' in etag_list:
        exc = NotModified({'pointer': ''}, 'Resource not modified')
        return make_response(json.dumps(jsonapi_errors([exc.to_dict()])),
                             exc.status,
                             headers)
```