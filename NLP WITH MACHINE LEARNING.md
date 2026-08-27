# NLP (Natural Language Processing) — Complete Notes

> Compiled from: `GEN_AI.md`, handwritten `NLP.pdf` notes, and `finalproject.ipynb` (emotion classification project).

---

## 1. What is NLP?

**NLP (Natural Language Processing)** is a branch of Artificial Intelligence that helps computers **understand, read, write, and talk** like humans.

- It allows machines to process human language (English, Hindi, etc.) — languages that are naturally full of ambiguity — and convert them into a form a computer can work with (ultimately, 0s and 1s).
- Computers only understand numbers, not words, feelings, or emojis — so a huge part of NLP is about **converting text into numbers in a smart way**.

### Real-World Applications
**Chatbots & Virtual Assistants** — e.g. ChatGPT, Alexa, Siri, Google Assistant
- NLP allows these tools to understand what you're saying or typing.
- They recognize your **intent** and generate a relevant response.
- This pipeline typically involves: speech recognition → text understanding → response generation.

Other common applications (filling in gaps from the notes):
- **Sentiment Analysis** — detecting positive/negative/neutral opinion in text (reviews, tweets).
- **Machine Translation** — Google Translate, DeepL.
- **Spam Detection** — email/SMS filtering.
- **Text Summarization** — condensing long documents.
- **Named Entity Recognition (NER)** — pulling out names, places, dates from text.
- **Search Engines** — Google Search uses vectorization techniques like TF-IDF for information retrieval.

---

## 2. Approaches to NLP

### 2.1 Rule-Based Approach (Old School NLP)
- Earliest method — relies on **handwritten rules and grammar logic**.
- Rules are manually written to define grammar, spelling corrections, sentence structure, etc.
- **Limitations:**
  - Can't handle complex, ambiguous, or new language patterns.
  - Not scalable for large datasets — every new case needs a new rule.

### 2.2 Statistical / Machine Learning Approach
- Language is treated **like data**; algorithms learn statistical patterns from it instead of following fixed rules.
- Common models: **Naive Bayes, Logistic Regression, SVM (Support Vector Machines)**.
- Text must first be converted into numbers using techniques like **Bag of Words, TF-IDF**, etc. (see Section 3).
- **Example:** Train a model on 10,000 product reviews → it learns which words indicate positive/negative sentiment → predicts sentiment on new, unseen reviews.
- **Used for:** Sentiment analysis, spam detection, text classification.
- **Limitation:** Loses the *context* of words — it can't detect things like sarcasm or word order/sequence.
  - Example: *"Yeah, because staying up all night debugging is exactly what I dreamed of."*
    - ML sees words like "dreamed" and "exactly" → predicts it sounds like passion/positivity.
    - Reality: it's sarcasm describing a painful dev-life moment. Statistical models miss this nuance.

### 2.3 Deep Learning-Based Approach (Modern NLP)
- Uses **neural networks** to automatically learn complex patterns and context from text — no manual rules or hand-crafted features needed.
- Models can understand **meaning, context, and word order**.
- **Popular architectures:**
  - RNN (Recurrent Neural Networks)
  - LSTM (Long Short-Term Memory)
  - GRU (Gated Recurrent Units)
  - **Transformers** (BERT, GPT, T5, etc.) — current state of the art.
- Text is still converted to numbers, but instead of Bag of Words/TF-IDF, deep learning uses:
  - **Word Embeddings** (Word2Vec, GloVe) or
  - **Token IDs fed through an Embedding Layer**
  - These capture **meaning and context**, not just frequency — e.g., "king" and "queen" end up close together in vector space because they're semantically related, unlike in BoW/TF-IDF where every word is treated as totally independent.

---

## 3. The NLP Pipeline

### Step 1 — Gather the Text (Data Collection)
Options for sourcing text data:
1. **Pre-built public datasets** (e.g., Kaggle, HuggingFace Datasets).
2. **Web Scraping** — when ready-made data isn't available.
3. **APIs** — a structured and legal way to get live data.
4. **Manual Collection / Crowdsourcing**.
5. **Data Augmentation** — using NLP tools to auto-generate more training data from existing data.

