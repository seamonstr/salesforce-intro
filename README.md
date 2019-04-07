# A developer's introduction to salesƒorce

## Introduction
I've recently had been motivated to get a broad overview of what salesƒorce is, the market problems it solves and how developers can go about integrating with it. It struck me that other developers (and interested people with technical backgrounds) would be interested in a nicely structured introduction in the same way that I was.

The thing about Salesforce is that its breadth of possible integration and customisation is staggering. The work that I've been doing has been focussed on integrating with salesƒorce's externally facing APIs in the smallest (yet non-trivial), useful/interesting way possible, and this page is heavily skewed in that direction.

In that spirit, I'll cover:
1. A quick look at what salesƒorce (the company) is and does, with a 50k foot view of the product set
1. salesƒorce's approach to the internal object model, its metadata, and how its exposed through the sobject API
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
