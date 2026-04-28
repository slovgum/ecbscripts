# Beginner Tutorial: ECB Monetary Policy Press Conference Textual Analysis

This tutorial is written for a beginner using an empty GitHub Codespace with VS Code. We will:

1. Set up a conda environment.
2. Learn the basic idea of web scraping.
3. Use browser Inspect tools to find the text we want.
4. Download the ECB page with Python.
5. Parse the HTML and store the text.
6. Run sentiment analysis.
7. Create a word cloud.

This guidance provides the ready-to-go example python scripts. However, you can also follow the prompt-based AI agent assisted workflow, which summarized in Lecture Note 2 - Basic Textual Analysis.pdf on our elearning.

Target page:

```text
https://www.ecb.europa.eu/press/press_conference/monetary-policy-statement/2026/html/ecb.is260319~93b1cbad97.en.html
```

Important note: web pages can change. The selectors below match the ECB page structure observed for this page, where the useful article text is inside:

```text
main div.section
```

## 0. What We Are Building

By the end, your project will look like this:

```text
ecb-text-analysis/
  environment.yml
  scripts/
    ecb_text_analysis.py
  data/
    ecb_press_conference_2026-03-19.txt
  outputs/
    ecb_sentiment_summary.csv
    ecb_top_words.csv
    ecb_wordcloud.png
```

The final Python script will do all the work in one run.

## 1. Set Up the Codespaces Environment

### 1.1 Open a Codespace

1. Create or open a GitHub repository.
2. Click `Code`.
3. Click `Codespaces`.
4. Click `Create codespace`.
5. Wait for VS Code in the browser to open.

### 1.2 Check Whether Conda Exists

Open the VS Code terminal:

```bash
conda --version
```

If you see something like this, conda is already installed:

```text
conda 24.x.x
```

If you see `command not found`, install Miniforge:

```bash
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
bash Miniforge3-Linux-x86_64.sh -b -p $HOME/miniforge3
source $HOME/miniforge3/etc/profile.d/conda.sh
conda init bash
```

Then close and reopen the terminal.

### 1.3 Create `environment.yml`

In VS Code, create a file named `environment.yml` and paste this:

```yaml
name: ecb-text
channels:
  - conda-forge
dependencies:
  - python=3.11
  - requests
  - beautifulsoup4
  - lxml
  - pandas
  - nltk
  - matplotlib
  - wordcloud
  - ipykernel
```

### 1.4 Create and Activate the Environment

Run:

```bash
conda env create -f environment.yml
conda activate ecb-text
python -m ipykernel install --user --name ecb-text --display-name "Python (ecb-text)"
python --version
```

Expected result:

```text
Python 3.11.x
```

### Copilot Prompt for This Step

Paste this into Copilot Chat:

```text
I am a beginner using GitHub Codespaces and VS Code. Help me set up a conda environment for a Python textual analysis project. I need requests, beautifulsoup4, lxml, pandas, nltk, matplotlib, wordcloud, and ipykernel. Explain each command slowly and tell me how to verify that the environment is active.
```

## 2. Packages We Will Use

Here is the job of each package:

| Package | Why we use it |
|---|---|
| `requests` | Downloads the HTML page from the ECB website. |
| `beautifulsoup4` | Parses HTML so we can extract text from tags. |
| `lxml` | A fast parser used by BeautifulSoup. |
| `pandas` | Stores summary results and word-frequency tables. |
| `nltk` | Provides VADER, a beginner-friendly sentiment analyzer. |
| `matplotlib` | Saves the word cloud image. |
| `wordcloud` | Creates the word cloud image. |
| `pathlib` | Standard Python library for file paths. |
| `re` | Standard Python library for regular expressions and text cleaning. |
| `collections.Counter` | Standard Python library for counting words. |

## 3. Web Scraping Basics

### 3.1 What Is Web Scraping?

Web scraping means writing code that visits a web page and extracts information from it.

A normal web page is mostly HTML. HTML is built from tags:

```html
<h1>PRESS CONFERENCE</h1>
<p>Good afternoon, the Vice-President and I welcome you...</p>
```

Common tags:

| Tag | Meaning |
|---|---|
| `<h1>` | Main heading. |
| `<h2>` | Section heading. |
| `<p>` | Paragraph. |
| `<a>` | Link. |
| `<div>` | Generic container. |
| `<main>` | Main page content. |

