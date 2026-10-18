# Bilingual Arabic–English PII Redaction Pipeline

An end-to-end PII detection and anonymization solution for bilingual Arabic and English text using **Microsoft Presidio**, **Stanza**, and **spaCy**.

---

### The Problems

1. **`spacy-stanza` Alignment Crash:** Using `spacy-stanza` to bridge Stanza into spaCy drops all Arabic entities (`doc.ents = []`). Arabic Multi-Word Token (MWT) segmentation separates attached clitics (e.g., `بـ`, `ـها`), throwing off character alignment and causing spaCy to discard all detected names and locations.


2. **Eastern Arabic Numerals (`٠-٩`):** Presidio's built-in numeric recognizers only inspect Western ASCII digits (`0-9`), completely ignoring Arabic numerals (`٠-٩`) across credit cards, national IDs, and phone numbers.


3. **No Date/Time NER in Arabic Stanza:** Stanza's Arabic model (`aqmar_charlm`) is strictly trained on `PER`, `LOC`, `ORG`, and `MISC`. It lacks any capability to detect dates or times.


4. **Presidio `KeyError: 'ar'`:** `AnalyzerEngine` defaults to an English-only spaCy instance, throwing a `KeyError: 'ar'` when invoked on Arabic text without a registered language pipeline.



---

### The Adopted Solution

* **`NativeArabicStanzaRecognizer`:** A custom `EntityRecognizer` that invokes `stanza.Pipeline("ar")` directly with its full morphological stack (`tokenize,mwt,pos,lemma,ner`). It extracts `ent.start_char` and `ent.end_char` directly from native Stanza output, eliminating token-drift and wrapper crashes.


* **Hybrid Engine with `spacy.blank("ar")`:** `AnalyzerEngine` is backed by `spacy.load("en_core_web_sm")` for English and `spacy.blank("ar")` for Arabic. This provides lightweight tokenization to satisfy Presidio's internal pipeline without duplicating heavy neural processing.


* **Numeral-Agnostic Regex Patterns:** Custom `PatternRecognizer` instances handle both Eastern (`٠-٩`) and Western (`0-9`) digits for KSA National IDs, Saudi phone numbers, credit cards, IBANs, driver's licenses, and SSNs.


* **Dedicated Arabic Date & Time Regex:** Regex patterns cover Hijri and Gregorian Arabic months (e.g., `أكتوبر`, `رمضان`) alongside standard 12-hour/24-hour time notations (`صباحًا`, `مساءً`).


* **Readable Anonymization Tags:** Custom `OperatorConfig` mappings replace detected entities with clear redactions (such as `[REDACTED_NAME]`, `[REDACTED_IBAN]`, and `[REDACTED_DATE_TIME]`) instead of generic Presidio tags.



---

### Quick Implementation

```python
import spacy
import stanza
from presidio_analyzer import AnalyzerEngine, RecognizerResult, EntityRecognizer
from presidio_analyzer.nlp_engine import SpacyNlpEngine
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

# 1. Native Stanza Recognizer (Direct extraction without spacy-stanza)
class NativeArabicStanzaRecognizer(EntityRecognizer):
    def __init__(self, stanza_pipeline=None):
        super().__init__(
            supported_entities=["PERSON", "LOCATION", "ORGANIZATION"],
            supported_language="ar",
            name="native_arabic_stanza_recognizer"
        )
        self.pipeline = stanza_pipeline or stanza.Pipeline(
            lang="ar", processors="tokenize,mwt,pos,lemma,ner", verbose=False
        )
        self.label_map = {"PER": "PERSON", "LOC": "LOCATION", "ORG": "ORGANIZATION"}

    def load(self):
        pass

    def analyze(self, text, entities, nlp_artifacts=None):
        doc = self.pipeline(text)
        return [
            RecognizerResult(
                entity_type=self.label_map[ent.type],
                start=ent.start_char,
                end=ent.end_char,
                score=0.85
            )
            for ent in doc.ents if ent.type in self.label_map and (not entities or self.label_map[ent.type] in entities)
        ]

# 2. Hybrid NLP Engine (en_core_web_sm + blank ar)
nlp_engine = SpacyNlpEngine()
nlp_engine.nlp = {
    "en": spacy.load("en_core_web_sm"),
    "ar": spacy.blank("ar"),
}

analyzer = AnalyzerEngine(nlp_engine=nlp_engine, supported_languages=["ar", "en"])
analyzer.registry.add_recognizer(NativeArabicStanzaRecognizer())

# Register custom patterns for numeric & datetime PII (cc_ar, nat_id_ar, phone_ar, etc.)
# ...

# 3. Anonymize with Custom Operator Tags
operators = {
    "PERSON": OperatorConfig("replace", {"new_value": "[REDACTED_NAME]"}),
    "LOCATION": OperatorConfig("replace", {"new_value": "[REDACTED_LOCATION]"}),
    "DEFAULT": OperatorConfig("replace", {"new_value": "[REDACTED]"}),
}

anonymizer = AnonymizerEngine()
results = analyzer.analyze(text=arabic_text, language="ar")
sanitized = anonymizer.anonymize(text=arabic_text, analyzer_results=results, operators=operators)

print(sanitized.text)

```
