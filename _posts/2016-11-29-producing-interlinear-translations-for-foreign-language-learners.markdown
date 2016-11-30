---
layout: post
title: "Producing interlinear translations for foreign-language Learners"
date: 2016-11-29T12:41:27-08:00
---

This work was inspired by commenter "SFReader" who [claims that it
possible to learn to read (but not to speak) a new foreign language in 10-12 hours](http://languagehat.com/dont-try-so-hard/#comment-1685834).
His method requires a long text in the language you want to learn,
a translation of that text into a language you can already read, and an audio
recording of the text in the original language. The two texts should be arranged such that
each small phrase or sentence in the original language is followed by the translation of
that phrase. Something like this:

<img src="{{ root_url }}/source/images/interlinear_example1.png" />

I call this an interlinear translation, though it's slightly different from a
[standard interlinear translation](https://en.wikipedia.org/wiki/Interlinear_gloss).

I was skeptical of this method, so I wanted to test it out. To do this, one needs
a very long interlinear text. How to get one?

### Solution 1: The Bible

One good place to start is the Bible. For two reasons. First, the Bible has been translated into more human
languages than any other book. Second, the chapter and verse structure of the Bible
means that it is already split into short segments, which makes aligning the texts trivial. (Also,
there are free audiobooks of the Bible available in many languages, which helps for
a language learner.)

The website [BibleGateway.com](https://www.biblegateway.com/) has translations
of the Bible into over 50 languages. I wrote a scraper to download the translations,
preserving chapter and verse information. From there it is easy to produce an interlinear
translation, like this [Hungarian and English version of the Gospel of Mark](https://rawgit.com/wlevine/translation_interleaver/master/texts/Mark_NT-HU_ESV.html).


### Solution 2: Aligning a human-translated text using machine translation

What if you're interested in reading something other than the Bible?

If you have
a source text in one language and a translation of that text in another, you can't
simply make an interlinear text by aligning the two texts sentence-by-sentence. This is
because a single sentence in the source may get split up into multiple sentences in
the translation or vice versa. We would like to have an automated procedure for
correlating sentences, taking into account that in general this can be a many-to-many
correspondence. This is called parallel-text alignment; a standard algorithm
for doing it is the [Gale-Church algorithm](https://en.wikipedia.org/wiki/Gale%E2%80%93Church_alignment_algorithm).

I chose a different approach, taking advantage of machine translation.
Due to the poor quality of machine translation, we don't just want to use
the machine-translated text directly. However, if a sentence
in the human-translated text shares a high number of words with the machine-translated
version of a sentence in the source text, we assume that these two sentences are correlated.
To account for non one-to-one correspondence, I test several possible alignments
for sentences or pairs of sentences and select the one with the highest likelihood.
[Here is an example of the output](https://rawgit.com/wlevine/translation_interleaver/master/texts/underground_chap1.html). Embarrasingly, it gets the first sentence wrong, but other
than that it does pretty well.

The code for this is [available on GitHub](https://github.com/wlevine/translation_interleaver).
