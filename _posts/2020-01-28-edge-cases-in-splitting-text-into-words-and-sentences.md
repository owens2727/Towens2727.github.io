---
layout: post
title: Edge Cases in Splitting Text into Words and Sentences
description: An overview of the challenges and corresponding strategies to splitting text into words and sentences.
---

This post provides an introduction to two common NLP tasks: splitting text into sentences and words. Both are often prerequisites for more advanced analysis, but they also introduce a great deal of hidden complexity. Specifically, we'll focus on the edge cases that make these problems challenging when parsing the English language.

Splitting text into sentences
-----------------------------

This process is formally referred to as Sentence Boundary Disambiguation — put simply, we want to know what separates one sentence from the next.

### A Naive Approach

In it's most naive form, splitting text into sentences is as simple as:

```python
text = "This is a sentence. Here is another one. And a third."
text.split(". ")

# OUT: ['This is a sentence', 'Here is another one', 'And a third.']
```

We scan for boundaries between sentences, denoted most often by a period followed by a space.

### Edge cases

This can go wrong in two ways: failing to detect a sentence boundary, or detecting a sentence boundary when in fact there is none. Our naive sentence splitter will quickly encounter both errors when faced with even a small amount of text to split.

Lets consider a few situations which the naive splitter would fail at:

1. Sentences can end in punctuation besides periods: question marks, exclamation points, quotations, ellipses, and so on...<br /><br />`He removed the director! Of the FBI? You're kidding...`

2. Abbreviations end in periods but are not sentence boundaries.<br /><br />`Mr. Comey was removed from his post as F.B.I. Director in May of 2017`

3. But sentences do sometimes end in abbreviations.<br /><br />`Mr. Comey used to the Director of the F.B.I. Now he is a private citizen.`

4. Initials end in periods but are not sentence boundaries.<br /><br />`James B. Comey was born on December 14, 1960.`

5. But sentences do sometimes end with a single letter that looks an awful lot like an intitial.<br /><br />`In college, James Comey never once earned a B. He got straight A's.`

6. Text can contain grammatical errors.<br /><br />```This text is missing a space.A human reader would still know it contains two sentences.```

### Advanced Strategies

There are two paths we can go down to improve upon our naive splitter and handle these edge cases:

1. Rules-based
2. Statistical (machine learning)

Rules-based approaches attempt to directly encode our knowledge of sentence boundaries. For sentence splitting, we might construct a regular expression with logic that respects various forms of punctuation, abbreviations, and so on.

This gets nasty quickly. The exceptions to sentence splitting rules are varied and require lots of contextual knowledge. Naive rules-based sentence splitters achieve around 95% correctness on most corpuses.

Statistical approaches can improve upon this 95% accuracy benchmark. We use a corpus to train a machine learning model to detect sentence boundaries. The quality of the splitter will depend on the quality of the training corpus, as well as the similarity between the training corpus and text the splitter will encounter in the wild.

The robustness of the model is entirely dependent on the quality of the training data. The training data may contain edge cases that we otherwise might never have considered. But it also could miss out on other classes of errors that are still quite obvious to a human reader.

### Error Rates

Depending on the use case, the 95% accuracy achieved by rules-based sentence splitters might be *good enough*. In academic and research settings, we're often looking for general trends. 95% correctness can help answer questions like *how long is the average sentence in the works of Shakespeare as compared to Stephen King?*

But in many consumer and enterprise applications, the acceptable error rate is near zero. Be mindful of the error tolerance for your use case and the specific questions you are trying to answer using the parsed text. In situtations where you cannot ensure accuracy, convey the level of certainty to the end user.

Splitting text into words
-------------------------

We'll turn now to word detection — specifically, the process of splitting a sentence into words.

### A Naive Approach

As with sentence splitting, the naive approach to this problem is straightforward and may appear satisfactory at first: we split the text on spaces.

```python
text = "This is a sentence"
text.split(" ")

# OUT: ['This', 'is', 'a', 'sentence']
```

The primary challenge for word detection comes down to punctuation. In some cases, like commas, the punctuation that follows a word should not be considered a part of the word. But in others, like abbreviations, the punctuation is considered part of the word.

### Edge cases

How many times does the word `left` appear in this sentence?

```
I left, and then they left after.
```

The naive splitter will parse `left,` and `left` as distinct words and claim that the word `left` only appears once. We might consider trimming trailing punctuation to account for this. But we wouldn't want to over-trim punctuation for abbreviations (`N.A.S.A.`), plural possessives (`Smiths'`), or informal contractions (`ain'`).