The goal is to find the tags that contain the text we care about.

### 3.2 What Is Inspect?

`Inspect` opens the browser's Developer Tools. It lets you see the HTML behind the page.

Try this:

1. Open the ECB page in your browser.
2. Find the paragraph that begins `Good afternoon`.
3. Right-click the paragraph.
4. Click `Inspect`.
5. The Developer Tools panel opens.
6. The browser highlights the matching HTML element.

You should see that the article text is inside the page's main content area. On this ECB page, the useful text is inside:

```html
<main>
  <div class="title">...</div>
  <div class="section">
    ...
    <p>Good afternoon, the Vice-President and I welcome you...</p>
    ...
  </div>
</main>
```

That means a good CSS selector for the article body is:

```text
main div.section
```

### 3.3 What Is a CSS Selector?

A CSS selector is a short pattern for finding HTML elements.

Examples:

| Selector | Meaning |
|---|---|
| `main` | Find the `<main>` tag. |
| `p` | Find all paragraph tags. |
| `.section` | Find elements with `class="section"`. |
| `div.section` | Find `<div>` tags with `class="section"`. |
| `main div.section` | Find a `div.section` somewhere inside `<main>`. |

In our project, this is the main selector:

```python
section = soup.select_one("main div.section")
```

This means: from the parsed HTML, find the first `<div class="section">` inside `<main>`.

### 3.4 What Is XPath?

XPath is another way to describe the location of an element in an HTML tree.

Example XPath:

```text
//main//div[contains(@class, "section")]
```

How to read it:

| Part | Meaning |
|---|---|
| `//main` | Find a `<main>` tag anywhere in the page. |
| `//div` | Then find a `<div>` below it. |
| `@class` | Look at the element's `class` attribute. |
| `contains(@class, "section")` | Match if the class contains the word `section`. |

Browsers can copy XPath automatically, but copied XPaths can be too fragile. For example, a copied XPath may look like:

```text
/html/body/div[2]/main/div[2]/p[1]
```

That can break if the website adds one extra `div`. For this tutorial, the CSS selector `main div.section` is easier and more stable.

### 3.5 Scraping Etiquette

For a class exercise with one public page, this is usually fine. Still, remember:

1. Do not overload a website with many rapid requests.
2. Identify your script with a reasonable `User-Agent`.
3. Store the source URL with your data.
4. Respect copyright and terms of use.
5. Use the text for analysis, not for republishing the whole page as your own.

### Copilot Prompt for This Step

Paste this into Copilot Chat:

```text
Explain web scraping to me like I am a beginner. Use the ECB monetary policy statement page as the example. Explain HTML tags, browser Inspect, CSS selectors, XPath, and why a selector like main div.section is better than a long copied XPath.
```

## 4. Create the Project Folders

In the VS Code terminal, make two folders:

```bash
mkdir -p scripts data outputs
```

Expected result:

```text
scripts/
data/
outputs/
```

### Copilot Prompt for This Step

```text
I am starting a small Python text analysis project. Suggest a simple folder structure with scripts, data, and outputs folders. Explain what belongs in each folder.
```

## 5. Step 1 Code: Request the Web Page

Create this file:

```text
scripts/ecb_text_analysis.py
```

Start with this code:

```python
from pathlib import Path

import requests


# Step 1: Store the webpage address in a variable so we only need to write it once.
URL = "https://www.ecb.europa.eu/press/press_conference/monetary-policy-statement/2026/html/ecb.is260319~93b1cbad97.en.html"

# Step 2: Create folder paths for saved files.
# We make these now even though Step 1 only downloads the page.
# This keeps the project structure consistent from the beginning.
DATA_DIR = Path("data")
OUTPUT_DIR = Path("outputs")

# Step 3: Create the folders if they do not already exist.
# exist_ok=True means Python will not crash if the folders are already there.
DATA_DIR.mkdir(exist_ok=True)
OUTPUT_DIR.mkdir(exist_ok=True)

# Step 4: Send a browser-like identity to the website.
# Some websites are happier to respond when a request looks like it comes from a real browser.
headers = {
    "User-Agent": "Mozilla/5.0 (beginner text analysis tutorial)"
}

# Step 5: Download the ECB webpage.
# timeout=30 means Python will wait up to 30 seconds before giving up.
response = requests.get(URL, headers=headers, timeout=30)

# Step 6: Stop the script immediately if the website returns an error code.
# For example, 404 means page not found and 500 means server error.
response.raise_for_status()

# Step 7: Print simple checks so we know the download worked.
print("Downloaded page successfully")
print("Status code:", response.status_code)
print("Number of characters in HTML:", len(response.text))
```

