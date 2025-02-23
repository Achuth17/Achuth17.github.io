+++
title = 'Hacker News post summarization with a thinking model'
date = 2025-02-23T13:22:27-08:00
draft = false
tags = ['LLM', 'HackerNews', 'Gemini']
+++

If you are anything like me, you spend a lot of time reading through the posts and discussions on the front page of Hacker News each day. Sometimes even after spending an hour or more on Hacker News, I come out with a sense that I haven’t fully caught up on the posts or the discussions.

For the uninitiated, Hacker News is a site that aggregates news and blogs about tech, science and innovation with a social element to it. Users can share stories they find interesting as text or links and have discussions about it.  I personally learn a lot by going through a handful of Hacker News posts and the discussions each day.  Posts that make it to the coveted front page often have thought provoking discussions, aspects of the topic missed by the original post. The contents of the post combined with the vibrant discussions give a more holistic view of the subject than reading just the post or just the discussion on Hacker News.

But the main issue, like I mentioned earlier, is the effort it takes to keep up with the sheer volume of discussions and postings that happen on Hacker News each day. One might wonder if it is even necessary to keep up with all the postings and that is true, one need not keep up with all the postings on the front page of Hacker News but even if you decided to follow one interesting thread on Hacker News, the number of comments can go well over a thousand. How do I get a gist of all the different discussions happening either in a thread or across the front page and selectively focus on the ones that interest me?

To solve this problem I started looking into some of the popular Large Language Models (LLMs) to distill the discussions into concise snippets. Thinking models (LLMs that can reason through a prompt before generating a result using techniques like chain-of-thought prompting or tree-of-thought reasoning) particularly seemed to fit the bill nicely. These models tend to do better when asked to reason through a posting and the perspectives brought forth by the top comments in a thread.

Before starting with the implementation, I had a few top levels goals for the result -

1. The result should include a summary of the Hacker News posting and this should be provided independent of the summary of the discussion.  
2. The summary of the discussion should be grouped into individual points based on the topic being discussed.  
3. Each point in the discussion summary should cite the comments it is summarizing. (This is important to avoid hallucinations.)  
4. The result should be in a human readable format, preferably Markdown.

### Implementation Goals

1. **Fetching the contents of the post:** For a provided Hacker News post id, fetch the contents of the URL shared in the post. A Hacker News post can be a URL or a block of text. For this task, I'm only focusing on the URL posts.  
2. **Converting the post to Markdown:** Since the posts usually tend to be very text heavy like a blog post or a product announcement post, converting the post to the Markdown format is a good way to share the post with an LLM.  
3. **Fetching the discussions from the Hacker News thread:** For a provided Hacker News post id, fetch the top comments from the discussion with the associated comment metadata.  
4. **Structuring the prompt:** Arrange the contents of the post and the discussion into a prompt template that instructs the model to generate a meaningful summary meeting the goals listed above.  
5. **Invoking the LLM:** Choose a model and supply the generated prompt template to the model with associated hyper-parameters to generate the summary. (More on model choice later.)  
6. **Output the summary:** Once the summary is returned by the LLM, output the results in a structured format. Bonus points if the LLM can output the result in a structured format.

