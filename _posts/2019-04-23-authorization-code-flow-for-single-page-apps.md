---
title: "OAuth2 Authorization COde flow for Single Page Apps"
layout: post
date: 2019-04-23
tags:
  - web-development
---

<div class="home-link-container">
  <a class="blog-home-link" href="/blog">Blog Home</a>
</div>

_You can implement the traditional OAuth **Authorization Code Flow** if you have access to both your SPA and backend server.  This may be useful for supporting older OAuth providers._


Recently I was tasked with integrating several 3rd party services into an application.   Most of these services use the OAuth2 protocol to handle authentication and authorization.  I have worked with OAuth many times, but this was my first time integrating OAuth authenticated services into a Single Page Application (SPA) / backend micro services (API and more) architecture.  Having to work with a backend API and SPA introduced some interesting wrinkles.

Single page apps might use the _Implicit Flow_ for OAuth.  The Implicit Flow is usually for client-side / SPA type applications when there is no access to the backend server.  Maybe the app uses something like Firebase as a backend.  The solution outlined in this article is using the _Authorization Grant Flow_.  This approach will not be easy unless one has access / control of the backend server.

__A FEW GROUND RULES:__
- This solution involves some redirecting and hence reloading of the SPA.  If this is problematic or unacceptable for your needs, feel free to jump ship right now.
- The registered redirect URI might not have the same domain as the SPA.
- This solution assumes we are using JWT or some kind of token-based authentication for our own SPA application.

### Implicit or Authorization Code Flow ?

This article is not attempting to address the question of which OAuth2 flow is best for your application.  For SPAs it seems the industry has moved away from the Implicit Flow and over to using Authorization Code flow, with a __public client__.  This flow is slightly different.  No secret authorization code is exchanged (this is in Step 2 - see below).  Since I had access and control over the backend services, I choose to go with the traditional Authorization Code flow.   

Does this all sound like Greek to you ?  If so, do not worry.  We are not getting into the weeds of OAuth2 nor will we disappear down the rabbit hole of arguing about which flow is “the one right way”.  The best solution depends on the goals of your application.  This article is going to describe one solution.  

### Brief OAuth2 Overview

Even if everything written above is completely new to you, chances are you have used OAuth2 enabled applications.  Especially if you’ve used any kind of Single Sign On buttons.  Think about those “Sign in with (Google|Facebook|Github|etc…)” buttons you see all over the place.

When you go to authorize one of these applications, the first thing that happens is that you must authenticate yourself via username and password.

