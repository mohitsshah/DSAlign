# DSAlign
DeepSpeech based forced alignment tool

## Installation

It is recommended to use this tool from within a virtual environment.
There is a script for creating one with all requirements in the git-ignored dir `venv`:

```bash
$ bin/createenv.sh
$ ls venv
bin  include  lib  lib64  pyvenv.cfg  share
```

`bin/align.sh` will automatically use it.

## Prerequisites

### Language specific data

Internally DSAlign uses the [DeepSpeech](https://github.com/mozilla/DeepSpeech/) STT engine.
For it to be able to function, it requires a couple of files that are specific to 
the language of the speech data you want to align.
If you want to align English, there is already a helper script that will download and prepare
all required data:

```bash
$ bin/getmodel.sh 
[...]
$ ls models/en/
alphabet.txt  lm.binary  output_graph.pb  output_graph.pbmm  output_graph.tflite  trie
```

### Dependencies for generating individual language models

If you plan to let the tool generate individual language models per text (you should!),
you have to get (essentially build) [KenLM](https://kheafield.com/code/kenlm/).
Before doing this, you should install its [dependencies](https://kheafield.com/code/kenlm/dependencies/).
For Debian based systems this can be done through:
```bash
$ sudo apt-get install build-essential libboost-all-dev cmake zlib1g-dev libbz2-dev liblzma-dev 
```

With all requirements fulfilled, there is a script for building and installing KenLM
and the required DeepSpeech tools in the right location:
```bash
$ bin/lm-dependencies.sh
```

If all went well, the alignment tool will find and use it to automatically create individual
language models for each document.

### Example data

There is also a script for downloading and preparing some public domain speech and transcript data.

```bash
$ bin/gettestdata.sh
$ ls data
test1  test2
```

## Using the tool

```bash
$ bin/align.sh --help
[...]
```

### Alignment using example data

```bash
$ bin/align.sh --output-max-cer 15 --loglevel 10 --audio data/test1/audio.wav --script data/test1/transcript.txt --aligned data/test1/aligned.json --tlog data/test1/transcript.log
```

## The algorithm

### Step 1 - Splitting audio

A voice activity detector (at the moment this is `webrtcvad`) is used
to split the provided audio data into voice fragments.
These fragments are essentially streams of continuous speech without any longer pauses 
(e.g. sentences).

`--audio-vad-aggressiveness <AGGRESSIVENESS>` can be used to influence the length of the
resulting fragments.

### Step 2 - Preparation of original text

STT transcripts are typically provided in a normalized textual form with
- no casing,
- no punctuation and
- normalized whitespace (single spaces only).

So for being able to align STT transcripts with the original text it is necessary
to internally convert the original text into the same form.

This happens in two steps:
1. Normalization of whitespace, lower-casing all text and 
replacing some characters with spaces (e.g. dashes)
2. Removal of all characters that are not in the languages's alphabet
(see DeepSpeech model data)

Be aware: *This conversion happens on text basis and will not remove unspoken content
like markup/markdown tags or artifacts. This should be done beforehand.
Reducing the difference of spoken and original text will improve alignment quality and speed.*

In the very unlikely situation that you have to change the default behavior (of step 1),
there are some switches:

`--text-keep-dashes` will prevent substitution of dashes with spaces.

`--text-keep-ws` will keep whitespace untouched.

`--text-keep-casing` will keep character casing as provided.

### Step 4a (optional) - Generating document specific language model

If the [dependencies][Dependencies for generating individual language models] for 
individual language model generation got installed, this document-individual model will
now be generated by default.

Assuming your text document is named `original.txt`, these files will be generated:
- `original.txt.clean` - cleaned version of the original text
- `original.txt.arpa` - text file with probabilities in ARPA format
- `original.txt.lm` - binary representation of the former one
- `original.txt.trie` - prefix-tree optimized for probability lookup

`--stt-no-own-lm` deactivates creation of individual language models per document and
uses the one from model dir instead.

### Step 4b - Transcription of voice fragments through STT

After VAD splitting the resulting fragments are transcribed into textual phrases.
This transcription is done through [DeepSpeech](https://github.com/mozilla/DeepSpeech/) STT.

As this can take a longer time, all resulting phrases are - together with their 
timestamps - saved as JSON into a transcription log file 
(the `audio` parameter path with suffix `.tlog` instead of `.wav`).
Consecutive calls will look for that file and - if found - 
load it and skip the transcription phase.

`--stt-model-dir <DIR>` points DeepSpeech to the language specific model data directory.
It defaults to `models/en`. Use `bin/getmodel.sh` for preparing it.  

### Step 5 - Rough alignment

The actual text alignment is based on a recursive divide and conquer approach:

1. Construct an ordered list of of all phrases in the current interval
(at the beginning this is the list of all phrases that are to be aligned),
where long phrases close to the middle of the interval come first.
2. Iterate through the list and compute the best Smith-Waterman alignment
(see the following sub-sections) with the document's original text...
3. ...till there is a phrase whose Smith-Waterman alignment score surpasses a (low) recursion-depth 
dependent threshold (in most cases this should already be the first phrase).
4. Recursively continue with step 1 for the sub-intervals and original text ranges
to the left and right of the phrase and its aligned text range within the original text.
5. Return all phrases in order of appearance (depth-first) that were aligned with the minimum 
Smith-Waterman score on their recursion level.

This approach assumes that all phrases were spoken in the same order as they appear in the
original transcript. It has the following advantages compared to individual
global phrase matching:

- Long non-matching chunks of spoken text or the original transcript will automatically and 
cleanly get ignored.
- Short phrases (with the risk of matching more than one time per document) will automatically
get aligned to their intended locations by longer ones who "squeeze" them in.
- Smith-Waterman score thresholds can be kept lower 
(and thus better match lower quality STT transcripts), as there is a lower chance for 
  - long sequences to match at a wrong location and for 
  - shorter sequences to match at a wrong location within their shortened intervals
  (as they are getting matched later and deeper in the recursion tree).

#### Smith-Waterman candidate selection

Finding the best match of a given phrase within the original (potentially long) transcript
using vanilla Smith-Waterman is not feasible.

So this tool follows a two-phase approach where the first goal is to get a list of alignment 
candidates. As the first step the original text is virtually partitioned into windows of the 
same length as the search pattern. These windows are ordered descending by the number of 3-grams
they share with the pattern.
Best alignment candidates are now taken from the beginning of this ordered list.

`--align-max-candidates <CANDIDATES>` sets the maximum number of candidate windows
taken from the beginning of the list for further alignment.

`--align-candidate-threshold <THRESHOLD>` multiplied with the number of 3-grams of the predecessor
window it gives the minimum number of 3-grams the next candidate window has to have to also be
considered a candidate.

#### Smith-Waterman alignment

For each candidate, the best possible alignment is computed using the 
[Smith-Waterman](https://en.wikipedia.org/wiki/Smith%E2%80%93Waterman_algorithm) algorithm
within an extended interval of one window-size around the candidate window.

`--align-match-score <SCORE>` is the score per correctly matched character. Default: 100

`--align-mismatch-score <SCORE>` is the score per non-matching (exchanged) character. Default: -100

`--align-gap-score <SCORE>` is the score per character gap (removing 1 character from pattern or original). Default: -100

The overall best score for the best match is normalized to a value of about 100 maximum by dividing
it through the maximum character count of either the match or the pattern.

### Step 6 - Gap alignment

After recursive matching of fragments there are potential text leftovers between aligned original
texts.

Some examples:
- Often: Missing (and therefore unaligned) STT transcripts of word-endings (e.g. English past tense endings _-d_ and _-ed_)
on phrase endings to the left of the gap
- Seldom: Phrase beginnings or endings that were wrongly matched on unspoken (but written) text whose actual
alignments are now left unaligned in the gap
- Big unmatched chunks of text, like
  - Preface, text summaries or any other kind of meta information
  - Copyright headers/footers
  - Table of contents
- Chapter headers (if not spoken as they appear)
- Captions of figures
- Contents of tables
- Line-headers like character names in drama scripts
- Dependent of the (pre-processing) quality: OCR leftovers like
  - page headers
  - page numbers
  - reader's notes
  
The basic challenge here is to figure out, if all or some of the gap text should be used to extend 
the phrase to the left and/or to the right of the gap.

As Smith-Waterman alignment led to the current (potentially incomplete or even wrong) result,
its score cannot be used for further fine-tuning. Therefore there is a collection of
so called test-distance algorithms to pick from using the `--align-similarity-algo`
parameter.

Using the selected distance metric, the gap alignment is done by looking for the best scoring 
extension of the left and right phrases up to their maximum extension.

`--align-stretch-factor <FRACTION>` is the fraction of the text length that it could get
stretched at max.  

For many languages it is worth putting some emphasis on matching to words boundaries 
(that is white-space separated sub-sequences).

`--align-snap-factor <FACTOR>` allows to control the snappiness to word boundaries.

If the best scoring extensions should overlap, the best scoring sum of non-overlapping
(but touching) extensions will win.

### Step 7 - Selection, filtering and output

Finally the best alignment of all candidate windows is selected as the winner.
It has to survive a series of filters for getting into the result file.

For each text distance metric there are two filter parameters:

`--output-min-<METRIC-ID> <VALUE>` only keeps utterances having the provided minimum value for the
metric with id `METRIC-ID`
                              
`--output-max-<METRIC-ID> <VALUE>` only keeps utterances having the provided maximum value for the
metric with id `METRIC-ID`

For each text distance metric there's also the option to have it added to each utterance's entry:

`--output-<METRIC-ID>` adds the computed value for `<METRIC-ID>` to the utterances array-entry

Error rates and scores are provided as fractional values (typically between 0.0 = 0% and 1.0 = 100%
where numbers >1.0 are theoretically possible).

### General options

`--play` will play each aligned sample using the `play` command of the SoX audio toolkit

`--text-context <CONTEXT-SIZE>` will add additional `CONTEXT-SIZE` characters around original
transcripts when logged

## Export

After files got successfully aligned, one would possibly want to export the aligned utterances
as machine learning training samples.

This is where the export tool `bin/export.sh` comes in.

### Step 1 - Reading the input

The exporter takes either a single audio file (`--audio`) 
plus a corresponding `.aligned` file (`--aligned`) or a series
of such pairs from a `.catalog` file (`--catalog`) as input.

All of the following computations will be done on the joined list of all aligned
utterances of all input pairs.

### Step 2 - (Pre-) Filtering

The parameter `--filter <EXPR>` allows to specify a Python expression that has access
to all data fields of an aligned utterance (as can be seen in `.aligned` file entries).

This expression is now applied to each aligned utterance and in case it returns `True`,
the utterance will get excluded from all the following steps. 
This is useful for excluding utterances that would not work as input for the planned
training or other kind of application.

### Step 3 - Computing quality

As with filtering, the parameter `--criteria <EXPR>` allows for specifying a Python 
expression that has access to all data fields of an aligned utterance.

The expression is applied to each aligned utterance and its numerical return 
value is assigned to each utterance as `quality`.

### Step 4 - De-biasing

This step is to (optionally) exclude utterances that would otherwise bias the data
(risk of overfitting).

For each `--debias <META DATA TYPE>` parameter the following procedure is applied:
1. Take the meta data type (e.g. "name") and read its instances (e.g. "Alice" or "Bob")
from each utternace and group all utterances accordingly
(e.g. a group with 2 utterances of "Alice" and a group with 15 utterances of "Bob"...)
2. Compute the standard deviation (`sigma`) of the instance-counts of the groups
3. For each group: If the instance-count exceeds `sigma` times `--debias-sigma-factor <FACTOR>`:
    - Drop the number of exceeding utterances in order of their `quality` (lowest first)
    
### Step 5 - Partitioning

Training sets are often partitioned into several quality levels.

For each `--partition <QUALITY:PARTITION>` parameter (ordered descending by `QUALITY`):
If the utterance's `quality` value is greater or equal `QUALITY`, assign it to `PARTITION`.

Remaining utterances are assigned to partition `other`.

### Step 6 - Splitting

Training sets (actually their partitions) are typically split into sets `train`, `dev` 
and `test` ([explanation](https://en.wikipedia.org/wiki/Training,_validation,_and_test_sets)).

This can get automated through parameter `--split` which will let the exporter split each
partition (or the entire set) accordingly.

Parameter `--split-field` allows for specifying a meta data type that should be considered 
atomic (e.g. "speaker" would result in all utterances of a speaker 
instance - like "Alice" - to end up in one sub-set only). This atomic behavior will also hold
true across partitions.

### Step 7 - Output

For each partition/sub-set combination the following is done:
 - Construction of a `name` (e.g. `good-dev` will represent the validation set of partition `good`).
 - Writing all utterance audio fragments (as `.wav` files) into a sub-directory of `--target-dir <DIR>`
 named `name` (using parameters `--channels <N>` and `--rate <RATE>`).
 - Writing an utterance list into `--target-dir <DIR>` named `name.(json|csv)` dependent on the
 output format specified through `--format <FORMAT>`
 
### Additional functionality

Using `--dry-run` one can avoid any writing and get a preview on set-splits and so forth
(`--dry-run-fast` won't even load any sample).

`--force` will force overwriting of samples and list files.

`--workers <N>` allows for specifying the number of parallel workers.

## File formats

### Catalog files (.catalog)

Catalog files (suffix `.catalog`) are used for organizing bigger data file collections and
defining relations among them. It is basically a JSON array of hash-tables where each entry stands
for a single audio file and its associated original transcript.

So a typical catalog looks like this (`data/all.catalog` from this project):

```javascript
[
  {
    "audio": "test1/joined.mp3",
    "tlog": "test1/joined.tlog",
    "script": "test1/transcript.txt",
    "aligned": "test1/joined.aligned"
  },
  {
    "audio": "test2/joined.mp3",
    "tlog": "test2/joined.tlog",
    "script": "test2/transcript.script",
    "aligned": "test2/joined.aligned"
  }
]
```

- `audio` is a path to an audio file (of a format that `pydub` supports)
- `tlog` is the (supposed) path to the STT generated transcription log of the audio file
- `script` is the path to the original transcript of the audio file
(as `.txt` or `.script` file)
- `aligned` is the (supposed) path to a `.aligned` file

Be aware: __All relative file paths are treated as relative to the catalog file's directory__.

The tools `bin/align.sh`, `bin/statistics.sh` and `bin/export.sh` all support parameter
`--catalog`:

The __alignment tool__ `bin/align.sh` requires either `tlog` to point to an existing
file or (if not) `audio` to point to an existing audio file for being able to transcribe
it and store it at the path indicated by `tlog`. Furthermore it requires `script` to
point to an  existing script. It will write its alignment results to the path in `aligned`.

The __export tool__ `bin/export.sh` requires `audio` and `aligned` to point to existing files.

The __statistics tool__ `bin/statistics.sh` requires only `aligned` to point to existing files.

Advantages of having a catalog file:

- Simplified tool usage with only one parameter for defining all involved files (`--catalog`).
- A directory with many files has to be scanned just one time at catalog generation.
- Different file types can live at different and custom locations in the system.
This is important in case of read-only access rights to the original data.
It can also be used for avoiding to taint the original directory tree.
- Accumulated statistics
- Better progress indication (as the total number of files is available up front)
- Reduced tool startup overhead
- Allows for meta-data aware set-splitting on export - e.g. if some speakers are speaking
in several files.

So especially in case of many files to process it is highly recommended to __first create
a catalog file__ with all paths present (even the ones not pointing to existing files yet).


### Script files (.script|.txt)

The alignment tool requires an original script or (human transcript) of the provided audio.
These scripts can be represented in two basic forms:
- plain text files (`.txt`) or
- script files (`.script`)

In case of plain text files the content is considered a continuous stream of text without
any assigned meta data. The only exception is option `--text-meaningful-newlines` which
tells the aligner to consider newlines as separators between utterances
in conjunction with option `--align-phrase-snap-factor`.

If the original data source features utterance meta data, one should consider converting it
to the `.script` JSON file format which looks like this
(except of `data/test2/transcript.script`): 

```javascript
[
  // ...
  {
    "speaker": "Phebe",
    "text": "Good shepherd, tell this youth what 'tis to love."
  },
  {
    "speaker": "Silvius",
    "text": "It is to be all made of sighs and tears; And so am I for Phebe."
  },
  // ...
]
```

_This and the following sub-sections are all using the same real world examples and excerpts_

It is basically again an array of hash-tables, where each hash-table represents an utterance with the
only mandatory field `text` for its textual representation.

All other fields are considered meta data 
(with the key called "meta data type" and the value "meta data instance").

### Transcription log files (.tlog)

The alignment tool relies on timed STT transcripts of the provided audio.
These transcripts are either provided by some external processing 
(even using a different STT system than DeepSpeech) or will get generated
as part of the alignment process.

They are called transcription logs (`.tlog`) and are looking like this
(except of `data/test2/joined.tlog`):

```javascript
[
  // ...
  {
    "start": 7491960,
    "end": 7493040,
    "transcript": "good shepherd"
  },
  {
    "start": 7493040,
    "end": 7495110,
    "transcript": "tell this youth what tis to love"
  },
  {
    "start": 7495380,
    "end": 7498020,
    "transcript": "it is to be made of soles and tears"
  },
  {
    "start": 7498470,
    "end": 7500150,
    "transcript": "and so a may for phoebe"
  },
  // ...
]
```

The fields of each entry:
- `start`: time offset of the audio fragment in milliseconds from the beginning of the
aligned audio file (mandatory)
- `end`: time offset of the audio fragment's end in milliseconds from the beginning of the
aligned audio file (mandatory) 
- `transcript`: STT transcript of the utterance (mandatory)

### Aligned files (.aligned)

The result of aligning an audio file with an original transcript is written to an
`.aligned` JSON file consisting of an array of hash-tables of the following form:

```javascript
[
  // ...
  {
    "start": 7491960,
    "end": 7493040,
    "transcript": "good shepherd",
    "text-start": 98302,
    "text-end": 98316,
    "meta": {
      "speaker": [
        "Phebe"
      ]
    },
    "aligned-raw": "Good shepherd,",
    "aligned": "good shepherd",
    "wng": 99.99999999999997,
    "jaro_winkler": 100.0,
    "levenshtein": 100.0,
    "mra": 100.0,
    "cer": 0.0
  },
  {
    "start": 7493040,
    "end": 7495110,
    "transcript": "tell this youth what tis to love",
    "text-start": 98317,
    "text-end": 98351,
    "meta": {
      "speaker": [
        "Phebe"
      ]
    },
    "aligned-raw": "tell this youth what 'tis to love.",
    "aligned": "tell this youth what 'tis to love",
    "wng": 92.71730687405957,
    "jaro_winkler": 100.0,
    "levenshtein": 96.96969696969697,
    "mra": 100.0,
    "cer": 3.0303030303030303
  },
  {
    "start": 7495380,
    "end": 7498020,
    "transcript": "it is to be made of soles and tears",
    "text-start": 98352,
    "text-end": 98392,
    "meta": {
      "speaker": [
        "Silvius"
      ]
    },
    "aligned-raw": "It is to be all made of sighs and tears;",
    "aligned": "it is to be all made of sighs and tears",
    "wng": 77.93921929148159,
    "jaro_winkler": 100.0,
    "levenshtein": 82.05128205128204,
    "mra": 100.0,
    "cer": 17.94871794871795
  },
  {
    "start": 7498470,
    "end": 7500150,
    "transcript": "and so a may for phoebe",
    "text-start": 98393,
    "text-end": 98415,
    "meta": {
      "speaker": [
        "Silvius"
      ]
    },
    "aligned-raw": "And so am I for Phebe.",
    "aligned": "and so am i for phebe",
    "wng": 66.82687893873339,
    "jaro_winkler": 98.47964113181504,
    "levenshtein": 82.6086956521739,
    "mra": 100.0,
    "cer": 19.047619047619047
  },
  // ...
]
```

Each object array-entry represents an aligned audio fragment with the following attributes:
- `start`: time offset of the audio fragment in milliseconds from the beginning of the
aligned audio file
- `end`: time offset of the audio fragment's end in milliseconds from the beginning of the
aligned audio file
- `transcript`: STT transcript used for aligning
- `text-start`: character offset of the fragment's associated original text within the
aligned text document
- `text-end`: character offset of the end of the fragment's associated original text within the
aligned text document
- `meta`: meta data hash-table with
  - _key_: meta data type
  - _value_: array of meta data instances coalesced from the `.script` entries that
  this entry intersects with
- `aligned-raw`: __raw__ original text fragment that got aligned with the audio fragment
and its STT transcript
- `aligned`: __clean__ original text fragment that got aligned with the audio fragment
and its STT transcript
- `<metric>` For each `--output-<metric>` parameter the alignment tool adds an entry with the
computed value (in this case `wng`, `jaro_winkler`, `levenshtein`, `mra`, `cer`)

## Text distance metrics

This section lists all available text distance metrics along with their IDs for
command-line use.

### Weighted N-grams (wng)

The weighted N-gram score is computed as the sum of the number of weighted shared N-grams
between the two texts.
It ensures that:
- Shared N-gram instances near interval bounds (dependent on situation) get rated higher than
the ones near the center or opposite end
- Large shared N-gram instances are weighted higher than short ones

`--align-min-ngram-size <SIZE>` sets the start (minimum) N-gram size

`--align-max-ngram-size <SIZE>` sets the final (maximum) N-gram size

`--align-ngram-size-factor <FACTOR>` sets a weight factor for the size preference

`--align-ngram-position-factor <FACTOR>` sets a weight factor for the position preference

### Jaro-Winkler (jaro_winkler)

Jaro-Winkler is an edit distance metric described
[here](https://en.wikipedia.org/wiki/Jaro%E2%80%93Winkler_distance).

### Editex (editex)

Editex is a phonetic text distance algorithm described
[here](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.18.2138&rep=rep1&type=pdf).

### Levenshtein (levenshtein)

Levenshtein is an edit distance metric described
[here](https://en.wikipedia.org/wiki/Levenshtein_distance).

### MRA (mra)

The "Match rating approach" is a phonetic text distance algorithm described
[here](https://en.wikipedia.org/wiki/Match_rating_approach).

### Hamming (hamming)

The Hamming distance is an edit distance metric described
[here](https://en.wikipedia.org/wiki/Hamming_distance).

### Word error rate (wer)

This is the same as Levenshtein - just on word level.

Not available for gap alignment.

### Character error rate (cer)

This is the same as Levenshtein but using a different implementation.

Not available for gap alignment.

### Smith-Waterman score (sws)

This is the final Smith-Waterman score coming from the rough alignment
step (but before gap alignment!).
It is described
[here](https://en.wikipedia.org/wiki/Smith%E2%80%93Waterman_algorithm).

Not available for gap alignment.

### Transcript length (tlen)

The character length of the STT transcript.

Not available for gap alignment.

### Matched text length (mlen)

The character length of the matched text of the original transcript (cleaned).

Not available for gap alignment.
