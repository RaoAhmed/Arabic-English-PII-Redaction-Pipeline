# Bilingual Arabic–English PII Redaction Pipeline (Microsoft Presidio & Stanza)
<b>NOTE: This Document Contains the Details of What Worked, What Didn't, Why It Worked, Why It Didn't, Things I Found, and what solution was adopted</b>

This repository contains an end-to-end PII (Personally Identifiable Information) detection and anonymization pipeline designed for bilingual (Arabic and English) legal, formal, and informal text. It leverages **Microsoft Presidio**, **Stanza (Stanford NLP)**, and **spaCy** to detect names, locations, financial identifiers, government IDs, phone numbers, dates, and times.

---

## 1. System Architecture

The pipeline uses a hybrid architecture that decouples heavy morphological and syntactic entity extraction from token stream artifact management:

<img width="2684" height="1084" alt="image" src="https://github.com/user-attachments/assets/81f55a3c-54fe-484c-af68-cd88762f384a" />

---

## 2. What Didn't Work (Engineering Pitfalls & Root Cause Analysis)

### A. The `spacy-stanza` Multi-Word Token (MWT) Alignment Crash

* **What was attempted:** Using `StanzaNlpEngine` or wrapping Stanza inside spaCy using `spacy-stanza` (`spacy_stanza.load_pipeline("ar")`).
* **The Failure:** Arabic text contains proclitics and enclitics (e.g., prepositions like `بـ` and possessive pronouns like `ـها` attached directly to words such as `بالمحفظة` and `رقمها`). Stanza’s morphological analyzer splits these into syntactic tokens via its Multi-Word Tokenizer (MWT). When `spacy-stanza` tried to map these expanded tokens into a spaCy `Doc`, character offsets desynchronized.
* **The Consequence:** spaCy raised `UserWarning: Can't set named entities because of multi-word token expansion...` and discarded all extracted entities, setting `doc.ents = []`. Presidio received an empty list and failed to detect any names or locations.

### B. Stripping MWT from Stanza Processors (`"processors": "tokenize,ner"`)

* **What was attempted:** Disabling MWT by passing `processors="tokenize,ner"` to prevent the alignment mismatch.
* **The Failure:** Stanza's Arabic NER model (`aqmar_charlm`) relies on morphological features derived from `mwt`, `pos`, and `lemma`. Stripping out MWT altered the token distribution, causing the NER model to detect zero entities (`[]`).
* **Configuration Overhead:** Presidio’s `StanzaNlpEngine` only passes the model name to `spacy_stanza.load_pipeline`, ignoring custom dictionary keys like `"stanza_processors"`.

### C. Standard RegEx / Built-in Presidio Recognizers on Eastern Arabic Numerals

* **What was attempted:** Relying on default Presidio recognizers for credit cards, phone numbers, and IDs.
* **The Failure:** Presidio's built-in recognizers assume ASCII digits (`0-9`). The Arabic input used Eastern Arabic numerals (`٠-٩`), rendering all standard regex rules inactive.

### D. Missing Date and Time Annotations in Arabic Models

* **What was attempted:** Relying on Stanza’s Arabic model to identify dates and times.
* **The Failure:** Stanza’s `aqmar_charlm` model is trained only on `PER` (Person), `LOC` (Location), `ORG` (Organization), and `MISC`. It does not support `DATE` or `TIME`. Presidio’s default date recognizer only recognizes Western date conventions and English month strings.

### E. Presidio Default Engine `KeyError: 'ar'`

* **What was attempted:** Initializing `AnalyzerEngine(supported_languages=["ar", "en"])` without passing an explicit `nlp_engine`.


* **The Failure:** Presidio fell back to its default English-only `SpacyNlpEngine`, crashing with `KeyError: 'ar'` when attempting to process Arabic text.



### F. Indentation Scope Bug in Custom Recognizers

* **What was attempted:** Declaring `def load(self)` and `def analyze(self, ...)` indented inside `def __init__(...)`.
* **The Failure:** Python scoped `analyze` as a local nested function rather than an instance method override. Presidio executed the inherited base method `EntityRecognizer.analyze()`, which returns `[]`.

