# lambda-cloudfront-cookies

> AWS Lambda to protect your CloudFront content with username/passwords
> NOTE: This is a less secure fork of [thumbsup/lambda-cloudfront-cookies](https://github.com/thumbsup/lambda-cloudfront-cookies), with all the KMS encryption stripped out. This version just stores the private key as an environment variable, unencrypted which is typically [a bad idea](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/). Use with caution.

## How it works

The first step is to have CloudFront in front of your S3 bucket.

```
Browser ----> CloudFront ----> S3 bucket
```

We then add a Lambda function responsible for logging-in.
When given valid credentials, this function creates signed session cookies.
CloudFront will verify every request has valid cookies before forwarding them.

```
Browser                   CloudFront             Lambda              S3
  |                           |                    |                 |
  | ---------- get ---------> |                    |                 |
  |                           |                    |                 |
  |                      [no cookie]               |                 |
  |                           |                    |                 |
  |                           |                    |                 |
  |                           |                    |                 |
  | <------ error page ------ |                    |                 |
  |                                                |                 |
  | -------------------- login ------------------> |                 |
  | <------------------- cookies ----------------- |                 |
  |                                                                  |
  | ---------- get ---------> |                                      |
  |                           |                                      |
  |                      [has cookie]                                |
  |                           |                                      |
  |                           | -----------------------------------> |
  |                           | <------------ html page ------------ |
  | <------ html page ------- |
```

## Pre-requisites

### 1. CloudFront key pair

- Logging in with your AWS **root** account, generate a CloudFront key pair
- Take note of the key pair ID
- Download the private key

### 3. Htpasswd

- Create a local `htpasswd` file with your usernames and passwords. You can generate the hashes from the command-line:

```
$ htpasswd -nB username
New password: **********
Re-type new password: **********
username:$2a$08$eTTe9DM5N0w50CxL5OL0D.ToMtpAuip/4TCSWCSDJddoIW9gaQIym
```

## Deployment

Create a configuration file called `dist/config.json`, based on [config.example.json](config.example.json).
Make sure you don't commit this file to source control (the `dist` folder is ignored).

NOTE: AWS Lambda functions do not play well with environment variables that contain newlines. To workaround,
this function will translate the character sequence `\n` to a newline.

It should contain the following info - minus the comments:

```js
[
  // -------------------
  // PLAIN TEXT SETTINGS
  // -------------------

  // the website domain, as seen by the users
  "websiteDomain=website.com",
  // how long the CloudFront access is granted for, in seconds
  // note that the cookies are session cookies, and will get deleted when the browser is closed anyway
  "sessionDuration=86400",
  // if false, a successful login will return HTTP 200 (typically for Ajax calls)
  // if true, a successful login will return HTTP 302 to the Referer (typically for form submissions)
  "redirectOnSuccess=true",
  // CloudFront key pair ID from step 2
  // This is not sensitive, and will be one of the cookie values
  "cloudFrontKeypairId=APK...",

  // Unencrypted CloudFront private key from step 2
  "cloudFrontPrivateKey=-----BEGIN RSA...",

  // Unencrypted contents of the <htpasswd> file from step 3
  "htpasswd=username:lksdjfalk..."
]
```

You can then deploy the full stack using:

```bash
export AWS_PROFILE="your-profile"
export AWS_REGION="ap-southeast-2"

# name of an S3 bucket for storing the Lambda code
./deploy my-bucket
```

The output should end with the AWS API Gateway endpoint:

```
Endpoint URL: https://0000000000.execute-api.ap-southeast-2.amazonaws.com/Prod/login"
```

Take note of that URL, and test it out!

```bash
# with a HTTP Form payload
curl -X POST -d "username=hello&password=world" -H "Content-Type: x-www-form-encoded" -i "https://0000000000.execute-api.ap-southeast-2.amazonaws.com/Prod/login"

# with a JSON payload
curl -X POST -d "{\"username\":\"hello\", \"password\":\"world\"}" -H "Content-Type: application/json" -i "https://0000000000.execute-api.ap-southeast-2.amazonaws.com/Prod/login"
```

Final steps:

- optionally setup CloudFront in front of this URL too, so you can use a custom domain like `https://website.com/login`
- once everything is working, change your CloudFront distribution to require signed cookies
and it will return `HTTP 403` for users who aren't logged in
- setup CloudFront to serve a nice login page for `403` errors, or use an existing page from your website to trigger the Lambda function
