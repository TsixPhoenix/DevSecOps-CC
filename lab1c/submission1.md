# Lab 1 Submission

## Task 1 - OWASP Juice Shop Deployment

### Triage Report - OWASP Juice Shop

#### Scope & Asset
- Asset: OWASP Juice Shop (local lab instance)
- Image: bkimminich/juice-shop:v19.0.0
- Release link/date: https://github.com/juice-shop/juice-shop/releases/tag/v19.0.0 - 2025-09-04
- Image digest (optional): sha256:2765a26de7647609099a338d5b7f61085d95903c8703bb70f03fcc4b12f0818d

#### Environment
- Host OS: Windows 10.0.26200 (win32)
- Docker: 28.3.3

#### Deployment Details
- Run command used: `docker run -d --name juice-shop -p 127.0.0.1:3000:3000 bkimminich/juice-shop:v19.0.0`
- Access URL: http://127.0.0.1:3000
- Network exposure: 127.0.0.1 only [x] Yes  [ ] No
- Container status: `docker ps` shows `127.0.0.1:3000->3000/tcp` and container is Up

#### Health Check
- Page load: added screenshot of home page, lab1c/artifacts/juice-shop-home.jpg
- API check: `/rest/products` returned "Unexpected path" on v19.0.0.
  - Verified product list via `curl -s http://127.0.0.1:3000/rest/products/search?q=` (first 10 lines):
```
{
  "status": "success",
  "data": [
    {
      "id": 1,
      "name": "Apple Juice (1000ml)",
      "description": "The all-time classic.",
      "price": 1.99,
      "deluxePrice": 0.99,
      "image": "apple_juice.jpg",
```

#### Surface Snapshot (Triage)
- Login/Registration visible: [x] Yes  [ ] No
- Product listing/search present: [x] Yes  [ ] No
- Admin or account area discoverable: [x] Yes  [ ] No
- Client-side errors in console: [x] Yes  [ ] No
- Security headers (quick look - optional): `curl -I http://127.0.0.1:3000`
  - Present: `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Feature-Policy: payment 'self'`
  - Observed: `Access-Control-Allow-Origin: *`
  - Not present: `Content-Security-Policy`, `Strict-Transport-Security` (HTTP, local)

#### Risks Observed (Top 3)
1) Missing CSP/HSTS in response headers increases XSS and transport risk for real deployments.
2) Public product API is accessible without auth, enabling data enumeration and scraping.
3) HTML content in product descriptions suggests potential for stored XSS if not sanitized in UI.

## Task 2 - PR Template Setup

### Creation Process
- Added `.github/pull_request_template.md` with required sections and checklist.

### Verification
- added proof-screenshot, lab1c/artifacts/auto-filled-PR.jpg

### Why Templates Help Collaboration
PR templates standardize expectations, reduce review back-and-forth, and ensure important checks (tests, docs, artifacts) are not skipped.

## Challenges & Solutions
- Challenge: `/rest/products` endpoint returned "Unexpected path" on v19.0.0.
- Solution: Used `/rest/products/search?q=` for API verification and documented the deviation.

## GitHub Community
Starring repositories signals useful projects to the community and helps maintainers gauge interest, while following developers keeps you informed about relevant work and strengthens collaboration.