---
layout: post
title: "How to compare audio in ruby"
date: 2013-12-19 19:51
comments: true
categories: ruby rspec audio sound chromaprint
---

## Or how to implement sound_like RSpec matcher

The problem I'm trying to solve in this article is comparison of two
audio files. We'll figure out how to verify that they sound similar.

I was developing an application that has a deal with audio processing and
I had to write a test to verify outcome audio file matches a one
from fixtures. Well, I've decided to compare audio binaries like these:

```ruby
expect(File.read('outcome.mp3')).to eq File.read('fixture.mp3')
```

And it worked!

But soon my colleagues let me know I had broken the build. It turned out
that `outcome.mp3`generated on their Mac books didn't match `fixture.mp3`
generated on my linux laptop, despite the fact that both sounded
absolutely the same. Probably we had different codecs.
So I had to come up with a better idea.


## Audio fingerprints and Chromaprint

After some investigation I found a term "audio fingerprint" or "acoustic fingerprint",
it was exactly what I was looking for. From Wikipedia:

> An acoustic fingerprint is a condensed digital summary, deterministically generated
from an audio signal, that can be used to identify an audio sample or quickly locate
similar items in an audio database

It's used by services like Shazam to identify songs.

So I started looking for open source implementations and found
[Chromaprint](http://acoustid.org/chromaprint) - a C library that calculates audio fingerprints
from raw audio files. It seemed to be simple, with good source documentation
and easy to get started.


## Integrate Chromaprint with Ruby

I found no already existing bindings, so I've implemented
[my own](https://github.com/TMXCredit/chromaprint). Instead of using C,
I gave [FFI](https://github.com/ffi/ffi) a shot and it worked perfect!
As result I had stuff that worked the following way:

```ruby
context     = Chromaprint::Context.new(44100, 1)
fingerprint = context.get_fingerprint(raw_audio_data)
fingerprint.raw # => [294890785, 328373552, 315802880, 303481088, ...]
```

According to Chromaprint's documentation a raw fingerprint is an array of 4 byte integers.
But how to compare to 2 fingerprints to detect similarity?

## Hamming distance

The answer was to calculate Hamming distance from binary representation of fingerprints.
Again according to Wikipedia:

> In information theory, the Hamming distance between two strings of equal length is
the number of positions at which the corresponding symbols are different. In another
way, it measures the minimum number of substitutions required to change one string
into the other, or the minimum number of errors that could have transformed one
string into the other.


To calculate Hamming distance for binary data we need to apply XOR operation and count
number of 1 in the result.

Here is a small example for 2 byte values:

    dec     bin
    11737   00101101 11011001
    27129   01101001 11111001

    XOR     01000100 00100000

    Hamming distance is 3

Basing on this I implemented an additional method `Fingerprint#compare(fingerprint)`
that calculates similarity in range from 0 to 1.


## Create RSpec matcher

Now I could compare raw audio data, but in real world almost always we have to have a deal
with compressed audio like mp3 or ogg. However wav files contain exactly raw audio data.
So I could convert compressed audio to wav, then read it to get raw audio and
calculate fingerprints for comparison. To convert audio I prefer using `sox`
command line tool, it's pretty powerful.

I have to explain that I did it all to avoid having a deal with
audio codecs within ruby, since it would make things be much more complicated.


Finally I got `sound_like` RSpec matcher:

```ruby
# Compare sound of two audio files.
# Based on the Chromaprint library and the +sox+ command like tool.
#
# @example
#   "/Airborne.mp3".should sound_like "/ACDC.mp3"
#   "/Children_of_Bodom.mp3".should_not sound_like "/Britney_Spears.mp3"
RSpec::Matchers.define :sound_like do |expected_file|
  match do |file|
    rate      = 96000
    channels  = 1
    threshold = 0.95

    if File.exists?(expected_file) && File.exists?(file)
      # Convert input files into raw 16-bit signed audio (WAV) to
      # process with Chromaprint:
      sox_command    = "sox %s -e signed -b 16 -t wav - " \
                       "rate #{rate} channels #{channels} 2> /dev/null"
      expected_audio = %x"#{sox_command % [expected_file]}"
      audio          = %x"#{sox_command % [file]}"

      # Get audio fingerprints:
      chromaprint = Chromaprint::Context.new(rate, channels)
      expected_fp = chromaprint.get_fingerprint(expected_audio)
      fp          = chromaprint.get_fingerprint(audio)

      # Compare fingerprints and compare result against threshold:
      expected_fp.compare(fp) > threshold
    else
      false
    end
  end
end
```

Note that I used threshold with value 0.95 because quite rare fingerprints
have 100% match.



## Links

* [Chromaprint ruby port on github](https://github.com/TMXCredit/chromaprint)
* [Chromaprint web page](http://acoustid.org/chromaprint)
* [Acoustic fingerprint in Wikipedia](http://en.wikipedia.org/wiki/Acoustic_fingerprint)
* [Hamming distance in Wikipedia](http://en.wikipedia.org/wiki/Hamming_distance)
* [Most efficient way to calculate Hamming distance in ruby](http://stackoverflow.com/a/6397116/1013173)
