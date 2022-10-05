---
title: "When To Chat 🕑"
date: 2019-01-08T13:48:09-08:00
Description: "Reconcile time zone differences, so chatting is easy"
Tags: ["gastby", "netlify", "javascript", "react", "google maps api", "lambda"]
Categories: ["code"]
banner: "/images/whentochat.gif"
draft: false

---
<style>
  .screenshot {
    display: block;
    margin: auto;
    max-width: 100%;
  }
  .text-center { margin-top: 0;
    text-align: center;
  }
  h1 {
    font-size: 1.8em
  }
</style>

In early December I [released](https://twitter.com/nathanwillson/status/1070349231800512512) a free tool called [When To Chat](https://whentochat.co/). It aims to help visualize time zone differences to make it easier to find time to chat.

I've actually found it to be super useful (yay) and often use it myself -- so mission accomplished.🎉

<img class='screenshot' src="/images/whentochat.gif" alt="When To Chat">

Anyways, in this post I want to elaborate on some of the inner workings.

<p class="text-center">
The <a href="https://github.com/nbw/whentochat">code is available here</a>.
</p>

---

# Written in Gatsby

[Gatsby](https://www.gatsbyjs.org/) is a site generator that uses React (Javascript). They have great documentation, a big community, and it's fast. I haven't met anyone that's used Gatsby and had complaints that are unusual for any React project.

I chose to write in Gatsby because:

* The complex (kinda) UI lends itself to a React environment
* I wanted a static website
* Netlify and Gatsby are good friends 🤞


---

# Hosted on Netlify

I have nothing but good things to say about Netlify. They make static website hosting a real treat. This blog that you're reading (right?) is hosted on Netlify. It's free, it's super easy to use, and it has a lot of cool features.

Also, putting my Gatsby project on Netlify took all of 2 minutes.


## Using Netlify Serverless Functions (AWS Lambdas)

Netlify has some tools to make integration with serverless lambda functions easy. In the case of When To Chat I included a Shorten button to generate a Bitly link.


<img class='screenshot' src="/images/whentochat_share.gif" alt="Share">

When a user clicks that button, they make a request to my serverless function that talks to Bitly's API and get's a shortened link back.

The advantage here was that I could insert my Bitly API_KEY at build time as an environment variable and hide it from the end user behind a serverless function.

[Here's the code for that function](https://github.com/nbw/whentochat/blob/master/src/functions/bitly.js).


---

# Location Pickers and Time Zones

I looked through a number of location api services (Microsoft even has one), but settled on Google's API for everything.

## Geosuggest Location Picker

Something I researched before choosing Gatsby was if there were any easy to use react-based location pickers out there. There are actually a few, but the best one I could find (based on downloads, stars on github, and community support) was [React Geosuggest](https://www.npmjs.com/package/react-geosuggest). Install it, include a link to the Google Maps API script above it (with your key) and you're good to go.

It's worth mentioning that you'll need to enable the API on Google's end. That part was a pain, but just how it is. There are a lot of tabs and buttons and nonsense that you have to trudge through to enable the right API.

## Retrieving time zones

Once two locations have been selected, the next step is to find the time zone of each location and calculate the offset. I ended up being able to leverage Google Map's API, but first I'll mention Geonames:

### Issue with Geonames

For a while I was using [Geonames' API](http://www.geonames.org/export/web-services.html) to fetch the time zone via latitude and longitude. It's free and doesn't require an API key. The issue I ran into was Day Light Savings time related. The times I received back were sometimes off by an hour and I was querying by Latitude and Longitude. This meant I was losing the context of the actual location and relying on geographical coordinates.

### Timezone from Google Maps' API

With React Geosuggest, you get the full API response back from Google. Embedded in the reponse is a "utc_offset" parameter which does the trick of calculating time zone differenes. I didn't notice it first, but it's a location name based time zone-- not a latitude longitude based, which is what I needed.


# Conclusion

Gastby and Netlify ended up working swimmingly.  I did my best to make a polished tool with minimal bells and whistles. There are definitely some features that would be nice to have, but as far as an MVP goes I'm satisfied with the end product.

Thanks to everyone that gave me feedback along the way.




