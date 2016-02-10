title: Making an autocomplete search that doesn't suck
date: Tue Feb 09 2016 15:25:43 GMT-0600 (CST)
author: Josh Duff

We index emails related to (as of February 2016) about 27m domains, 4m brands, and 50k companies.  It's important that our customers be able to easily bring up the domain or brand they care about.

We use search dropdowns with results coming from ElasticSearch in the back-end.  In the early days of our app, people would have difficulties finding some domains or company names - you couldn't easily find companies with periods or dashes in their name, and short domain names or some awkward subdomains.

This is how we made our autocomplete searches excellent.

# Searching

So we have indexes of domain names and company names.  There are two ways to determine what results you get back for a user's search string: filtering down to matching results, and then ordering the remaining list so that the most relevant are at the top.

<img src="content/image/brand-autocomplete.png" width="195" height="203">

See examples of our ElasticSearch schemas [here](https://gist.github.com/TehShrike/4dbc9fbad7243faae615).  As you can see, we give ourselves several different analyzed strings to use for searching.

## ngram

We originally used ElasticSearch's [standard](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-tokenizer.html) tokenizer for company and brand names, like you're supposed to for English words.  It didn't work well at all for us - people searching for "GlobalMarket Group" expect to be able to type "market" and find what they're looking for.

What people expect, at least in our autocomplete dropdowns, is that their strings be able to find matches anywhere in the company/brand/domain name.  To accomplish this we use an [ngram](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-ngram-tokenizer.html) tokenizer - the word "group" would be indexed with the tokens "gr", "ro", "ou", "up", "gro", "rou", "oup", "grou", "roup", and "group".

# Ordering

Originally we were doing pretty liberal matching of search terms, and would try to filter out the irrelevant ones by ordering based on match strength.  This did not work well.

For example: searching for "amazon email" would match email.amazon.com, but it would also match amazon.com.  We would order by the ElasticSearch score in order to get email.amazon.com to rank higher than amazon.com for that search string.

This did not work well.  "email" matches tens of thousands of domains in our index, and it was always frustrating to see irrelevant results like "Groupon, Inc." show up when you searched for "Amazon Inc".

Our search results improved drastically when we realized that we had a proxy for customer relevance - we know how much email is being sent out by each domain!

Once we started keeping our domain/company/brand indexes up to date with recent email volume, filtered the results to contain every search term the user had entered, and sorted those results by email volume, our results became instantly more useful.

We do still do some other intelligent ordering for the cases where people are searching for new entries or entries without any email volume:

```
BoolQueryBuilder groupQuery = QueryBuilders.boolQuery();
query = query.toLowerCase();

groupQuery.must(makeQuery(tokenize(query)).boost(0));

groupQuery.should(QueryBuilders.termQuery("ngramName", stripWhitespace(query))); // raise the score slightly when an ngram contains all characters in the right order

groupQuery.should(QueryBuilders.prefixQuery("name", query).boost(80)); // raise the score when the name starts with the query string
groupQuery.should(QueryBuilders.termQuery("name", query).boost(100)); // raise the score when the name matches the query string exactly

groupQuery.should(QueryBuilders.prefixQuery("alphanumName", stripNonAlphanum(query)).boost(40));
groupQuery.should(QueryBuilders.termQuery("alphanumName", stripNonAlphanum(query)).boost(70));

```

# Matching

As you can see from the code snippet above, there is only one query that result sets `must` match, hidden behind the enigmatic `tokenize` and `makeQuery` methods.

Most of what it does is make sure our query best fits our ngram analyszed data.  Since we only index ngrams between 2 and 10 characters, by default searching for "professional network" would only return matches for "network" - "professional" has 12 characters, and would not match any indexed tokens.

So we emulate what our `ngram_2_10` tokenizer and `alphanum` filters do in ElasticSearch (splitting into tokens on whitespace, filtering out special characters), and then break up any long words into tokens that we can actually search by.

If someone searches for "professional", we actually run a query that filters results matching three 10-character tokens: "profession", "rofessiona", and "ofessional".

All of the other optional queries are there for scoring purposes only, to hopefully make your results make more sense when you're searching for obscure domains without much data.

<img src="content/image/domain-autocomplete.png" width="206" height="200">

# Life lessons

So this is how we made our dropdown searches pretty useful:

1. toss our best metric for "useful to users" into the index and sort results by that
2. use ngram matching to find matches at any point in the word
	a. manually tokenize searches before sending them to ElasticSearch so they will find ngram matches
3. add in some helper boosts to make edge-case searching more likely to be useful