Run it:

```bash
conda activate ecb-text
python scripts/ecb_text_analysis.py
```

Expected result:

```text
Downloaded page successfully
Status code: 200
Number of characters in HTML: many thousands
```

### What This Code Does

```python
requests.get(URL)
```

asks the ECB server for the page.

```python
response.raise_for_status()
```

stops the script if the page download failed.

```python
headers = {"User-Agent": "..."}
```

tells the website that the request comes from a browser-like script.

### Copilot Prompt for This Step

```text
Write beginner-friendly Python code using requests to download this ECB page:
https://www.ecb.europa.eu/press/press_conference/monetary-policy-statement/2026/html/ecb.is260319~93b1cbad97.en.html
Use a User-Agent header, a timeout, and raise_for_status. Print the status code and the length of the HTML. Explain every line.
```

## 6. Step 2 Code: Parse the Page and Extract the Text

Now replace the code in `scripts/ecb_text_analysis.py` with this larger version:

```python
from pathlib import Path
import re

import requests
from bs4 import BeautifulSoup


# Step 1: Store the webpage address in one variable.
URL = "https://www.ecb.europa.eu/press/press_conference/monetary-policy-statement/2026/html/ecb.is260319~93b1cbad97.en.html"

# Step 2: Define where saved files should go.
DATA_DIR = Path("data")
OUTPUT_DIR = Path("outputs")

# Step 3: Create those folders if needed.
DATA_DIR.mkdir(exist_ok=True)
OUTPUT_DIR.mkdir(exist_ok=True)


def clean_whitespace(text: str) -> str:
    # Step 4: Clean messy spacing.
    # HTML text can contain extra spaces, tabs, and line breaks.
    # This function turns repeated whitespace into a single clean space.
    """Turn repeated spaces, tabs, and newlines into single spaces."""
    return re.sub(r"\s+", " ", text).strip()


# Step 5: Identify our request as coming from a browser-like script.
headers = {
    "User-Agent": "Mozilla/5.0 (beginner text analysis tutorial)"
}

# Step 6: Download the webpage HTML.
response = requests.get(URL, headers=headers, timeout=30)

# Step 7: Crash early if the download failed.
response.raise_for_status()

# Step 8: Parse the HTML with BeautifulSoup so we can search inside it.
# "lxml" is the parser engine. It is fast and commonly used.
soup = BeautifulSoup(response.text, "lxml")

# Step 9: Find the main content block that holds the press conference text.
# We inspected the page in the browser and found that the useful content is in main div.section.
section = soup.select_one("main div.section")
if section is None:
    raise RuntimeError("Could not find the article section. The page structure may have changed.")

# Step 10: Remove page elements we do not want in the analysis text.
# This avoids mixing navigation or page metadata into the script text.
for unwanted in section.select('script, style, a[href="#qa"], .ecb-publicationDate'):
    unwanted.decompose()

# Step 11: Prepare an empty list to collect each cleaned heading or paragraph.
text_blocks = []

# Step 12: Loop through headings and paragraphs inside the section.
# We use both h2 and p because the section titles are meaningful text too.
for element in section.find_all(["h2", "p"]):
    classes = element.get("class", [])

    # Step 13: Skip the subtitle line with speaker names.
    # That line is useful on the webpage but not central to the analysis text.
    if "ecb-pressContentSubtitle" in classes:
        continue

    # Step 14: Extract visible text from the HTML tag and clean its spacing.
    text = clean_whitespace(element.get_text(" ", strip=True))

    # Step 15: Only keep non-empty text blocks.
    if text:
        text_blocks.append(text)

# Step 16: Combine all text blocks into one long script.
# We put blank lines between blocks to keep the text readable.
full_text = "\n\n".join(text_blocks)

# Step 17: Choose the output filename for the cleaned text.
text_path = DATA_DIR / "ecb_press_conference_2026-03-19.txt"

# Step 18: Save the full script as a UTF-8 text file.
text_path.write_text(full_text, encoding="utf-8")

# Step 19: Print a small report so we can check what happened.
print("Saved text to:", text_path)
print("Number of extracted text blocks:", len(text_blocks))
print()
print("Preview:")
print(full_text[:800])
```

