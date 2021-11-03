---
title: "Cloudfront Response Headers Policies"
date: 2021-11-03T00:14:03+02:00
---

Another day, another blog post about adding security headers to Cloudfront HTTP responses. Actually, it's my third post about this topic, which is the same as the number of AWS services that can be used to modify Cloudfront headers. As of today we have: Lambda@Edge, Cloudfront Functions, and the newly introduced Response Headers Policies

Again, ability to easily add HTTP headers to Cloudfront was very commonly requested feature:
{{< tweet 1329134380183392256 >}}

 Almost all Cloudfront distributions require custom security headers and adding them with Lambda@Edge simply wasn't a very elegant solution. So, you can imagine how happy I was when I heard that AWS announced Cloudfront Functions. I tried and [wrote]({{< ref "/posts/cloudfront-functions.md" >}})  about this feature. However, it was not what I expected. Firstly, it still requires writing and  maintaining a piece of code(although extremely simple). Secondly, it is not suitable for javascript applications (react/angular/vue) hosted on S3 (I explained why in my [2nd post]({{< ref "/posts/cloudfront-functions.md#further-reading" >}})).

 Therefore, after migrating my Lambda@Edge to Cloudfront Functions, security headers weren't added as expected. I checked with AWS support but they just confirmed that this functionality is by design and I was forced to go back to Lambda@Edge.

## Enter a new era with Cloudfront Response Headers Policies

As of today Cloudfront Response Headers Policies are also available. Looking  through the [launch blog](https://aws.amazon.com/blogs/networking-and-content-delivery/amazon-cloudfront-introduces-response-headers-policies/) it looks like it should be able to add response headers even if Cloudfront origin returns HTTP 403 status code. But it's time to check if it actually works!

As a starting point I have already a cloudfront distribution with a custom error response for 403 error codes:
{{< figure src="/images/cf/1.jpg" >}}
It's not really required but I also have a Cloudfront function that  associated with my CF distribution from when I was testing it. Cloudfront Response Headers Policies can be used together with Cloudfront/Lambda@Edge function and are applied before them. Therefore, I decided to keep the CF function. Now when I run curl on my CF distribution I get the current results:
```sh
# Curl on index.html security headers added by CF Function as expected
# Not relevant HTTP headers were redacted from the responses
$ curl -I https://d33vf9zl6um5ej.cloudfront.net/
HTTP/2 200
server: AmazonS3
content-security-policy: default-src 'self'
# Curl on object that doesn't exist on s3, no security header added
# by CF Function.
$ curl -I https://d33vf9zl6um5ej.cloudfront.net/sfdsfs
HTTP/2 200
server: AmazonS3
```

As you can see, Cloudfront Function adds the response security header only when object exists in the s3 origin. Therefore, security header is not added to the second response.

Let's check if adding Cloudfront Response Headers Policies(OK, I will call it RHP from now) would help. I can associate an RHP with a Cloudfront distribution by clicking on the Behaviors tab and then clicking edit:
{{< figure src="/images/cf/2.jpg" >}}
There, I can see the available RHPs. Let's select SecurityHeadersPolicy that should add all the required security headers:
{{< figure src="/images/cf/3.jpg" >}}
Next, let's save the changes to see if it actually helps with our security headers. Again let's use curl for testing:

```sh
# Not relevant HTTP headers were redacted from the responses
$ curl -I https://d33vf9zl6um5ej.cloudfront.net/
HTTP/2 200
x-xss-protection: 1; mode=block
x-frame-options: SAMEORIGIN
referrer-policy: strict-origin-when-cross-origin
x-content-type-options: nosniff
strict-transport-security: max-age=3153600
content-security-policy: default-src 'self'

$ curl -I https://d33vf9zl6um5ej.cloudfront.net/sfdsfs
HTTP/2 200
x-xss-protection: 1; mode=block
x-frame-options: SAMEORIGIN
referrer-policy: strict-origin-when-cross-origin
x-content-type-options: nosniff
strict-transport-security: max-age=3153600
```

We can see that the security headers are successfully added for both responses. However, as you can see the second response doesn't include the content-security-policy header. This happens because the managed SecurityHeadersPolicy leaves this header empty.
{{< figure src="/images/cf/4.jpg" >}}

To fix this problem, I can simply create my own custom Security Headers Policy and assign it to my distribution. After doing that, I can finally get rid of the Cloudfront Function code because it's no longer needed!

## Summary

It looks like AWS listened to feedback and provided a really simple and intuitive way to add HTTP headers to Cloudfront responses/requests. It was very easy to setup and it worked even for scenarios where Cloudfront functions failed me.

I haven't tested the performance of this but it should probably be faster then lambda@edge and maybe even that CF functions.

It was very productive week for me and this is my third blog post this week. If you liked the post please check two other posts: [how to block access to certain aws regions and limit your blast radius]({{< ref "/posts/limit-access-to-aws-regions-with-iam.md" >}})  and another post where I go through the [differences between RDS read replicas and Multi-AZ deployments] ({{< ref "/posts/rds-multi-az-vs-read-replica.md" >}}).
