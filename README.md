# VaultBase — Broken Access Control Lab

> A hands-on simulation of one of the most common and dangerous vulnerabilities in web security.  
> Built for learning. Referenced from [PortSwigger Web Security Academy](https://portswigger.net/web-security/access-control).

---

## What Is This Project?

VaultBase is a fake "secure asset manager" — the kind of dashboard a fintech startup might build to let users view their investment portfolio. It looks polished, it has a login page, it has different user roles, and it has a restricted admin panel.

The catch? **The restriction doesn't actually work.**

Any regular user — without knowing the admin password, without any hacking tools, just by editing the URL — can walk right into the admin panel and see everything: internal secrets, all user accounts, and destructive controls.

This is not a made-up scenario. This exact class of vulnerability is responsible for some of the biggest data breaches in history. It sits at **#1 on the OWASP Top 10** list of web application security risks.

---

## The Core Idea — What Is Access Control?

Think of a hotel. Every guest gets a keycard. That keycard opens their room and the gym, but not the manager's office or the server room. The hotel's security system enforces those boundaries — it's not just about hiding the manager's door from guests.

In web apps, **access control** is the exact same idea:

- **Authentication** = Proving who you are (your keycard)
- **Authorization** = Deciding what you're allowed to do (which doors your keycard opens)

Most developers get authentication right. A login form with hashed passwords is standard. What often goes wrong is authorization — specifically, *where* that check happens.

---

## The Two Worlds: Client vs. Server

Every web application lives in two places simultaneously.

**The Client** is your browser — the thing you can see, click, and directly interact with. JavaScript runs here. The URL bar lives here. LocalStorage lives here. Crucially, you — the user — have *full control* over everything on the client side. You can open DevTools, edit JavaScript, change stored values, and type any URL you want.

**The Server** is the machine in a data center somewhere. It receives requests and decides what to send back. You cannot directly touch the server. You can only *ask* it for things.

This distinction is everything in security. If a rule only exists in your browser, you can break it. If a rule exists on the server, only the server enforces it — and you have no way around it.

---

## What VaultBase Does Wrong

When you log in, your session is stored in the browser's `localStorage` like this:

```json
{
  "username": "wiener",
  "role": "user",
  "display": "Wiener"
}
```

When you try to visit the admin panel, the app runs a check:

```js
function route() {
  const hash = window.location.hash.replace('#', '') || 'login';

  if (!session) {
    navigate('login');  // Not logged in? Go back.
    return;
  }

  // Checks if the page exists — but NEVER checks your role
  if (['login', 'dashboard', 'admin'].includes(hash)) {
    navigate(hash);  // Any logged-in user reaches any page
  }
}
```

See the problem? The function asks one question: *"Are you logged in?"*

It never asks the second, more important question: *"Are you **allowed** here?"*

So the flow for a regular user trying to access `/admin` looks like this:

```
Are you logged in?   →  YES
Is #admin a valid page?  →  YES
Are you an admin?    →  [question was never asked]
                         ↓
                    FULL ACCESS GRANTED
```

---

## The Hidden-Link Trap

Here's something clever (and deceptive) about how VaultBase is built — the Admin Panel link in the navigation bar is **invisible to regular users**:

```js
document.getElementById('nav-admin').style.display =
  session.role === 'admin' ? 'inline' : 'none';
```

At first glance this seems like a security measure. Regular users don't see the link, so how would they even know the page exists?

This thinking is called **Security Through Obscurity** — the idea that hiding something is enough to protect it. It is not. It has never been enough.

A page isn't protected because its link is hidden. It's protected because the **server refuses to serve it** to unauthorized users. Hiding the link is like covering a door handle with a curtain — the door is still unlocked.

Any attacker can:
- Type the URL directly
- Look at the page's JavaScript source code to find all routes
- Use browser DevTools to inspect the navigation logic
- Simply guess common admin paths like `/admin`, `#admin`, `/dashboard/admin`

---

## What the Admin Panel Exposes

Once inside, the unprotected admin panel reveals:

| Data Exposed | Real-World Impact |
|---|---|
| All user accounts + emails | Identity exposure, targeted phishing |
| Database connection string | Direct database compromise |
| JWT signing secret | Forge any user's authentication token |
| API master key | Full programmatic control of the system |
| Encryption salt | Weaken or reverse password hashing |
| Delete/reset user controls | Account takeover, service disruption |

In a production system, this is a full breach. One URL. No tools. No exploit code.

---

## The Fix — Server-Side Enforcement

The correct approach is simple in principle: **the server must verify your role on every single request, independently, using its own trusted data.**

Here is what the broken code looks like versus the secure version:

```js
// ❌ BROKEN — The browser decides who's an admin
// An attacker can modify localStorage and make themselves admin
function route() {
  const user = JSON.parse(localStorage.getItem('session'));
  if (user.role === 'admin') {
    showAdminPage();
  }
}
```

```js
// ✅ SECURE — The server decides, and the browser has no input
// Express.js example with middleware
function requireRole(role) {
  return (req, res, next) => {
    const user = getUserFromToken(req.headers.authorization);
    if (!user || user.role !== role) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

// The admin route is blocked before the page is ever sent
app.get('/admin', requireRole('admin'), (req, res) => {
  res.render('admin');
});
```

The key difference: in the broken version, the check happens in code the user controls. In the secure version, the check happens in code the user can never touch.

---

## Attack Walkthrough — Step by Step

Here's the complete path to exploit VaultBase:

**Step 1 — Log in as a regular user**
```
Username: wiener
Password: peter
```
You land on the Dashboard. You see your portfolio, your holdings, nothing unusual.

**Step 2 — Observe the nav bar**
No admin link is visible. A developer might think: "good, the user can't find it."

**Step 3 — Edit the URL**
Change the hash from `#dashboard` to `#admin`.

**Step 4 — Access granted**
The browser runs `route()`, confirms you're logged in, confirms `admin` is a valid page name, and renders the admin panel — no role check ever fires.

**Step 5 — Exfiltrate data**
All user records, internal credentials, and privileged controls are now visible and usable.

Total time: under 10 seconds. No special tools required.

---

## Why This Happens in Real Projects

This vulnerability is so common because of how web apps are typically built:

1. A developer builds the frontend first — they add a role check in JavaScript because it's fast and easy.
2. The feature works in testing — admins see the admin panel, users don't.
3. The developer ships it, assuming the UI restriction equals security.
4. Nobody thinks to test: *"What if I just type the admin URL directly?"*

The mistake isn't stupidity — it's a gap in mental model. **Hiding something is not the same as securing it.**

---

## OWASP & PortSwigger References

This vulnerability is formally documented as:

- **OWASP Top 10 — A01:2021 Broken Access Control**
  The most common web application vulnerability class. Occurs when users can act outside their intended permissions.

- **CWE-284 — Improper Access Control**
  The Common Weakness Enumeration entry covering failures to restrict access to resources.

- **PortSwigger Web Security Academy**
  Full interactive labs on this exact topic:
  https://portswigger.net/web-security/access-control

---

## Test Credentials

| Username | Password | Role |
|---|---|---|
| `wiener` | `peter` | Regular user |
| `carlos` | `montoya` | Regular user |
| `administrator` | `admin123` | Admin |

The admin credentials are intentionally not hidden — the point is that you shouldn't **need** them to reach the admin panel, and that's exactly the vulnerability.

---

## Key Takeaways

**Never trust the client.** The browser is in the hands of the user. Any check that lives only in JavaScript can be bypassed.

**Authorization ≠ hiding links.** A page must be blocked at the server level, not just made invisible in the UI.

**Every route needs its own check.** It's not enough to verify a user's role once at login. Each request to a sensitive resource must independently verify permission.

**Assume attackers know your URL structure.** Obscurity is not security. If a route exists, assume it will be found.

---

*Built as a mini-project to demonstrate Broken Access Control vulnerabilities. For educational use only.*