Run:

```bash
python scripts/ecb_text_analysis.py
```

Expected result:

```text
Saved text to: data/ecb_press_conference_2026-03-19.txt
Number of extracted text blocks: many text blocks

Preview:
Good afternoon, the Vice-President and I welcome you to our press conference.
...
```

### What This Code Does

This line parses the raw HTML:

```python
soup = BeautifulSoup(response.text, "lxml")
```

This line finds the main text container:

```python
section = soup.select_one("main div.section")
```

This line removes page elements we do not want:

```python
for unwanted in section.select('script, style, a[href="#qa"], .ecb-publicationDate'):
    unwanted.decompose()
```

This loop extracts headings and paragraphs:

```python
for element in section.find_all(["h2", "p"]):
```

This stores the result as one clean text file:

| File | Why it is useful |
|---|---|
| `.txt` | Easy to read, score as one whole script, and use for word clouds. |

### Copilot Prompt for This Step

```text
Help me parse the ECB page with BeautifulSoup. The useful article text is inside the CSS selector main div.section. Extract h2 and p tags, skip the subtitle and publication date, clean whitespace, combine everything into one full script, and save it as a plain text file. Write beginner-friendly code and explain each block.
```

## 7. Step 3 Code: Sentiment Analysis for the Whole Press Conference

We will use VADER from NLTK.

VADER gives four scores:

| Score | Meaning |
|---|---|
| `neg` | Negative share of the text. |
| `neu` | Neutral share of the text. |
| `pos` | Positive share of the text. |
| `compound` | Overall sentiment from `-1` to `+1`. |

Common beginner interpretation:

| Compound score | Label |
|---|---|
| `>= 0.05` | Positive |
| between `-0.05` and `0.05` | Neutral |
| `<= -0.05` | Negative |

Important caution: VADER is a general sentiment tool. Central bank text is technical, so treat the score as exploratory, not as a perfect measure of economic optimism or policy stance.

In this first tutorial, we do sentiment analysis once for the whole press conference script. That means we combine the full text into one string and score that full string.

Replace your script with this version:

```python
from pathlib import Path
import re

import nltk
import pandas as pd
import requests
from bs4 import BeautifulSoup
from nltk.sentiment import SentimentIntensityAnalyzer


# Step 1: Store the ECB webpage address.
URL = "https://www.ecb.europa.eu/press/press_conference/monetary-policy-statement/2026/html/ecb.is260319~93b1cbad97.en.html"

# Step 2: Define folders for saved data and outputs.
DATA_DIR = Path("data")
OUTPUT_DIR = Path("outputs")

# Step 3: Create the folders if they are missing.
DATA_DIR.mkdir(exist_ok=True)
OUTPUT_DIR.mkdir(exist_ok=True)


def clean_whitespace(text: str) -> str:
    # Step 4: Replace repeated spaces and line breaks with a single space.
    """Turn repeated spaces, tabs, and newlines into single spaces."""
    return re.sub(r"\s+", " ", text).strip()


def sentiment_label(compound: float) -> str:
    # Step 5: Convert the numeric VADER compound score into a simple label.
    # These cutoffs are the common beginner-friendly VADER rules.
    """Convert a VADER compound score into a simple label."""
    if compound >= 0.05:
        return "positive"
    if compound <= -0.05:
        return "negative"
    return "neutral"


# Step 6: Add a User-Agent header so the request looks browser-like.
headers = {
    "User-Agent": "Mozilla/5.0 (beginner text analysis tutorial)"
}

# Step 7: Download the page.
response = requests.get(URL, headers=headers, timeout=30)

# Step 8: Stop if the download failed.
response.raise_for_status()

# Step 9: Parse the downloaded HTML.
soup = BeautifulSoup(response.text, "lxml")

# Step 10: Find the main container that holds the press conference content.
section = soup.select_one("main div.section")
if section is None:
    raise RuntimeError("Could not find the article section. The page structure may have changed.")

# Step 11: Remove pieces of the page that we do not want in the text analysis.
for unwanted in section.select('script, style, a[href="#qa"], .ecb-publicationDate'):
    unwanted.decompose()

# Step 12: Collect clean text blocks from headings and paragraphs.
text_blocks = []
for element in section.find_all(["h2", "p"]):
    classes = element.get("class", [])

    # Skip the subtitle line with names and roles.
    if "ecb-pressContentSubtitle" in classes:
        continue

    # Convert the HTML element into plain text.
    text = clean_whitespace(element.get_text(" ", strip=True))
    if text:
        text_blocks.append(text)

# Step 13: Combine all blocks into one document.
full_text = "\n\n".join(text_blocks)

# Step 14: Save the clean full script so we have the source text on disk.
text_path = DATA_DIR / "ecb_press_conference_2026-03-19.txt"
text_path.write_text(full_text, encoding="utf-8")

# Step 15: Download the VADER lexicon.
# This is the word list NLTK uses behind the scenes for sentiment scoring.
nltk.download("vader_lexicon", quiet=True)

# Step 16: Create the sentiment analyzer object.
sentiment_analyzer = SentimentIntensityAnalyzer()

# Step 17: Score the entire press conference as one text.
# The result is a dictionary with neg, neu, pos, and compound values.
scores = sentiment_analyzer.polarity_scores(full_text)

# Step 18: Add a simple label and keep the source URL with the result.
scores["sentiment_label"] = sentiment_label(scores["compound"])
scores["source_url"] = URL

# Step 19: Turn the dictionary into a one-row table.
# We use pandas so it is easy to save and inspect the result.
sentiment_summary = pd.DataFrame([scores])

# Step 20: Save the summary table as a CSV file.
sentiment_path = OUTPUT_DIR / "ecb_sentiment_summary.csv"
sentiment_summary.to_csv(sentiment_path, index=False)

# Step 21: Print a short report for the user.
print("Saved text to:", text_path)
print("Saved sentiment summary to:", sentiment_path)
print()
print("Whole-document sentiment scores:")
print(sentiment_summary[["neg", "neu", "pos", "compound", "sentiment_label"]])
```