### Step 2 — Text Cleaning / Preprocessing
Depending on the task, common cleaning steps include:

| # | Step | Notes |
|---|------|-------|
| 1 | **Lowercasing** | Convert all text to lowercase so "Cat" and "cat" are treated the same. |
| 2 | **Remove Punctuation** | Symbols like `. , ! ? @ # $ % ^ & * ( )` usually don't add meaning for classic ML models. **Exception:** in deep learning models (BERT, GPT, T5), punctuation *can* carry meaning and is often kept. |
| 3 | **Remove Numbers** | Optional — depends on task. E.g. "I bought 5 phones" — you may or may not need the number. |
| 4 | **Remove URLs/Links** | Very common in social media data. |
| 5 | **Remove HTML Tags** | If scraping the web, strip tags like `<div>`, `<p>`, etc. |
| 6 | **Remove Emojis & Special Characters** | Emojis can break vectorization if not handled. |
| 7 | **Remove Stopwords** | Remove common low-information words like "is", "the", "was", "and", "in". Requires a library like **NLTK** or **spaCy** (there's no simple manual list-based way that scales well). |
| 8 | *(Optional)* **Spelling Correction** | Fix typos using libraries like `TextBlob`. |

### Key Terminology
- **Corpus** — the entire collection of text your model reads/learns from. E.g., a folder of 10,000 tweets is your corpus. *("Text is one message. Corpus is the whole collection.")*
- **Document** — a single piece of text inside your dataset (could be one sentence, paragraph, or article). If your corpus is 100 movie reviews, each review = 1 document.
- **Vocabulary** — the list of all **unique** words present in your corpus. Acts like a dictionary the model uses to convert words into numbers.
  - Example: corpus = `["I love pizza", "I love pasta"]` → vocabulary = `["I", "love", "pizza", "pasta"]`.
- **Sentence** — a group of words forming a meaningful statement. Some pipelines split long text into sentences before breaking them into words.
- **Token** — each individual word or symbol in text. Example: `"I love NLP"` → 3 tokens = `["I", "love", "NLP"]`.
- **Tokenization** — the process of splitting text into tokens.

---

## 4. Feature Extraction / Vectorization

> **Why?** Text is not numbers, but ML models only understand numbers — not feelings, not words, not emojis. So we need to convert words into numbers in a smart way. This is called **feature extraction**.

Techniques (roughly in order of increasing sophistication):
```
One-Hot Encoding → Bag of Words (BoW) → TF-IDF → Word2Vec → BERT/Transformers
```

- **Machine Learning-based NLP** mostly uses traditional vectorization: **Bag of Words (BoW)** and **TF-IDF**.
- **Deep Learning-based NLP** relies on **embeddings** (Word2Vec, GloVe, FastText) which convert words into **dense vectors** that capture relationships and context — e.g. "king" and "queen" end up related in vector space, unlike in BoW/TF-IDF/OHE where every word is orthogonal (totally unrelated) to every other word.

### Running Example (used throughout the notes)
Corpus:
- D1: *"Akarsh watch Sheryians"*
- D2: *"Harsh also watch Sheryians"*
- D3: *"Sheryians Teach Akarsh"*

Vocabulary (6 unique words): `[Akarsh, watch, Sheryians, Harsh, also, teach]`

---

### 4.1 One-Hot Encoding (OHE)

Each word in the vocabulary gets its own vector, with a `1` in the position corresponding to that word and `0`s everywhere else. Each **document** becomes a matrix (one one-hot row per word in the sentence).

Vocabulary index: `[Akarsh, watch, Sheryians, Harsh, also, teach]`

D1 = "Akarsh watch Sheryians" → shape (3 × 6):
```
Akarsh    → [1, 0, 0, 0, 0, 0]
watch     → [0, 1, 0, 0, 0, 0]
Sheryians → [0, 0, 1, 0, 0, 0]
```

D2 = "Harsh also watch Sheryians" → shape (4 × 6):
```
Harsh     → [0, 0, 0, 1, 0, 0]
also      → [0, 0, 0, 0, 1, 0]
watch     → [0, 1, 0, 0, 0, 0]
Sheryians → [0, 0, 1, 0, 0, 0]
```

**Pros:**
- Intuitive.
- Easy to implement.

**Cons:**
1. **Sparsity** — too many zeros; wasteful for large vocabularies.
2. **OOV (Out of Vocabulary)** — any new/unseen word has no representation.
3. **Size difference** — different sentences produce differently-shaped matrices (hard to feed into fixed-size ML models directly).
4. **No semantic meaning** — every word is equally "different" from every other word (e.g., in the geometric view, "Time," "Success," and "1-Pod" all sit at 90° / √2 distance apart from each other — no notion of similarity).

---

### 4.2 Bag of Words (BoW)

Instead of one-hot encoding every word position, BoW builds **one vector per document**, where each entry counts **how many times** each vocabulary word appears in that document. Grammar and word order are ignored — only frequency matters (hence "bag" = jumbled, unordered collection of words).

Vocabulary: `[Akarsh, watch, Sheryians, Harsh, also, teach]`

| | Akarsh | watch | Sheryians | Harsh | also | teach |
|---|---|---|---|---|---|---|
| D1: "Akarsh watch Sheryians" | 1 | 1 | 1 | 0 | 0 | 0 |
| D2: "Harsh also watch Sheryians" | 0 | 1 | 1 | 1 | 1 | 0 |
| D3: "Sheryians Teach Sheryians" | 0 | 0 | 2 | 0 | 0 | 1 |

(Note: **MF = 1** in the notes stands for **Minimum Frequency** — a threshold parameter that can filter out very rare words from the vocabulary.)

A new sentence like D4 = *"Sheryians is cool"* can then be plotted geometrically alongside D1/D2 — documents that share more common words end up **closer together** (smaller angle) in vector space, which is the basis of similarity comparisons (e.g. cosine similarity).

**Pros:**
1. Intuitive.
2. Easy to implement.

**Cons:**
1. **Sparsity** — still mostly zeros for large vocabularies.
2. **OOV** — unseen words still can't be represented.
3. **Size Difference** across raw sentence representations (though the final BoW vectors are fixed-length, equal to vocab size).
4. **Loses semantic meaning** — "good" and "great" are just as unrelated as "good" and "car."

---

### 4.3 N-Grams (extension of BoW)

Instead of counting single words (**unigrams**), N-grams count sequences of **N consecutive words**, which helps recover some word-order/context information that plain BoW loses.

- **Unigram** (N=1): single words → `[Akarsh, watch, Sheryians, Harsh, also, teach]`
- **Bigram** (N=2): pairs of consecutive words → `["Akarsh watch", "watch Sheryians", "Harsh also", "also watch", "Sheryians teach", "teach Akarsh"]`
- **Trigram** (N=3): triplets of consecutive words → `["Akarsh watch Sheryians", "Harsh also watch", "watch also Sheryians"]`

**Example — bigram vectorization** for D1/D2/D3 against the bigram vocabulary `[Akarsh watch, watch Sheryians, Harsh also, also watch, Sheryians teach, teach Akarsh]`:

| | Akarsh watch | watch Sheryians | Harsh also | also watch | Sheryians teach | teach Akarsh |
|---|---|---|---|---|---|---|
| D1 | 1 | 1 | 0 | 0 | 0 | 0 |
| D2 | 0 | 1 | 1 | 0 | 0 | 0 |
| D3 | 0 | 0 | 0 | 0 | 1 | 1 |

**Second example (from the notes) — measuring similarity with bigrams:**
- D1: "Cricket is very good"
- D2: "Cricket is not good"

Unigram vocab: `[Cricket, is, very, good, not]`
```
D1 → [1, 1, 1, 1, 0]
D2 → [1, 1, 0, 1, 1]
```
Out of 5 dimensions: **3 are the same, 2 are different** → sentences look fairly similar at the unigram level, even though they have *opposite* meaning.

Bigram vocab: `[Cricket is, is very, very good, is not, not good]`
```
D1 → [1, 1, 1, 0, 0]
D2 → [1, 0, 0, 1, 1]
```
Out of 5 dimensions: only **1 is the same, 4 are different** → bigrams correctly capture that these two sentences are much more dissimilar than unigrams suggested. This demonstrates why n-grams add semantic/contextual signal that unigram BoW misses.

**"N-gram = can be called Bag of N-grams."**

**Pros:**
1. Captures a **little bit** of semantic meaning / word order.

**Cons:**
1. **Dimensions increase** rapidly as N grows (a much larger, sparser vocabulary of n-grams).
2. **Out of Vocabulary (OOV)** problem persists.

---

### 4.4 TF-IDF (Term Frequency – Inverse Document Frequency)

TF-IDF builds on OHE/BoW/n-grams by **weighting** each word based on how important it is to a specific document relative to the whole corpus — rather than just counting raw frequency.

**Formulas:**

```
TF (Term Frequency) = (Number of occurrences of a term in a Document) / (Total number of terms in that Document)

IDF (Inverse Document Frequency) = loge( Total number of documents in corpus / Number of documents containing the term )

TF-IDF = TF × IDF
```

**Worked example** using D1 = "Akarsh watch Sheryians", D2 = "Harsh also watch Sheryians", D3 = "Sheryians Teach Akarsh":

For the word **"Akarsh"** in D1:
- TF = 1/3 (appears once, out of 3 terms in D1)
- IDF = loge(3/2) — because "Akarsh" appears in 2 of the 3 documents (D1 and D3) → ≈ 0.405
- TF-IDF = (1/3) × loge(3/2) ≈ **0.135** (notes round intermediate factor to ~0.7 for loge(3/2))

For a word appearing in **every** document (e.g. if a word were in all 3 docs):
- IDF = loge(3/3) = loge(1) = **0**
- → TF-IDF = 0, meaning words common to *all* documents get **zero weight** — they're considered uninformative (similar in spirit to how stopwords are removed).

Intuition: **TF-IDF gives high weight to words that are frequent in one document but rare across the whole corpus** — i.e., words that are actually *distinctive* for that document.

**Pros:**
- **Information Retrieval** — this is exactly how search engines like Google Search rank document relevance to a query.

**Cons:**
1. **Sparsity** — still a large, mostly-zero matrix for big vocabularies.
2. **OOV** — new/unseen words are still not handled.
3. **Dimensions** — vector size still equals vocabulary size, which grows with corpus size.

---

### 4.5 Word2Vec / GloVe / FastText / BERT (Deep Learning Embeddings)


Unlike OHE/BoW/TF-IDF, these techniques represent words as **dense, low-dimensional vectors** (e.g., 100–300 dimensions) learned from large corpora, where:
- Semantically similar words end up **close together** in vector space (e.g., "king" and "queen", "good" and "great").
- Relationships can even be captured algebraically: `king - man + woman ≈ queen`.
- **Word2Vec** — learns embeddings via a shallow neural network using either the **CBOW** (predict a word from its context) or **Skip-gram** (predict context from a word) architecture.
- **GloVe** — learns embeddings from global word co-occurrence statistics across the corpus.
- **FastText** — extends Word2Vec by representing words as bags of character n-grams, which helps handle OOV words (misspellings, rare words) better.
- **BERT/Transformers** — go a step further: instead of a *fixed* embedding per word, they generate **contextual embeddings**, meaning the same word gets a different vector depending on its surrounding sentence (solving polysemy, e.g. "bank" of a river vs. a "bank" that holds money).

These solve the biggest weaknesses of the classic techniques: **sparsity, OOV, and lack of semantic meaning** — at the cost of needing much more data and compute to train (or using pre-trained versions).

---

## 5. Project: Emotion Classification from Text (`finalproject.ipynb`)

This is an end-to-end applied NLP project: classifying text messages into **emotion categories** (e.g. sadness, anger, love, joy, etc.) using classic ML pipelines (Bag of Words / TF-IDF + Naive Bayes / Logistic Regression).

### 5.1 Setup

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
```

### 5.2 Load Data

```python
df = pd.read_csv('train.txt', sep=';', header=None, names=['text', 'emotion'])
df.head()
```

Sample rows:
```
                                            text   emotion
0                       i didnt feel humiliated   sadness
1  i can go from feeling so hopeless to so damned...  sadness
2  im grabbing a minute to post i feel greedy wrong   anger
3  i am ever feeling nostalgic about the fireplac...   love
4                            i am feeling grouchy    anger
```

```python
df.isnull().sum()
# text       0
# emotion    0
# → No missing values, dataset is clean of nulls.
```

### 5.3 Encode Labels

Convert emotion string labels into numeric IDs (needed since ML models require numeric targets):

```python
unique_emotions = df['emotion'].unique()
emotion_numbers = {}
i = 0
for emo in unique_emotions:
    emotion_numbers[emo] = i
    i += 1

df['emotion'] = df['emotion'].map(emotion_numbers)
```

### 5.4 Text Cleaning (applying the Section 3 pipeline in practice)

**Lowercasing:**
```python
df['text'] = df['text'].apply(lambda x: x.lower())
```

**Remove punctuation:**
```python
import string

def remove_punc(txt):
    return txt.translate(str.maketrans('', '', string.punctuation))

df['text'] = df['text'].apply(remove_punc)
```

**Remove numbers:**
```python
def remove_numbers(txt):
    new = ""
    for i in txt:
        if not i.isdigit():
            new = new + i
    return new

df['text'] = df['text'].apply(remove_numbers)
```

**Remove emojis / non-ASCII characters:**
```python
def remove_emojis(txt):
    new = ""
    for i in txt:
        if i.isascii():
            new += i
    return new

df['text'] = df['text'].apply(remove_emojis)
```

**Remove stopwords (using NLTK):**
```python
import nltk
from nltk.corpus import stopwords
from nltk.tokenize import word_tokenize

nltk.download('punkt')
nltk.download('stopwords')

stop_words = set(stopwords.words('english'))

def remove(txt):
    words = txt.split()
    cleaned = [w for w in words if w not in stop_words]
    return ' '.join(cleaned)

df['text'] = df['text'].apply(remove)
```

**Before/after example** (row index 1):
```
Before: "i can go from feeling so hopeless to so damned hopeful just from being around someone who cares and is awake"
After:  "go feeling hopeless damned hopeful around someone cares awake"
```
→ Stopwords like "i", "can", "from", "so", "to", "just", "being", "who", "and", "is" are removed, leaving only the content-bearing words.

### 5.5 Train/Test Split

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(
    df['text'], df['emotion'], test_size=0.20, random_state=42
)
```
- 80% training data, 20% test data, with a fixed `random_state` for reproducibility.

### 5.6 Model 1 — Bag of Words + Multinomial Naive Bayes

```python
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.metrics import accuracy_score

bow_vectorizer = CountVectorizer()
X_train_bow = bow_vectorizer.fit_transform(X_train)
X_test_bow = bow_vectorizer.transform(X_test)

nb_model = MultinomialNB()
nb_model.fit(X_train_bow, y_train)

pred_bow = nb_model.predict(X_test_bow)
print(accuracy_score(y_test, pred_bow))
```
**Result: Accuracy ≈ 0.768 (76.8%)**

> Note: `fit_transform` is called on `X_train` (learns vocabulary + transforms), while only `.transform()` (no fit) is called on `X_test` — this is correct practice, since the test set must be encoded using the vocabulary learned from training data only, to avoid data leakage.

### 5.7 Model 2 — TF-IDF + Multinomial Naive Bayes

```python
tfidf_vectorizer = TfidfVectorizer()
X_train_tfidf = tfidf_vectorizer.fit_transform(X_train)
X_test_tfidf = tfidf_vectorizer.transform(X_test)

nb2_model = MultinomialNB()
nb2_model.fit(X_train_tfidf, y_train)

y_pred = nb2_model.predict(X_test_tfidf)
print(accuracy_score(y_test, y_pred))
```
**Result: Accuracy ≈ 0.661 (66.1%)**

> Interesting result: TF-IDF performed *worse* than plain BoW with Naive Bayes here. This can happen because Naive Bayes (Multinomial variant) is built around raw/discrete count statistics, so TF-IDF's continuous re-weighting doesn't always play to its strengths — whereas it often helps more with models like Logistic Regression or SVM.

### 5.8 Model 3 — TF-IDF + Logistic Regression

```python
from sklearn.linear_model import LogisticRegression

logistic_model = LogisticRegression(max_iter=1000)
logistic_model.fit(X_train_tfidf, y_train)

log_pred = logistic_model.predict(X_test_tfidf)
print(accuracy_score(y_test, log_pred))
```
**Result: Accuracy ≈ 0.863 (86.3%)** — the best-performing model of the three.

### 5.9 Results Summary

| Vectorizer | Model | Accuracy |
|---|---|---|
| Bag of Words | Multinomial Naive Bayes | 76.8% |
| TF-IDF | Multinomial Naive Bayes | 66.1% |
| TF-IDF | Logistic Regression | **86.3%** ✅ Best |

**Takeaway:** For this emotion-classification task, **TF-IDF + Logistic Regression** clearly outperforms the other combinations — showing that vectorizer and model choice need to be paired thoughtfully (TF-IDF works best here paired with a linear model rather than Naive Bayes).

### 5.10 Suggested Next Steps (gaps to fill in — not present in the notebook)
To take this project further, consider adding:
1. **Confusion matrix / classification report** (`sklearn.metrics.classification_report`, `confusion_matrix`) to see per-emotion precision/recall, not just overall accuracy.
2. **Class balance check** — plot `df['emotion'].value_counts()` to check if any emotion class is underrepresented (accuracy alone can be misleading on imbalanced data).
3. **Try n-grams** in `CountVectorizer(ngram_range=(1,2))` / `TfidfVectorizer(ngram_range=(1,2))` to capture bigram context, per Section 4.3.
4. **Try more models**: SVM, Random Forest, or a simple neural network (Embedding layer + LSTM) to compare against the classical baselines.
5. **Hyperparameter tuning** — e.g., `max_features`, `min_df`/`max_df` in the vectorizers, or `C` in Logistic Regression.
6. **Save the trained model & vectorizer** (`pickle`/`joblib`) for reuse/deployment.
7. **Error analysis** — manually inspect a few misclassified examples to understand model weaknesses (e.g., sarcasm, mixed emotions).

---

## 6. Quick-Reference Summary Table

| Technique | Captures Semantic Meaning? | Handles OOV? | Fixed Size Output? | Sparse? | Typical Use |
|---|---|---|---|---|---|
| One-Hot Encoding | ❌ No | ❌ No | ❌ No (varies with sentence length) | ✅ Very | Basic/legacy, teaching concept |
| Bag of Words | ❌ No | ❌ No | ✅ Yes (= vocab size) | ✅ Very | Simple text classification |
| N-Grams | ⚠️ A little | ❌ No | ✅ Yes (larger vocab) | ✅ Very | Adds some word-order context |
| TF-IDF | ❌ No (but weights importance) | ❌ No | ✅ Yes | ✅ Very | Search/IR, classic ML classification |
| Word2Vec / GloVe / FastText | ✅ Yes | ⚠️ Partial (FastText best) | ✅ Yes (dense, small) | ❌ No | Deep learning inputs |
| BERT / Transformers | ✅ Yes (contextual) | ✅ Yes (subword tokenization) | ✅ Yes (dense, contextual) | ❌ No | Modern state-of-the-art NLP |

---

## 7. Full Glossary

- **NLP** — Natural Language Processing.
- **Corpus** — the whole collection of text/documents used.
- **Document** — one unit of text within the corpus.
- **Vocabulary** — set of all unique words across the corpus.
- **Token / Tokenization** — a single unit (word/symbol) of text / the act of splitting text into tokens.
- **Stopwords** — common, low-information words filtered out during cleaning (e.g. "is", "the", "and").
- **OHE** — One-Hot Encoding.
- **BoW** — Bag of Words.
- **N-gram** — a contiguous sequence of N words (unigram=1, bigram=2, trigram=3...).
- **TF** — Term Frequency.
- **IDF** — Inverse Document Frequency.
- **OOV** — Out Of Vocabulary (a word not seen during training/vocabulary building).
- **Embedding** — a dense numeric vector representation of a word/token that captures meaning.
- **RNN / LSTM / GRU** — sequential neural network architectures for processing text.
- **Transformer** — modern deep learning architecture (attention-based) underlying BERT, GPT, T5.
