---
title: "OLD AWS Cloudfront response headers with Cloudfront functions"
date: 2021-05-04T13:51:56+03:00
draft: true
---
Adding custom response headers to cloudfront is quite a common task. E.g.: Your website might need a [Content-Security-Policy header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy).

Doing this task in Cloudfront was not easy. You needed to setup a Lambda@Edge function that intercepts either request or response requests and adds the required headers. It\'s not that difficult to setup Lambda@Edge but using Lambda functions to add a simple http header to a response feels like an overkill. Actually, I wrote about this in my tweet when Cloudfront team asked for feature suggestions:
{{< tweet 1329134380183392256 >}}
I was quite surprised when this morning I finally heard about new Feature called CloudFront Functions.

## Cloudfront Functions vs Lambda@Edge

Cloudfront Functions are actually quite similar to Lambda@Edge. The main difference is that Cloudfront Functions run at Cloudfront Edge locations while despite having the word **Edge** in their name Lambda@Edge runs at specific AWS regions.  Lambda@Edge supports Node.js  while Lambda@Edge allows running javascript(ECMAScript 5.1 so `let` and `const` keywords are not supported). I\'m a bit surprised on why AWS decided to use such old Javascript standard but maybe since the functions are supposed to be lightweight and short it\'s OK. Honestly, it\'s not a solution I expected. I thought it could be possible to simply have a section in cloudfront console where you can specify the custom headers without needing to write any Javascript: Lambda or Cloudfront Function. Since Cloudfront functions run at Cloudfront edge locations and use more lightweight runtime it should be faster than Lambda@Edge. Cloudfront Functions also have some limitations like no network or file system access, limited execution time, etc. Adding a custom response headers doesn\'t require any of that so let\'s do that using a Cloudfront function.

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

Before writing our own code let\'s try to run the example code that is kindly provided by cloudfront team:

```js
function handler(event) {
    // NOTE: This example function is for a viewer request event trigger.
    // Choose viewer request for event trigger when you associate this function with a distribution.
    var response = {
        statusCode: 200,
        statusDescription: 'OK',
        headers: {
            'cloudfront-functions': { value: 'generated-by-CloudFront-Functions' }
        }
    };
    return response;
}
```

Not sure why they chose to provide an example that completely rewrites the response instead of simply modifying headers(If by accident I applied this example to my production cloudfront it will start returning this custom response instead of content). Let\'s click save and publish to publish this function. Finally click associate, select your distribution and Viewer request as an Event Type:
{{< figure src="/images/cf2.png" title="Cloudfront Function association" >}}

Now when I try to use curl on my distribution instead of my `index.html` page I get this empty response with some cloudfront headers:
```sh
curl -i d33vf9zl6um5ej.cloudfront.net
HTTP/1.1 200 OK
Server: CloudFront
Date: Tue, 04 May 2021 17:06:36 GMT
Content-Length: 0
Connection: keep-alive
Cloudfront-Functions: generated-by-CloudFront-Functions
X-Cache: FunctionGeneratedResponse from cloudfront
Via: 1.1 bc362383b5c95fa821ce42f151e2a4aa.cloudfront.net (CloudFront)
X-Amz-Cf-Pop: ARN54-C1
X-Amz-Cf-Id: 0UrFohRsd3uEUcS3knuqJcTSUFhqugTs4DeR5Ros3jgR5HRZkDD19Q==
```


Let\'s remove this function association by clicking Remove association.
Now let\'s finally modify our code to to modify the cloudfront response and include the CSP header:

```js
function handler(event) {
    var response = event.response;
    //add SCP header to the response
    response.headers['content-security-policy'] = {value: "default-src 'self'"};
    return response;
}
```

 After clicking save, let's click Test.

 Since this function modifies requests after they reach origin let's select Viewer response as event type. After clicking test we can see that content-security-policy header is successfully added as a response header.
{{< figure src="/images/cf3.png" title="Cloudfront Functions Test">}}

Let\'s click publish to publish a new version of your function. Finally, click associate, select your distribution Viewer Response as an event type and click add association. After using curl on your cloudfront distribution you should receive this:
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

Cloudfront functions is a great tool if you need to do simple tasks like modifying response or request headers. It\'s more simple to use than Lambda@Edge and also provides simplified debugging, deployment, and monitoring options. I would like to be able to migrate all my lambda@edge code to Cloudfront Functions but as of now it\'s not supported by any infrastructure as a code tool.

Update:

I wanted to write this blogpost as I found no resources on how to add security headers using Cloudfront functions. As of now AWS also have a very cool page with example [cloudfront functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-example-code.html).
