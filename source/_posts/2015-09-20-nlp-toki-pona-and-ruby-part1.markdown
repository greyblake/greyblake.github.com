---
layout: post
title: "NLP, Toki Pona and Ruby: part 1"
date: 2015-09-20 20:36
comments: true
categories: nlp tokipona ruby
---
## Intro
During last few years, I spent a lot of time learning foreign languages like Esperanto, Spanish and German.
After a while, I came up with an idea that I can apply this knowledge in computer science.

When I decided this I was completely new to Computational Linguistics(CL) and Natural Language Processing(NLP).
However after reading a number of articles I got some basic ideas.

## What I am gonna do
To dive into CL/NLP I've decided implement Toki Pona -> English translator from scratch.
It's interesting to see which issues I will face and how I will solve them.
It will make me go through number of stages of language processing:

* Lexical analysis
* Language detection (I want to distinguish Toki Pona from other languages)
* Morphological analysis (actually will be skipped because of simplicity of Toki Pona)
* Syntax analysis
* Word translation
* Syntax tree conversion
* Generation of final translation with respect to English grammar.

Anyway, this list is not strict, and probably it will be modified in the future.

## What I am not gonna do
There are many tools and libraries that already exist in Ruby for NLP.
I am not gonna use any of them here neither cover them in the articles.
If you need something like that, please take a look at [ruby-nlp](https://github.com/diasks2/ruby-nlp).
It's a document that gathers a variety of NLP tools implemented in ruby.

## What is Toki Pona?

[Toki Pona](https://en.wikipedia.org/wiki/Toki_Pona) is a constructed language created by Sonja Lang in 2001.
What is so special about it? Its vocabulary is limited and contains only **125 words**.
The grammar is regular (anyway there will be some pitfalls). The language itself simple and can be learned in 1-2 nights,
and I believe it allows to express 80-90% of daily human communication. Also, it has some philosophical background:
speaking the language you realize what things really are.

Example: there is no word like "friend", one would say "jan pona", what literally  means "good person/human".
In similar way "an ocean" is "telo suli" (big water), "juice" is "telo kili" (water of fruit or vegetable), etc.

So, even Toki Pona is not real _natural_ language, it's good to experiment with, and it gives me some hope that my
goal can be achieved :)

And the end of this article you'll find number of useful links if you want to get into the language.


## First step: lexical analysis

The first step in processing natural or programming language is **lexical analysis**. It means splitting sequence of
characters into some meaningful units: **tokens**. Sometimes the process is called **tokenization** and
the tools that do it are **tokenizers** or **lexical analyzers**.

Let's see an example. Given a sentence:
```
jan suli li pona.
```
Translation: "Big man is good"
(_jan_ - human/man, _suli_ - big, _li_ - is/are, _pona_ - good).

Note: in Toki Pona the main word goes first, so noun(_jan_) is on the first position,
and on the second position is adjective(_suli_) that modifies the noun.


Expected list of tokens is
```ruby
["jan", "suli", "li", "pona", "."]
```

Let's implement class `Tokipona::Tokenizer` with a class method `.tokenize` that returns an array of
tokens for a given text. We start with tests first.

```ruby
describe Tokipona::Tokenizer do
  describe ".tokenize" do
    context "only words" do
      it "returns array of words" do
        text = "toki mi li toki pona"
        tokens = described_class.tokenize(text)
        expect(tokens).to eq ["toki", "mi", "li", "toki", "pona"]
      end
    end

    context "words with multiple spaces in between" do
      it "returns array of words" do
        text = "toki   mi   li   toki   pona"
        tokens = described_class.tokenize(text)
        expect(tokens).to eq ["toki", "mi", "li", "toki", "pona"]
      end
    end

    context "words with special characters" do
      it "returns array of words and characters" do
        text = "sina wile lape anu seme, jan lane?"
        tokens = described_class.tokenize(text)
        expect(tokens).to eq ["sina", "wile", "lape", "anu", "seme", ",", "jan", "lane", "?"]
      end
    end

    it "does not change input text" do
      text = "toki mi li pona"
      described_class.tokenize(text)
      expect(text).to eq "toki mi li pona"
    end
  end
end
```

Usually lexical analysis for programming languages is based on [finite-state automata](http://web.cse.ohio-state.edu/~gurari/course/cse756/html/cse756se2.html).
But in our simple case we can easily handle it with one regular expression:

```ruby
module Tokipona
  class Tokenizer
    def self.tokenize(text)
      @text.scan(/\w+|[^\s]/)
    end
  end
end
```

This implementation looks very naive, but specs pass, so we leave it as it is.
Probably in the future we will modify.

## Conclusion

It is the first article and the beginning of the journey. The next step will be an implementation
of Toki Pona language detector. It's not necessary to know Toki Pona to follow me,
but in case you are interested, here below I provide some useful links, so you can learn yourself
and start communicating.

I've created a github repository where you can access the code: [greyblake/tokipona](https://github.com/greyblake/tokipona).

P.S.

Thanks for reading. The subject is new for me, so your comments, suggestions and feedback can be very helpful.


## Links

* [Toki Pona at Wikipedia](https://en.wikipedia.org/wiki/Toki_Pona)
* [Official Toki Pona site](http://tokipona.org/)
* [Toki Pona lessons](http://rowa.giso.de/languages/toki-pona/english/lessons.php) - here you can start learning the language
* [Toki Pona visual vocabulary](http://x-raizor.github.io/visual-tokipona/index.html)
* \#tokipona - IRC channel where you can communicate with other people

