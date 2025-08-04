# Vercel Shield

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A simple, drop-in middleware firewall to protect your Vercel apps from API abuse and rate-limit bypasses.

---

## The Problem

Vercel's security infrastructure has a flaw: it blindly trusts requests coming from a Cloudflare Worker, allowing attackers to bypass the WAF and rate limits. This can be used to:

*   **Run up your API bills** by forcing thousands of unauthorized, billable calls.
*   **Shut down your app** with a Denial of Service attack.

This affects any Vercel project using metered APIs (like AI providers or Vercel's own KV store).

## The Solution

This middleware acts as a smart firewall. It runs before your application code and correctly identifies the **real user's IP address**, blocking abusive traffic before it can do any harm.

### Features

*   **Smart IP Rate Limiting:** Correctly blocks abusive traffic, even from a proxy.
*   **Trusted Key Bypass (VIP Pass):** Allows your own trusted services (like a webhook) to bypass the rate limit by providing a secret key.
*   **Easy Configuration:** All settings are in one simple config object.
*   **Effectively Free:** Uses Vercel KV's free tier (500,000 requests/month).

---

## Getting Started: Protect Your App in 3 Minutes

#### 1. Install KV Package

In your project's terminal, run:
```bash
npm install @vercel/kv
```

#### 2. Connect Database

In your Vercel dashboard, go to the **Storage** tab for your project.
*   Select **"Upstash for Redis"** and connect the **free-tier** database.
*   Vercel will automatically create and link the necessary environment variables to your project.

#### 3. Create the Firewall

In the **root** of your project, create a new file named `middleware.ts`. Copy the code from the `middleware.ts` file in this repository and paste it into your new file.

That's it! Your application is now protected. Just push your changes to redeploy.

---

## Configuration

You can easily customize the firewall by editing the `FIREWALL_CONFIG` object at the top of your `middleware.ts` file.

```typescript
// --- MAIN CONFIGURATION ---
const FIREWALL_CONFIG = {
  
  // A smart rate limit for all untrusted public traffic.
  ipRateLimit: {
    limit: 30,
    window: 20, // in seconds
  },

  // A list of secret tokens for your trusted services.
  // Any request with 'Authorization: Bearer <token>' will bypass the IP rate limit.
  trustedSecrets: [
    process.env.MY_WORKER_SECRET || '', // Example of using an env var
  ],

  // An array of paths to exclude from all firewall protections.
  // For example: ['/', '/about']
  allowedPaths: [] as string[], 

};
```
*   **`ipRateLimit`**: Sets the request limit (`limit`) per time window (`window` in seconds).
*   **`trustedSecrets`**: Add your own long, random secret keys here.
*   **`allowedPaths`**: Add any public paths you want to exclude from the firewall.

### Using Trusted Secrets (The VIP Pass)

If you have your own service (like another Cloudflare Worker or a webhook) that needs to call your Vercel app without being rate-limited, you must make it send the secret key in an `Authorization` header.

Here is an example of how to do this in a Cloudflare Worker:

```javascript
// Inside your own "good" Cloudflare Worker...

// Get the secret from your worker's environment variables
const MY_SECRET_TOKEN = env.MY_SECRET_TOKEN;

const response = await fetch('https://your-vercel-app.vercel.app/api/some_endpoint', {
  method: 'POST',
  headers: {
    // This is the VIP Pass
    'Authorization': `Bearer ${MY_SECRET_TOKEN}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ your: 'data' })
});
```

### ⚠️ Security Best Practice: Protect Your Secrets

**Do NOT hardcode your secrets directly in the `middleware.ts` file.** This is a major security risk. Instead, you should always store them in **Environment Variables**.

**For Local Development:**
1.  Create a file named `.env.local` in the root of your project.
2.  Add your secrets to this file:
    ```
    MY_WORKER_SECRET=k$tF_8#Z!pQvY7@eR_wJ6x*uG9hL4m&
    STRIPE_WEBHOOK_SECRET=s3C!bN_9^zP_qR$tW*vXyZ_2&4@6*8(0
    ```
3.  Make sure `.env.local` is listed in your `.gitignore` file so you never accidentally commit it.

**For Production (Vercel):**
1.  Go to your project's **Settings -> Environment Variables** in the Vercel dashboard.
2.  Add the same secrets there.

Your middleware can then access them safely like this: `process.env.MY_WORKER_SECRET`.

## License

This project is licensed under the **MIT License**. See the `LICENSE` file for details.
