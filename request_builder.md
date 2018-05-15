To know what you want in a secure way, S3 requires a signature. The signature is created by:

1. assembling a canonical requst
2. putting together some parts of the request together with a hashed version of the full request - this is the `string to sign`
3. creating a signing key
4. using the signing key to create a signature from the `string to sign`


**Step 1**

Create a canonical request first. S3 uses this as its canonical request:
```
<HTTPMethod>\n            # the HTTP method, such as GET, PUT, HEAD, and DELETE.
<CanonicalURI>\n          # the part of the url between the host and the end of the string (or the ?, if you have query parameters)
<CanonicalQueryString>\n  # anything after the ?, if you have query parameters. If no query, include an empty string ("")
<CanonicalHeaders>\n      # list of request headers, with their values, such as host, Content-Type, and any x-amz-* headers. 
<SignedHeaders>\n         # alphabetically sorted, semicolon-sorted list of the request headers we just used
<HashedPayload>           # the hexidecimal value of the SHA256 has of the request payload.
````

Some notes:

* If you are using temporary security credentials, you will include x-amz-security-token in your request, and include the header in the list of `CanonicalHeaders`.
* If POSTing, the data to be POSTed is put in the payload. If GETting, use an empty string.

**Step 2**

Now we use that canonical request to create a string to sign. The string to sign is a concatenation of the following pieces:
```
"AWS4-HMAC-SHA256" + "\n" +
timeStampISO8601Format + "\n" +
<Scope> + "\n" +
Hex(SHA256Hash(<CanonicalRequest>))
````

The `scope` is make up of `date.Format(<yyyyMMdd>) + "/" + <region> + "/" + <service> + "/aws4_request"`.

**Step 3**

Now we calculate the signature. First we have to create a signing key. The key is made up of the following items:
```
DateKey              = HMAC-SHA256("AWS4"+"<SecretAccessKey>", "<yyyymmdd>")
DateRegionKey        = HMAC-SHA256(<DateKey>, "<aws-region>")
DateRegionServiceKey = HMAC-SHA256(<DateRegionKey>, "<aws-service>")
SigningKey           = HMAC-SHA256(<DateRegionServiceKey>, "aws4_request")
```

Amazon has thoughtfully written up a key creation function in Ruby.

```
def getSignatureKey key, dateStamp, regionName, serviceName
    kDate    = OpenSSL::HMAC.digest('sha256', "AWS4" + key, dateStamp)
    kRegion  = OpenSSL::HMAC.digest('sha256', kDate, regionName)
    kService = OpenSSL::HMAC.digest('sha256', kRegion, serviceName)
    kSigning = OpenSSL::HMAC.digest('sha256', kService, "aws4_request")

    kSigning
end
```

Now, we can create the final signature, which is, in pseudo-code:
```
HMAC-SHA256(SigningKey, StringToSign)
```

**All at once, now**

An example, from Amazon, using the object `test.txt` from `examplebucket`:

```
GET /test.txt HTTP/1.1
Host: examplebucket.s3.amazonaws.com
Date: Fri, 24 May 2013 00:00:00 GMT
Authorization: SignatureToBeCalculated
Range: bytes=0-9 
x-amz-content-sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
x-amz-date: 20130524T000000Z 
```

Creating the Canonical Request, which will become the last line of the string to sign:
```
GET                                     # HTTPMethod
/test.txt                               # Canonical URI
                                        # Canonical Query String (empty, because there's no query)
host:examplebucket.s3.amazonaws.com     # Canonical Header
range:bytes=0-9                         # Canonical Header
x-amz-content-sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855       # Canonical Header
x-amz-date:20130524T000000Z             # Canonical Header

host;range;x-amz-content-sha256;x-amz-date  # Signed Headers
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855    # Hashed empty string

```

The string to sign, then, is:
```
AWS4-HMAC-SHA256                            # Algorithm we're using
20130524T000000Z                            # Date and time of request
20130524/us-east-1/s3/aws4_request          # Scope
7344ae5b7ee6c3e7e6b0fe0640412a37625d1fbfff95c48bbb2dc43964946972    # Hash of the Canonical Request
```

Creating the signing key:
```
signing key = HMAC-SHA256(HMAC-SHA256(HMAC-SHA256(HMAC-SHA256("AWS4" + "<YourSecretAccessKey>","20130524"),"us-east-1"),"s3"),"aws4_request")
```
which results in
```
f0e8bdb87c964420e857bd35b5d6ed310bd44f0170aba48dd91039c6036bdb41
```

Finally, we create the header:
```
AWS4-HMAC-SHA256 Credential=AKIAIOSFODNN7EXAMPLE/20130524/us-east-1/s3/aws4_request,SignedHeaders=host;range;x-amz-content-sha256;x-amz-date,Signature=f0e8bdb87c964420e857bd35b5d6ed310bd44f0170aba48dd91039c6036bdb41
```

Done!




But...

If we want to ask S3 for things using the URL itself, we'll need to use query parameters to authenticate the request. This is called a "presigned URL". Presigned URLs allow you to grant temporary access to S3 resources, which is especially good if, for example, you want to serve images from S3 on a non-S3 hosted site.

S3 presigned URLs are constructed very similarly to the above way of requesting resources. There are a few differences in how we create the canonical request that goes into the signature:

* You don't include a payload hash in the Canonical Request, because when you create a presigned URL, you don't know anything about the payload. Instead, you use a constant string "UNSIGNED-PAYLOAD".
* The CanonicalQueryString must include several query parameters: `X-Amz-Algorithm`, `X-Amz-Credential`, `X-Amz-Date`, `X-Amz-Expires`, and `X-Amz-SignedHeaders`. The `X-Amx-Credential` is similar to the `<scope>` in step 2 above, but is in the format of `<your-access-key-id>/<date>/<AWS-region>/<AWS-service>/aws4_request`. (When creating the request in the URL, `/`s need to be in the form of `%2F`.
* Canonical Headers must include the HTTP host header. If you plan to include any of the x-amz-* headers, these headers must also be added for signature calculation. You can optionally add all other headers that you plan to include in your request. For added security, you should sign as many headers as possible.

So, requesting `test.txt` from `examplebucket` request would end up going like this:

Canonical Request
```
GET
/test.txt
X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20130524%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20130524T000000Z&X-Amz-Expires=86400&X-Amz-SignedHeaders=host
host:examplebucket.s3.amazonaws.com

host
UNSIGNED-PAYLOAD
```
You'll notice the Canonical Query String has a bunch of information we didn't need in the first `test.txt` request version.

The Canonical Request goes into the string to sign, as the hash on the last line:
```
AWS4-HMAC-SHA256
20130524T000000Z
20130524/us-east-1/s3/aws4_request
3bfa292879f6447bbcda7001decf97f4a54dc650c8942174ae0a9121cf58ad04
```

Creating the signing key:
```
signing key = HMAC-SHA256(HMAC-SHA256(HMAC-SHA256(HMAC-SHA256("AWS4" + "<YourSecretAccessKey>","20130524"),"us-east-1"),"s3"),"aws4_request")
```
gets us
```
aeeed9bbccd4d02ee5c0109b86d86835f995330da4c265957d157751f604d404
```

And, finally, our presigned URL would be:
```
https://examplebucket.s3.amazonaws.com/test.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20130524%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20130524T000000Z&X-Amz-Expires=86400&X-Amz-SignedHeaders=host&X-Amz-Signature=aeeed9bbccd4d02ee5c0109b86d86835f995330da4c265957d157751f604d404
```