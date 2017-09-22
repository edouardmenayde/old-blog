---
title: Dead links
tags:
  - dev
  - websites
  - reflections
photos:
  - /images/dead-links.png
abbrlink: 36019
date: 2017-09-17 00:00:00
---

It's been some times i'm creating websites and one recurring problem I didn't bother finding a good solution for is *dead links*.

## The problem

**Dead links** are bad in many ways :
- They make the user experience very poor
- They (may) impact SEO
- They are hard to spot because you don't control sites you link to
- They can come back from the dead without telling it to you

## A possible solution

For the last website I created I decided I would try to deal with the problem, I came up with a poor's man solution but that can be inspiration for others to do way better.

I didn't want and couldn't have a CI for that website thus I opted out for a `pre-commit` hook.
In practice it means each time I commit a change a script is gonna be fired which job is to check for dead links.
My site being a sails application I ended up with this code :

```js
const Sails = require('sails').Sails;
const BLC   = require('broken-link-checker');

const sails = new Sails();

sails.lift({log: {level: 'warn'}}, error => {
  if (error) {
    console.error(error);
    return process.exit(1);
  }

  const errors = [];

  // uses the SiteChecker method which is by default recursive
  const blc = new BLC.SiteChecker({
    retry            : 0,
    rateLimit        : 0,
    maxSockets       : Infinity,
    maxSocketsPerHost: Infinity,
    retry405Head     : true,
  }, {
    link: (result) => {
      if (result.broken) {
        const reason = result.broken.reason ? ` (${result.broken.reason})` : '';
        const url    = result.url.resolved ? result.url.resolved : result.url.original;

        errors.push(new Error(`BROKEN: ${url} ${reason}`));
      }
    },
    end : () => {
      if (errors.length > 0) {
        errors.forEach(error => {
          console.error(error);
        });

        sails.lower();
        return process.exit(1);
      }

      sails.lower();
      process.exit(0);
    },
  });

  blc.enqueue('http://localhost:1337'); // Add site's home page to the broken link checker queue
});
```

Basically the sails application is **lifted** (compared to loading which does not expose it on http) and then I use the `broken-link-checker` on the whole site from the homepage to test links.

Putting everything together is then really easy :

1. Install the broken link checker package `npm i --save-dev broken-link-checker`
2. Install the pre commit package `npm i --save-dev pre-commit`
3. Add a `precommit` section to the `package.json` : 
```json
"precommit": [
    "test:blc"
]
```
4. Add a new script to the `package.json` :
```json
"scripts": {
    "test:blc": "NODE_ENV=test node test/blc.js",
}
```
5. You are done

Now back to reality.

## Drawbacks

This approach has __many__ drawbacks :

1. This makes commiting slow because the pre commit hook is slow
2. This is not a *real* broken link check : the tests occur in a dev environment biased by many factors (hosts file, firewall, antivirus...)
3. This does not account for links which breaks between commits
4. This does not account for links which comes back online : the main point of this script is to comment out or remove broken links in your source code before commiting thus thoses links won't get rechecked.

## The better solution

First we need to get rid off the pre commit hook and setup a continuous integration workflow so that our dead link test maybe part of the process which remove problem 1 and 2.
There still are problem 3 and 4 that needs to be solved and it's not trivial.

## The perfect solution aka the perfect service we need

If we want to continuously monitor for dead link we need some kind of services that regularly check your site for dead link and if a dead link is proven to be dead (meaning it's been some amount of time) the service would create a pull request on your website's git repository where it would mark your link as inactive (comment it out or add a special class to it...).
It would also be able to check if a website came back online and create a pull request to mark it as active.

## Conclusion

I think adding a broken link checker to your `CI` is a no brainer and should be done for any site that needs it however using a specialised service may not be needed but our perfect solution would be the perfect fit.