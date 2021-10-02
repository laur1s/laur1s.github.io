---
title: "Replace Lambda@Edge With Cloudfront Functions"
date: 2021-09-09T16:53:11+03:00
draft: false
---

In my previous [post]({{< ref "/posts/cloudfront-functions.md" >}} "Cloudfront Functions") I discussed about newly released Cloudfront functionality - Cloudfront Functions and Compared them to Lambda@Edge.

In this post I will show how to migrate Cloudfront distribution that uses Lambda@Edge to  set custom security headers to Cloudfront Functions using Terraform. I've discussed differences of these two AWS functions previously so I'm not going go into that and I'll jump straight to the code.

## Replacing Lambda@Edge with Cloudfront Functions using Terraform

Replacing Lambda@Edge with Cloudfront Functions using Terraform is quite straightforward.
You simply have to create a equivalent Cloudfront function. Luckily AWS provides bunch of examples of [commonly used functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-example-code.html)

```js
//function.js
function handler(event) {
    var response = event.response;
    var headers = response.headers;

    // Set HTTP security headers
    // Since JavaScript doesn't allow for hyphens in variable names, we use the dict["key"] notation
    headers['strict-transport-security'] = { value: 'max-age=63072000; includeSubdomains; preload'};
    headers['content-security-policy'] = { value: "default-src 'none'; img-src 'self'; script-src 'self'; style-src 'self'; object-src 'none'"};
    headers['x-content-type-options'] = { value: 'nosniff'};
    headers['x-frame-options'] = {value: 'DENY'};
    headers['x-xss-protection'] = {value: '1; mode=block'};

    // Return the response to viewers
    return response;
}
```
This javascript code modifies cloudfront responses and adds security headers.

Next, simply add Cloudfront Function Terraform resource:

```Terraform
resource "aws_cloudfront_function" "response" {
  name    = "cf-response-security-headers"
  runtime = "cloudfront-js-1.0"
  comment = "Function that adds Cloudfront Security Headers"
  publish = true
  code    = file("${path.module}/function.js")
}
```

Finally, modify the Cloudfront distribution, comment out `lambda_function_association` and add `function_association` block:

```Terraform
resource "aws_cloudfront_distribution" "example" {
  # ... other configuration ...


    # Comment out or delete lambda function association
    # lambda_function_association {
        #   event_type   = "origin-response"
        #   include_body = false
        #   lambda_arn   = aws_lambda_function.this.qualified_arn
        # }


    function_association {
      event_type   = "viewer-response"
      function_arn = aws_cloudfront_function.response.arn
    }
  }
}
```

I think is good idea not to remove the Lambda Function resource straight away as you might need it, in case something goes wrong with Cloudfront Function.

After running terraform plan you can see that Cloudfront Function will be added and associated with Cloudfront Distribution and Lambda Function association will be removed:

```Terraform
  + function_association {
              + event_type   = "viewer-response"
              + function_arn = (known after apply)
            }

    - lambda_function_association {
                - event_type   = "origin-response" -> null
                - include_body = false -> null
                - lambda_arn   = "arn:aws:lambda" -> null


    + resource "aws_cloudfront_function" "response" {
```

Next, simply run terraform apply and your cloudfront distribution will start to use cloudfront function to modify the response headers.

## Gotchas when working with Terraform and Cloudfront functions

Since, Terraform associates Cloudfront function with a Cloudfront distribution you won't be able to do any actions that force Terraform to delete and recreate a function e.g.: Change the function's name. In order to fix that, you can add Terraform lifecycle policy to create a new function and associate it with Cloudfront before destroying old one.


```Terraform
resource "aws_cloudfront_function" "response" {
    ###
    lifecycle {
        create_before_destroy = true
    }
}
```

## Conlusion

That's it for this short blog post. As you can hopefully see replacing Lambda@Edge with Cloudfront Functions is quite straightforward. If there's anything else I'd notice while working with CF Functions I will update this post.

## Update

After using Cloudfront functions for a while I realized that there's an important limitation to it that's is not obvious without digging into AWS documentation:
>CloudFront does not invoke edge functions for viewer response events when the origin returns HTTP status code 400 or higher.

That makes CF Functions not suitable for adding headers to  any Frontend code that uses Angular, react, Vue, or any similar framework.

For example, if a React application is deployed on S3 Cloudfront first tries to look for an object in S3 instead of using React router. E.g.: for url `example.com/login/` it tries to look for `login` file in the s3 bucket. When the file is not found Cloudfront receives error (usually 403) and then you have to setup a redirection in Cloudfront to redirect all 403 errors to `index.html`  After setting up this redirection, React router works fine but since S3 origin still returns 403 error code no headers could be set on the file.
Therefore, if you host any application that uses Javascript based routing you can unfortunately limited to Lmabda@Edge.


You can read about all Cloudfront function restrictions [here](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-restrictions.html#edge-function-restrictions-all). It's important to learn about these limitations because they can limit you to use Cloudfront Functions.