Run:

```bash
python scripts/ecb_text_analysis.py
```

Expected result:

```text
Saved text to: data/ecb_press_conference_2026-03-19.txt
Saved sentiment summary to: outputs/ecb_sentiment_summary.csv

Whole-document sentiment scores:
    neg   neu   pos   compound   sentiment_label
    ...   ...   ...   ...        ...
```

Your exact numbers can differ if the page changes or if package versions change.

### How to Read the Result

Open:

```text
outputs/ecb_sentiment_summary.csv
```

There is one row for the whole press conference script. Useful columns:

| Column | Meaning |
|---|---|
| `neg` | Negative score. |
| `neu` | Neutral score. |
| `pos` | Positive score. |
| `compound` | Overall sentiment score. |
| `sentiment_label` | Simple positive, neutral, or negative label. |
| `source_url` | The original ECB page URL. |

### Copilot Prompt for This Step

```text
Add sentiment analysis to my ECB text scraping script. Use NLTK VADER. Score the whole press conference as one text, create neg, neu, pos, and compound values, add a simple positive/neutral/negative label, save the result as outputs/ecb_sentiment_summary.csv, and explain the limitations for central bank text.
```

## 8. Step 4 Code: Make a Word Cloud

A word cloud shows frequent words in larger font.

Before making one, we remove common words such as `the`, `and`, and `of`. We also remove some ECB-specific words that would otherwise dominate the image.

Now replace your script with the final full version:

