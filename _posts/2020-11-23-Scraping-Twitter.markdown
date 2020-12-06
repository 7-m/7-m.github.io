---
layout: post
title:  "Lets scrape some tweets!!1"
date:   2020-11-23 00:00:00
categories: scraping
---

Want to scrape a handful of tweets? but don't want to sign up for twitter developer or break your head going through tons of API's or copy paste one tweet after the other? Then you have come to the right place!

#### Things to get started
- Basic understanding of Rest API's
- Your favourite Rest API tool
- 10 minutes

#### In brief
1. `api.twitter.com/1.1/guest/activate.json` to get the `x-guest-token` header
2. `api.twitter.com/2/timeline/profile/{rest_id}.json?count={numTweets}` to get the actual tweets
3. `api.twitter.com/graphql/esn6mjj-y68fNAj45x5IYA/UserByScreenName?variables={"screen_name":"{userHandle}","withHighlightedLabel":false}` to get `rest_id` from a twitter handle
4. `authentication` header for bearer token

#### How?
The `authorization : Bearer AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I8xnZz4puTs%3D1Zv7ttfk8LF81IUq16cHjhLTvJu4FA33AGWWjCpTnA` header seems to be invariant, incase if it ever changes, just inspect one of the API requests headers in your browser to get it, make sure to include it in every request. The next important bit is the `x-guest-token` header, that's what permits you to make any tweet fetching API requests. Acquire it by making a `POST` the `/1.1/guest/activate.json` endpoint, don't forget to include the `authorization` header.

 At this point you're ready to fetch some tweets but first you need `rest_id` which identifies a twitter user. To acquire it, `GET` the `/esn6mjj-y68fNAj45x5IYA/UserByScreenName` endpoint. Include `x-guest-token` and `authorization` headers. Substitute `userHandle` in the query part `variables={"screen_name":"{userHandle}","withHighlightedLabel":false}` with the twitter handle of the user, for ex. `variables={"screen_name":"theasf","withHighlightedLabel":false}` (Apache Foundation's twitter handle). The `esn6mjj-y68fNAj45x5IYA` in the path seems to be a graphQl query hash so its invaraint, but, if it does change, inspect the API requests in your browser to acquire the new value. The response is a JSON, and you can get `rest_id` using the JSONPath `$.data.user.rest_id`.

Now we're truly ready to fetch some tweets, armed with the 2 headers and `rest_id`, `GET` the `api.twitter.com/2/timeline/profile/{rest_id}.json?count={numTweets}` API to fetch `numTweets` number of tweets ( sometimes a few more) . Include both `authorizaton` and `x-guest-token` headers. The tweets are contained as key-value pairs, with the tweet id being the key, I'll leave the exploration of the value object up to you. Use `$.globalObjects.tweets` to fetch the key-value  tweets map. 

That's all folks!

