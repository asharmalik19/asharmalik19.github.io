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

I first built my solution and used it locally. Then I actorified my code and published on Apify. Apify docs are pretty comprehensive about explaining how to [build your actor](https://docs.apify.com/academy/getting-started/creating-actors) but configuring your input and output can get confusing if this is your first actor. Apify has made everything so accessible by bringing AI assistance to their docs. Its a great way to get quick answers and pointers in the right direction. The AI assistant has been tremendously helpful to me and I hope you find it helpful as well. To configure the input for actor, i did this

```python
actor_input = await Actor.get_input() or {}
start_urls = actor_input.get("startUrls", [])
```
This reads the actor's input which is a json object and retrieves a list which contains the input urls.

```python
urls_list = []
for start_url in start_urls:
    if "requestsFromUrl" in start_url:
        response = requests.get(start_url["requestsFromUrl"])
        urls_list.extend(response.text.splitlines())
    else:
        urls_list.append(start_url["url"])
```
A user can provide the input urls in the form of a text file which contains urls separated by a newline or he can specify urls individually in the input form. The `if` block handles the user upload a text file containing the input while the `else` block here handles the user input provided through the input UI (which can be a form or a json). In addition to reading the input in your code, you also need to configure a `json` file named `input_schema.json` and put that inside the `.actor` folder at root. This file tells Apify what to show the user to input. Here is the minimum version of your `input_schema.json` file

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
This schema shows the user that the actor expects an array of urls and he can use multiple ways to enter his input.
![Alt text](/assets/images/bizsleuth/actor_input_schema.png)

After you have configured the input for your actor, then comes the part where you define output and you have to configure two json files for that, `dataset_schema.json` and `output_schema.json`. The `dataset_schema.json` file provides validation for your output. Here you specify the output fields for your actor with their expected datatypes. For example see my `dataset_schema` file

```json
{
    "actorSpecification": 1,
    "fields": {
        "$schema": "http://json-schema.org/draft-07/schema#",
        "type": "object",
        "properties": {
            "business_name": {
                "type": ["string", "null"]
            },
            "business_owner_name": {
                "type": ["string", "null"]
            },
            "contact_email": {
                "type": ["string", "null"]
            }
        }
    }
}
```
This specifies that the expected output contains 3 fields and each field has type *string* or *null*. If any other data type of field is pushed to the output, the Apify API will return *400* error code and discard the output.

The `output_schema.json` is what actually shapes the output tab in the Apify console
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
The `{{links.apiDefaultDatasetUrl}}` gets replaced by your actual actor dataset url at runtime and the `/items` provides an endpoint to access or view the items or fields in your output. As you can see, you can also set icon for each field and it will be displayed in the output tab. 

After configuring your input and output schemas as above, you also need to configure 1 other file called `actor.json`. This file is essential because it contains metadata about your actor which gives identity to your actor on the Apify platform. It contains the name and version of your actor and links the input and output schema files.
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
This defines your actor name and its version and also lists the path of the input and output schema files you defined earlier. 

Apify runs an actor inside a container so you need to configure a Dockerfile as well. You can put the Dockerfile at the project root folder or inside the `.actor` folder but i like to keep it at the root folder. Depending upon your solution, you can choose a base image provided by Apify. For example, if your solution uses simple python and doesn't use a browser, you can start with `actor-python` base image. If it uses playwright, then you can start with `actor-python-playwright`. These [docker images](https://docs.apify.com/platform/actors/development/actor-definition/dockerfile) are optimised for the Apify platform.

So once you have all the elements in place to turn your solution into an Apify [actor](https://docs.apify.com/platform/actors/development/actor-definition), there are 2 main ways to deploy your actor to Apify platform, using the *apify cli* or *linking a github repo*. Linking a github repo is the recommended approach and that is what I have used for my actor. Here is how to do so:

**Step 1** — Go to the Apify console and click *Develop new* to start creating a new actor.

![Alt text](/assets/images/bizsleuth/create_actor.png)

**Step 2** — Select the *Github* option to link your repository.

![Alt text](/assets/images/bizsleuth/2_link_repo.png)

**Step 3** — Pick the repo that contains your actor code and confirm.

![Alt text](/assets/images/bizsleuth/3_pick_repo.png)

That's it! you have deployed your first actor. I like deploying via *Github repo* because that gives me complete control over my project. For example, if I want to test some changes, I can push the changes to the project repo but instead of building my production actor which is used by real users, I can set up another actor which pulls from the same Github repo and use that for staging and testing purpose before pushing anything to the main production actor. I can also push any changes to the github repo while working through multiple machines or if collaborating with other developers on a project, perform any CI/CD or anything as such before creating a *build* on Apify. In short, the Github repo provides an intermediate space between your machine and Apify platform and it gives me a lot of control over my project. You can also set the automatic build so that everytime you push some code to your Github repo, the actor automatically rebuilds. If you are interested in reading more about deploying an actor, you can read [here](https://docs.apify.com/academy/deploying-your-code/deploying)

When you make your first build, Apify creates a docker image from your `Dockerfile` and runs that image as a container when you run the actor. If your actor relies on an API key, you need to provide it in the *Environment variables* section located in the *source* tab of your actor's console
[!Alt text](/assets/images/bizsleuth/env_variable.png)

Run your actor on different samples. Observe the actor performance and note down the costs and execution time across different input sizes. Test your actor with different run configurations (like 1GB or 2GB RAM) to find optimal configuration for your actor. Apify RAM configuration also has *cpu* usage tied to it, 1GB RAM allocates 0.25 cpu core, 2GB RAM allocates 0.5 cpu core and so on. Find more on that [here](https://docs.apify.com/platform/actors/running/usage-and-resources). Apify charges you for the compute resources used by your actor but your actor might be using external APIs so make sure that to take those costs into account as well. This step gives you a good estimate for how to price your actor. If your code is using any unnecessary dependency (like a browser), make sure to remove that dependency or have a good reason to keep that. In my case, I found that I don't really need a browser in my solution and I can get good results with only `httpcrawler`, so I removed the browser and it exponentially increased the speed and reduced the cost of my actor. This step is important before making your actor public as this will give you confidence in your actor, surface any edge-cases or bugs. The monetization part can be done later once you are ready to price your actor and you have even real users but keep in mind that it will take 14 days to take effect after your changes.

#### Pricing
Once you have a good cost estimate, make a *pricing plan* that covers all your costs (Apify platform as well as any third-party API) and makes some profit. Apify provides 2 main ways to monetize your actor, *Pay Per Event (PPE)* and *Pay Per Result (PPR)*. 
*PPE (Recommended)*: You can create events and whenever those events happen in your code, user will get a charge. For example, I created an event called *process-url* which charges a small specified amount for processing each url which includes crawling the website and then AI parsing. You can create an event called *request-url* which charges the user for every successful `http request` or whatever suits your use-case. Extrapolate these small event charges over a useful output. I set my initial pricing as $5/1000 urls because that covers my costs and makes me some profit. My initial pricing is simple. It's also easy to cut down costs of your Apify actor as it takes effect immediately and any sane user would like it but if you increase the cost or make major changes to your pricing, it will take 14 days to take effect and you can't do it more than once in a month. So any positive change in pricing for a user takes effect immediately but any negative change for a user takes [14 days](https://docs.apify.com/platform/actors/publishing/monetize/pay-per-event#pass-platform-usage-to-users) to take effect.  
 
One other thing to be aware of is that the Apify will also take 20% of the profit you make (because they also need to run the company), so your actual profit is calculated using this formula
    `profit = (0.8 * revenue) - platform costs`

Read more about PPE [here](https://docs.apify.com/platform/actors/publishing/monetize/pay-per-event)

#### README
An easy to read README file is a must have for your actor. It serves as a documentation for your actor. Clearly define the input, output, how to use the actor and anything that the user should be aware of. My README defines clear steps for the user on how to use the actor along with a note that the user should be aware of
[!Alt text](/assets/images/bizsleuth/readme.png)
It would also be a great help if you can record a demo video to clearly show the value that your actor provides and how to effectively use the actor. Remember that your users might be from a diverse set of backgrounds and most probably non-technical as well as it might be their first time on Apify platform, so you want to make using your actor as simple as possible for them.

Once your actor is published and properly working, monetization is set, a clear README is in place, now is the time for some marketing so that the wold knows about what you have built. After publishing my actor, I posted about it in the Apify [discord](https://discord.com/invite/jyEM2PRvMU) community. It was very helpful and I got some users. Apify has a pretty comprehensive checklist on how to market your actor that you can find [here](https://docs.apify.com/academy/actor-marketing-playbook/promote-your-actor/checklist)

One of my user complained about the actor failing on a large input. I went to the insights tab in my actor's console
[!Alt text](/assets/images/bizsleuth/insights.png)
then clicked on the Debugging section and selected my respective actor that I wanted to debug.
[!Alt text](/assets/images/bizsleuth/actor-debug.png)
[!Alt text](/assets/images/bizsleuth/2_debugging.png)
I found the run that resulted in a failure and clicked on it which showed me all the logs, RAM and CPU usage and all the useful information I could use for debugging. 
[!Alt text](/assets/images/bizsleuth/run.png)
The issue due to which the run failed was Out Of Memory (OOM). When the RAM usage exceeds the allocated maximum RAM, it causes the container to stop. A good thing about Apify is that it lets users *resurrect* the actor to continue running in case of such failures by storing the state and output but bugs like these clearly needs to be fixed. After fixing the potential causes for the issue, I rebuilt the actor with the updated code

#### What I have learned and what I would do differently when building my next actor
- *Don't wait until pefection* Your initial actor build will fail in some scenarios that you didn't anticipate. Actorify an MVP of your idea and let the real users use it, iteratively improve upon the feedback from users. You can never anticipate all the problems that will occur when different users will use your actor on different kind of inputs so don't wait for the perfect moment, simply publish a good enough version and not the perfect one.
- *Efficient Solution* I would try to make my solution as efficient and lightweight as possible. This increases the speed and reduces the costs for me and my user which is something very important for an Apify Actor. 

What surprised me about building on Apify is that how quickly I was able to reach real users and how much is already taken care for you. You just need to publish your Actor and Apify takes care of 
- Cloud hosting
- Automatic Scaling
- User-facing features such as usage via an API
- Billing
- Data Storage
- Logging and Monitoring
- Automated Testing
- SEO

This is honestly so much and makes building good solution effortless.