```python
from collections import Counter
from pathlib import Path
import re

import matplotlib.pyplot as plt
import nltk
import pandas as pd
import requests
from bs4 import BeautifulSoup
from nltk.sentiment import SentimentIntensityAnalyzer
from wordcloud import STOPWORDS, WordCloud


# Step 1: Store the ECB webpage address.
URL = "https://www.ecb.europa.eu/press/press_conference/monetary-policy-statement/2026/html/ecb.is260319~93b1cbad97.en.html"

# Step 2: Define folders for saved text and output files.
DATA_DIR = Path("data")
OUTPUT_DIR = Path("outputs")

# Step 3: Create the folders if they do not exist yet.
DATA_DIR.mkdir(exist_ok=True)
OUTPUT_DIR.mkdir(exist_ok=True)


def clean_whitespace(text: str) -> str:
    # Step 4: Turn messy spacing into clean spacing.
    """Turn repeated spaces, tabs, and newlines into single spaces."""
    return re.sub(r"\s+", " ", text).strip()


def sentiment_label(compound: float) -> str:
    # Step 5: Map the numeric compound score to a simple word label.
    """Convert a VADER compound score into a simple label."""
    if compound >= 0.05:
        return "positive"
    if compound <= -0.05:
        return "negative"
    return "neutral"


def tokenize_words(text: str, stopwords: set[str]) -> list[str]:
    # Step 6: Break the text into lowercase word tokens.
    # We keep alphabetic words and allow apostrophes or hyphens inside them.
    # Then we remove short words and stopwords.
    """Make a simple list of lowercase words, excluding stopwords."""
    tokens = re.findall(r"[A-Za-z][A-Za-z'-]+", text.lower())
    return [
        token
        for token in tokens
        if len(token) > 2 and token not in stopwords
    ]


# Step 7: Add a User-Agent so the request looks browser-like.
headers = {
    "User-Agent": "Mozilla/5.0 (beginner text analysis tutorial)"
}

# Step 8: Download the ECB page.
response = requests.get(URL, headers=headers, timeout=30)

# Step 9: Stop if the request failed.
response.raise_for_status()

# Step 10: Parse the HTML so we can find elements inside it.
soup = BeautifulSoup(response.text, "lxml")

# Step 11: Locate the main article section.
section = soup.select_one("main div.section")
if section is None:
    raise RuntimeError("Could not find the article section. The page structure may have changed.")

# Step 12: Remove elements that are not part of the real script text.
for unwanted in section.select('script, style, a[href="#qa"], .ecb-publicationDate'):
    unwanted.decompose()

# Step 13: Extract clean text from headings and paragraphs.
text_blocks = []
for element in section.find_all(["h2", "p"]):
    classes = element.get("class", [])

    # Skip the subtitle line with speaker names.
    if "ecb-pressContentSubtitle" in classes:
        continue

    # Pull out plain text from the HTML tag and clean the spacing.
    text = clean_whitespace(element.get_text(" ", strip=True))
    if text:
        text_blocks.append(text)

# Step 14: Join everything into one full document.
full_text = "\n\n".join(text_blocks)

# Step 15: Save the extracted text as a plain text file.
text_path = DATA_DIR / "ecb_press_conference_2026-03-19.txt"
text_path.write_text(full_text, encoding="utf-8")

# Step 16: Download the VADER lexicon and build the sentiment analyzer.
nltk.download("vader_lexicon", quiet=True)
sentiment_analyzer = SentimentIntensityAnalyzer()

# Step 17: Score the full press conference as one document.
scores = sentiment_analyzer.polarity_scores(full_text)
scores["sentiment_label"] = sentiment_label(scores["compound"])
scores["source_url"] = URL

# Step 18: Save the sentiment result as a one-row CSV summary.
sentiment_summary = pd.DataFrame([scores])
sentiment_path = OUTPUT_DIR / "ecb_sentiment_summary.csv"
sentiment_summary.to_csv(sentiment_path, index=False)

# Step 19: Start from the default English stopwords used by the wordcloud package.
custom_stopwords = set(STOPWORDS)

# Step 20: Add domain-specific words that would otherwise dominate the figure.
# These words are common in ECB texts, so removing them helps other themes stand out.
custom_stopwords.update(
    {
        "ecb",
        "euro",
        "area",
        "monetary",
        "policy",
        "inflation",
        "per",
        "cent",
        "will",
        "would",
        "could",
        "also",
        "question",
        "questions",
        "answer",
        "answers",
        "think",
        "going",
    }
)

# Step 21: Tokenize the clean text and remove stopwords.
tokens = tokenize_words(full_text, custom_stopwords)

# Step 22: Count how often each remaining word appears.
word_counts = Counter(tokens)

# Step 23: Save the top 30 words as a CSV table.
top_words = pd.DataFrame(
    word_counts.most_common(30),
    columns=["word", "count"],
)
top_words_path = OUTPUT_DIR / "ecb_top_words.csv"
top_words.to_csv(top_words_path, index=False)

# Step 24: Build the word cloud image from the full text.
# The display settings control the size, colors, and reproducibility of the figure.
wordcloud = WordCloud(
    width=1200,
    height=700,
    background_color="white",
    stopwords=custom_stopwords,
    colormap="viridis",
    random_state=42,
).generate(full_text)

# Step 25: Choose the output path for the word cloud image.
wordcloud_path = OUTPUT_DIR / "ecb_wordcloud.png"

# Step 26: Draw the word cloud with matplotlib and save it as a PNG file.
plt.figure(figsize=(12, 7))
plt.imshow(wordcloud, interpolation="bilinear")
plt.axis("off")
plt.tight_layout(pad=0)
plt.savefig(wordcloud_path, dpi=200, bbox_inches="tight")
plt.close()

# Step 27: Print a short summary so we can confirm all output files were created.
print("Saved text to:", text_path)
print("Saved sentiment summary to:", sentiment_path)
print("Saved top words table to:", top_words_path)
print("Saved word cloud to:", wordcloud_path)
print()
print("Whole-document sentiment scores:")
print(sentiment_summary[["neg", "neu", "pos", "compound", "sentiment_label"]])
print()
print("Top 10 words after stopword removal:")
print(top_words.head(10))
```