![](https://thepracticaldev.s3.amazonaws.com/i/hgaxbho5fw827t8ut2k3.png)


After you have proven that you are yourself, you must give the OAuth2 application permission to do various things on your behalf.

![](https://thepracticaldev.s3.amazonaws.com/i/7z9qmkdeyqz715760ktx.png)

Once you have gone through the “OAuth2 song and dance”, the response from the OAuth2 provider should include an access token that we can use to access the 3rd party application.  Maybe that includes giving our application access to a user’s Google contacts or emails.  Or giving our application access to a user’s Facebook group.

One important OAuth2 detail.  This redirect URL is something that needs to be registered ahead of time with the OAuth2 provider.  

Here is an example from a Google OAuth2 config page, where localhost is used in the redirect URI.  Note the "Authorized Redirect URIs section".

![](https://thepracticaldev.s3.amazonaws.com/i/6pzuxhs3xwk6arpwk913.png)

_NOTE_: Some OAuth2 providers will not allow "localhost" as a redirect domain.  SSH tunneling / port forwarding can be used for development purposes to get around that issue.

The entire simplified flow looks like this:

![](https://docs.google.com/drawings/d/e/2PACX-1vQnYFLXp2VUBCUxbj5SL9_tXYhrqg_1Q1w7nc-dO6zXyvcJ7UdE0izWjMR2qzdFoGYIOaQThIVXrf9d/pub?w=1440&h=1080)


You could imagine that this takes place somewhere on a configuration or settings page.  When the user decides to "enable" a 3rd party service, behind the scenes the OAuth song and dance is happening.  In the end we have an access token.  And sometimes more...

### Open ID Connect (OIDC)

Oh no!  Another acronym.   In official terminology OIDC is a “Profile” of OAuth2.  That just means, it is like a “flavor” of OAuth2.  What is important about OIDC is that we not only get back an access token, but we also get back an “ID Token”.  This is a unique identifier that can be linked to our user.

When we get the response from our OIDC-enabled provider, we will store the id token and associate it with our user.  In our workflow (see the next image), the user is already “logged in”, thus we can associate the ID Token with our user.

### Single Sign On

The simple OAuth2 flow (described above) has already been handled.   Therefore, we should already have stored the ID Token that was returned from Step 2.  Now we have all the pieces in place, here is the big plan:

![](https://docs.google.com/drawings/d/e/2PACX-1vS9rFUUkIViEfnAzQEXFrdughwuMxz8dScTodDwNeOrYCKhJld9EUlx44EYMPthEFjJSlcmaaFUzsKl/pub?w=1440&h=1080)

Now, let’s start from the beginning.  The user clicks on our SSO button (“Login with Google”).  If the user is already logged into Google, Steps 1 and 2 will happen transparently.  If not, the user will have to authenticate with username / password.

The ID Token from Step 2 can be used to identify our user!

`SELECT * FROM users WHERE <xxxx>.id_token = <ID Token>`  
_This is pseudo-SQL.  This article is not addressing how or where to store tokens_

We now have our user!  Our SPA is stateless and uses its own set of JWT tokens to authenticate requests.  That means, for our user to be “logged in”, their requests need to be accompanied by a valid JWT token each time.

At this point, many cookie/session based solutions could be implemented, instead of this article’s solution.  We could create and stuff the JWT token into the cookie and then the user will be “logged in”.  However, there are some limitations with the cookie-based solution.  For one, it only works easily if the SPA and Backend Server are on the same domain (frontend.myapp.com and api.myapp.com).  There are ways around that challenge, however, our solution bypasses the need for cookies.  We can also avoid other security concerns for cookies with this approach.  Ok, we are not using cookies.  What’s next ?

**Step 3:**  We have identified the user.  Why not just create the JWT and include it with the redirect back to our SPA ?  

It turns out there is a major problem with that approach.  The JWT token can be stolen and reused by nefarious users to impersonate your user.  Therefore, redirecting our initial GET request back to our SPA (Step 3) with the JWT token included in the URL is unsafe.  That URL will be in the browser history among other places.

Since we do not want to redirect back to the SPA with anything sensitive in the URL, we can create a short-lived, one-time-use token.  

**Step 4:**  The SPA will see this token in the URL.  The SPA will immediately make a POST request to the Backend Server with the SSO token.

**Step 5:**  The Backend Server will use the SSO token to identify the user, immediately invalidate the SSO token, and then respond with the JWT token securely in the POST body.  User is “logged in.”


### Good / Bad from this solution

__Bad:__
- Redirects ?  In many SPA situations the backend server has one job: output JSON.  Since we are redirecting here, we break that simple rule.  Now our backend server only outputs JSON…. except for this _one time_.   In our case, I think this sacrifice was worth the gain.
- An extra token to manage.  It _should_ only exist briefly, however, it is another moving piece that can break.

__Good:__
- No cookies !  
- Stateless which is consistent with how many SPAs operate.
- Using Authorization Code flow assures that older OAuth providers (who might not use encrypted data transfers) may only be accessible through this flow.  Implicit flow (and OAuth2 in general) requires encrypted data transfer.  __This was the winning point in choosing this approach__.  It turns out that the project needed to support some smaller OAuth providers who were outdated and did not always adhere to OAuth's security standards.

#### Links:
- [OAuth 2.0 — OAuth](https://oauth.net/2/)
- [OAuth 2 Simplified • Aaron Parecki](https://aaronparecki.com/oauth-2-simplified/)
- This is outlines a similar approach but with a PUBLIC client.   This can be implemented when there is no access to a backend server.  Also great security notes.  [SECURELY USING THE OIDC AUTHORIZATION CODE FLOW AND A PUBLIC CLIENT WITH SINGLE PAGE APPLICATIONS](https://medium.com/@robert.broeckelmann/securely-using-the-oidc-authorization-code-flow-and-a-public-client-with-single-page-applications-55e0a648ab3a)
