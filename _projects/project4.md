---
title: "Tik-Tok style news webapp"
date: 2025-10-07
category: Web-Development
description: "Scrolling Tik-Tok but with an upside."
status: Complete
tools: [LLM, Web-development]
github: "Unable to provide as it is currently deployed as an app by a friend"
---

The modern consumption of information is a paradox: we have access to more news than ever before, yet staying genuinely informed feels increasingly exhausting. The default user behavior has shifted heavily toward the "infinite scroll" paradigm popularized by TikTok. Recognizing this friction, a friend and I collaborated on a project back in the day to bridge the gap between long-form journalism and short-form attention spans.

The goal was simple in theory but complex in execution: aggregate top global news, synthesize it into objective, bite-sized bullet points using a custom-trained Large Language Model (LLM), and serve it up in a highly swipeable, frictionless user interface.

I've since handed the reins of this project over to my friend, but looking back at the underlying codebase offers a great case study in data pipelining, concurrent I/O operations, and practical, custom AI integration. Here is a deep dive into the architecture that powered the app.

---

## Phase 1: The Ingestion Engine (RSS Aggregation)

Before we could summarize the news, we had to source it. Relying on a single API is often expensive and limits journalistic diversity. Instead, we opted to build a custom aggregator that taps directly into the RSS feeds of major publications (Reuters, BBC, Al Jazeera, NYT, etc.).

Because fetching XML data from dozens of endpoints synchronously would be agonizingly slow, we utilized Python's `concurrent.futures.ThreadPoolExecutor` to parallelize the network requests. This I/O-bound task is a textbook use case for multi-threading.

```python
def get_all_news() -> list:
    all_news = []
    with ThreadPoolExecutor(max_workers=len(RSS_FEEDS)) as executor:
        future_to_category = {
            executor.submit(get_news_for_category, category, urls): category
            for category, urls in RSS_FEEDS.items()
        }
        for future in as_completed(future_to_category):
            category = future_to_category[future]
            try:
                category_news = future.result()
                all_news.extend(category_news)
            except Exception as exc:
                logging.error(f'{category} generated an exception: {exc}')

    random.shuffle(all_news)
    return all_news[:TARGET_ARTICLES]
```

**The Academic Takeaway:** Network latency is the enemy of data aggregation. By mapping our RSS endpoints to separate threads, we effectively decoupled the total execution time from the sum of individual request times, binding it instead to the duration of the slowest single request.

---

## Phase 2: The Cognitive Layer (Custom AI Summarization)

Raw RSS descriptions are often just the first paragraph of an article — hardly a complete summary. To generate actual value, we needed to pass the content through an LLM.

Here is where the architecture gets particularly interesting. While it is easy to plug in an off-the-shelf API, the `GROQ_API_KEY` in our environment actually pointed to a **custom LLM that I trained myself**, hosted on Groq's high-speed inference infrastructure.

Why train a custom model instead of just prompting a standard one? Off-the-shelf models often struggle to maintain a strict, objective, non-editorialized tone across thousands of diverse news categories. By fine-tuning my own model, I could guarantee that the output adhered perfectly to our strict 3-4 bullet point format, eliminating the conversational "fluff" that usually plagues LLM outputs.

```python
def summarize_content(article_key, content, title, url, source, category, image_url, pub_date, max_retries=5, initial_delay=5):
    prompt = f"""
    Title: {title}
    Source: {source}
    Category: {category}
    Content: {content}

    Please provide a concise summary of the news article in a single paragraph. The summary should:
    1. Capture the main point and key details of the article.
    2. Be written in a clear, objective manner without editorializing.
    3. Be about 3-4 bullet points that build upon one another.
    4. Avoid using phrases like 'this article' or 'the author states'.
    """

    delay = initial_delay
    for attempt in range(max_retries):
        try:
            chat_completion = client.chat.completions.create(
                messages=[{"role": "user", "content": prompt}],
                model="mixtral-8x7b-32768",
                max_tokens=300,
                temperature=0.3
            )
            return chat_completion.choices[0].message.content

        except requests.exceptions.RequestException as e:
            if "429" in str(e):
                retry_after = e.response.headers.get('Retry-After', delay)
                time.sleep(int(retry_after))
            # ... exponential backoff logic
```

**The Academic Takeaway:** Hosting a custom model on Groq was a strategic necessity. A TikTok-style feed requires content generation at scale. Groq's architecture processes language tokens exponentially faster than traditional GPU clusters, meaning our background summarizer could chew through 50+ articles in seconds without bottlenecking the data pipeline. We paired this speed with an exponential backoff algorithm to gracefully handle any API rate limits.

---

## Phase 3: The Delivery Mechanism (Flask Backend)

With our data nicely packaged into `points.json`, we needed a way to serve this to the frontend. Because the heavy AI processing occurs asynchronously in the background, the backend does not need to do any heavy lifting at runtime.

```python
from flask import Flask, jsonify
from flask_cors import CORS
import json
import os

app = Flask(__name__)
CORS(app)

@app.route('/api/articles')
def get_articles():
    try:
        file_path = os.path.join(os.path.dirname(__file__), '..', 'points.json')
        with open(file_path, 'r') as file:
            articles = json.load(file)
        return jsonify(articles)
    except FileNotFoundError:
        return jsonify({"error": "Articles file not found"}), 404
```

This decoupled architecture — a cron-style background worker generating a JSON file, paired with a fast API serving that file — ensures that the end-user never experiences the latency of the RSS fetching or the LLM inference.

---

## Phase 4: The Presentation (React Frontend)

While the heavy engineering lived in the Python backend, the user experience was driven by a React frontend built to ingest the Flask API's array of objects. The frontend logic maps the data into a vertical, snap-scrolling container where each card displays the title, source, and AI-generated bullet points. By constraining the reading experience to the exact dimensions of a mobile screen, the app removes the cognitive load of navigating traditional news sites.

---

## Final Thoughts

Building this app was a masterclass in pipeline orchestration. It is one thing to write a script that summarizes a text file; it is an entirely different engineering challenge to train a custom LLM, deploy it on high-speed infrastructure, and build a fault-tolerant system that constantly polls the web and serves data instantaneously to a client interface.

While my friend is now the one steering the ship and optimizing the frontend, the underlying architecture remains a testament to the power of combining traditional web scraping techniques with custom generative AI. The result is a system that filters out the noise, leaving only the signal.
