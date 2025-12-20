# Yahoo Japan User Sentiment Analysis Pipeline

A pipeline that scrapes Yahoo Japan news articles and user comments, performs Japanese-language sentiment analysis on user comments and JP -> EN translation and summarization of articles, and visualizes sentiment distributions across news categories through a Streamlit dashboard. 

Over 400+ articles and 200,000+ user comments processed and analyzed using Transformer Models from Hugging Face and Ollama.

Data source: https://news.yahoo.co.jp/


### What the Pipeline Does

- Automatically scrapes Yahoo Japan news articles and user comments every 4 hours
- Classifies sentiment on Japanese user comments (positive/negative/neutral) using a Japanese-pretrained BERT classifier
- Translates articles JP to EN and generates summaries
- Categorizes articles into 10 topics using local LLM (Ollama)
- Aggregates sentiment statistics and engagement metrics by category
- Visualizes sentiment distributions and trends through dashboard

---

## Dashboard

<img src="assets/dashboard_1.png" width="900">
<img src="assets/dashboard_2.png" width="900">
<img src="assets/dashboard_3.png" width="900">

---

## Yahoo Japan Site
- Article
<img src="assets/yahoojp_1.png" width="400">

- Comments
<img src="assets/yahoojp_2.png" width="400">

---

## Architecture

<img src="assets/architecture_diagram.png" alt="architecture diagram" width="800">



---

## Technical Implementation

### Data Pipeline Flow


#### 1. SCRAPING 
     Yahoo JP Trending → Selenium → BeautifulSoup → MongoDB
        

#### 2. PROCESSING 
    Step 1: MongoDB → Comments → Sentiment Analysis (BERT) → MongoDB
    Step 2: MongoDB → Articles → Translation (NLLB) → Summarization (BART) → Categorization (LLM) → MongoDB
                                      
    (Article-level sentiment is derived via aggregation of comment sentiment)

#### 3. VISUALIZATION 
    MongoDB → Streamlit Dashboard → Analytics

<br>


### NLP Tools

 - Models and Tokenizer from Hugging Face
 - Ollama

#### 1. Comment Sentiment Analysis
**Model:** `christian-phu/bert-finetuned-japanese-sentiment`
- Performed on **raw Japanese text** to preserve linguistic nuances
- Returns: Positive/Negative/Neutral with confidence scores


#### 2. Article Translation
**Model:** `facebook/nllb-200-distilled-600M` (configurable)
- Japanese → English translation
- Handles long articles via chunking (~800 characters per chunk)
- Preserves context across chunks


#### 3. Article Summarization
**Model:** `facebook/bart-large-cnn`
- Summarizes translated English content
- Skips summarization for short articles (<50 words)


#### 4. Article Categorization
**Model:** `qwen2.5:7b-instruct` (via Ollama)
- LLM-based classification on **Japanese text**
- 10 predefined categories (Politics, Economy, Crime, International Relations, etc.)

```python
# From categorize_articles.py
CATEGORIES = [
    "政治・行政",        # Politics & Government
    "経済・ビジネス",     # Economy & Business
    "犯罪・司法",        # Crime & Justice
    "国際関係",          # International Relations
    "社会問題",          # Social Issues
    "テクノロジー・科学",  # Technology & Science
    "健康・環境",        # Health & Environment
    "教育",             # Education
    "エンタメ・文化",     # Entertainment & Culture
    "スポーツ"           # Sports
]
```



---

### Design Decisions

- **Separate collections** for articles and comments enable efficient querying
- **`processed` flag** ensures idempotent pipeline execution (can safely re-run)
- **Embedded sentiment summary** in articles improves dashboard performance
- **Indexed on `article_id`** for fast comment lookups 
- **Append-only ingestion** maintains data integrity



**Index Rationale:**
- **Unique indexes** prevent duplicate entries during scraping
- **Partial indexes** on `processed: false` reduce index size proportional to the number of matching documents, often dramatically smaller than a full index after most documents are processed.
- **Compound index** on comments ensures no duplicate comments from the same user at the same time
- **article_id lookup** is implicit in the compound index for fast comment retrieval

**Idempotent Design**
- Only processes documents where `processed: false`
- Safe to re-run without duplication
- Handles interruptions gracefully
- Checkpoint-based resumption

---

## Streamlit Dashboard

Analytics interface built with Streamlit and Plotly:

### Features

**Category-Level Analysis**
- Average positive/negative sentiment per news category
- Identifies most/least positively received topics
- Grouped bar charts for comparison
- Sortable table showing articles per category

**Article-Level Insights**
- Paginated article list (10 per page)
- Search by title/summary keywords
- Filter by dominant sentiment (Positive/Neutral/Negative)
- Sort by comment volume or sentiment polarity

**Comment Deep-Dive**
- Expandable view for each article
- Display up to 20 comments per article
- Show engagement metrics (likes, insightful votes, disagrees)
- Color-coded borders by sentiment
- Confidence scores displayed

**Visualizations**
- Pie chart: Overall sentiment distribution across all comments
- Stacked bar: Top 4 articles by comment volume with sentiment breakdown
- Category comparison: Positive vs negative trends
- Real-time statistics (processed vs unprocessed articles/comments)

---

## Challenges & Solutions

- Separating ingestion from processing improved resource utilization and fault isolation
- Separating ingestion from processing also improved overall processing time. Since the scraper must throttle requests to avoid bot detection (30–60 minutes per article), decoupling allows parsing and analysis of already-collected data in under 20 minutes for batch processing.
- `processed` flags made the pipeline safe to re-run after failures
- Local LLM inference (Ollama) reduced costs and provided acceptable latency
- Japanese-specific BERT model outperformed general multilingual models
- Performing sentiment analysis before translation preserved linguistic nuances
- Confidence scores helped identify unreliable predictions
- Batch processing balanced speed and memory usage
- Per-comment error handling improved reliability

