# A developer's introduction to salesƒorce

## Introduction
I've recently had been motivated to get a broad overview of what salesƒorce is, the market problems it solves and how developers can go about integrating with it. It struck me that other developers (and interested people with technical backgrounds) would be interested in a nicely structured introduction in the same way that I was.

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


## salesƒorce (the company) - what do they do?

salesƒorce is a very large company by anybody's standards.  According to [Owler](https://www.owler.com/company/salesforce), salesƒorce has an annual revenue of $22bn and a headcount of 35,000.  For such a large company they have managed to stay very focussed on their core business: helping businesses manage their relationships with their customers.

This isn't an obvious conclusion to draw if you look at the breadth of their product set, but if you map the major Salesforce products onto the customer lifecycle then it makes a lot of sense:
![Salesforce products vs. product lifecycle](https://docs.google.com/drawings/d/e/2PACX-1vQ0DIkP0FPIZGw-82-igAeiaKb4ZdP3o22mZ-kAfhPq0tPYeqmYvKY9p72NLp_Pokt6qP-8P5HdFRXa/pub?w=971&h=545)
* Sales Cloud **manages sales leads**, right from the first contact through to a closed deal.  It also *provides analytics* for forecasting sales, and performance managing the sales process and team.
* Service Cloud **manages the immediate needs of customers**, from connecting them with each other for mutual assistnace through to getting support tickets opened and managed through to closure. It also manages an in-the-field support force (with scheduling and work-order management), as well as automated monitoring of customer health for early prediction of when service might be needed.
* Marketing cloud is a **multi-channel marketing product suite that enables communication to all customers no matter where they are on the customer lifecycle**.  This enables campaigns for
    * Demand and lead generation
    * Lead conversion
    * Automated early-days engagement for new customers
    * Ongoing customer engagement targeted to customers who may be a retention risk
    * Automated, low-cost communication to keep customers with in-flight service tickets informed and close

These three major products tell a tight, holistic story of customer management; of course, the picture isn't quite this clean - Salesforce has other products in addition to these, and these products tend to be used for much more than their CRM use cases! In broad strokes, though, it's a great picture.

## salesƒorce's core platform

The core salesƒorce platform comprises two major classes of thing, which together form the basis of salesƒorce's application portfolio:
1. The **data model** defines the business domain of the application, which objects are in the domain and how they relate to each other. For example, the [Sales domain](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_erd_majors.htm) comprises objects such as Accounts, each of which have several Opportunities.  Each Opportunity can be in one several OpportunityStages, and have a number of known OpportunityCompetitors.
1. **Logic** operates on the objects in the data model to automate processes. Application behaviour can be triggered by user action, external events or by changes to objects.

On top of these two principal parts, there are other layers of technology to make the system more useable:
* An MVC framework to decouple the UI from the underlying data and logic; this enables good UX on multiple platforms
* Declarative vs. code-based logic; a hybrid clickable/codeable approach miimises the cost of development and maintenance
* A detailed set of APIs that allow customers to tightly integrate the platform into home-grown apps; more on this shortly!
* Technology frameworks allowing Salesforce customers to use their apps (including customisations) over mobile platforms
* A Salesforce appstore to enable third-party vendors to write and sell their own third-party apps
...and probably a lot more besides.

The above layers of technology form the basis of most of Salesforce's applications, with some exceptions: Marketing Cloud, for example, was an acquisition of a company named ExacTarget, and their tech was a combination of home-build and acquisition in turn.  While this suite of products has been tightly integrated into the salesƒorce core application set, it doesn't seem to be built on salesƒorce's core platform.

## Integrating with salesƒorce's APIs

Right, let's get on to the interesting bit! 

Historially, salesƒorce has had most of their APIs available over horrible old SOAP, and they all still exist in that form.  However, a recent development has been salesƒorce Lightning, which as far as I can tell was a rewrite of the UI platform. In addition to a new UI approach, though, it also brought a suite of [REST APIs](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_rest_resources.htm) which are far more friendly than SOAP.

This is the API that I'll be exploring in the remainder of this post.

### Creating a playground instance of salesƒorce for experimenting with APIs

salesƒorce is deeply engineered to be mult-tenant, such that each customer has their own instance of salesƒorce that is separate from other customers' instances. Under the covers they share infrastructure, but from the customer's point of view they might as well have their own salesƒorce instance. salesƒorce documentation often refers to an instance as an "org".

There are two ways to get an instance of your own to play with.  The first is to create a [developer edition account](https://developer.salesforce.com/signup) for yourself, which will give you your own salesƒorce instance to play with.

A better way, though, is through [Trailhead](https://trailhead.salesforce.com). Trailhead is salesƒorce's e-learning site, and it's rammed with free, high-quality, gamified learning materials to get you started with working in salesƒorce. 

Many of the Trailhead exercises require access to a reasonably well set-up salesƒorce instance, and so salesƒorce has built a "playground" set of infrastructure into Trailhead that can rapidly and cheaply spin up such salesƒorce instances for use in experimentation and training. These "playgrounds" come ready configured with a set of default accounts, and even a set of default test data to make them a bit interesting.

When you've created a Trailhead account and logged in to it, go to your [hands-on orgs management page](https://trailhead.salesforce.com/users/profiles/orgs), and you should see your playground listed there.  You can just click through to it and it'll log you straight in - you don't need a password if you click through from Trailhead.

Note that to fool around with APIs you will actually need your password!  If you go to Setup (gear icon, top right) -> Users then you'll see the main admin account listed.  It will have been set up with your email address (ie. your salesforce account name); if you ask to reset the password then it'll send the reset mail to that address.

### The APIs and their documentation

The documentation for core platform REST APIs are available [here](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_what_is_rest_api.htm), and there are also a number of Trailhead trails with good materials.

An interesting starting point to experiment with is the `services/data` API. To call it, you can use `https://{your-instance-name}.lightning.force.com/services/data/` without authentication.

It will return you a list of the available API versions with each one's root endpoint; to start calling those endpoints you'll need an authenticated HTTP session.

### Getting access via API to your salesƒorce instance - adding a new Connected App

### OAUTH and grant types

### Querying for metadata

### Querying for data
