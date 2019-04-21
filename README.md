# A developer's introduction to salesƒorce

## Introduction
I've recently been motivated to get a broad overview of what salesƒorce is, the market problems it solves and how developers can go about integrating with it. It struck me that other developers (and interested people with technical backgrounds) would be interested in a nicely structured introduction in the same way that I was.

The thing about Salesforce is that its breadth of possible integration and customisation is staggering. The work that I've been doing has been focussed on integrating with salesƒorce's externally facing APIs in the smallest (yet non-trivial), useful/interesting way possible, and this page is heavily skewed in that direction.

In that spirit, I'll cover:
1. A quick look at what salesƒorce (the company) is and does, with a 50k foot view of the product set
1. salesƒorce's approach to the internal object model, its metadata, and how it's exposed through the sObject API
1. A focussed look at integrating with salesƒorce's APIs
    * The APIs and their documentation
    * Creating a playground instance of salesƒorce for experimenting with APIs
    * Getting access via API to your salesƒorce instance - adding a new Connected App
    * OAUTH and grant types
    * Querying for metadata
    * Querying for data

## This Repo

If you check out this repo and build it, it'll give you a silly web app that you can use to authenticate to your Salesforce instance, and then run some Salesforce APIs to look at the data model.

It uses the OAUTH2 authorization_code flow [(see the appendix, below)](#OAUTH-Appendix) to authenticate, which is the correct way to do a server-to-server integration.

Before this will work for you, you'll have to change the following:
1. `src/main/resources/application.yml`: the top section configures the server to use a self-signed certificate in a PKCS12 key store, and that key store is actually checked in to the repo.  This makes it really convenient to use the app, but an attacker could intercept your traffic - you'd need to generate your own, properly signed certificate.
1. Salesforce client ID and client secret: These are specified as environment variables to the server using the envars `sf-client-id` and `sf-client-secret`. Ideally they'd be in a secure vault and retrieved at startup, but envar is a good second-best.

Bits of code to look at:
* `src/main/java/org/redcabbage/salesforceintro/salesforceauth/LoginController.java`: this is the crux of the auth flow; it does the initial redirect to Salesforce to authenticate, and triggers the subsequent call to get the access code.
* `src/main/java/org/redcabbage/salesforceintro/salesforceauth/SalesForceApiConnectorImpl.java`: wraps calls to the Salesforce APIs; shows the call to translate the auth code into the access code.

## salesƒorce (the company) - what do they do?

salesƒorce is a very large company by anybody's standards.  According to [Owler](https://www.owler.com/company/salesforce), salesƒorce has an annual revenue of $22bn and a headcount of 35,000.  For such a large company they have managed to stay very focussed on their core business: helping businesses manage their relationships with their customers.

This isn't an obvious conclusion to draw if you look at the breadth of their product set, but if you map the major Salesforce products onto the customer lifecycle then it makes a lot of sense:
![Salesforce products vs. product lifecycle](https://docs.google.com/drawings/d/e/2PACX-1vQ0DIkP0FPIZGw-82-igAeiaKb4ZdP3o22mZ-kAfhPq0tPYeqmYvKY9p72NLp_Pokt6qP-8P5HdFRXa/pub?w=971&h=545)
* Sales Cloud **manages sales leads**, right from the first contact through to a closed deal.  It also *provides analytics* for forecasting sales, and performance managing the sales process and team.
* Service Cloud **manages the immediate needs of customers**, from connecting them with each other for mutual assistance through to getting support tickets opened and managed through to closure. It also manages an in-the-field support force (with scheduling and work-order management), as well as automated monitoring of customer health for early prediction of when service might be needed.
* Marketing cloud is a **multi-channel marketing product suite that enables communication to all customers no matter where they are on the customer lifecycle**.  This enables campaigns for
    * Demand and lead generation
    * Lead conversion
    * Automated early-days engagement for new customers
    * Ongoing customer engagement targeted to customers who may be a retention risk
    * Automated, low-cost communication to keep customers with in-flight service tickets informed and close

These three major products tell a tight, holistic story of customer management; of course, the picture isn't quite this clean - Salesforce has other products in addition to these, and the flagship products tend to be used for much more than their CRM use cases! In broad strokes, though, it's a great picture.

## salesƒorce's core platform

Sales Cloud, Service Cloud and a lot of the smaller products are built on Salesforce's core platform. This core platform comprises two major classes of thing:
1. The **data model** defines the business domain of the application, which objects are in the domain and how they relate to each other. For example, the [Sales domain](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_erd_majors.htm) comprises objects such as Accounts, each of which have several Opportunities.  Each Opportunity can be in one several OpportunityStages, and have a number of known OpportunityCompetitors.
1. **Logic** (application behaviour) operates on the objects in the data model to automate processes. Application behaviour can be triggered by user action, external events or by changes to objects. Application behaviour is specified using a combination of declarative, clickable development and actual code (using Salesforce's proprietary Apex programming language).

On top of these two principal parts, there are other layers of technology which make the system more useable:
* An MVC framework to decouple the UI from the underlying data and logic; this enables good UX on multiple platforms
* Declarative vs. code-based logic; the hybrid clickable/codeable approach miimises the cost of development and maintenance
* A detailed set of APIs that allow customers to tightly integrate the platform into home-grown apps; more on this shortly!
* Technology frameworks allowing Salesforce customers to use their apps (including customisations) over mobile platforms
* A Salesforce appstore to enable third-party vendors to write and sell their own third-party apps
...and probably a lot more besides.

The above layers of technology form the basis of most of Salesforce's applications, with some exceptions: Marketing Cloud, for example, was an acquisition of a company named ExacTarget, and their tech was a combination of home-build and acquisition in turn.  While this suite of products has been tightly integrated into the salesƒorce core application set, it doesn't seem to be built on salesƒorce's core platform.

## Integrating with salesƒorce's APIs

Right, let's get on to the interesting bit! When I started looking at Salesforce in depth, it seemed like it would be an interesting exercise to dig around in the core data model using APIs.

Historially, salesƒorce has had most of their APIs available over horrible old SOAP, and they all still exist in that form.  However, a recent development has been salesƒorce Lightning, which as far as I can tell was a rewrite of the UI platform. In addition to a new UI approach, though, it also brought a suite of [REST APIs](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_rest_resources.htm) which are far more friendly than SOAP.

This is the API that I'll be exploring in the remainder of this post.

### Creating a playground instance of salesƒorce for experimenting with APIs

salesƒorce is deeply engineered to be mult-tenant, such that each customer has their own instance of salesƒorce that is separate from other customers' instances. Under the covers they share infrastructure, but from the customer's point of view they might as well have their own salesƒorce instance. salesƒorce documentation often refers to an instance as an "org".

There are two ways to get an instance of your own to play with.  The first is to create a [developer edition account](https://developer.salesforce.com/signup) for yourself, which will give you your own salesƒorce instance to play with.

A better way, though, is through [Trailhead](https://trailhead.salesforce.com). Trailhead is salesƒorce's e-learning site, and it's rammed with free, high-quality, gamified learning materials to get you started with working in salesƒorce. 

Many of the Trailhead exercises require access to a reasonably well set up salesƒorce instance, and so salesƒorce has built a "playground" set of infrastructure into Trailhead that can rapidly and cheaply spin up such salesƒorce instances for use in experimentation and training. These "playgrounds" come ready configured with a set of default accounts, and even a set of default test data to make them a bit interesting.

When you've created a Trailhead account and logged in to it, Trailhead will have created you a playground. Go to your [hands-on orgs management page](https://trailhead.salesforce.com/users/profiles/orgs), and you'll see your playground listed there.  You can just click through to the playground instance and it'll log you straight in - you don't need a password if you click through from Trailhead.

Note that to call any of the APIs on your instance you will actually need your password to authenticate your session!  If you go to ```Setup (gear icon, top right) -> Users``` then you'll see the main admin account listed.  It will have been automatically set up with your email address (ie. your salesforce account name); if you ask to reset the password then it'll send the reset mail to your address.

### The APIs and their documentation

The documentation for core platform REST APIs are available [here](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_what_is_rest_api.htm), and there are also a number of Trailhead trails with good materials.

An interesting starting point to experiment with is the `services/data` API. To call it, you can use `https://{your-instance-name}.lightning.force.com/services/data/` without authentication.  You can get an XML or JSON version of it by using an `Accept` http header.

It will return you a list of the available API versions with each one's root endpoint; to start calling those endpoints you'll need an authenticated HTTP session.  Let's have a look at doing that!

### Getting access via API to your salesƒorce instance 

The Salesforce documentation includes a [really detailed treatment of this topic](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_understanding_authentication.htm).  We'll go through the skinny version here, though.

Before you can call the APIs, you have to have an authenticated HTML session with your Salesforce instance.  I believe this can be done with SAML or Oauth2, but I didn't explore the SAML route at all.

A full treatment of OAUTH2 is clearly outside the scope of this post, but we can cover the minimum bits we need!

OAUTH2 is a protocol that allows a _client_ (ie. a piece of software) to get hold of an _access token_ (a unique key that proves the client is allowed to call a service) on behalf of a _resource owner_ (user, to you and me!).

#### Adding a new Connected App

OAuth2 includes the concept of a client (remember that's the client software - our app.  It's not the user) being pre-registered with the service before it can ask for access tokens. This way the service (Salesforce) can trust that the client (our app) is at least semi-legit because some sort of pre-vetting process has taken place.

To pre-register a client app with your Salesforce instance, go to ```Setup (cog icon, top right) -> Home -> Apps -> App Manager```.  This will list all of the apps already registered to your instance.  You want to add a new one of type "connected", which means that your app will be connecting via APIs. Click "New connected app", top right.

There's very little to set up in the resulting page.  Create a name (whatever you choose) in the "Connected app name"; when you tab out of that field the system will add a sanitised version of it into the "API Name" field for you.

Further down, check the "Enable OAUTH Settings" box.  In the fields that open out, you'll have to supply
* **A callback URL:** this is for more complicated OAUTH2 flows than we'll use here; you can just put a placeholder URL, but make sure it's "https"!  This is a requirement.
* **OAuth scopes:** The permissions that your connected app has. I've not tried to minimise the scope I supplied to my application, although this is clearly desireable.  I moved the "Enable full access" item across; if I write a connected app in anger for production then I'll do the research to figure out what the various scopes are and only enable the minimum.

If you save that page then Salesforce will first warn you that it'll take some time for your app to be connectable, and then it'll give you your new app's setup page.

The two items that you'll need here are the "Consumer Key" and the "Consumer Secret":
1. **The "consumer key"** is a semi-public identifier for your app. It's not a great idea to share it around, as it's part of the credential to authenticate to Salesforce, but it' just an identifier.
1. **The "consumer secret"** is a different story!  This is essentially your app's password to identify itself to Salesforce, and you should treat it exactly like a password.

Don't worry about copying these items anywhere; they'll be available here when you need them.  

Bizarrely, to access these details again you have to go to ```Setup (cog icon, top right) -> Home -> Apps -> App Manager -> drop down by your app -> view```.  Going to ```Setup (cog icon, top right) -> Home -> Apps -> Connected Apps -> Manage connected apps``` takes you to a different page that doesn't show the connection details!

#### Authenticating to Salesforce with the Password grant type

Next, we'll use the OAUth2 password flow to connect to Salesforce. Please note that this is fine for fooling around in scripts like we're doing here, but it's categorically not okay for product use via third-party applications. Please see the section below on OAuth Grant Types for details about why.

I'm going to do this using curl, simply because it shows exactly the HTTP interactions involved and because it means not having to get involved in language and library differences.

First, we need to call the Salesforce instance's authentication endpoint using a POST, passing:
* **A grant_type of "password"**, to specify the OAuth2 flow we want to use
* **our client_id and client_secret** (in Salesforce, the  consumer ID and consumer secret) - to identify our connected app
* **The user's username and password** - to authenticate the user 

So:

```
curl -X POST -d 'grant_type=password&client_id=your_consumer_id&client_secret=your_consumer_secret&username=your_uname&password=passwd'
https://login.salesforce.com/services/oauth2/token
```

This (assuming you've done it right!) will give you an uglier version of the following.  I've pretty-printed it to make it legible.  If you want to do the same, pipe the output of curl through something like `json_pp`.

```
{
    "access_token": "a_long_string_of_odd_characters",
    "instance_url": "https://angry-rodent-r4psqn-dev-ed.my.salesforce.com",
    "id": "https://login.salesforce.com/id/more_chars/odd_chars",
    "token_type": "Bearer",
    "issued_at": "1554670862917",
    "signature": "2zZDrh3PlPht6gl4JHyywsq2Fgf8Bvet75BlefkhLmQ="
}
```

The `access_token` field is the one you want.  If you include that in subsequent API calls as the access token, all will be well - see the next sections!

### Querying for metadata

Right, we're ready to rumble!  Access key in hand, we can query an API for something interesting.  We have to add an HTTP header called `Authorization`, with a value of `Bearer {your-access-token}`:

```
curl -H "Authorization: Bearer {your-access-token}" https://{your-instance}.my.salesforce.com/services/data/v45.0/sobjects/
```

If you run this then you will probably get back an extraordinary amount of text - the "services/data/v45.0/sobjects/" sends you a dump of the whole data model from your saleforce instance, all 150kb of it!  It's interesting for a browse around, and it includes a bunch of relative URLs that you can prepend your Salesforce instance name to and call.  This lets you drill in to the specifics of some of the data types.

### Querying for data

Finally, let's try an API call to retrieve some actual data - how about a list of all Accounts objects in your instance?  This endpoint uses Salesforce's SOQL language to retrieve the names of all the Account objects (which represent sales accounts) in your instance.

```
curl -H 'Authorization: Bearer {your_access_token}' https://curious-shark-r4psqn-dev-ed.my.salesforce.com/services/data/v20.0/query/?q=SELECT+name+from+Account
```
You'll see that each of the returned Account JSON blocks includes a URL that can use to query the whole object for more experimentation.

## Summary

In this post we've covered a lot of ground very quickly! I hope I kept it to the point. The simple points to remember are:

1. Salesforce is based on a relational-style object model, and you can query the data and metadata of the whole object model through some pretty intuitive REST APIs
1. You can create test instances of Salesforce at will to mess around with, either by creating a developer edition account or by spinning up playgrounds
1. Before calling Salesforce APIs you have to register a connected app to get the consumer ID and consumer secret.
1. You can authenticate to your instance of Salesforce using any of a number of OAuth2 flows; `password` is simple but dangerous! Use `authorization_code` instead (see below).
1. After authenticating, include the access code in the Authorization header on subsequent API calls.

I plan to write some actual Java code to create a web app that does this stuff properly, rather than just cobbled together through `curl` calls. That code will be in this repo, so check back soon!

## <a name="OAUTH-Appendix"></a>Appendix: OAUTH grant types

We used basic username and password authentication to get an access key from Salesforce, above.  This is a bad idea in general, but this post has enough in it without opening that particular can of worms! 

However, as an appendix (because it's really important) I've included this section to talk about what we should have done instead.

As Oauth2 is a protocol, it's defined in terms of how the interactions between the client and the server need to proceed - who says what and supplies what, in what order.  OAuth2 has several different "flows", depending on the kind of client (remember that's the client *system*!) that's authenticating.  Each flow is subtly different from the other flows. You specify the flow you're using as part of the initial authentication call by specifying the "grant_type" parameter

The flows that we care about here are as follows:

### Resource Owner Password Credentials flow

Also known as the **password flow** (```grant_type=password```), this enables an application to directly pass a username and password to Salesforce and receive an access code in return. This access code can be used to access Salesforce in subsequent API calls.  This is what we did above.

This approach is very hazardous!
* The client software has full visibility of the user's username and password.
* The client software has full visibility of the user's access token.

This flow is only applicable for first-party applications - applications written and maintained by the user, the user's company or by Salesforce themselves.  The other OAuth2 flows are all designed to enable users to log in and get access tokens without the client system ever seeing the username and password, or even the access code.

### Authorization code flow

The **Web server flow** (```grant_type=authorization_code```) enables a user to use their Salesforce username and password to give a third-party web server the ability to access Salesforce data on their behalf.

For example, imagine we write a cool analytics package that can pull data out of Salesforce and magically predict which deals are likely to close in the next week. Our user would need to enter their Salesforce username and password to enable our app (embodied by our web server, which is being used by our user) to access the user's Salesforce data.

OAuth2 was designed specifically to enable our webserver to get an access token without us ever seeing the user's username and password.  The way this happens is:

1. We forward the user to the Salesforce login page for our Salesforce instance. We supply a URL that Salesforce will forward our user to when they've successfully logged in.
1. The user logs in to Salesforce with their username and password.  Salesforce generates an authorisation code (note - NOT an access code!), and then forwards the user back to us at the URL we supplied (with the authorization code).
1. Our web server receives the authorisation code and makes a server-to-server call to Salesforce to convert the authorization code into an access code. This call includes:
    * Our consumer ID and consumer secret to prove to Salesforce that we're who we say we are.
    * A nonce- and hash-based mechanism to prove to Salesforce that we're the same party that asked for the authorization code in the first place; this prevents an attacker that intercepts the authorization code from converting it into an access code.
1. Salesforce generates the access code and returns it to our web server.  We store this access code in the server-side session data for our user, but we never share it with the user! We only send it to Salesforce when we make API calls.

The advantages of this approach are that the username and password are only ever seen by the user and by Salefore, and th access code (which is highly sensitive) never actually goes to the client's device. This is the flow that we should use in our app!
