---
draft: false
title: "A better way of receiving emails within end-to-end testing"
snippet: "Many websites that I’ve worked on send emails. For example, user registrations, password resets, form submissions, and so forth. And ideally, this functionality needs to be tested. Within unit- and integration tests it’s easy to swap out the email service with a test double (mock/stub/fakes), but when doing end-to-end testing this becomes more of a challenge, and you should want to test the real deal, right?"
image: {
    src: "_d44aff32-c942-40e7-9b90-3fbc21326017.jpeg",
    alt: "full stack web development"
}
publishDate: "2023-01-08 11:39"
category: "Tutorial"
author: "Johan Benschop"
tags: [webdev, tailwindcss, frontend]
---

Many websites that I’ve worked on send emails. For example, user registrations, password resets, form submissions, and so forth. And ideally, this functionality needs to be tested. Within unit- and integration tests it’s easy to swap out the email service with a test double (mock/stub/fakes), but when doing end-to-end testing this becomes more of a challenge, and you should want to test the real deal, right?

## A brittle way to test

Often, I’ve seen testers using Selenium (a web browser puppeteer) to go to Outlook or Gmail, sign in using some test account they have set up beforehand, wait for the email to appear and grab it. There are a couple of problems with this approach. Primarily, this makes the test brittle as we’re mainly testing the third-parties email client, and that is subject to change at any time. Often, we also have the one inbox, and emails are bound to get mixed up if multiple tests are running. Gmail has this excellent feature where everything after the + is automatically an alias. For example, john+random᠎@gmail.com will work as if the part after the plus-sign isn’t there, which can be used to mitigate these mix-ups. And lastly, it just makes the test needlessly slower.

## A better way to test

Observing this begged the question, there must be some service out there that can receive the emails and make them available through an API? As I couldn’t find such a service that I liked enough (admittingly, I didn’t look all that hard), I spent two evenings coming up with a custom solution. It turned out to be surprisingly simple, zero lines of code simple.

The requirements for my solution are the following:

*   Support for a catch-all for near-infinite email addresses
*   Emails must be accessible via an API
*   No misbehaving spam filters
*   A custom domain name for the email addresses

I first looked at getting an SMTP mail server running on Azure, but I quickly discovered that this would require a VM, and that’s not the way. Looking further, I saw the Azure docs describing the use of SendGrid for sending emails, but I need to receive, not send them. But as luck have it, SendGrid also supports receiving emails; it’s just a wee bit hidden when looking at their website.

