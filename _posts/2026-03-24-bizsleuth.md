---
layout: design-post
title:  "Bizsleuth"
tags: [ai, web-scraping]
---

Bizsleuth is a solution to a real problem I saw in B2B businesses. I saw that the sales and marketing teams get potential leads data from different sources and directories like `Google Maps` but that information is often incomplete for targeted campaigns and lack essential information for lead-generation such as email address and highly useful information such as the business owner name. They often have to manually open each website url to find this missing info. This takes up huge time and effort. So I build bizsleuth to solve this problem.

Bizsleuth is an AI-powered lead-generation and lead-enrichment tool which scrapes key information about a business from its website's content. 

Bizsleuth consists of a crawler that crawls the domain and internal pages upto maximum 10 pages, gathers the text content from all these pages and then use AI to parse key info. This is why its important to feed the homepage url as an input instead of some random page url of the domain so that the crawler can crawl the navigation pages instead of some random links.  

Initially when I built the solution, I didn't think of actorising the solution but when I saw its impact, I saw a common pain point for B2B businesses and a gap in the market. I couldn't find any tool in the market that can extract commonly required information for lead-generation using the power of AI to get the information reliably. The existing solutions were using regular expressions or techniques which can result in lower accuracy, higher false positives and false negatives. But nothing good ever happens without challenges, right? So here are some challenges I faced

#### Challenges
- AI context: 
    Crawling websites and then using AI to parse required information seems intuitive but the website content across multiple pages can exceed the maximum context length. To solve this problem, I used an open source library by *Zyte* called `html_text` to extract clean text from html source pages and used truncation mechanism to ensure that the context doesn't exceed a limit of 10000 tokens for any given website. 10000 is not some magic number but I found 10k to be an average of a sample of business websites (about 1k websites) in my case where I was dealing with fitness. But regardless of type of business, 10k tokens represent a good amount of text content for any type of business if not exceed it will most likely contain the information we are looking for.

- AI cost: (modern solutions incur modern costs)
    AI API is not free for a developer like me (and you unless you are someone who works at one of the AI labs replacing developers). Cleaning the website content and truncating the text reduced not only helped manage the context but also reduce the cost of AI API. With 10k tokens per website, you get about $1 with gemini-flash-lite model for 1000 websites which is pretty inexpensive. 

- Apify platform cost:
    Initially I used a browser-based crawler instead of simple `httpcrawler` to crawl which was very slow. Apify charges per Compute Units (CU) that your actor consumes. The more time or memory your actor takes, the more your actor costs. Using browser can be beneficial for some websites which displays our required information after javascript rendering but it wasn't worth the time it takes and the costs it incurs. So I switched to simple `httpcrawler` to crawl the business websites. 

- Concurrency control:
    The actor processes given websites asyncronously. It was ok on a small input but when a large input was given like (1000+ urls), then the actor would get into Out Of Memory (OOM) issue because firing up so many tasks at once is a guaranteed recipe for exhausting the available resources. I implemented a concurrency control mechanism using `asyncio.Semaphore` to process 10 websites at a time.

I wanted my actor to be such that it is achieves the task in a reasonable amount of time and cost. This is why I decided to use `httpcrawler`, put a cap on tokens per website so that I have a good idea of AI usage and costs. It takes my actor 25-30 mins to crawl and process 1000 websites which is reasonable given the fact that the websites are crawled deeply and processed through AI. 

Setting pricing of the tool is challenging if its your first SaaS tool. You need to price the tool in a way that covers all the costs as well as make some profit while also keeping it affordable for the user. After some research on the Apify platform and some research on the web about the type of tool I was providing, I found that $5/1000 urls would be a good starting point because that covers all my cost and makes me a little profit and it is also affordable for the end user. If it turns out to be expensive, then I can tweak the plan to make it cheaper. Reducing the price is easy, increasing the price is not easy because you can do that only once a month on Apify and it will take effect after 14 days. Also, an existing customer doesn't like a price increase and may churn but reducing the price will make him only happier. 

But how do you actually charge a user for your actor. Apify provides a few different ways to monetize your actor and you can read more about them [here](https://docs.apify.com/academy/actor-marketing-playbook/store-basics/how-actor-monetization-works). I have personally used *PPE* and I like this method. In *PPE* method, you create an event and whenever that event happens, user will be charged the amount you have set. For example, you can set `request_url` event which charges the user everytime your actor makes an `http_request` to a url. Apify has also made it very easy to implement. All you have to do is to use `Actor.Charge()` method in your code to charge for the event. You can read more about this [here](https://docs.apify.com/platform/actors/publishing/monetize/pay-per-event)    

