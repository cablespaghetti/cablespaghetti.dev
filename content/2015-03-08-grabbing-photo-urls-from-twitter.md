---
layout: post
title:  "Grabbing Photo URLs from Twitter"
description: "Yesterday I helped out at #CodeOff2015, an event that my employer Snowflake Software runs every year at the Electronics and Computer Science department at the University of Southampton. We give students the day to write software to solve a problem that we set, they get judged at the end of the day and then the winner gets a paid summer internship with our development team. A great way to find talented developers, I'm sure you'll agree!"
date:   2015-03-08 18:00
tags: scripts python
redirect_from:
  - /2015/03/08/grabbing-photo-urls-from-twitter.html
---
Yesterday I helped out at [#CodeOff2015](https://twitter.com/search?q=%23codeoff2015), an event that my employer [Snowflake Software](http://snowflakesoftware.com) runs every year at the [Electronics and Computer Science department at the University of Southampton](http://www.ecs.soton.ac.uk/) (which also happens to be where I studied). We give students the day to write software to solve a problem that we set, they get judged at the end of the day and then the winner gets a paid summer internship with our development team. A great way to find talented developers, I'm sure you'll agree!

Here are the students tucking in to the free pizza:
![Students tucking in to free pizza](/media/codeoff2015.jpg)

The morning was a bit hectic, helping everyone to get started and solving a few connectivity problems with the resources we had provided. However after they were all busy coding I thought I'd set myself a little (or so I thought) challenge, to brush up on my (very ropey) Python skills. 

I had set up an internal website within ECS to host the various resources using [Bootstrap](http://getbootstrap.com/) and as the day went on, there were lots of photos posted under the hashtag on Twitter. I thought I'd use Twitter's API and Bootstrap's carousel/slideshow feature to show the latest photos on the website.

Initially I used [Twython](https://github.com/ryanmcgrath/twython) to get the tweets from the API and print out any 'media_url' attributes it found. However whatever I did to tweak the parameters for the Twitter search, I still only got 4 or 5 unique images, which is a fraction of what was posted.
```python
#!/usr/bin/env python3
from twython import Twython, TwythonError
APP_KEY = ''
APP_SECRET = ''
OAUTH_TOKEN = ''
OAUTH_TOKEN_SECRET = ''

# Requires Authentication as of Twitter API v1.1
twitter = Twython(APP_KEY, APP_SECRET, OAUTH_TOKEN, OAUTH_TOKEN_SECRET)
try:
    search_results = twitter.search(q='CodeOff2015', count=100, result_type="recent")
except TwythonError as e:
    print(e)

for tweet in search_results['statuses']:
#    print (tweet['text'])
    if 'media' in tweet['entities'].keys():
         media = tweet['entities']['media']
         for mediaitem in media:
             if 'media_url' in mediaitem.keys():
                 print(mediaitem['media_url'])

```

I did some thinking and took a different (significantly more hacky) approach. The ['Photos' view](https://twitter.com/search?q=%23codeoff2015&src=typd&mode=photos) on the Twitter website shows many more photos that what I was getting through the API, so I looked into grabbing the HTML and processing it. This uses [urllib3](https://github.com/shazow/urllib3) to get the HTML and [Beautiful Soup 4](http://www.crummy.com/software/BeautifulSoup/) to process it. For our hashtag it seems to get the most recent 12 photos which is much better than the API.
```python
#!/usr/bin/env python3

#Grab the html
import urllib3
http = urllib3.PoolManager()
twitterreq = http.request('GET', 'https://twitter.com/search?v=stream&q=%23CodeOff2015&src=typd&mode=photos')
twitterpage = twitterreq.data

from bs4 import BeautifulSoup
#Create a BeautifulSoup object from the HTML source code
twittersoup = BeautifulSoup(twitterpage)

#Make a list of the link elements which have photos associated with them
linklist = twittersoup.find_all("a", class_="media media-thumbnail twitter-timeline-link media-forward is-preview ")

#Iterate over the list of links
for link in linklist:
    imageurl = link.get('data-resolved-url-large')
    print(imageurl)
```