The code associated with this post can be found on Github [here](https://github.com/Achuth17/py-hn-summarizer). The repository contains the main Python script, prompt templates, and sample summaries.

### Technical Implementation

#### Step 1: Fetching the top posts, the post contents and the discussion from Hacker News

Hacker News has a great REST API with [documentation](https://github.com/HackerNews/API). Since I work with a Python script, I decided to use a Python wrapper for the Hacker News API called [Haxor](https://github.com/avinassh/haxor). The unofficial python client provides a simple interface to access all the top posts with its external URL, top comments and the comment metadata.

```py
from hackernews import HackerNews  
from hackernews import Item

def get_top_story_urls(self, count: int):          
        hn = HackerNews()  
        top_stories = self.hn.top_stories(limit=count)  
        urls = []  
        for story in top_stories:  
            if story.item_type == 'story':  
                urls.append(story.url)  
        return urls

def get_top_story_comments(self, item: Item, count: int = 50):  
        comments = []  
        if not item.kids:  
            return comments  
        for comment in item.kids:  
            if len(comments) == 50:  
                break  
            if not comment.text or comment.deleted or comment.dead:  
                continue  
            comments.append(comment)  
        return comments  
```

Once the URL is available, Python’s requests library is great for fetching a web page in the HTML format.

```py
import requests

def _fetch_url_contents(self, url: str):      
        response = requests.get(url)  
        if response.status_code == 200:  
            return response.text  
        else:  
            ""  
```

#### Step 2: Converting the contents of the post as a Markdown (But why do it?)

The post fetched from Hacker News URL is almost always in HTML format but Markdown is a better  representation to share with an LLM. Markdown is more concise, human readable and leads to less confusion without all the additional noise. Avoiding all the tags and the noise also helps us keep the LLM token usage down.

The html2markdown library is great for converting HTML files to Markdown but the library is not perfect. It does leave the headers and footers behind and there are some edge cases where image descriptions are left behind with weird formatting. I have not fixed these issues in the first iteration.

```py
import html2text

def _convert_contents_to_markdown(self, contents: str):  
        h = html2text.HTML2Text()  
        h.ignore_links = True  
        h.ignore_images = True  
        return h.handle(contents)  
```

#### Step 3: Structuring all the data into a prompt

So far, we have the contents of the post in the Markdown format, the top comments with its metadata. We need an LLM to summarize the post, summarize and group the comments but topic and include the citations. I decided to take the easy road for this step and all all the requirements into a single prompt. You can find the latest version of this prompt [here](https://github.com/Achuth17/py-hn-summarizer/tree/main/prompts).

Sample prompt -

```
System prompt instructions:

    1. Summarize the article into bullet points with the title - "Summary".  
    2. Group Comments that carry a similar message.  
    3. For each group of Comments, summarize the Comments about the article into bullet points with the title - "Discussion Summary".  
        a. For each group of comments being summarized, include attribution to the original Comments in the format: (<Comma separated list of https://news.ycombinator.com/item?id=<comment_id>>)

Article Contents:

ARTICLE_CONTENTS

Top comments in the format 'comment_id : comment contents'

COMMENT_CONTENTS  
```

Is there any room for improvement in the prompt? Yes. But does the first version of the prompt already do what I want? Also Yes. So I will kick the “prompt improvements” can down the road.

#### Step 4: Which LLM to use?

I decided to go with `gemini-2.0-flash-thinking-exp-01-21` (rolls off the tongue) for the first iteration. Why did I go with Gemini? Keeping my personal biases aside, I picked Gemini models for a few reasons -

1. **Generous free tier for prototyping:** As I write this, the thinking models are free up to 1500 requests per day on aistudio.google.com. The quota is always free, unlike Claude, for example, whose limits are set dynamically based on daily demand rather than being explicitly stated.  
2. **Low latency:** The latency for even the frontier and experimental thinking models is surprisingly low in my tests. I prefer not waiting for minutes to iterate on my prompts.  
3. **Price to performance ratio of paid tier:** Short of hosting the reasoning model myself locally, which I don’t plan to do at any point in the future,  the price to performance ratio of Gemini was the best in the market in-case I wanted to scale this up as a service.

Gemini invocation is fairly simple, I have a small snippet of what I did below -

```py
import os  
import google.generativeai as genai

genai.configure(api_key=os.environ["GEMINI_API_KEY"])

class GeminiClient:  
    def __init__(self):  
        # Create the model  
        generation_config = {  
            "temperature": 0.7,  
            "top_p": 0.95,  
            "top_k": 64,  
            "max_output_tokens": 65536,  
            "response_mime_type": "text/plain",  
        }  
        self.model = genai.GenerativeModel(  
            model_name="gemini-2.0-flash-thinking-exp-01-21",  
            generation_config=generation_config,  
        )

    def ask_gemini(self, query: str) -> str:          
        chat_session = self.model.start_chat(history=[])

        response = chat_session.send_message(query)  
        return response.text  
```

#### Step 5: Structuring the output

The prompt had specific instructions on how to structure the output in Markdown including specific instructions around including citations in the output. Gemini is surprisingly good at following my instructions and I did not run into any issues in the testing.

The final summaries are written as a text file in the `library/` folder for now. I have a few sample summaries generated by the script [here](https://github.com/Achuth17/py-hn-summarizer/tree/main/library)

### Final Thoughts 

The instruction following capabilities are great in the Gemini thinking model I used. I tried citing the urls of the comments that contributed to each point in the comment summarization block and Gemini was able to do this with minimal prompting and examples.

There is plenty of room for improvement in the prompting, error handling and Markdown conversion areas. The goal of this initial exercise was to hack together a solution to check the feasibility of this approach. I will explore these areas next and share more here.

### Future Work

1. This whole process could have been faster if I used a tool like Windsurf or Cursor to code. I will explore this too in the next iteration.  
2. Checking if open weight models like DeepSeek R1 can be an option for this task. I have a version of R1 running locally using Ollama, I plan to explore if R1 can replace Gemini thinking for this use case.  
3. Exploring other reasoning LLMs like O1, xAI’s Grok 3 or Perplexity Sonar based Deepseek R1 1776.  
4. Turning this script into a service that users can invoke locally or visit as a website to get a daily or weekly gist of events happening on Hacker News.
