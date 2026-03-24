---
layout: design-post
title:  "Bizsleuth"
tags: [ai, web-scraping]
---

Bizsleuth is a solution to a real problem I saw in B2B businesses. I saw that the sales and marketing teams get potential leads data from different sources and directories like `Google Map` but that information is often incomplete for targeted campaigns and lack essential information for lead-generation such as email address and highly useful information such as the business owner name. They often have to manually open each website url to find this missing info. This takes up huge time and effort. So I build bizsleuth to solve this problem.

Bizsleuth is an AI-powered lead-generation and lead-enrichment tool which searches for key information about a business in its website's content. 

Bizsleuth consists of a crawler that crawls the domain and internal pages upto max 10, gathers the text content from all these pages and then use AI to parse key info. This is why its important to feed the homepage url as an input instead of some random page url of the domain. 