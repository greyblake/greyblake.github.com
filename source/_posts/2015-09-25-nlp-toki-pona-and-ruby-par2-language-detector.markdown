---
layout: post
title: "NLP, Toki Pona and Ruby. Part 2: language detector"
date: 2015-09-25 23:55
comments: true
categories: nlp tokipona ruby
---

Previous articles:

* [Part 1: Intro. Tokenizer](/blog/2015/09/20/nlp-toki-pona-and-ruby-part1)

In the first article we created a simple tokenizer, today we're going to create a
language detector to identify Toki Pona text among other texts.

First I want to say that are at least few good libraries for detecting natural languages in ruby:

* [language_detector](https://github.com/feedbackmine/language_detector) - detector based on [N-grams](https://en.wikipedia.org/wiki/N-gram)
* [whatlanguage](https://github.com/peterc/whatlanguage) - detector based on [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter)

But those are for mainstream: French, English, German... We want Toki Pona!
Also, since we are focused on Toki Pona only, we can get much more precise results.

<!--more-->

## The algorithm

My initial idea was dead simple: since there are only 125 Toki Pona words, we can
simply check whether a token is a Toki Pona word or something else. Then it's
easy to calculate a density of Toki Pona words in a given phrase and compare it
against some threshold, where 0 < threshold <= 1.

Here is an example of how such algorithm would work with threshold=0.75 and given
phrase _"mi moka e kala suli"_.

```
Phrase:   mi   moka   e    kala    suli
Weights:  1    0      1    1       1

Sum weight: 1 + 0 + 1 + 1 + 1 = 4
Words count: 5
Density: 4 / 5 = 0.8

0.8 > threshold => true (it's Toki Pona)
```

The phrase has a typo: instead of word `moka` there should be word `moku`.
_"mi moku e kala suli"_ means "I am eating a big fish".

I decided to make the algorithm face real data: I picked ~100 random message
from \#tokipona IRC channels and split them into three groups:

1. Messages in Toki Pona
2. Messages in other languages (mostly English)
3. Mixed messages (half Toki Pona and half English)

I wrote a spec, I expected `LanguageDetector.toki_pona?(text)` to return `true` for the first group of messages
and `false` for others. After playing with the value of the threshold, I made it
work almost for all cases.

One of messages looked like this:
```
Moku pona xD
```

It's obvious that it's pure Toki Pona, then what's wrong with it?
The issue was in the `Tokenizer` that we had implemented in the [previous article](/blog/2015/09/20/nlp-toki-pona-and-ruby-part1).

For this message it returns 4 tokens: `["Moku", "pona", "x", "D"]`. While
`Moku` and `pona` belong to Toki Pona  vocabulary,  `x` and `D` don't.

## Updating Tokenizer

We, as humans, can see that `xD` actually must be one token, and it's not a regular word, but a smile.
So I had to update `Tokenizer` to distinguish words and smiles.

That's the point where I had to introduce the difference between **tokens** and **lexemes**.

As the famous [Dragon Book](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools) says about lexemes:

{% blockquote %}
A lexeme is a sequence of characters in the source program that matches the pattern
for a token and is identified by the lexical analyzer as an instance of that token.
{% endblockquote %}

And about tokens:

{% blockquote %}
A token is a pair consisting of a token name and an optional attribute value.
The token name is an abstract symbol representing a kind of lexical unit, e.g.,
a particular keyword, or sequence of input characters denoting an identifier.
The token names are the input symbols that the parser processes.
{% endblockquote %}

So I've updated `Tokenizer` to return array of hashes with `:lexeme` and `:type` keys.
By now I have 3 types of tokens: _word_, _smile_ and _punctuation_.

Example:
```ruby
Tokenizer.tokenize "moku li ike :("  # Translation: food is bad
# => [ {:lexeme=>"moku", :type=>:word}, {:lexeme=>"li", :type=>:word},
#      {:lexeme=>"ike", :type=>:word}, {:lexeme=>":/", :type=>:smile}]
```

The implementation of `Tokenizer` now is the following:

```ruby
class Tokenizer
  SMILE_REGEXP = /
    (?:
      (?: : | ; | = )                            # eyes
      -?                                         # nose
      (?: \) | \| | \\ | \/ | D | P | p | \* )   # mouth
    ) | (?:x|X)D                                 # other (e.g. XD)
  /x
  WORD_REGEXP = /\w+/
  PUNCTUATION_REGEX = /[^\s]*/
  LEXEME_REGEXP = /#{SMILE_REGEXP}|#{WORD_REGEXP}|#{PUNCTUATION_REGEX}/

  def self.tokenize(text)
    new(text).tokenize
  end

  def initialize(text)
    @text = text
  end

  def tokenize
    lexemes = @text.scan(LEXEME_REGEXP)
    lexemes.map { |lex| lexeme_to_token(lex) }
  end

  private def lexeme_to_token(lexeme)
    { lexeme: lexeme, type: token_type(lexeme) }
  end

  private def token_type(lexeme)
    case lexeme
    when SMILE_REGEXP then :smile
    when WORD_REGEXP  then :word
    else :punctuation
    end
  end
end
```

Now tokenization process contains two stage: detecting lexemes and defining tokens
for those lexemes. Note, that the order of regular expressions in `token_type(lexeme)`
method is important. Since `SMILE_REGEXP` and `WORD_REGEXP` can overlap (e.g. "XD"),
we want `SMILE_REGEXP` to have higher priority. The same with punctuation: everything
what is not a smile or a word we consider as punctuation.

## Levenshtein distance

So I've I updated the language detection algorithm to count only words. Still
there are number of things to improve.

What if some words are meant to be Toki Pona words but they contain a typo?
Actually we can detect them and adjust the algorithm to count them as well.

So to detect a word with a typo we need somehow to calculate word similarity.
And actually what we need is called [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance).

{% blockquote %}
Levenshtein distance between two words is the minimum number of single-character
edits (i.e. insertions, deletions or substitutions) required to change one word into the other.
{% endblockquote %}

Levenshtein distance is used in computer science (e.g. for spell checkers), genetics (comparison of gens)
and likely in some other areas. The algorithm isn't the simplest one, but is not hard to understand, so
I encourage you to take a look at [wikipedia](https://en.wikipedia.org/wiki/Levenshtein_distance) to
uderstand it better.

Now back to our case: how do we measure the difference between words `moka` and `moku`?
The levenshtein distance for them is 1, because only one edit needs to be performed to make `moka` become
`moku`: replace `a` with `u`.

We can implement it based on the algorithm ourself (actually I did it for fun, but then replaced it
with [this implementation](https://github.com/threedaymonk/text/blob/master/lib/text/levenshtein.rb)) or google
for existing solutions.
One of suprises was to find it in
[rubygems](https://github.com/rubygems/rubygems/blob/45966be372d85520630143090b82b455d287cec6/lib/rubygems/text.rb#L42-L72) gem.
I guess it's used when one mistypes name of gem in order to have "Did you mean ...?" feature.


## Updating the detection algorithm

Now, we can adjust the LanguageDetector to be not so strict with typos, and give a word score `0.5` if it has
levenshtein distance 1 with one of Toki Pona words. Considering the previous example with phrase _"mi moka e kala suli"_,
the density will be 0.9:

```
Phrase:   mi   moka   e    kala    suli
Weights:  1    0.5    1    1       1

Sum weight: 1 + 0.5 + 1 + 1 + 1 = 4.5
Words count: 5
Density: 4.5 / 5 = 0.9

0.9 > threshold => true (it's Toki Pona)
```

In reality, apart from the mentioned 125 words, the Toki Pona vocablurary includes names of languages, countries and cities.
So the real detector is slightly more complected. You can check it
[here](https://github.com/greyblake/tokipona/blob/60d8ec72f2da6af26440239e8cb1f0fed5bea8a5/lib/tokipona/language_detector.rb).

Btw, the entire implementation of what is being described can be found as the project at
github [greyblake/tokipona](https://github.com/greyblake/tokipona).

That's it for now! In the next part we will try to implement grammar and parser for Toki Pona!

## Links
* [Part 1: implementing Tokenizer](blog/2015/09/20/nlp-toki-pona-and-ruby-part1)
* [Official Toki Pona site](http://tokipona.org/)
* [Levenshtein distance at Wikipedia](https://en.wikipedia.org/wiki/Levenshtein_distance)
* [My tokipona project at Github](https://github.com/greyblake/tokipona)
