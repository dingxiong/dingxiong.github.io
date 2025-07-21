---
layout: post
title: AWS -- Authentication and Authorization
date: 2025-07-20 16:00 -0700
categories: [aws]
tags: [aws]
---

## Two Cache Locations

As an AWS sso user, I find two cache locations for authentication/authorization
`~/.aws/sso/cache` and `~/.aws/cli/cache`. Example contents follow as below.

```
$ cat ~/.aws/sso/cache/4fbbfb5d40afa2f001d7c45107652897179a34ad.json | jq .
{
  "startUrl": "https://d-9a6772b5aa.awsapps.com/start",
  "region": "us-east-2",
  "accessToken": "...",
  "expiresAt": "2025-07-21T06:54:05Z",
  "clientId": "qGLAlyjg3Fz26xGjHyARG3VzLWVhc3QtMg",
  "clientSecret": "...",
  "registrationExpiresAt": "2025-10-18T22:54:00Z"
}

$ cat ~/.aws/cli/cache/8357a0f61cba55874869cd6b6190f15ffa3ad8c0.json | jq .
{
  "ProviderType": "sso",
  "Credentials": {
    "AccessKeyId": "...",
    "SecretAccessKey": "...",
    "SessionToken": "..."
    "Expiration": "2025-07-21T10:56:20Z",
    "AccountId": "242230929264"
  }
}
```

`~/.aws/cli/cache` is where the AWS CLI stores temporary credentials, such as
those obtained when using an IAM role with temporary security credentials. This
caching mechanism improves performance by avoiding the need to re-authenticate
with every command. On the other hand, `~/.aws/sso/cache` is where the AWS CLI
stores cached access tokens for AWS IAM Identity Center (formerly AWS SSO)
logins.

To run a regular aws-cli command, the authentication flow is that

1. If the cache `~/.aws/cli/cache/<cache-key>.json` exits and is not expired,
   then use the `AccessKeyId` and the `SecretAccessKey` in the cache directly.
   We are done.
2. Otherwise, we need to generate a new `AccessKeyId` and `SecretAccessKey`
   pair. To do this, we need prove that we are a true human that has the
   permission to do it. This part is controlled by the AWS IAM Identity Center.
   If the cache `~/.aws/sso/cache/<cache-key>.json` exists and is not expired,
   then aws-cli will use it to generate the `AccessKeyId` and `SecretAccessKey`
   pair automatically.
3. If not, then we need to do `aws sso login` to authenticate with IAM Identity
   center first.

One immediate question is how the `cache-key` is generated. For sso cache, the
logic is
[here](https://github.com/aws/aws-cli/blob/9bb6a90ea6ed98f382ffedce67120809b94e3873/awscli/botocore/utils.py#L3742).
Basically, it is only determined by sso `startUrl` or `session_name`. A little
bit background here. There are two ways to configure sso.

```
[profile admin]
sso_start_url = https://my-sso-portal.awsapps.com/start
sso_region = us-east-2
sso_account_id = ...
sso_role_name = admin
region = us-east-2
```

or

```
[sso-session my-sso]
sso_region = us-east-2
sso_start_url = https://my-sso-portal.awsapps.com/start

[profile admin]
sso_session = my-sso
sso_account_id = ...
sso_role_name = admin
region = us-east-2
```

So the sso cache can be used across multiple accounts. A common misconception
is that we need to do `aws sso login --profie <profile-name>` for each profile
or account. This is absolutely wrong. Realizing this fact helps me save a lot
of time! We can do a little verification here.

```
In [4]: import hashlib
In [7]: input_str = 'https://d-9a6772b5aa.awsapps.com/start' # My employer's sso
In [8]: hashlib.sha1(input_str.encode('utf-8')).hexdigest()
Out[8]: '4fbbfb5d40afa2f001d7c45107652897179a34ad'
```

`4fbbfb5d40afa2f001d7c45107652897179a34ad` is what I show at the top, right?
Also, this cache is not associated with the machine. You can happily copy this
hash file to a remote box to use it. No need to sso in the remote box!

For cli cache, the cache key is generated
[here](https://github.com/aws/aws-cli/blob/9bb6a90ea6ed98f382ffedce67120809b94e3873/awscli/botocore/credentials.py#L2226).
It is determined by `roleName`, `accountId` and `startUrl`. So this cache is
(account, role) specific. Let's verify it

```
# I masked out the account id below.
# Note, the accountId should be a string, not a number.

In [16]: args = {'roleName': 'admin', 'accountId': '...', 'startUrl': 'https://d-9a6772b5aa.awsapps.com/start'}
In [17]: args = json.dumps(args, sort_keys=True, separators=(',', ':'))
In [18]: argument_hash = sha1(args.encode('utf-8')).hexdigest()
In [19]: argument_hash
Out[19]: '8357a0f61cba55874869cd6b6190f15ffa3ad8c0'
```

That is what is shown above!

One nasty thing about the `aws-cli` repo is that the default branch is
`develop`, but this branch is not the latest branch, neither is `master`
branch. Instead, the latest release branch contains the latest code. Branch
`develop` and `master` lag way behind. This wasted me at 30 min when I was
digging around this repo.

Another side note is that if you install `aws-cli` using homebrew in macos,
then it does not use the site package in the global python 3rd party folder. It
is inside somewhere like
`/opt/homebrew/Cellar/awscli/2.27.32/libexec/lib/python3.13/site-packages/awscli/`.