---

## 3. What Worked & The Adopted Solution

### 1. `NativeArabicStanzaRecognizer` (Bypassing `spacy-stanza`)

I implemented a custom `EntityRecognizer` that wraps `stanza.Pipeline("ar")` directly.

* Retains the full morphological pipeline: `tokenize,mwt,pos,lemma,ner`.


* Extracts entities using Stanza’s native character offsets (`ent.start_char`, `ent.end_char`).


* Maps Arabic tags (`PER`, `LOC`, `ORG`) directly into Presidio’s standard schema (`PERSON`, `LOCATION`, `ORGANIZATION`).



### 2. High-Performance Hybrid NLP Engine (`spacy.blank("ar")`)

Instead of loading Stanza twice (once via `StanzaNlpEngine` and once via the custom recognizer), I configured:

* `ar`: `spacy.blank("ar")` — a lightweight tokenizer providing the required token stream for regex pattern evaluation with near-zero latency and minimal memory overhead.


* `en`: `spacy.load("en_core_web_sm")` — provides full syntactic and NER capabilities for English text.



### 3. Comprehensive Regex Pattern Suite with Eastern Numeral Support

Custom `PatternRecognizer` instances were registered to handle:

* **KSA National ID / Iqama:** 10 digits starting with `1` or `2` (supporting `[12١٢][0-9٠-٩]{9}`).


* **Saudi Mobile Numbers:** Capturing country codes (`+966`, `٠٠٩٦٦`), local prefixes (`05`, `٠٥`), and standard 8-digit extensions.


* **IBAN:** Validating standard prefixes (`SA`, `PL`) followed by account numbers.


* **Credit Cards:** 12–19 digit groupings formatted with spaces or hyphens across both Western and Eastern numerals.


* **Driver's Licenses & SSNs:** Pattern matching for 9-digit formats and `XXX-XX-XXXX` structures.


* **Bilingual Dates & Times:** Regex patterns supporting both Western and Hijri/Gregorian Arabic months (e.g., `أكتوبر`, `رمضان`) and time markers (`صباحًا`, `مساءً`).



### 4. Custom Anonymizer Operators

Configured `OperatorConfig` rules to produce standardized, human-readable redaction tags rather than raw default placeholders:

* `PERSON` $\rightarrow$ `[REDACTED_NAME]`

* `LOCATION` $\rightarrow$ `[REDACTED_LOCATION]`

* `DATE_TIME` $\rightarrow$ `[REDACTED_DATE_TIME]`

* `CREDIT_CARD` $\rightarrow$ `[REDACTED_CREDIT_CARD]`

* `IBAN_CODE` $\rightarrow$ `[REDACTED_IBAN]`

* `PHONE_NUMBER` $\rightarrow$ `[REDACTED_PHONE]`

* `EMAIL_ADDRESS` $\rightarrow$ `[REDACTED_EMAIL]`

* `DEFAULT` $\rightarrow$ `[REDACTED]`


---

## 4. Environment Setup & Installation

Dependencies can be installed using `uv` or standard `pip`:

```bash
# Using uv (recommended for fast dependency resolution)
uv pip install "presidio-analyzer[stanza]" presidio-anonymizer spacy

# Download English core model
python -m spacy download en_core_web_sm

```

> **Note on Stanza Models:** Stanza automatically downloads the Arabic model package (`padt`, `aqmar_charlm`) to `~/.cache/stanza/` on first execution. For offline environments, pre-download via `stanza.download("ar")` and set `download_method=None` in `stanza.Pipeline`.

---

## 5. Usage Example

