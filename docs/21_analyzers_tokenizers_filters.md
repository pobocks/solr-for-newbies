## Analyzers, Tokenizers, and Filters  

The `indexAnalyzer` section defines the transformations to perform *as the data is indexed* in Solr and `queryAnalyzer` defines transformations to perform *as we query for data* out of Solr. It's important to notice that the output of the `indexAnalyzer` affects the terms *indexed*, but not the value *stored*. The [Solr Reference Guide](https://lucene.apache.org/solr/guide/7_0/analyzers.html) says:

    The output of an Analyzer affects the terms indexed in a given field
    (and the terms used when parsing queries against those fields) but
    it has no impact on the stored value for the fields. For example:
    an analyzer might split "Brown Cow" into two indexed terms "brown"
    and "cow", but the stored value will still be a single String: "Brown Cow"

When a value is *indexed* for a particular field the value is first passed to the `tokenizer` and then to the `filters` defined in the `indexAnalyzer` section for that field type. Similarly, when we *query* for a value in a given field the value is first processed by the `tokenizer` and then by the `filters` defined in the `queryAnalyzer` section for that field.

If we look again at the definition for the `text_general` field type we'll notice that "stop words" (i.e. words to be ignored) are handled at index and query time (notice the `StopFilterFactory` filter appears in the `indexAnalyzer` and the `queryAnalyzer` sections.) However, notice that "synonyms" will only be applied at query time since the filter `SynonymGraphFilter` only appears on the `queryAnalyzer` section.

We can customize field type definitions to use different filters and tokenizers via the Schema API which we will discuss later on this tutorial.

### Tokenizers

For most purposes we can think of a tokenizer as something that splits a given text into individual tokens or words. The [Solr Reference Guide](https://lucene.apache.org/solr/guide/7_0/tokenizers.html) defines Tokenizers as follows:

    Tokenizers are responsible for breaking
    field data into lexical units, or tokens.

For example if we give the text "hello world" to a tokenizer it might split the text into two tokens like "hello" and "word".

Solr comes with several [built-in tokenizers](https://lucene.apache.org/solr/guide/7_0/tokenizers.html) that handle a variety of data. For example if we expect a field to have information about a person's name the [Standard Tokenizer](https://lucene.apache.org/solr/guide/7_0/tokenizers.html#standard-tokenizer) might be appropriated for it. However, for a field that contains e-mail addresses the [UAX29 URL Email Tokenizer](https://lucene.apache.org/solr/guide/7_0/tokenizers.html#uax29-url-email-tokenizer) might be a better option.

I believe you can only have [one tokenizer per analyzer](https://lucene.apache.org/solr/guide/7_0/tokenizers.html)


### Filters

Whereas a `tokenizer` takes a string of text and produces a set of tokens, a `filter` takes a set of tokens, process them, and produces a different set of tokens. The [Solr Reference Guide](https://lucene.apache.org/solr/guide/7_0/about-filters.html) says that

    in most cases a filter looks at each token in the stream sequentially
    and decides whether to pass it along, replace it or discard it.

Notice that unlike tokenizers, whose job is to split text into tokens, the job of filters is a bit more complex since they might replace the token with a new one or discard it altogether.

Solr comes with many [built-in Filters](https://lucene.apache.org/solr/guide/7_0/filter-descriptions.html) that we can use to perform useful transformations. For example the ASCII Folding Filter converts non-ASCII characters to their ASCII equivalent (e.g. "México" is converted to "Mexico"). Likewise the English Possessive Filter removes singular possessives (trailing 's) from words. Another useful filter is the Porter Stem Filter that calculates word stems using English language rules (e.g. both "jumping" and "jumped" will be reduced to "jump".)


### Putting it all together

If we look at the tokenizer and filters defined for the `text_general` field type at *index time* we would see that we are tokenizing with the `StandardTokenizer` and filtering with the `StopFilter` and the `LowerCaseFilter` filters.

Notice that the at *query time* an extra filter, the `SynonymGraphFilter`, is applied for this field type.

If we *index* the text "The television is broken!" the tokenizer and filters defined in the `indexAnalyzer` will transform this text to four tokens: "the", "television", "is", and "broken". Notice how the tokens were lowercased ("The" became "the") and the exclamation sign was dropped.

Likewise, if we *query* for the text "The TV is broken!" the tokenizer and filters defined in the `queryAnalyzer` will convert the text to the following tokens: "the", "television", "televisions", "tvs", "tv", "is", and "broken". Notice that an additional transformation was done to this text, namely, the word "TV" was expanded to four synonyms. This is because the `queryAnalyzer` uses the `SynonymGraphFilter` and a standard Solr configuration comes with those four synonyms predefined in the `synonyms.txt` file.

The "Analysis" option in the [Solr Admin](http://localhost:8983/solr/#/bibdata/analysis) tool is a great way to see a particular text is either indexed or queried by Solr *depending on the field type*. Here is a few examples to try:

* Enter "The quick brown fox jumps over the lazy dog" in the "Field Value (*index*)", select `string` as the field type and see how is indexed. Then select `text_general` and see how is indexed. Lastly, select `text_en` and see how is indexed.

* With the text still on the "Field Value (*index*)" text box, enter "The quick brown fox jumps over the LAZY dog" on the "Field Value (*query*)" and try the different field types (`string/text_general/text_en`) again to see how each of them shows different matches.

* Try changing the text on the "Field Value (*query*)" text box to "The quick brown foxes jumped over the LAZY dogs". Try again with the different field types to see the matches.

* Now enter "The TV is broken!" on the "Field Value (*index*)" text box, blank the "Field Value (*query*)" text box, select `text_general`, and see how the value is indexed. Then do the reverse, blank the indexed value and enter "The TV is broken!" on the "Field Value (*query*)" text box and notice synonyms being applied.

* Now enter "The TV is broken!" on the "Field Value (*index*)" text box and "the television is broken" on the "Field Value (*query*)". Notice how they are matched because the use of synonyms applied for `text_general` fields.

Quiz: When we tested the text "The television is broken" with the `text_general` field type we probably expected the word "the" to be dropped since it's a stop word in the English language. Can you guess why it was not dropped? Hint: try the same text but with the `text_en` field type instead and see what happens.
