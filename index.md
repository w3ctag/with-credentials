# Reducing the pain of the credentials mode

## Use cases

These proposed new values come out of discussions in the W3C TAG related to [issue #76](https://github.com/w3ctag/design-reviews/issues/76).  At the root of that discussion are concerns about the ease of using and mixing open data in script that is part of a web page.  The [fetch](https://fetch.spec.whatwg.org/) specification for fetching resources defaults to behaving as a mechanism for fetching a resources that is located at the given URL.  However, in some cases, the user of the fetch API wishes to use the ambient authority in the browser (such as HTTP authentication associated with the origin or cookies associated with the domain) as part of the network request, because the request will not succeed unless this authority is included.

1. Including ambient authority in a request can be done by setting the request's [credentials mode](https://fetch.spec.whatwg.org/#concept-request-credentials-mode) to `include`.  (For legacy reasons, [XMLHttpRequest](https://xhr.spec.whatwg.org/) defaults to including this ambient authority for same-origin requests.) Because of [the way CORS works](https://fetch.spec.whatwg.org/#cors-protocol-and-credentials), it is possible for requests that would fail without the ambient authority to succeed with it (because the authority is needed), and also possible for requests that would fail with it to succeed without it (because `Access-Control-Allow-Origin` and `Access-Control-Allow-Credentials` do not allow responses to requests made with credentials to be shared with another origin).  Furthermore, because of security restrictions, it is difficult for a developer who gets these failures to debug them, because most network error details are obscured for cross-origin requests.  (**FIXME**: Add detail here?)  Together, these problems make it difficult for developers to know the best way to access data on the web, and difficult to understand whether failures they encounter are easily fixable. Functionally, a fetch call is now dependent on not only the URL but also the context.

2. It is currently rather difficult to do the server side configuration needed to make data that requires the browser's ambient authority accessible to other origins.  It requires both setting the `Access-Control-Allow-Credentials` header and (the harder part) echoing the `Origin` request header into the `Access-Control-Allow-Origin` response header, since `Access-Control-Allow-Origin: *` is not accepted in this case.  This also requires setting `Vary: Origin`, without which intermittent caching-related failures will occur.

Therefore, given the goal of making it easier to mix sources of open data on the Web and do useful things with the data, the TAG would like these issues to be less of a problem for developers trying to do this.

There's another question that's relevant to consider here:  why are there sites where ambient authority is required to access the data, but the data are actually public and the site is comfortable with them being accessed in mashups hosted on other domains?  Consider the case of Elizabeth hosting data in such a situation, and Alice has credentials to access the data.  In this case an attacker Sid who runs a website that Alice visits can use Alice's visit to the site to extract the data from Elizabeth's site using Alice's credentials.  If Elizabeth is fine with that, then why did Elizabeth require credentials to access the data in the first place?  One common answer is that sites may want to be able to understand who their users are, or be able to contact them (for example, if they cause too much traffic). Another reason would be the use of credentials (instead of custom URLs, like [capability URLs](https://w3ctag.github.io/capability-urls/)) to deliver custom content. (Are there other common reasons?)  It's then worth considering whether these sites expect (or can accept) that someone (Sid, in this example) being able to access the data through somebody else's (Alice's) credentials by getting them (Alice) to visit a site (written by Sid) that steals the data?  Or would these sites consider that a serious security or privacy vulnerability?

## Design criteria

One important design criterion that we don't want to break is that [it is safe](https://annevankesteren.nl/2012/12/cors-101) for any server on the public internet (i.e., not behind a firewall) to send the `Access-Control-Allow-Origin: *` response header on all responses.  This design means that the simple advice to set that header on all public servers can help to make a significant portion of the data on the web shareable without having to do much thinking about security.

## Potential improvements

### Libraries that try both ways

One solution to problem (1) above is a library that tries requests with one credentials mode (either `omit` or `include`) and then, if they fail, retries them with the other one.  This would cover up the difficulty of knowing the right thing to do.  The existence and use of such a library would provide evidence for the importance of this problem.

The main issue is that such library would act blindly on every network error, making lots of unnecessary requests. As a result, such libraries are very unlikely to happen.

### Change the error type when ACAO is `*` and the request is using ambient authority.

By changing the generic "network error" to a better defined one, it would allow libraries to do a retry without ambient authority by looking at the error. Given that the target URL being publicly accessible, there is no high need to hide its presence (which is a property of using a generic error), so the impact on security should be almost nonexistant. 

### `Access-Control-Allow-Origin: *public-deauth*`

Another proposal for problem (1)  is the addition of `Access-Control-Allow-Origin: *public-deauth*`.  This would be similar to `Access-Control-Allow-Origin: *` in terms of safety, but it would also require that if a fetch implementation were going to reject a crossorigin request because the credentials flag is set, it would instead retry the request with the credentials flag unset.  (This could save a roundtrip, relative to the library approach above, if the request were one that required a preflight, since the second preflight could be avoided.)

An issue raised with this proposal is that it's not clear that it's worth the overhead of putting this functionality into implementations given the lack of widespread use of a library that solves problem (1), but see the point about Libraries above.

### `Access-Control-Allow-Origin: *public-auth*`

A proposal for problem (2) is the addition of `Access-Control-Allow-Origin: *public-auth*`, which says that the resource is public even if credentials were used, avoiding the requirement for echoing the `Origin` header into `Access-Control-Allow-Origin` (`*` would be sufficient) and the related need to set the `Vary` header (or face intermittent cache-related failures) and *maybe* also avoiding the need for `Access-Control-Allow-Credentials`.  (**FIXME**: What was the plan here exactly?)

Concerns that have been raised with this proposal are:
* It's potentially quite dangerous since it is making it easy to do an operation that is often unsafe; because that is often unsafe, it was intentionally made difficult (by requiring echoing of the `Origin` header).  It's not as bad as `crossdomain.xml` was, though, since it's per-resource rather than origin-wide.
* It would be good to understand whether the use cases that drive this need really require the use of the ambient authority in the browser.  (Apparently echoing of the `Origin` header is somewhat widespread today, though, so it would be good to understand why.)
