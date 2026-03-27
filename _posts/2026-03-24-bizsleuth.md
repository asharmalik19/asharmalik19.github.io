---
layout: design-post
title:  "Bizsleuth"
tags: [ai, web-scraping]
---

Bizsleuth is a solution to a real problem I saw in B2B businesses. I saw that the sales and marketing teams get potential leads data from different sources and directories like `Google Map` but that information is often incomplete for targeted campaigns and lack essential information for lead-generation such as email address and highly useful information such as the business owner name. They often have to manually open each website url to find this missing info. This takes up huge time and effort. So I build bizsleuth to solve this problem.

Bizsleuth is an AI-powered lead-generation and lead-enrichment tool which scrapes key information about a business from its website's content. 

Bizsleuth consists of a crawler that crawls the domain and internal pages upto maximum 10 pages, gathers the text content from all these pages and then use AI to parse key info. This is why its important to feed the homepage url as an input instead of some random page url of the domain so that the crawler can crawl the navigation pages instead of some random links.  

Initially when I built the solution, I didn't think of actorising the solution but when I saw its impact, I saw a common pain point for B2B businesses and a gap in the market. I couldn't find any tool in the market that can extract commonly required information for lead-generation using the power of AI to get the information reliably. The existing solutions were using regular expressions or techniques which can result in lower accuracy, higher false positives and false negatives. But nothing good ever happens without challenges, right. So here are some challenges I faced

#### Challenges
- AI context: 
    Crawling websites and then using AI to parse required information seems intuitive but the website content across multiple pages can exceed the maximum context length. To solve this problem, I used an open source library by *Zyte* called `html_text` to extract clean text from html source pages and used truncation mechanism to ensure that the context doesn't exceed a limit of 10000 tokens for any given website. 10000 is not some magic number but I found 10k to be an average of a sample of business websites (about 1k websites) in my case where I was dealing with fitness. But regardless of type of business, 10k tokens represent a good amount of text content for any type of business if not exceed it will most likely contain the information we are looking for.

- AI costs: (modern solutions incur modern costs)
    AI API is not free for a developer like me (and you unless you are someone who works at one of the AI labs replacing developers). Cleaning the website content and truncating the text reduced 
    