Run:

```bash
python scripts/ecb_text_analysis.py
```

Expected result:

```text
Saved text to: data/ecb_press_conference_2026-03-19.txt
Saved sentiment summary to: outputs/ecb_sentiment_summary.csv
Saved top words table to: outputs/ecb_top_words.csv
Saved word cloud to: outputs/ecb_wordcloud.png

Whole-document sentiment scores:
...

Top 10 words after stopword removal:
...
```

Open this image in VS Code:

```text
outputs/ecb_wordcloud.png
```

### Copilot Prompt for This Step

```text
Add a word cloud to my ECB text analysis script. Use the wordcloud package and matplotlib. Remove common English stopwords and also remove ECB-specific words like ecb, euro, area, monetary, policy, inflation, per, and cent. Save outputs/ecb_wordcloud.png and outputs/ecb_top_words.csv. Print the top 10 words. Explain how stopwords affect the result.
```

## 9. How to Explain Your Results

A simple write-up could look like this:

```text
I scraped the ECB monetary policy statement and Q&A from the ECB website. I used the browser Inspect tool to identify that the main article text is located inside main div.section. I downloaded the page with requests, parsed it with BeautifulSoup, and saved the extracted text as one clean plain text file.

For sentiment analysis, I used NLTK's VADER sentiment analyzer. I first scored the whole press conference as one document and saved the summary scores in a CSV file. Because central bank language is technical, I interpret these sentiment scores as a rough exploratory measure rather than a final economic indicator.

For the word cloud, I removed common English stopwords and several domain-specific terms that would otherwise dominate the visualization. The resulting word cloud highlights recurring themes in the press conference text.
```
 

## 10. Common Problems and Fixes

### Problem: `conda: command not found`

Conda is not installed or not initialized. Install Miniforge using the setup instructions, then reopen the terminal.

### Problem: `ModuleNotFoundError`

The environment may not be active.

Run:

```bash
conda activate ecb-text
```

Then run the script again.

### Problem: `Could not find the article section`

The page structure may have changed.

Go back to the browser:

1. Open the ECB page.
2. Right-click the target paragraph.
3. Click `Inspect`.
4. Look for the nearest parent container around all article paragraphs.
5. Update this line:

```python
section = soup.select_one("main div.section")
```

### Problem: Word Cloud Has Boring Words

Add more words to `custom_stopwords`.

Example:

```python
custom_stopwords.update({"rate", "rates", "growth"})
```

But be careful: removing too many topic words can hide the meaning of the text.

### Problem: Sentiment Scores Look Strange

This is common for policy text. Words like `risk`, `shock`, `tightening`, or `uncertainty` may influence sentiment scores, but they have technical meanings in monetary policy.

For a more advanced project, compare VADER with a finance-specific sentiment dictionary.

## 11. Extension: Separate the Monetary Policy Statement and the Q&A

In the first tutorial, we treat the whole press conference as one text. In the extension, we split it into two parts:

1. The prepared monetary policy statement.
2. The Q&A transcript.

On this ECB page, there is a jump link:

```html
<a href="#qa" class="arrow">Jump to the transcript of the questions and answers</a>
```

That tells us the page contains a Q&A anchor called `qa`. A beginner-friendly extension is:

1. Find the element with id `qa`.
2. Collect text before that point as the monetary policy statement.
3. Collect text after that point as the Q&A.
4. Run one sentiment score for each part.
5. Create one word cloud for each part.

You do not need this extension in the first version. It is a clean second step after the whole-document workflow works.

