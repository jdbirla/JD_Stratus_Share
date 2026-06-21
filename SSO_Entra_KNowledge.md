# SSO understanding for redirect and react to BFF to Entra id

The confusion comes from thinking that **the browser needs to identify itself to Entra ID**.

It doesn't.

The **browser is just transporting the user between websites**. The **OIDC client is your Spring Boot backend**, not the browser and not the React app.

Let's walk through it carefully.

## Think of it like visiting a bank

Imagine you want to access a secure office building:

1. You arrive at the office reception (**your React app**).
2. The receptionist says, "Please get your identity verified by the security desk" (**Spring Boot**).
3. The security desk sends you to the central company identity office (**Microsoft Entra ID**).
4. The identity office checks whether you already have a company badge.
5. If you do, you walk straight through.
6. The identity office sends you back to the security desk with proof of your identity.
7. The security desk creates a visitor pass for this building.
8. You enter the office.

The identity office doesn't need to know anything about the reception area. It only knows the security desk.

---

## What Entra ID actually knows

When you register your application, Entra ID stores something like this:

```text
Client ID: abc-123

Redirect URI:
https://api.company.com/login/oauth2/code/entra
```

That's all it needs.

It does **not** need to know:

* Your React URL
* Your React routes
* Your React components

---

## What the browser actually does

Let's say your user opens:

```text
https://app.company.com
```

The browser loads your React application.

When the user clicks **SSO Login**, React redirects the browser to:

```text
https://api.company.com/login
```

Spring Boot responds:

```http
HTTP/1.1 302 Found
Location: https://login.microsoftonline.com/tenant/oauth2/v2.0/authorize?client_id=abc-123&redirect_uri=https://api.company.com/login/oauth2/code/entra&scope=openid+profile&response_type=code
```

The browser sees the `Location` header and navigates there.

The browser is not "talking as React."

It's simply following redirects.

```text
Browser ───► api.company.com/login
           ◄── 302 Redirect

Browser ───► login.microsoftonline.com
```

---

## How Entra ID knows where to send the user back

Look at the authorization URL:

```text
https://login.microsoftonline.com/.../authorize?
    client_id=abc-123
    &redirect_uri=https://api.company.com/login/oauth2/code/entra
```

Entra ID reads:

* `client_id`
* `redirect_uri`

It verifies that the redirect URI matches the one registered for that client.

If it matches, Entra ID accepts the request.

After authentication, Entra ID sends the browser to:

```text
https://api.company.com/login/oauth2/code/entra?code=xyz
```

Again, the browser simply follows the redirect.

---

## Where does the SSO experience come from?

When the browser arrives at:

```text
https://login.microsoftonline.com
```

it automatically sends cookies for that domain:

```text
login.microsoftonline.com
```

If the user already logged in to Teams or Outlook, the browser already has those cookies.

```text
Browser ───► login.microsoftonline.com
            (includes Entra session cookie)
```

Entra ID sees:

> "This user is already authenticated."

No credentials are required.

---

## The complete flow

```text
1. Browser opens app.company.com

2. React redirects to:
   api.company.com/login

3. Spring Boot redirects to:
   login.microsoftonline.com/authorize?client_id=abc-123&redirect_uri=https://api.company.com/login/oauth2/code/entra

4. Browser automatically sends Entra cookies.

5. Entra ID finds existing session.

6. Entra ID redirects browser to:
   api.company.com/login/oauth2/code/entra?code=xyz

7. Spring Boot exchanges code for tokens.

8. Spring Boot creates its own session cookie.

9. Spring Boot redirects browser to:
   app.company.com

10. React calls:
    api.company.com/api/me

11. Browser sends Spring Boot session cookie automatically.
```

The important point is:

* **Entra ID knows only your Spring Boot application.**
* **The browser knows all the URLs because it follows redirects.**
* **React is not part of the OIDC protocol.**

React only starts the login process and later calls your backend APIs.
