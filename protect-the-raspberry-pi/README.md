# Protect the Raspberry Pi

![diagram](diagram.drawio.png)

**Scenario:**
A public web server runs on a Raspberry Pi (RPI) on a home network. A SOHO router forwards all HTTP(S) traffic to the RPI. Legitimate users access the web server using the subdomain `rpi.nicewebsite.net`. This is linked from `nicewebsite.net`, which is an S3 static website with a CloudFront distribution.

**Problem:**
Over 99% of traffic is illegitimate, such as web crawling and GET requests for non-existent files or malicious payloads. Besides the risk of intrusion, the unwanted traffic also takes up bandwidth and fills up logs, making it harder to analyze legitimate traffic.

**Solution:**
Have the RPI reject traffic that doesn't access it via the link in the S3 website. In S3 website, remove any direct references to the RPI's IP or subdomain.

**Implementation:**
In the S3 website, replace the subdomain with `rpi/`. In the CloudFront distribution, add a behavior. Configure it with a path `/rpi*`, a custom origin for the RPI's subdomain, and a custom header `X-Origin-Verify`. The RPI can refuse connections that don't include the header.

## 1. Given

### S3

An S3 bucket `nicewebsite.net` is created in the us-west-2 region and configured as an S3 static website, with the endpoint `nicewebsite.net.s3-website-us-west-2.amazonaws.com`.

The bucket has an object `index.html`, which is used as the default home page. The html code includes this button that links to the RPI endpoint:
```
<button onclick="window.location.href='rpi.nicewebsite.net'">
...
</button>
```

### CloudFront

The CloudFront distribution has the domain name `xxxxxxxxxxxxxx.cloudfront.net`. In Route 53, a public hosted zone for `nicewebsite.net` has an A Alias record that points to the distribution.

The distribution has an Origin of type "S3 static website" with the S3 website's domain. The default Behavior uses this origin, meaning CloudFront looks in the bucket for any files that the client requests.

### RPI

The same Route 53 hosted zone has an A record for subdomain `rpi.nicewebsite.net` that points to the SOHO router's public IP.

The RPI runs an Nginx web server listening on ports 80 and 443, and proxies to a Flask app listening on port 8000.

It's hardened to protect against intrusion attempts and further movement within the network should it become comprimised. 

## 2. Custom Origin

### S3

Update the button code with the path `rpi/`.
```
<button onclick="window.location.href='rpi/'">
...
</button>
```

### CloudFront

Create another origin.
- Name "rpi"
- Origin domain `rpi.nicewebsite.net`

Create another behavior.
- Path pattern `/rpi*`
- Origin "rpi"

## 3. Custom Header

### CloudFront

Add a custom header in the origin.
- Header name: X-Origin-Verify
- Value: 36ed1dda-15ef-11f1-85f7-08bfb876c5cc

The value is a random UUID, which can be generated with a couple lines of python.
```
import uuid
print(uuid.uuid1())
```

### RPI

In the Nginx config file `/etc/nginx/sites-available/rpi.nicewebsite.net` add logic to check for the custom header, and return HTTP status code 403 if it's missing.
```
server {
    listen 443 ssl;
    server_name dronetrain.symbolfigures.io;

    if ($http_x_origin_verify != "36ed1dda-15ef-11f1-85f7-08bfb876c5cc") {
        return 403;
    }

    ...
}
```