---

Does the following sentence contain the word `should`?

```
The owner shouldn't enter the rental property without providing 24 hours' notice.
```

Strictly speaking, the sentence contains the word `shouldn't`, a contraction with is distinct from the word `should`. But for kinds of textual analysis, we might be more concerned with the root word, and would want to treat both `should` and `shouldn't` as equivalent.

More broadly, the English language often allows us to combine two words into one. We have to decide if we want to retain the combined form of the word, or attempt to parse it into its components. This is a design choice that depends on the kind of analysis we want to perform on the text.

### Advanced Strategies

Rules-based strategies tend to perform quite well on most corpuses. The rules-based methods included in the Natural Language Toolkit (NLTK) have an approximate error rate of 0.27%. They tend to rely on regular expressions that embed our understanding of when punctuation should be included or excluded from a word.

By default, NLTK is over-eager on splitting words based on punctuation. To give an example of the default behavior:

```python
from nltk.tokenize.punkt import PunktLanguageVars

tokenizer = PunktLanguageVars()
tokenizer.word_tokenize("Hey, why can't we try this instead?")

# OUT: ['Hey', ',', 'why', 'can', "'t", 'we', 'try', 'this', 'instead', '?']
```

### Modifying the NLTK Regex to improve word detection

The NLTK source includes the Regex that NLTK uses to parse text into words. We can modify that Regex to be more tolerant of punctuation within a word in cases of contractions, abbreviations, numbers, and so on.

The following snippet relies heavily on the Regex strategies of [lazy quantifiers](https://www.rexegg.com/regex-quantifiers.html#lazy), and [positive and negative lookaheads](https://www.rexegg.com/regex-lookarounds.html).

```python
import re

# Lazily match non-whitespace characters. This sequence does the
# bulk of the work for this regex and is what actually consumes
# characters.
WORD_CHARACTERS = r"\S+?"

# These characters MAY mark the beginning or the end of the word.
# Some of them can also be found within words, though. Usually,
# this part of the Regex will match against a space character.
WORD_BOUNDARY_CANDIDATES = r"\s?!\"“”‘’`—'().,:;=\[\]{}"

# Optionally consume (include in the word) commas and apostrophes
# as long as they're followed by an alphanumeric character.
#
# Match: "We [couldn't] believe it"
# Match: "The [N.A.S.A.] launch is scheduled"
# Match: "Welcome to [N.A.S.A.'s] control center"
# No Match: "Do not include 'closing [punctuation).']"
OPTIONAL_PUNCTUATION_IN_WORD = r"(?:(?:[\,\'\’\.]|\.\')[\w%]+)*"

# Optionally consume the period at the end of the word.
# Do not include the period if it is followed by two more periods
# (which would denote an ellipses).
#
# Match: Welcome [Mr.] Bond
#
# Note: This tends to include the periods at the end of sentences.
# Accordingly, this Regex expects to receive strings that are
# sentences where the closing punctuation has been stripped.
OPTIONAL_PERIOD_AT_END_OF_WORD = r"(?:\.(?!\.\.))?"

# Use a negative lookahead (?!) to detect the beginning of words.
BEGINNING_OF_WORD_MARKERS = f"(?![{WORD_BOUNDARY_CANDIDATES}])"

# Use a positive lookahead (?=) to detect the end of words.
# Also, match against the end of the string.
END_OF_WORD_MARKERS = f"(?=[{WORD_BOUNDARY_CANDIDATES}]|$)"

WORD_REGEX = f"""
(
    {BEGINNING_OF_WORD_MARKERS}
    {WORD_CHARACTERS}
    {OPTIONAL_PUNCTUATION_IN_WORD}
    {OPTIONAL_PERIOD_AT_END_OF_WORD}
    {END_OF_WORD_MARKERS}
)
"""

matcher = re.compile(WORD_REGEX)
matcher.findall("Hey, why can't we try this instead?")

# OUT: ['Hey', 'why', "can't", 'we', 'try', 'this', 'instead']
```

### Further reading

 - [How to Split Sentences (Grammarly)](https://tech.grammarly.com/blog/how-to-split-sentences)
 - [Natural Language Toolkit (NLTK) Sentence and Word Tokenizer Source](https://www.nltk.org/_modules/nltk/tokenize/punkt.html)