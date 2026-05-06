# Personal Portfolio Site

A static personal portfolio site hosted on AWS, demonstrating a production-grade pattern for serving static content globally with HTTPS, low latency, and a private origin.

**Live site:** https://d3chtckz98igx7.cloudfront.net

\---

## Architecture

!\[Architecture diagram](./architecture.png)

A request flows through three AWS services:

1. The user's browser requests the page from the **CloudFront** distribution URL.
2. **CloudFront** serves the page from the nearest edge location. If it isn't cached, CloudFront fetches it from the origin.
3. The origin is a private **S3 bucket**. Direct access is blocked at the bucket level — only this specific CloudFront distribution can read from it, enforced by an Origin Access Control (OAC) and a bucket policy.

HTTPS is provided by CloudFront's default certificate.

\---

## Services used

* **Amazon S3** — stores the static HTML file. Bucket is private; public access is blocked.
* **Amazon CloudFront** — CDN that caches and serves the site globally over HTTPS, redirecting all HTTP traffic to HTTPS.
* **Origin Access Control (OAC)** — modern replacement for OAI; grants CloudFront permission to read from the private bucket while keeping the bucket otherwise inaccessible.
* **IAM** — the bucket policy defines a single allowed principal (the CloudFront service) restricted to this distribution's ARN.

\---

## Key configuration decisions

* **Bucket is private, not a "static website" endpoint.** This is the modern, secure pattern. The S3 static-website endpoint is HTTP-only and exposes the bucket publicly; using CloudFront + OAC instead gives HTTPS and origin lockdown.
* **Origin Access Control over the legacy Origin Access Identity.** OAC supports SigV4 signing and all S3 features in all regions; OAI is being phased out.
* **`Redirect HTTP to HTTPS` viewer protocol policy.** Any HTTP request gets a 301 redirect to HTTPS — there's no plain-HTTP path to the content.
* **Default root object set to `index.html`.** Without it, requests to the root URL (`/`) return an error instead of the home page.
* **Bucket region: `eu-central-1` (Frankfurt).** Lowest latency to the primary expected user (me, in Poland) for content updates. CloudFront caches globally regardless.

\---

## What I learned

* **Default-deny is the default for a reason.** S3's "Block all public access" stays on, and CloudFront is granted access narrowly through a bucket policy that names this exact distribution. Anything broader is a footgun.
* **CloudFront is a global service** — the region selector locks to "Global" while you're working with it. ACM certificates that are *attached* to CloudFront have to live specifically in `us-east-1`, regardless of where everything else is. Worth knowing if I add a custom domain later.
* **Cache invalidation matters.** After updating `index.html` in S3, CloudFront still serves the old cached version until the cache expires (default TTL is hours). For active development, a `/index.html` invalidation forces CloudFront to refetch.
* **Filename hygiene is a real thing.** Windows hides "known file extensions" by default, which means an HTML file saved through Notepad can silently end up as `index.html.html` and break the site. Turning on "File name extensions" in File Explorer is now part of my standard setup.

\---

## What I would do differently

* **Add a custom domain.** The CloudFront URL works but isn't memorable. Buying a domain at a third-party registrar and pointing it at CloudFront via Route 53 (or directly via CNAME) would make this look like a real personal site.
* **Set up CI/CD for deployments.** Right now, updating the site means manually re-uploading to S3 and creating an invalidation. A simple GitHub Actions workflow on push to `main` could sync the bucket and invalidate the CloudFront cache automatically.
* **Tighten the cache policy.** The default `CachingOptimized` policy is fine for static HTML but uses long TTLs. For a site I plan to update frequently, a shorter TTL or an explicit cache-busting strategy on filenames would prevent stale-content surprises.
* **Add monitoring.** A CloudWatch alarm on 4xx/5xx error rates from the distribution would tell me if something breaks in production before I notice manually.

\---

## Deploying this yourself

The high-level steps:

1. Save `index.html` to a private S3 bucket in any region (with "Block all public access" enabled).
2. Create a CloudFront distribution with the bucket as the origin. In the new console wizard, tick "Allow private S3 bucket access to CloudFront" — this auto-creates the OAC and updates the bucket policy.
3. In the distribution settings, set **Default root object** to `index.html` and **Viewer protocol policy** to **Redirect HTTP to HTTPS**.
4. Wait 5–15 minutes for CloudFront to deploy, then visit the distribution URL.

Total cost at idle traffic: effectively $0 within AWS Free Tier and Always Free allowances.

\---

## About me

IT Support Specialist with 8+ years of international experience, AWS Certified Cloud Practitioner, transitioning into cloud-focused roles. This project is the first of three AWS portfolio projects.

* AWS Certified Cloud Practitioner — issued May 2026

