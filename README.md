
# Interoperability of Locality-Sensitive Hashing Schemes

This (work-in-progress) document proposes specific document fingerprinting
algorithms, configuration, and practices for use in identifying and
cross-referencing nearly identical documents in several medias. For example,
web content across various HTML encodings and web design revisions, or research
publications across PDF, HTML, and XML formats.

The motivation to come to a public consensus on specific details of these
techniques is to enhance collaboration in efforts like web archiving,
automated document identification, verifying repository contents, and
near-duplicate-detection across distributed datasets.

These techniques are variously refered to as "near duplicate detection",
"locality-sensitive hashing", and "similarity hashing". The major advances in
this area historically came from commercial search engine development and the
field of information retrieval in academia.

Contributions, corrections, and feedback to this document welcomed! You can use
github features (issues and pull-requests) or email contributors directly.

## Background

The techniques described here usually complement both regular content hashing
and full-document similarity measures. Regular hashing (or digests) generally
have the property that exact byte-for-byte file matches will have equal hashes,
but any small difference between files results in a completely different hash.
The hash is of a fixed length of some 40 to 64 bytes (very small compared to
document size).  This is useful for fast equality checking and verify the
integrity of files, but is too brittle for use identification across media.
Similarity measures (such as cosine similarity or Hamming distance) often take
a full document (or an extract set of tokens from the document) and give a
fractional measure of similarity. They have the property of being relatively
inefficient for lookups in large corpuses because each document must be
compared one-to-one.

Similarity hashing is often used as an optimization on top of similarity
measures: the hashes of all documents are pre-computed and then sorted, and
only those documents within a fixed distance in the sorted list need to be
compared. This reduces identification of near-duplicates in a corpus from
`O(N^2)` similarity computations to `O(N)` similarity compurations. Indexed
lists also reduce work in the context of individual lookup query for large
corpuses.

Some notable algorithms to consider are:

- simhash (aka, Charikar hashing)
- minhash
- weighed minhash

However, the papers describing these algorithms have a few parameters (eg, hash
bit length, token hash algorithm), and leave several important steps up to the
implementor, such as:

- media-specific content extraction (eg, stripping some or all HTML tags)
- input encoding normalization (UTF-8, etc)
- tokenization (particularly for non-latin languages)
- stop words
- string serialization (eg, base16, base32, or base64 encoding)
- prefixes or short identifiers for specific schemes

These parameters are usually left to be tuned by implementors. Some parameters
can still be left to tuning by implementors, because they happen at comparison
time, not during batch indexing:

- number of segmented rotations when sorting and distance of match (in bit
  flips) to seach for 
- algorithm or measure used for final similarity measurement

Note that unlike regular hashing algorithms, the details for similarity hashes
don't necessarily need to be specified in completeness: if differences between
implementations only result in (statistically) tiny changes in the hash, hashes
should still compare somewhat accurately. Complete reproducibility (exact
similarity hash output for exact document input) between implementations is
desirable, but not strictly necessary in all cases, leaving some wiggle room
for implementors and optimization.

## Proposed Schemes

Similar to the `blake2` and `SHA` families of hashes, I can imagine that a
small (1-4) set of variants might be useful for use in different contexts. It's
also good practice to build software, protocols, and data models to permit
swapping out algorithms for future flexibility, though of course the whole
concept here is to settle on a small number of consensus schemes for
interoperability.

TODO: summary table here

### simhash-doc

The roughly defined scope for this scheme would be "documents" of length
between one and fifty pages when printed. If it could scale down to 2-3
paragraphs and up to thousand-page documents that would be great, but it should
be optimized for the narrower scope. It should be useful across media (web
pages, PDFs, plain text, etc), and across the most popular languages.

Proposed extraction and tokenization:

- strip all metadata and styling when extracting. include figure captions,
  table contents, quoted text. Do include reference lists, but do not include
  tokens from URLs or identifiers.
- UTF-8 encoded tokens
- fallback to unicode word-character boundaries for tokenization if a
  language-specific tokenizer is not available
- tokens should include only "word characters", as commonly included in
  unicode-aware regex libraries. Specifically including the cateogires: `Ll Lu
  Lt Lo Lm Mn Nd Pc`. They must include at least one letter/"Alphabetic"
  character.
- specifically, no zero-width or non-printing unicode modifiers
- numbers (unless part of an alphanumeric string, eg an acronym) should not be
  included
- TODO: instead, strip all numeric characters?
- OPTIONALLY, a language-specific stop-list appropriate for search-engine
  indexing may be used.

Proposed algorithm and parameters:

- `simhash` algorithm
- TODO: fast token hashing algorithm?
- TODO: 64x signed 8-bit buckets during calculation
- 64bit length final form
- base32 (non-capitalization-sensitive) when serialized as a string
- TODO: little-endian? does base32 specify this?

Recommended lookup parameters:

- TODO: how many columns in rotation?
- k=3 (number of bits to filter on)
- TODO: Hamming distance
- TODO: 0.XYZ considered a loose match, 0.XYZ considered a close match

## Existing Libraries and Implementations

These don't necessarily implement the schemes above, but they do implement the
basic algorithms.

- [seomoz/simhash-cpp](https://github.com/seomoz/simhash-cpp) (C++): simhash,
  used in production for web de-dupe
- [simhash](https://github.com/JanX2/simhash) (C++): Google's simhash
  implementation (old?)
- [datasketch](https://ekzhu.github.io/datasketch/) (Python): minhash, Jaccard
  similarity, minhash-based indexing with Jaccard threshold, Top-K, or
  containment threshold.
- [sean-public/python-hashes](https://github.com/sean-public/python-hashes)
  (Python)
- [sing1ee/simhash-java](https://github.com/sing1ee/simhash-java) (Java)
- [vkandy/simhash-js](https://github.com/vkandy/simhash-js) (Javascript)

Other resources:

- [seomoz/simhash-db-py](https://github.com/seomoz/simhash-db-py)
- [codelibs/elasticsearch-minhash](https://github.com/codelibs/elasticsearch-minhash):
  plugin for a general-purpose search engine
- [bbalet/stopwords](https://github.com/bbalet/stopwords) (Golang): for a
  dozen+ languages. also does HTML stripping

## References

"Near-Duplicate Detection", Lecoc. 2015.
[https://moz.com/devblog/near-duplicate-detection/]()

> Moz blogpost. Good place to start.

"Charikar: Similarity Estimation Techniques from Rounding Algorithms", in
Proceedings of the thiry-fourth annual ACM symposium on Theory of computing,
ACM Press, 2002"

"Manku, Jain, Sarma: Detecting Near-Duplicates for Web Crawling". in
Proceedings of the 16th international conference on World Wide Web, ACM Press,
2007

> Google paper

"Identifying and Filtering Near-Duplicate Documents", Broder. 2000.

> AltaVista paper

"Near Duplicate Detection in an Academic Digital Library", Williams, Giles.
2013.
[http://www.personal.psu.edu/kiw5209/papers/2013/williams_doceng2013.pdf]()

> CiteseerX paper

"Probabilistic Near-Duplicate Detection Using Simhash", Sood and Loguinov.
2011.

"In Defense of MinHash over SimHash"
[http://proceedings.mlr.press/v33/shrivastava14.pdf]()

"Finding Similar Files in a Large File System", Manber. 1993.
[http://webglimpse.net/pubs/TR93-33.pdf]()

For a bibliography of older work, see
[https://github.com/JanX2/simhash/blob/master/paper/simHashBiblio.bib]().
