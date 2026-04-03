---
layout: design-post
title: "How I Built and Monetized Bizsleuth: An AI-Powered Lead Enrichment Actor on Apify"
tags: [ai, web-scraping, apify]
published: false
---

I was working with a sales team in SaaS company who provides services in the fitness industry. The team had thousands of leads data acquired through web scraping extensions or third-party data providers. These leads lacked some key information such as business owner, emails and phone numbers (partially missing) but had website urls so they had to manually open each url and find the missing info. That's when I decided to build Bizsleuth

Bizsleuth is an AI-powered lead enrichment tool. You give it a list of business URLs, it crawls each domain across up to 10 pages, extracts the text content, and uses an LLM to parse out the key information a sales team actually needs: business name, owner name, contact email and more. The reason I require a homepage URL as input rather than any random page is that crawling from the homepage means the crawler follows navigation links to the most useful pages (About, Contact, Team) rather than crawling dead-end internal pages.

I initially built this for my own use. When I saw how much time it saved and realized I couldn't find anything in the market that did this using AI rather than regex patterns, I decided to publish it as an Apify Actor.

---

## Challenges I ran into

### AI context limits

Crawling multiple pages per domain and feeding that content to an LLM sounds straightforward, but website content across 10 pages can easily overflow an LLM's context window. To handle this, I used an open-source library by Zyte called `html_text` to extract clean text from raw HTML, then applied a truncation mechanism to cap the input at 10,000 tokens per website.

The 10,000 figure wasn't arbitrary. I sampled around 1,000 business websites in the fitness space and found that 10k tokens represented a reasonable average, enough to contain the information I was looking for without blowing past context limits. For other business types, the same ceiling holds reasonably well. If the key information exists on the site, it's almost always within the first 10k tokens of clean text.

### AI cost

At 10k tokens per website with the Gemini Flash Lite model, processing 1,000 websites costs roughly $1 in API fees. That's inexpensive enough to build a viable pricing model around. The truncation approach served double duty: it kept costs predictable and kept the context manageable.

### Apify platform cost

My first implementation used a browser-based crawler. It was slow and expensive. Apify charges by Compute Units (CU), which are a function of time and memory, so browser overhead directly hits your bill. Most business websites don't need JavaScript rendering to surface contact info, so the browser wasn't earning its cost. Switching to `HttpCrawler` increased speed and reduced CU consumption by at least 3x.

### Concurrency and OOM

The Actor processes URLs asynchronously, which worked fine on small inputs. On larger inputs, 1,000+ URLs which are also deeply crawled, it would crash with an Out of Memory (OOM) error. Firing up that many concurrent tasks at once guaranteed exhausting available RAM.

I fixed this with `asyncio.Semaphore` to cap concurrency at 10 simultaneous websites. Processing 1,000 websites now takes 25–30 minutes, which is reasonable given that each site is crawled deeply and run through AI parsing.

After publishing the Actor, a user reported a failure on a large input. I went to the Insights tab in the Apify console, opened the Debugging section, and found the specific failed run. The logs showed RAM usage spiking past the allocated maximum, which confirmed the OOM issue I'd suspected. Apify's resurrection feature let the user continue from where the run stopped, but it was a bug I needed to fix properly, not just work around.

![Insights tab in the Apify console](/assets/images/bizsleuth/insights.png)
![Debugging view](/assets/images/bizsleuth/actor-debug.png)
![Run detail showing logs and resource usage](/assets/images/bizsleuth/run.png)

---

## Turning a local script into an Apify Actor

I built the solution locally first, verified it worked, then actorified it. The Apify docs are comprehensive on the mechanics, but configuring input and output schemas tripped me up the first time. Their AI assistant in the docs was useful for getting quick answers when I was going in circles. Here's what the configuration actually looks like.

### Reading Actor input

```python
actor_input = await Actor.get_input() or {}
start_urls = actor_input.get("startUrls", [])
```

```python
urls_list = []
for start_url in start_urls:
    if "requestsFromUrl" in start_url:
        response = requests.get(start_url["requestsFromUrl"])
        urls_list.extend(response.text.splitlines())
    else:
        urls_list.append(start_url["url"])
```

The `if` block handles users uploading a text file of URLs (one per line). The `else` block handles URLs entered directly through the input UI. You need to support both because users come in with different workflows.

### input_schema.json

This file tells Apify what to render in the input UI. It lives inside the `.actor` folder at your project root.

```json
{
    "title": "bizsleuth input",
    "description": "This is the input for the bizsleuth actor.",
    "type": "object",
    "schemaVersion": 1,
    "properties": {
        "startUrls": {
            "title": "Start URLs",
            "type": "array",
            "description": "URLs to start with",
            "editor": "requestListSources",
            "prefill": [
                { "url": "https://www.example.com" },
                { "url": "https://www.example.com/some-path" }
            ]
        }
    }
}
```

![Actor input UI rendered from the schema](/assets/images/bizsleuth/actor_input_schema.png)

### dataset_schema.json and output_schema.json

Output configuration requires two files. `dataset_schema.json` validates what gets pushed to the dataset: if a field comes in with the wrong type, Apify returns a 400 and discards that item.

```json
{
    "actorSpecification": 1,
    "fields": {
        "$schema": "http://json-schema.org/draft-07/schema#",
        "type": "object",
        "properties": {
            "business_name": { "type": ["string", "null"] },
            "business_owner_name": { "type": ["string", "null"] },
            "contact_email": { "type": ["string", "null"] }
        }
    }
}
```

`output_schema.json` shapes what the user sees in the Output tab of the console.

