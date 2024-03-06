---
title: "Setting up SSO on Vercel (with Next.js App Router and TypeScript)"
date: 2024-02-03 15:49:16
categories: ["vercel", "sso", "nextjs"]
---

## Introduction

In this post, we will learn how to set up SSO (Single Sign-On) entirely on Vercel using the new Next.js App Router and TypeScript. For the SSO, we will be using Jackson by Boxy HQ. I believe this is the easiest and cheapest (completely free) way to set up SSO for your Next.js app whether you want to just allow your users to sign in with their Google account or you want to set up a full-fledged SSO for your organization or enterprise clients. Plus you get to learn and use all the latest features of Next.js with TypeScript. So, let's get started.

## Set up the SSO Server

First, let's set up the SSO server. For this, we will be using Jackson by Boxy HQ. Jackson is a free and open-source SSO server that you can use to set up SSO for your Next.js app. To set up Jackson, we're just going to navigate to the github repo (https://github.com/boxyhq/jackson) and use the 1 click deployment button for vercel in the readme. This will present you with a page where you can choose the git repository you want to deploy Jackson to, assuming you have already linked your github and vercel accounts. Choose a name for your new repo and make sure the repo is private. Once you've chosen the repository, you will need to set some environment variables. There are a lot of supposedly "required" environment variables, but if you look at the (documentation)[https://boxyhq.com/docs/jackson/deploy/env-variables], most of them actually have defaults. But the way their vercel template is configured you will need to set a fair few. These are the most important environment variables you will actually need to set:

### Environment Variables

- `NEXTAUTH_URL`: This is the URL that the SSO server will be running on. You should set this to something like https://auth.mydomain.com. You will need to of course set up a custom domain for this project in vercel once the deployment is complete.
- `NEXTAUTH_SECRET`: This is a secret key that is used to encrypt the session token. You can generate a random string and use it as the value for this environment variable e.g. by running `openssl rand -base64 32` on the command line
- `CLIENT_SECRET_VERIFIER`: I used the password generator in 1password to generate a "memorable" password consisting of 7 random words and used that as the value for this environment variable.
- `SAML_AUDIENCE`: This is the audience that the SAML response will be sent to. This should be the URL of your Next.js app that you want to set up SSO for e.g. https://mydomain.com
- `IDP_ENABLED`: This should be set to `false` as recommended by Boxy HQ.
- `EXTERNAL_URL`: This should be the same as `NEXTAUTH_URL`.
- `JACKSON_API_KEYS`: Set this to a random password that you will use to authenticate with the Jackson API in case you need/want to use the API. In this tutorial, we won't be using the API and will be configuring everything through the Jackson UI.
- `DB_CLEANUP_LIMIT`: 1000
- `DB_TTL`: 300
- `DB_ENCRYPTION_KEY`: This is a key that is used to encrypt the database. You can generate a random string and use it as the value for this environment variable e.g. by running `openssl rand -base64 24` on the command line
- `DB_TYPE`: postgres
- `DB_ENGINE`: sql
- `NEXTAUTH_ACL`: \*@mydomain.com

### Custom domain

For the rest of the environment variables that are marked as required, simply put empty quotes for now and click the deploy button. Once the deployment is complete, you will likely need to set up a custom domain for the project in vercel to match the one you set for `NEXTAUTH_URL`. You will also need to set up a CNAME record in your DNS settings to point to the vercel domain for the project.

### Database

Once we've done that, it's time to set up the database that Jackson will use. For this, we will be using Vercel's own free postgres database service. It doesn't offer a lot of storage but it is more than enough for our needs. To set up the database, we're just going to navigate to the vercel project and click on the "Storage" tab and create a new postgres database. Ensure you choose the same region for the database deployment as the Jackson deployment for best performance. Once the database is deployed, you will need to set the `DB_URL` environment variable in the Jackson deployment to the connection string for the database which should be visible front and centre. It will be something like: `postgres://default:<password>@<db-domain>.us-east-1.postgres.vercel-storage.com:5432/verceldb`. You will also need to set the `DB_SSL` environment variable to `true`. This wasn't presented as an option when deploying jackson in the first step but it is required for the database connection to work.

### Admin credentials

And finally, the last environment variable you will need to set is `NEXTAUTH_ADMIN_CREDENTIALS` which will allow you to actually log in to the jackson admin portal. The value should be of the form `<username>:<password>`. It's recommended you use a very strong password for this as it will give you access to the admin portal where you can configure the SSO settings. Alternatively it's also possible to set up SSO _for_ the SSO Admin portal but that is rather too meta for this tutorial. Once you've set all these environment variables, you will need to redeploy the Jackson project for the changes to take effect. Once, the service has redeployed, you can navigate to https://auth.mydomain.com and log in with the admin credentials you just set.

<img class="login_page" src="/assets/images/boxyhq-login.png">

:tada: You now have a fully functioning SSO server that you can use to set up SSO for your Next.js app.

## Create the SSO Connection

Now that we have the SSO server set up, it's time to set up the SSO connection for the Next.js app. To do this, we're just going to navigate to the Jackson admin portal and click on the "Connections" tab within "Enterprise SSO" on the left hand side panel. Then we're going to click on the green button that says "New Connection". This will present us with a form where we can set up the SSO connection. For this tutorial, we'll be using [MockSAML](https://mocksaml.com) as our Identity Provider (IdP) which is used just for testing purposes. For this reason, we're only going to setup the SSO connection for local development i.e. against localhost rather than the app's deployed domain. We will need to fill in the following fields:

- `Type`: This is the type of the connection. You can choose between "SAML" and "OIDC". For this tutorial, we will be using "SAML".
- `Name`: This is the name of the connection. You can set this to something like "My Next.js App".
- `Description`: This is a description of the connection. You can set this to something like "SSO for my Next.js app".
- `Tenant`: This is the tenant that the SSO connection will be for. You can set this to something like "mydomain.com". Typically, this is the domain of your Next.js app but only by convention. It doesn't actually have to be the same as the domain of your Next.js app and in this case it won't be since we're setting up the connection for localhost.
- `Product`: This is the product that the SSO connection will be for. We'll be using "Demo" for this tutorial.
- `Allowed redirect URLs`: This is the URL that the SAML response will be sent to. We'll be using "http://localhost:3000" for this tutorial.
- `Default redirect URL`: This is the URL that the user will be redirected to after they have successfully logged in. We'll be using "http://localhost:3000/login/saml" for this tutorial.
- `Metadata URL`: This is the URL of the IdP metadata. We'll be using "https://mocksaml.com/api/saml/metadata" for this tutorial. This can be provided instead of the `Raw IdP XML` for simplicity.

That's all! Once you've filled in these fields, you can click the "Save Changes" button to create the SSO connection which will redirect you back to the "Connections" page. If you click the edit button on the connection we just created, you will then see a `Client ID` and `Client Secret` in a panel on the right hand side which we will need for the next step.

<img class="login_page" src="/assets/images/sso-connection.png">

## Set up the Next.js App

Now that we have the SSO server and connection set up, it's time to set up the Next.js app. For this, we will be using the new Next.js App Router which became stable in v13.4. The Next.js App Router is a new feature in Next.js that completely changes how you set up pages and API routes. The old way is still supported but it is recommended for new projects to use the App router. To create this project we are going to use the Next.js CLI. To do this, we're just going to run the following command in the terminal:

```bash
yarn create next-app
```

This will present you with some options for how the repo should be configured - ensure you definitely select "Yes" for the TypeScript and App Router options.

We will be using the `next-auth` package to set up the SSO for the Next.js app. To install this package, we're just going to run the following command in the terminal:

```bash
yarn add next-auth
```

Boxy HQ has an example repo for setting up `next-auth` with Jackson (here)[https://github.com/boxyhq/jackson-examples/tree/main/apps/next-auth] but it's a little old so they're not using the App Router. However it is still a valuable resource and is actually deployed here: https://saml-demo.boxyhq.com/ so you can see what the end result could look like.

## Pro-tips

- Save passwords and secrets in a password manager like 1password and use the password generator to generate strong passwords.

## Useful Links

- Jackson vercel info
- Jackson youtube video
