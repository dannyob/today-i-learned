---
layout: default
title: "Simple asteroid impact probability display using Cloudflare Workers"
date: 2025-02-24
tags:
  - JavaScript
  - Cloudflare
  - NASA API
author: Danny O'Brien <danny@spesh.com>
---

# Simple asteroid impact probability display using Cloudflare Workers

I wanted to create a [one-off webpage](http://arewedoomedyet.org/)that would display the current probability of the asteroid 2024 YR4 hitting Earth, based on NASA's Sentry API data. I originally asked Claude to write it in Guile, but it got confused about libraries. We got a working version in Python, and then I asked for it to be turned into JavaScript suitable for Cloudflare Workers to cache and serve the data. The code fetches the latest impact probability from NASA's API, formats it into human-readable form (converting probabilities into "1 in X" ratios). It includes caching to minimize API calls and handles errors gracefully. The only bit I struggled with was how to point a domain purchased outside of Cloudflare at the worker. In the end, I just bought another one from Cloudflares registrar, and attached that in their dashboard. It took a few minutes to resolve (ah, DNS), but once it did, everything worked.

Here's the [complete code for the site](https://arewedoomedyet.org/). I'm still unsure as to whether this will trigger any limits on Cloudflare's side, or whether I should really be bothered to renew the domain in a year. I guess we'll see if it gets much more likely -- it may end up a popular site if it does!

```javascript
// Cache configuration
const CACHE_KEY = 'asteroid-data';
const CACHE_TTL = 3600; // 1 hour in seconds

// Utility functions
function formatDate(dateStr) {
  const [baseDate] = dateStr.split('.');
  const date = new Date(baseDate);
  return date.toLocaleDateString('en-US', { 
    month: 'long', 
    day: 'numeric', 
    year: 'numeric' 
  });
}

function formatProbability(probStr) {
  const prob = parseFloat(probStr);
  const ratio = Math.round(1 / prob);
  const percentage = prob * 100;
  
  // Simple number formatting for large numbers
  function intword(num) {
    const units = ['', 'thousand', 'million', 'billion', 'trillion'];
    const k = 1000;
    let magnitude = Math.floor(Math.log10(num) / 3);
    if (magnitude > units.length - 1) magnitude = units.length - 1;
    
    const scaled = num / Math.pow(k, magnitude);
    return `${Math.round(scaled)} ${units[magnitude]}`;
  }
  
  return {
    ratio: `1 in ${intword(ratio)}`,
    percentage: percentage
  };
}

async function getAsteroidData() {
  const url = new URL('https://ssd-api.jpl.nasa.gov/sentry.api');
  url.searchParams.set('des', '2024 YR4');
  
  const response = await fetch(url);
  const data = await response.json();
  
  // Find entry with maximum impact probability
  const maxImpact = data.data.reduce((max, current) => {
    return parseFloat(current.ip) > parseFloat(max.ip) ? current : max;
  });
  
  return {
    probability: maxImpact.ip,
    date: maxImpact.date
  };
}

function generateHTML(probability, date) {
  const readableDate = formatDate(date);
  const { ratio: readableProb, percentage } = formatProbability(probability);
  
  return `<!DOCTYPE html>
<html>
<head>
    <title>Asteroid Impact Probability</title>
    <style>
        body { 
            font-family: Arial, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            background-color: #1a1a1a;
            color: #ffffff;
        }
        h1 {
            font-size: 2.5em;
            text-align: center;
            margin-bottom: 0.5em;
            padding: 0 1em;
        }
        .probability {
            font-size: 3.5em;
            font-weight: bold;
            color: #ff6b6b;
            text-align: center;
            line-height: 1.2;
        }
        .subtitle {
            font-size: 0.4em;
            opacity: 0.8;
            margin-top: 0.5em;
        }
    </style>
</head>
<body>
    <div style="font-size: 1.5em; margin-bottom: 1em; opacity: 0.9;">The <a href="https://cneos.jpl.nasa.gov/sentry/details.html#?des=2024%20YR4">asteroid 2024 YR4</a> has a</div>
    <div class="probability">
        ${readableProb}
        <div class="subtitle">chance of hitting Earth (${percentage.toFixed(4)}%) on ${readableDate}.</div>
    </div>
</body>
</html>`;
}

export default {
  async fetch(request, env, ctx) {
    try {
      // Try to get cached data
      const cache = caches.default;
      let response = await cache.match(request);
      
      if (!response) {
        // If no cached data, fetch new data
        const { probability, date } = await getAsteroidData();
        const html = generateHTML(probability, date);
        
        response = new Response(html, {
          headers: {
            'Content-Type': 'text/html;charset=UTF-8',
            'Cache-Control': `public, max-age=${CACHE_TTL}`,
          },
        });
        
        // Store in cache
        ctx.waitUntil(cache.put(request, response.clone()));
      }
      
      return response;
    } catch (error) {
      const html = generateHTML('Error fetching data', 'Unknown date');
      return new Response(html, {
        headers: { 'Content-Type': 'text/html;charset=UTF-8' },
        status: 500,
      });
    }
  },
};
```