Receiving emails with SendGrid requires setting up an [inbound parse webhook](https://sendgrid.com/docs/for-developers/parsing-email/setting-up-the-inbound-parse-webhook/ "Setting Up The Inbound Parse Webhook"). The idea is that when SendGrid receives an email, they trigger a webhook to POST the email to. This webhook then needs to parse this request and store the email somewhere to access them with an API. I decided to use Azure Logic Apps for the webhook implementation and Azure Table Storage to store the emails.

## This is the way

Let’s back up for a minute. Why is this important? At the customers where I observed this brittle way of testing last, they have several services that work together to send emails. There is one service for sending emails like the password reset and activate account emails. This service depends on another service that generates those one-time tokens used in those emails. There is also a service used for the templates, which looks up content in a content management system (CMS). Once the message is ready, it gets thrown on a queue, where yet another service hands it off to the local email server. Oh, and the form builder skips the template service and goes directly to the CMS. And these services need to be configured correctly for them to talk to each other. There’s more going on, but you should get the idea. The point is that there are many points of failure here, and while there are some unit- and integration tests around this, end-to-end tests are essential too. And we like those tests to be reliable.

## Follow the way

The game plan is to have the system-under-test send the email to \[random\]@mail.example.org. The DNS of this subdomain is configured to point to Sendgrid. On inbound emails, Sendgrid triggers the Azure Logic App webhook. This Logic App then stores the contents of that email to an Azure Table Storage table. The test runner polls this table to verify that the email was sent correctly.

![mermaid-diagram-20210508222643](//images.ctfassets.net/9gzi1io5uqx8/3cQPyNaoogfaziFaJ17oq9/132dce89251d9c7ebccc440fa18bebbf/mermaid-diagram-20210508222643.svg?fit=scale&w=825)

For the rest of the article, I will assume that you have a basic understanding of Azure and how to configure DNS records. You will need an Azure subscription and access to the DNS of a domain name if you want to follow along.

### Create the Storage Account

The first thing we need to do is create the Storage Account resources in Azure. This is where the Logic App will store the incoming emails in a table. Besides table service, it also provides file, blob, and queue services in the same resource. We’ll only use the table service.

If you haven’t done so already, sign in to the Azure Portal. Once there, create a Storage Account resource. Choose or create a resource group, location, and account name. This name must be unique across all storage accounts in Azure. When this is done, go to the newly created resource. Under the ‘Table service’ section of the resource menu, click on Tables and create a new table named emails.

Using the Azure CLI, these steps look like this:

    // Creates a resource group named 'e2e-mail-rg'
    az group create --name e2e-mail-rg --location westeurope
    
    // Creates a storage account named 'mailstorage42'
    az storage account create --name mailstorage42 --location westeurope --resource-group e2e-mail-rg
    
    // Creates a table named 'emails' in storage account 'mailstorage42'
    az storage table create --name emails --account-name mailstorage42
    

### Create the Logic App

Here we come to the heart of the solution. Azure Logic Apps is a serverless offering that gives you pre-built components to connect to many other services. You use a graphical designer to put the components together in any combination you need, and Logic Apps will run your process automatically in the cloud, at scale. A Logic App starts with a single trigger and then executes one or more actions.

Now create a Logic App resource in the portal. Choose the same resource group as before (or don’t, it doesn’t matter to me) and give it a good name. I’ve named mine e2e-mail-inbound. After the deployment has finished, go to the new resource. Once there, you’ll be greeted by the Logic Apps Designer where you can choose to start with a common trigger or template. You’ll want to select the ‘When a HTTP request is received’ common trigger.

Now you should see the actual designer with the HTTP trigger expanded in the center. Please note that the ‘HTTP POST URL’ will be generated after saving the Logic App. You may do this now and note this down somewhere; you will need it when configuring SendGrid. At the bottom, there is an option to use a sample JSON payload to generate a schema, but our payload will be form data, unfortunately.

Now click on ‘New step’ and search for ‘Azure Table Storage’. From that connector, choose the ‘Insert Entity’ action. Now choose the storage account we’ve created earlier, give it a connection name, and click ‘Create’. Now you should be able to choose the ‘emails’ table.

The ‘Entity’ field is where things get interesting. Since the incoming requests from SendGrid use the form-data content type, we’ll need to use the `triggerFormDataValue(‘field-name’)` expression to acquire our required values. Finding how to get the form data from the trigger took me a little while to figure out. To save you some work, you can copy-paste the JSON below into the ‘Entity’ field.

    {
      "DKIM": "@{triggerFormDataValue('dkim')}",
      "From": "@{json(triggerFormDataValue('envelope')).from}",
      "HTML": "@{triggerFormDataValue('html')}",
      "PartitionKey": "@{json(triggerFormDataValue('envelope')).to[0]}",
      "RawFrom": "@{triggerFormDataValue('from')}",
      "RawTo": "@{triggerFormDataValue('to')}",
      "RowKey": "@{workflow().run.name}",
      "SPF": "@{triggerFormDataValue('SPF')}",
      "Subject": "@{triggerFormDataValue('subject')}",
      "Text": "@{triggerFormDataValue('text')}",
      "To": "@{json(triggerFormDataValue('envelope')).to[0]}"
    }
    

Now the ‘Insert Entity’ card should look like this. Don’t forget to save.

![Azure Logic App with HTTP trigger and Insert Entity action cinfigured with the JSON](//images.ctfassets.net/9gzi1io5uqx8/7cQ3EX20MGvWfyNgzajoPK/625a5155a459cc4e8f7449ddc999460d/Picture1_HQ.png?fit=scale&w=825)

### Configure SendGrid

#### Configure Domain Authentication

If you don’t have an SendGrid account, you can create one from the Azure portal for easy management or signup at their website. SendGrid is free for our use case.

Before you can receive emails with SendGrid, you must configure and authorize your domain with them first. You can do this from the SendGrid dashboard, open the ‘Settings’ dropdown on the bottom of the menu and go to ‘Sender Authentication’ and click on ‘Authenticate Your Domain’. Here you can choose your DNS host and follow the instructions. I choose to register a subdomain only (mail.example.org) since it serves a specific use case. This ended up having me adding three CNAMES and a single MX record. If you don’t know how to do this or don’t have access to the domain, SendGrid can send the records to a coworker.

Using Cloudflare, for example, for DNS management, you should have four new records that should look something like this.

![DNS management in Cloudflare](//images.ctfassets.net/9gzi1io5uqx8/67948hdlUlVQ8yIuLgaVKl/37c82d718b66ca0687189386aa278ecf/Picture2_HQ.png?fit=scale&w=825)

#### Configure Inbound Parse

The final step to make this all work is to configure the inbound parse webhook. This can be done on the ‘Inbound Parse’ page under the ‘Settings’ menu. If you have configured SendGrid with just a subdomain, you can select it under the ‘Domain’ dropdown; otherwise, configure it as you want. The ‘Destination URL’ is the ‘HTTP POST URL’ from the HTTP trigger of the Azure Logic App. Optionally you can enable a spam filter. After saving this, it may take some time for the DNS changes to propagate through.

![Sendgrid inbound parse settings page](//images.ctfassets.net/9gzi1io5uqx8/7uOJgOwC7hWV2iKPuUVUmL/c58d0dad7bba69bd510a9be8f4b6e351/Picture3_Hq.png?fit=scale&w=825)

#### Retrieve the emails

To see the emails, we need to access the Table Service. We can do this directly from the Azure Portal by using the Storage Explorer, which is a great option to test whether this whole thing works or not. If you haven’t done so already, send an email to your new webhook using an address like da4dcd8d᠎@mail.example.org with ‘hello world’ as the subject. The email should appear within a minute.

![Recieved emails on the Storage Explorere in the Azure Portal](//images.ctfassets.net/9gzi1io5uqx8/3QyqYlwVVvrxcAjrUqWnvT/1b473cc68e3b8859708a104a61aa95c8/Picture4_HQ.png?fit=scale&w=825)

The Storage Explorer isn’t great for automated tests, though. For this, we should query the table by using HTTP API requests. Alternatively, there are API clients for .NET, Node, Python, and Java available. I’ve chosen to use the REST API here because it’s language agnostic and transparent for the reader.

Before we can make any REST API requests, we need to be able to authenticate. For this, you need to generate a Shared Access Signature (SAS) token. This token is used to allow a client granular access to a resource. To create a SAS token via the Azure portal, navigate to the ‘Shared access signature’ page under the ‘Settings’ section of the resource menu. You can allow many permissions here, but the minimum needed are the ‘Table’ service, ‘Object’ resource type, with ‘Read’ permissions. The token expiry is set to 8 hours by default, but it’s a good idea to increase this. The maximum is two years. The generate button should now be active. Once clicked, several fields should appear. Copy the ‘Table service SAS URL’ field.

![Shared access signature settings page of the storage account in the Azure Portal](//images.ctfassets.net/9gzi1io5uqx8/7C7N8aATQmmM5q3ZYokdv6/906a81a5f71ec31b96e72e82a337ce88/Picture5_HQ.png?fit=scale&w=825)

You should now have a URL that looks like the following:

`https://mailstorage42.table.core.windows.net/?sv=2020-02-10&ss=t&srt=o&sp=r&se=2021-03-13T23:00:00Z&st=2021-03-12T23:00:00Z&spr=https&sig=udDMRvYDr15ShkXRZ4oFeIMV4K8oD2DaIEbccyqJj00%3D`

Alternatively, to create a SAS token via the Azure CLI, use the command below. This token will expire two years from execution and only has read permissions on the ‘emails’ table of the ‘mailstorage42’ Storage Account.

    az storage table generate-sas --account-name mailstorage42 --name emails --permissions r --expiry $(date -d "2 years" --utc +%FT%TZ)
    

The output of this should look something like below. Note that this is only the SAS token part and not the full URL. An example of that can be found above.

`se=2023-03-14T12%3A37%3A50Z&sp=r&sv=2017-04-17&tn=emails&sig=%2BkaajUOqxUTC1xbDzdaF86H3u3xm9Hx6mWA9GalACRw%3D`

To access the ‘emails’ table, you need to slightly change the URL to include the path to the table. If you don’t do this, you will get an InvalidUri error back.

`https://mailstorage42.table.core.windows.net/emails?sv=2020-02-10&ss=t&srt=o&sp=r&se=2021-03-13T23:00:00Z&st=2021-03-12T23:00:00Z&spr=https&sig=udDMRvYDr15ShkXRZ4oFeIMV4K8oD2DaIEbccyqJj00%3D`

Accessing this URL in a browser or doing a GET request using your favorite API testing tool should now show the email’s table’s contents. Don’t forget to replace ‘mailstorage42’ with the name of your Storage Account.

#### Filtering

When there are too many records in the table, the service will limit the number of records returned per API request. You can use OData filters to filter the data. This is as simple as appending a query parameter to the URL.

Let’s say we want to filter on emails send to da4dcd8d᠎᠎@mail.example.org. To do this, append `&$filter=To eq 'da4dcd8d@mail.example.org'` to the URL, and you should now only get emails to that specific address.

You can also filter on other properties in the same way. For example, to filter on the subject append `&$filter=Subject eq 'Password reset'` and all emails with that subject should be returned. To combine the two filters, append `&$filter=To eq 'da4dcd8d@mail.example.org' and Subject eq 'Password reset'`, and only mails matching both conditions are returned.

At the moment of writing, filtering using a contains function isn’t supported. But this can be done on the client-side. For example, using a filter method in JavaScript or using LINQ in .NET.

For more information, take a look at the documentation [Querying tables and entities (REST API) - Azure Storage | Microsoft Docs](https://docs.microsoft.com/en-us/rest/api/storageservices/querying-tables-and-entities "Querying tables and entities (REST API) - Azure Storage | Microsoft Docs").

##  Conclusion

The problem I was attempting to solve was how to enable the testers to reliably check emails from an end-to-end testing scenario without them worrying about tests breaking due to third-party changes in their email clients. The support for the catch-all means that the chance of one test interfering with another is non-existent.

Thanks to the power of using a simple to use low-code platform to integrate various services, I was able to build this solution in just two short evenings. And that was with my limited knowledge of Azure Logic Apps and Sendgrid.

We have been using this solution for more than a year now at a customer without issue. These tests helped us identify problems with the infrastructure for sending emails a lot more quickly, and more importantly, before they hit production.

I hope this article may help you improve your own testing scenarios or give some ideas on how to use the services discussed in new and exciting ways to help solve your problems. Please share your insights and thoughts in the comments section below.