```python
import spacy
from presidio_analyzer import AnalyzerEngine
from presidio_analyzer.nlp_engine import SpacyNlpEngine
from presidio_anonymizer import AnonymizerEngine

# 1. Initialize Hybrid NLP Engine
nlp_engine = SpacyNlpEngine()
nlp_engine.nlp = {
    "en": spacy.load("en_core_web_sm"),
    "ar": spacy.blank("ar"),
}

# 2. Build Analyzer
analyzer = AnalyzerEngine(
    nlp_engine=nlp_engine,
    supported_languages=["ar", "en"]
)

# 3. Register Custom Recognizers
analyzer.registry.add_recognizer(NativeArabicStanzaRecognizer())
analyzer.registry.add_recognizer(arabic_date_recognizer)
analyzer.registry.add_recognizer(arabic_time_recognizer)
analyzer.registry.add_recognizer(cc_ar)
analyzer.registry.add_recognizer(license_ar)
analyzer.registry.add_recognizer(ssn_ar)
analyzer.registry.add_recognizer(iban_ar)
analyzer.registry.add_recognizer(phone_ar)
analyzer.registry.add_recognizer(nat_id_ar)

# 4. Analyze & Anonymize
text = "اسمي جون دو، وسُرقت محفظتي في كيلمارنوك بتاريخ ١٩ أكتوبر ٢٠٢١."
results = analyzer.analyze(text=text, language="ar")

anonymizer = AnonymizerEngine()
anonymized = anonymizer.anonymize(
    text=text,
    analyzer_results=results,
    operators=operators
)

print(anonymized.text)
# Output: اسمي [REDACTED_NAME]، وسُرقت محفظتي في [REDACTED_LOCATION] بتاريخ [REDACTED_DATE_TIME].

```

---

## 6. Verification & Test Results

### Arabic Text Anonymization Output

```text
التاريخ:[REDACTED_DATE_TIME]
الشاهد: [REDACTED_NAME]
الموضوع: شهادة بشأن فقدان محفظة

محتوى الشهادة:

حضرة الضابط،

اسمي [REDACTED_NAME]، وفي[REDACTED_DATE_TIME]، سُرقت محفظتي بالقرب من [REDACTED_LOCATION] أثناء رحلة بالدراجة.
تحتوي المحفظة على بطاقة ائتماني رقمها[REDACTED_CREDIT_CARD]، وهي مسجلة باسمي ومرتبطة بحسابي البنكي رقم [REDACTED_IBAN].
إضافةً إلى ذلك، كانت المحفظة تحتوي على رخصة قيادة رقم[REDACTED_LICENSE] صادرة باسمي. كما تحتوي على رقم الضمان الاجتماعي الخاص بي،[REDACTED_SSN].
... علاوة على ذلك، كانت بطاقة [REDACTED] موجودة هناك، ورقمها [REDACTED].

أعتقد أنها سُرقت في تمام الساعة[REDACTED_DATE_TIME].
يُرجى التواصل معي على رقم هاتفي [REDACTED_PHONE]، أو عبر بريدي الإلكتروني الشخصي [REDACTED_EMAIL].
ممثلتي هناك هي [REDACTED_NAME] (رقم هاتفها الخاص بالعمل: [REDACTED_PHONE]).

```

(Verified: Caught Arabic names `جون دو` and `فيكتوريا تشيري`, location `كيلمارنوك`, formatted Eastern digits, dates, emails, and credentials).

### English Text Anonymization Output

```text
Date: [REDACTED_DATE_TIME]Witness: [REDACTED_NAME]: Testimony Regarding the Loss of Wallet

Hello Officer,
My name is [REDACTED_NAME] and on [REDACTED_DATE_TIME], my wallet was stolen in the vicinity of [REDACTED] during a bike trip.
Firstly, the wallet contains my credit card with number [REDACTED_CREDIT_CARD], which is registered under my name and linked to my bank account, [REDACTED_IBAN].
Additionally, the wallet had a driver's license - DL No: [REDACTED_DATE_TIME] issued to my name. It also houses my [REDACTED], [REDACTED_SSN].
I believe It was stolen at [REDACTED_DATE_TIME].
Reach out to me on my phone number, [REDACTED_PHONE], or through my personal email, [REDACTED_EMAIL].
My representative there is [REDACTED_NAME] (her business phone: [REDACTED]).

```

(Verified: Redacted English names, emails, card numbers, IBAN, and dates).