```json
{
    "actorOutputSchemaVersion": 1,
    "title": "Output schema of bizsleuth",
    "properties": {
        "business_name": {
            "type": "string",
            "title": "Business Name 🏢",
            "template": "{{links.apiDefaultDatasetUrl}}/items?view=business_name"    
        },
        "business_owner_name": {
            "type": "string",
            "title": "Business Owner Name 👤",
            "template": "{{links.apiDefaultDatasetUrl}}/items?view=business_owner_name"
        },
        "contact_email": {
            "type": "string",
            "title": "Contact Email 📧",
            "template": "{{links.apiDefaultDatasetUrl}}/items?view=contact_email"
        }
    }
}
```

`{{links.apiDefaultDatasetUrl}}` gets replaced with your actual dataset URL at runtime. The `/items` endpoint then lets users view or download specific fields.

### actor.json

This is the metadata file that gives your Actor identity on the platform. It links all the schema files together.

```json
{
    "actorSpecification": 1,
    "name": "bizsleuth",
    "version": "0.0",
    "input": "./input_schema.json",
    "required": ["startUrls"],
    "output": "./output_schema.json",
    "storages": {
        "dataset": "./dataset_schema.json"
    }
}
```

### Dockerfile

Apify runs Actors inside containers. For a Python Actor that doesn't use a browser, the `actor-python` base image is the right starting point. If your solution needs Playwright, use `actor-python-playwright`. These base images are optimized for the Apify platform, so use them rather than rolling your own.

---

## Deploying via GitHub

There are two ways to deploy an Actor: Apify CLI or linking a GitHub repo. I use GitHub and I'd recommend it. Here's why: it gives you a staging layer between your machine and production. I run a second Actor pulling from the same repo that I use for testing. Changes go there first. When I'm confident, I rebuild the production Actor. You also get the option to enable auto-builds on push, which fits naturally into any CI/CD workflow.

**Step 1:** In the Apify console, click *Develop new*.

![Create actor screen](/assets/images/bizsleuth/create_actor.png)

**Step 2:** Select the GitHub option.

![Link repo option](/assets/images/bizsleuth/2_link_repo.png)

**Step 3:** Pick your repository.

![Pick repo](/assets/images/bizsleuth/3_pick_repo.png)

If your Actor uses an API key, add it in the *Environment Variables* section under the *Source* tab. Don't hardcode it in your Dockerfile or code.

![Environment variables section](/assets/images/bizsleuth/env_variable.png)

Before making the Actor public, run it against different input sizes and note the CU consumption and execution time at each. Test with 1GB and 2GB RAM configurations (Apify ties CPU allocation to RAM: 1GB = 0.25 cores, 2GB = 0.5 cores). This testing step will surface edge cases, give you real cost numbers to price against, and let you catch anything broken before real users encounter it.

---

## Pricing

Figuring out what to charge for your first SaaS tool is genuinely difficult. You're estimating costs that move (API usage, CU consumption) and guessing at what users will pay.

My starting point was $5 per 1,000 URLs. That covers my Apify platform costs plus the Gemini API fees and leaves some margin. It's also cheap enough that a sales team can justify it. I went with this as a starting point knowing I could adjust, but it's worth knowing that adjustments are not symmetric on Apify.

Reducing your price takes effect immediately. Increasing your price takes 14 days to kick in, and you can only change it once per month. Existing users don't like price increases and may churn. So price conservatively at first, with room to cut if you need to, not room to raise.

Apify also takes 20% of your revenue, so your actual profit formula is:

```
profit = (0.8 × revenue) - platform costs
```

Factor that in before you set your price.

### Pay Per Event (PPE)

Apify supports a few monetization models. I use [Pay Per Event (PPE)](https://docs.apify.com/platform/actors/publishing/monetize/pay-per-event) and it suits this kind of Actor well. You define events in your code and set a charge per event. I created a `process-url` event that fires once per URL processed (crawling + AI parsing combined). Implementation is a single method call:

```python
Actor.charge(event_name="process-url")
```

You can also use events like `request-url` that fire per HTTP request, depending on what maps most cleanly to your Actor's value delivery.

---

## README

Your README is the first thing a user reads and serves as the documentation for your actor. Write it clearly: what the Actor does, what the input format is, what the output contains, and anything the user needs to know before running it.

![Bizsleuth README in the Apify console](/assets/images/bizsleuth/readme.png)

A demo video is worth making if you can. Your users will come from different backgrounds. Many won't have used Apify before, and a 3-minute video showing a real run is often clearer than any amount of written documentation.

---

## Getting your first users

After publishing, I posted in the [Apify Discord community](https://discord.com/invite/jyEM2PRvMU). It was more effective than I expected for getting early users. Apify also has a [marketing checklist for actors](https://docs.apify.com/academy/actor-marketing-playbook/promote-your-actor/checklist) that's worth going through systematically.

---

## What I'd do differently

**Ship earlier.** My initial Actor had the OOM bug. I didn't know it existed until a real user hit it on a large input. You can't anticipate every failure case from a local environment. You need real users on real inputs. Ship a working MVP, not a perfect one, and fix bugs as they surface.

**Start with the lightest possible crawler.** I wasted time and money on browser-based crawling before switching to `HttpCrawler`. The default assumption should be that you don't need a browser. If you find a site that requires JavaScript rendering for the content you need, handle it as a special case. Don't build your whole Actor around it.

---

## What Apify handles for you

The thing that surprised me most about building on Apify was how much infrastructure is just taken care of. Once your Actor is published, Apify takes care of:

- Cloud hosting and execution environment
- Automatic scaling
- API access for users
- Billing and payment processing
- Dataset storage and retrieval
- Logging and monitoring
- SEO for your Actor

For a solo developer building a tool that needs to reach real users quickly, that list matters. The gap between "working script on my machine" and "product that strangers can pay to use" is usually enormous but Apify makes that gap minimal.