### Copilot Prompt for the Extension

```text
I already have a working script that scrapes the full ECB press conference and does one whole-document sentiment analysis plus one word cloud. Now help me extend it so that it separates the monetary policy statement from the Q&A using the qa anchor on the page. I want one sentiment score and one word cloud for the statement, and one sentiment score and one word cloud for the Q&A. Keep the code beginner-friendly and explain the logic step by step.
```

### Possible Extension Output Files

```text
outputs/ecb_statement_sentiment_summary.csv
outputs/ecb_qa_sentiment_summary.csv
outputs/ecb_statement_wordcloud.png
outputs/ecb_qa_wordcloud.png
```

## 12. Create a Repository and Push Your Work to GitHub

This part shows how to put your tutorial files into a GitHub repository and push them online.

If you already created the repository before opening Codespaces, you can still follow the `git add`, `git commit`, and `git push` parts below.

### 12.1 What Git and GitHub Do

| Tool | What it does |
|---|---|
| `git` | Tracks changes to your files on your computer or Codespace. |
| `GitHub` | Stores the repository online so you can back it up, share it, and keep working later. |
| `git add` | Tells git which changed files you want to include in the next save point. |
| `git commit` | Creates a save point with a message. |
| `git push` | Uploads your commits from the Codespace to GitHub. |

### 12.2 Create a New Repository on GitHub

If you do not already have a repository:

1. Go to GitHub in your browser.
2. Click `New repository`.
3. Give it a name such as `ecb-text-analysis`.
4. Choose `Public` or `Private`.
5. Click `Create repository`.

After GitHub creates the repository, you can open it in Codespaces.

### 12.3 Check Which Files You Want to Save

In the VS Code terminal, run:

```bash
git status
```

This shows which files are new or changed.

For this tutorial project, you may want to save files such as:

```text
environment.yml
scripts/ecb_text_analysis.py
ecb_textual_analysis_beginner_tutorial.md
```

You might or might not want to save generated files like:

```text
data/ecb_press_conference_2026-03-19.txt
outputs/ecb_sentiment_summary.csv
outputs/ecb_top_words.csv
outputs/ecb_wordcloud.png
```

For a classroom tutorial repo, it is often fine to keep the script and tutorial, and optionally keep one sample output image.

### 12.4 Add Files to Git

To stage specific files:

```bash
git add environment.yml
git add scripts/ecb_text_analysis.py
git add ecb_textual_analysis_beginner_tutorial.md
```

If you want to add everything in the folder:

```bash
git add .
```

Be careful with `git add .` because it stages all changed files in the current project.

### 12.5 Make a Commit

Create a commit message that explains the work clearly:

```bash
git commit -m "Add ECB textual analysis beginner tutorial"
```

Think of a commit as a named save point in the project history.

### 12.6 Push to GitHub

Upload your commit to GitHub:

```bash
git push
```

If this is the first push for a new branch, Git may ask for:

```bash
git push -u origin main
```

or:

```bash
git push -u origin master
```

The exact branch name depends on your repository.

### 12.7 Verify That the Push Worked

1. Refresh the GitHub repository page in your browser.
2. Check that the files are visible.
3. Click the Markdown file to confirm the tutorial is online.
4. Check the latest commit message.

### 12.8 A Simple Beginner Workflow

Each time you make progress, repeat this cycle:

1. Edit files in VS Code.
2. Run `git status`.
3. Run `git add ...`.
4. Run `git commit -m "your message"`.
5. Run `git push`.

### Copilot Prompt for This Step

```text
Explain to me like I am a beginner how to use git and GitHub in Codespaces. I want to check changes, add my tutorial file and Python script, make a commit, and push to GitHub. Explain what git status, git add, git commit, and git push do.
```

## 13. Final Checklist

Before you finish, make sure you can answer yes to each question:

1. Did I create and activate the `ecb-text` conda environment?
2. Did I install the required packages?
3. Did I inspect the ECB page and identify `main div.section`?
4. Did I download the page with `requests`?
5. Did I parse the page with `BeautifulSoup`?
6. Did I save the extracted text?
7. Did I create one whole-document sentiment score for the full press conference?
8. Did I create a word cloud image?
9. Did I remember that VADER sentiment is exploratory for central bank text?
10. Did I save or push my work to GitHub?

If all are yes, you have completed the tutorial.
