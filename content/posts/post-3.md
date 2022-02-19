---
title: "Implementing domain based Multi-tenancy - 1"
date: 2022-02-19T02:01:58+05:30
description: "This is a tutorial on implementing a domain based multi-tenant application using NextJs and Openresty."
tags: [multi-tenant, "nextjs", "openresty"]
---

In normal scenerios, you build a web-application and host it on a server. Then you purchase a domain which you then link to this server's IP. Other users can then access the web-application using this domain of yours.

However, often times you also want to give the users the ability to map their own domains to a web application which is hosted on a server which you control. Now the users can open the web-application on their own domains as well. We call this types of web applications as _domain based multi-tenant_ applications and the users as the _tenants_.

_Add note about multi-tenant applications_

In this tutorial we will show you how you can build a website which can also be accessed by users using a custom domain of their own choosing.

I'll make this a 3 part series, in each part we will cover the following things separately-

1. Set up NextJs to programmatically create unique pages based on user's custom domain
2. Installing and setting up Openresty as your webserver
3. Dynamically provisioning SSL certifcates in Openresty

So, let's get started.

### Setting up NextJS

Let's first set up a NextJs application which can server unique content page based on the custom domain.

Inside the _pages_ directory create a new dynamic page called - _\_sites/[site]/index.tsx_.  
Inside the _pages_ directory create a file called _\_middleware.ts_.

Your _pages_ directory should now look something like this -

```
.
├── _app.tsx
├── _middleware.tsx
├── _sites
│   └── [site]
│       ├── index.tsx

```

By creating the _\_middleware_ file you can run code before a request is completed. Based on the incoming request, you can modify the response by rewriting, redirecting, adding headers, or even streaming HTML. [Read more about NextJs middlewares](https://nextjs.org/docs/middleware)

Inside the \_middleware.tsx add the following code

```
import { NextRequest, NextResponse } from 'next/server'

export default function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl
  // Get hostname
  const hostname = req.headers.get('host')

  // Prevent security issues – users should not be able to canonically access
  // the pages/sites folder and its respective contents. This can also be done
  // via rewrites to a custom 404 page
  if (pathname.startsWith(`/_sites`)) {
    return new Response(null, { status: 404 })
  }
    return NextResponse.rewrite(`/_sites/${currentHost}${pathname}`)
}
```

Let's understand what the code does when it receives a request for the URL - https://arpit.example.com/about

1. The path which is _/about_ is stored inside the _pathname_ variable
2. We extract the domain value from the _host_ header of the HTTP request and store it in the _hostname_ variable.
3. NextJs supports this awesome ability to rewrite a request path to a different destination path. The line `NextResponse.rewrite(`/\_sites/${currentHost}${pathname}`) }` does exactly that. For our example the the new path is - `_sites/arpit.example.com/about`

Inside the _\_sites/[sites]_ directory for the pages that you set up, you can get the domain inside your _getServerSideProps_ function. In normal scenerios each domain would map to a unique user in your database. You can now fetch data specific to this user using this domain value. You can do it something like this -

```
const getServerSideProps: GetServerSideProps = async (context) => {
  let domain = context.req.headers.host;
  if (!domain) {
    return {
      redirect: "/",
      notFound: true,
      props: {},
    };
  }
    // Make API call and fetch data for this user of this domain and render page
    // const data = await fetchUserData(domain);

  return {
    props: {
        data
    }
  };
};

```

Now you have the idea about how you can configure your Frontend application for domain based multi-tenant applications in NextJs. In the next part we will cover how to setup the webserver with Openresty.

Hope you liked it, See you in the next blog :)
