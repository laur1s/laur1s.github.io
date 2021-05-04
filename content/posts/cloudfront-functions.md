---
title: "Modifying AWS Cloudfront response headers with Cloudfront functions"
date: 2021-05-04T21:51:56+03:00
---
Adding custom HTTP response headers to cloudfront is quite a common task. E.g.: Your website might need a [Content-Security-Policy header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy).

Doing this task in Cloudfront was not easy. You needed to setup a Lambda@Edge function that intercepts response requests and adds the required headers. It\'s not that difficult to setup Lambda@Edge but using Lambda functions to add a simple HTTP header to a response feels like an overkill. I even wrote about this when the Cloudfront team asked for suggestions:
{{< tweet 1329134380183392256 >}}
I was quite surprised when this morning I heard about a new feature called CloudFront Functions. This feature is supposed to make tasks like adding a custom HTTP header simpler. Let\'s see how it works!

## Cloudfront Functions vs Lambda@Edge

Cloudfront Functions are actually quite similar to Lambda@Edge. The main difference is that Cloudfront Functions run at Cloudfront Edge locations. Wile despite the word **Edge** in Lambda@Eddge it runs at specific AWS regions.  Lambda@Edge supports Node.js while Lambda@Edge allows running javascript(ECMAScript 5.1 so `let` and `const` keywords are not supported). I\'m a bit surprised on why AWS decided to use older Javascript standard. Maybe since the functions are supposed to be lightweight and short that's fine. Since Cloudfront functions run at Cloudfront edge locations and use more lightweight runtime it should be faster than Lambda@Edge. Cloudfront Functions also have some limitations like no network or file system access, limited execution time, etc. Adding a custom response headers doesn\'t require any of that so let\'s do that using a Cloudfront function.

## Adding custom security header using a Cloudfront Function

The Cloudfront function that we will build will intercept the response from the Cloudfront origin(s3) and will modify it by adding a content-security-policy header.

I prefer to use [Terraform](https://www.terraform.io/) to define all my AWS resources as code. At this time the feature is not supported yet. However, AWS go sdk already supports Cloudfront Functions so it should be available in Terraform soon. Despite AWS promise to make their services available in AWS Cloudformation from day 1 cloudformation is not supported too. So let\'s try to create a cloudfront function using AWS console.

I already have a Cloudfront distribution set up with S3 bucket as origin. Inside S3 bucket I have a simple index.html page. If I run `curl` command now I can see the following response:
```sh
$ curl -i d33vf9zl6um5ej.cloudfront.net
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 63
Connection: keep-alive
Date: Tue, 04 May 2021 16:25:59 GMT
Last-Modified: Tue, 04 May 2021 16:24:27 GMT
ETag: "1cf70af742cc5258f37ac61772a0554f"
Accept-Ranges: bytes
Server: AmazonS3
X-Cache: Hit from cloudfront
Via: 1.1 2f7792bdc67f7953e2dce93aea1bb9ee.cloudfront.net (CloudFront)
X-Amz-Cf-Pop: ARN54-C1
X-Amz-Cf-Id: Vr8Td6stRKhJVAk2rA5XoitG_eKKlm3_TJq-Aob1isHfrF5hQ_HoSQ==
Age: 6

<!DOCTYPE html>
<html>
<body>
  <h1>Hello</h1>
</body>
</html>
```

Let\'s add a custom Content Security Policy(CSP) header that controls what resources and from where the browser is allowed to load from. It\'s used for preventing Cross Site Scripting attacks.

First  let\'s go to Cloudfront Console and create a new function:
{{< figure src="/images/cf-1.png" title="Cloudfront Functions Console" >}}

Let\'s give our function a name `add-scp-headers` and click Continue.

Now let\'s replace the example code with this:

```js
function handler(event) {
    var response = event.response;
    //add SCP header to the response
    response.headers['content-security-policy'] = {value: "default-src 'self'"};
    return response;
}
```

It simply modifies the cloudfront response and adds the CSP header to the response.

 After clicking save, let\'s click on the Test tab.

 Since this function modifies requests after they reach origin let\'s select Viewer response as event type. After clicking test we can see that content-security-policy header is successfully added as a response header.
{{< figure src="/images/cf2.png" title="Cloudfront Function Test" height="500px">}}

Let\'s click publish to publish a new version of your function.

Finally, click Associate, select your distribution Viewer Response as an event type and click add association.
{{< figure src="/images/cf3.png" title="Cloudfront Function Association" >}}

After using curl on your cloudfront distribution you should receive this:
```sh
curl -i d33vf9zl6um5ej.cloudfront.net
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 63
Connection: keep-alive
Date: Tue, 04 May 2021 16:25:59 GMT
Last-Modified: Tue, 04 May 2021 16:24:27 GMT
Etag: "1cf70af742cc5258f37ac61772a0554f"
Accept-Ranges: bytes
Server: AmazonS3
Via: 1.1 990c1aa70667fe4e8f93d88ac8400fc5.cloudfront.net (CloudFront)
Content-Security-Policy: default-src 'self'
X-Cache: Hit from cloudfront
X-Amz-Cf-Pop: ARN54-C1
X-Amz-Cf-Id: 18gD7OMeCKCE-SieGM2ScxOSTgeBQaSXXYdvQ5L6VvmBXDb8j_s69w==

<!DOCTYPE html>
<html>
<body>
  <h1>Hello</h1>
</body>
</html>
```

## Conclusion
It\'s not a solution what I expected when I tweeted about custom security headers. I thought it could be possible to simply have a section in cloudfront console where you can specify the custom headers without needing to write any Javascript code.

However, Cloudfront functions is still a more convenient tool than Lambda@Edge if you need to do simple tasks like modifying response or request headers. It\'s more simple to use and provides simplified debugging, deployment, and monitoring options. I would like to be able to migrate all my Lambda@Edge code to Cloudfront Functions but as of now it\'s not supported by any infrastructure as a code tool. Personally, I think that having multiple ways to work with Cloudfront(Lambda@Edge and CF Functions) adds some complexity but CF functions looks like a great tool when you don\'t need full capabilities of Lambda and also provides a nicer developer experience.

Update:

I wanted to write this blogpost as I found no resources on how to add security headers using Cloudfront functions. As of now AWS also have a very cool page with example [cloudfront functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-example-code.html).
