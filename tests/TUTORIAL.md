# Real World Serverless Part 1: Fast, Cheap & Global Frontend 

Deploying a React, Vue or Angular app is fairly simple. But doing so **well** can be a different story; where well is defined as:

* With a CDN & Cache (Fast & Global for your users and SEO)
* With HTTPS & your own domain (SSL certs and config)
* Developer friendly (no hassle to deploy and update, works with any backend, reliable)
* Infinitely scalable (handle 0 traffic or millions well)
* For free (at least until you have *tons* of users)

Achieving this entire list without things getting too complicated is quite the feat! [Serverless Framework](https://serverless.com) will make it easy. This tutorial is design to gradually show you how to use Serverless to spend more time building your apps and less time in making them run.

### RealWorld - "The Mother of all demo apps"

We'll deploy an already-built fully working **fullstack** medium.com clone; checkout [RealWorld example apps on Github](https://github.com/gothinkster/realworld). It has frontends in Vue, Angular, React and many more. So you'll be learning how to deploy and run an app that is a bit more.. ***real world***, than your typical todo or hello world.

You can see the final deployed result of this tutorial here: [mediumserverless.com](https://mediumserverless.com)



#### 1 Setting up our RealWorld Frontend

For this tutorial I'll be using the [Vue.js RealWorld Frontend](https://github.com/gothinkster/vue-realworld-example-app) however you can follow along just as well with React, Angular or whatever else you fancy :)

* Create a new folder for the frontend and clone the RealWorld Vue repo contents into it `https://github.com/gothinkster/vue-realworld-example-app.git .`
* Install dependencies `npm install`
* Run the project locally `npm run serve`

You should see a message saying your app is running on http://localhost:8080/ . Click on the link and you should see the Vue app:

// first screenshot

This app fetches data from RealWorlds test API. We'll connect it to our own backend at the end. 



#### [FYI] Prerequisite: Serverless Framework & AWS Account

You will need the Serverless Framework installed and an AWS account. Setting this up is super simple and this 5-min video will show you how:

https://www.youtube.com/watch?v=ts26BVuX3j0

**What is Serverless Framework?**

Everything we'll do here could be done using the AWS web interface. But clicking around and setting configurations etc.. its very tedious. Serverless Framework lets you easily configure your cloud in your repo, along with your code, and will then automagically "do all the clicking for you". 

In the first part of the tutorial we'll merely scratch the surface of what it can do. All you need to know for now, is that it's going to make deploying web apps extremely simple for you.



#### 2 Configuring our Serverless file

There's no need to re-invent the wheel here. Deploying some javascript and html has been done a million times and should be simple. For this we'll use the [Serverless website component](https://github.com/serverless-components/website)/ A pre-made "recipe" for  "AWS S3 & AWS Cloudfront powered serverless website w/ custom domain."

Create a serverless.yml file in your root directory `touch serverless.yml` and open it, copy paste the below:

```
# serverless.yml

name: medium-serverless # change to your name
stage: dev

myWebsite:
  component: "@serverless/website"
  inputs:
    code:
      root: ./
      src: ./dist # after running "npm run build" our app will be in dist folder
      hook: npm run build
    region: eu-central-1 # use same region as your backend from part1 tutorial!
    bucketName: medium-serverless-tutorial
                      
    domain: www.mediumserverless.com # replace with your own domain! or remove this line and aws will generate a generic url for you.  
```

You can have a look at the [Serverless Website Component README](https://github.com/serverless-components/website#3-configure) for detailed comments on each option and more info. 

#### 3. Using your own domain

Purchase your domain (or transfer your existing domain to AWS Route 53). If you can see it under Hosted Zones in [AWS Console Route 53](https://console.aws.amazon.com/route53) you don't need to do anything else and you're good to go!

// hosted zone image

#### 4. Deploy

Lets push our Vue app to the internet with command `serverless` and you will see it starting to work on deploying. Its going to configure your domain (including SSL certifictate), CloudFront CDN and S3 bucket for your static files and finally you will recieve your urls:

```
myWebsite: 
    url:    http://medium-serverless-tutorial.s3-website-eu-central-1.amazonaws.com
    env: 
    domain: https://mediumserverless.com

  43s › myWebsite › done
```

If you open the links they likely won't work yet. It will take some time (20 min, an hour.. depends!) for DNS and CloudFront to go into effect. However if you open [AWS S3](https://s3.console.aws.amazon.com/s3/) (make sure you have the same region selected as you specified in serverless.yml), search for you bucket name and open it and you should see all your dist files available. 

You can view my deployed app on [mediumserverless.com](https://mediumserverless.com). You can also try [www.mediumserverless.com](https://www.mediumserverless.com) and why not the non-SSL url [http://mediumserverless.com](http://mediumserverless.com), see how it redirects to https? :) 



#### 6. Updating our Frontend

Let's make a small code change and publish that update. Let's edit the title of our site in src/views/Home.vue;

```
<div class="home-page">
    <div class="banner">
      <div class="container">
        <h1 class="logo-font">MEDIUM SERVERLESS</h1>
        <p>A place to share your knowledge.</p>
      </div>
    </div>
```

Change the contents of the h1 tag to whatever you want. Save your change, then enter command `serverless` to deploy the update! Once done visit your url. Can you see your update? Probably not. Why?

#### [FYI] Caching & Updates

Try opening an incognito window and reloading your page. Now you see the update right? Load it in non-incognito, doesn't work. Close all your browser tabs (really, all of them, or just close the browser) and open again, and it works. 🤔

Since it works in incognito, new visitors to your site will recieve the new version. However, returning users might continue to use an outdated version. With just a changed H1 tag, that isn't much of a problem. However, with bigger updates, having a bunch of users with outdated frontends they could start seeing problems, especially with JavaScript and changes in the backend. So if you're going to work in an agile way and update often, this is a problem.

*So what is going on?* And how do we fix it **well**?

There are three places where caching could affect your frontend app; the CDN (CloudFront), browser cache and service worker. 

* When you deploy with `serverless` a *CloudFront invalidation* is created, meaning your CDN will serve new files from S3. 
  * // cloudfront image
* When you build your vue app, if you look in the dist folder, you'll notice that all the filenames change, which takes care of browser cache. 

If you open DevTools and check the Network tab we can see that our files are being served from (ServiceWorker). Hence, ServiceWorker is to blame! If you are building a **Progressive Web App** with something like **Vue**, **Angular** or **React** your frontend will use a Service Worker. 

// devtools service worker image.

#### 7.  Updating our Service Worker

The default Service Worker that ships with Vue is configured to cache assets and it correctly detects that there are updates available - however no logic is implemented for actually updating. Checkout [VueJS PWA: Cache Busting](https://medium.com/js-dojo/vuejs-pwa-cache-busting-8d09edd22a31) for details on the solution I'll be implementing.

**Update the service-worker.js file**

```javascript
// service-worker.js

workbox.core.setCacheNameDetails({ prefix: 'd4' })

//Change this value every time you update your service worker
const LATEST_VERSION = 'v1.0'

self.addEventListener('activate', (event) => {
  console.log(`%c ${LATEST_VERSION} `, 'background: #ddd; color: #0000ff')
  if (caches) {
    caches.keys().then((arr) => {
      arr.forEach((key) => {
        if (key.indexOf('d4-precache') < -1) {
          caches.delete(key).then(() => console.log(`%c Cleared ${key}`, 'background: #333; color: #ff0000'))
        } else {
          caches.open(key).then((cache) => {
            cache.match('version').then((res) => {
              if (!res) {
                cache.put('version', new Response(LATEST_VERSION, { status: 200, statusText: LATEST_VERSION }))
              } else if (res.statusText !== LATEST_VERSION) {
                caches.delete(key).then(() => console.log(`%c Cleared Cache ${LATEST_VERSION}`, 'background: #333; color: #ff0000'))
              } else console.log(`%c Great you have the latest version ${LATEST_VERSION}`, 'background: #333; color: #00ff00')
            })
          })
        }
      })
    })
  }
})

workbox.skipWaiting()
workbox.clientsClaim()

self.__precacheManifest = [].concat(self.__precacheManifest || [])
workbox.precaching.suppressWarnings()
workbox.precaching.precacheAndRoute(self.__precacheManifest, {})
```

Notice the `const LATEST_VERSION='v1.0'`. If you update your Service Worker again, before you build the project, update this variable. That way the service worker itself would also get updated. 

**update src/registerServiceWorker.js updated() function**  

```javascript
/* eslint-disable no-console */

import { register } from "register-service-worker";

if (process.env.NODE_ENV === "production") {
  ...
    updated() {
      console.log('New content is available; Refresh...')
      setTimeout(() => {
        window.location.reload(true)
      }, 1000)
    },
   ...
```

This will force a refresh of the browser window, which due to the script placed in the service-worker.js file will clear the cache and load updated assets.

**vue.config.js**

The RealWorld Vue project comes without a vue.config.js file. So first create one in the root project directory containing the below:

```javascript
const manifestJSON = require('./public/manifest.json')module.exports = {
  pwa: {
    themeColor: manifestJSON.theme_color,
    name: manifestJSON.short_name,
    msTileColor: manifestJSON.background_color,
    appleMobileWebAppCapable: 'yes',
    appleMobileWebAppStatusBarStyle: 'black',
    workboxPluginMode: 'InjectManifest',
    workboxOptions: {
      swSrc: 'service-worker.js',
    },
  },
}
```



**Deploy!**

`serverless`

Then try navigating to your site. You may need to update your service worker manually the first time in devtools

// devtools sw img

Following that try updating the h1 tag again and revist your site. You will see the old site load first, then the browser should automatically reload and your update should show. 



#### [FYI] What you've just deployed

Whilst the Serverless Framework has done everything for you, this setup could've been achieved by clicking around in the AWS Console. You can logon to [console.aws.amazon.com](https://console.aws.amazon.com) to see everything you've deployed!

**CloudFront** - your global Content Delivery Network & Cache

[console.aws.amazon.com/cloudfront](https://console.aws.amazon.com/cloudfront)

You can get basic website usage and viewers stats here. 

// cloudfront image

**S3** - static file storage [s3.aws.amazon.com/s3](https://s3.console.aws.amazon.com/s3)





# Next Steps

We've only scratched the surface of what can be done with AWS and the Serverless Framework. In Part 2 of this tutorial [link]() I will take one of the RealWorld Backends and deploy it to AWS 🚀⭐️









