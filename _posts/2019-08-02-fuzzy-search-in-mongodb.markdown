---
layout: post
title:  "Fuzzy / Partial Text search in MongoDB using n-grams"
date:   2020-07-30 15:45:32 +0530
categories: mongodb
---
Level: `intermediate`

Pre-requisites: mongodb `CRUD` operations, `$text` operator in mongodb, `indexing` queries

### Search Techniques
Consider the following `the sneaky fox` JSON document:
{% highlight json %}
{
  "name": "sneaky fox",
  "description": "I'm a sneaky fox. Sneaked in your house. Sat on your couch. Ate your pasta. Deleted your browser history"
}
{% endhighlight%}



To really put this problem statement into perspective, let's look at the two types of search techniques:
- **Full Text Search** - Will return results that contain full and exact matches of the search query. For example, the sneaky fox document would be returned in the results only if we search for `sneaky` or `house` or `pasta`, but not for `pas`, `hou`, `ouse`, `hou`, `sne` and so on, so forth.
- **Fuzzy / Partial Text Search** - Will return results that contain partial matches of the search query. Now, the sneaky fox document would be returned in the results for all the search queries stated in the previous definition, including the *partial ones*.

MongoDB's default text search functionality matches entire words only, *i.e.* the full text search.
Now for the ones who really need a partial text search functionality, there are various ways to achieve this:
- Use a solution like Elastic search for all data reading operations. Elastic search, *umm* is a search server, and caches a copy of the data for faster retrievals and ***search***
- Use regular expressions to search, which are memory expensive, and on larger datasets would do more harm than good
- Or do a custom implementation of **Partial Text search**

### Search Example
Let's first walk through the example provided in the [Offical MongoDB docs][mongo_docs_search_link]

-   Run the following in your Mongo shell to insert the documents in database:
  {% highlight javascript %}
  db.stores.insert(
    [
      { _id: 1, name: "Java Hut", description: "Coffee and cakes" },
      { _id: 2, name: "Burger Buns", description: "Gourmet hamburgers" },
      { _id: 3, name: "Coffee Shop", description: "Just coffee" },
      { _id: 4, name: "Clothes Clothes Clothes", description: "Discount clothing" },
      { _id: 5, name: "Java Shopping", description: "Indonesian goods" }
    ]
  )

  /* Output */
  // BulkWriteResult({
  // 	"writeErrors" : [ ],
  // 	"writeConcernErrors" : [ ],
  // 	"nInserted" : 5,
  // 	"nUpserted" : 0,
  // 	"nMatched" : 0,
  // 	"nModified" : 0,
  // 	"nRemoved" : 0,
  // 	"upserted" : [ ]
  // })
  {% endhighlight %}
-   To use text search, we need to create a text index:
  {% highlight javascript %}
  db.stores.createIndex( { name: "text", description: "text" } )

  /* Output */
  // { 
  //     "createdCollectionAutomatically" : false, 
  //     "numIndexesBefore" : 1.0, 
  //     "numIndexesAfter" : 2.0, 
  //     "ok" : 1.0
  // }

  {% endhighlight %}
-    Now, let's try some search queries:
  {% highlight javascript %}
  db.stores.find( { $text: { $search: "java coffee" } } ) // full text
  /* Output */
  // { 
  //   "_id" : 3.0, 
  //   "name" : "Coffee Shop", 
  //   "description" : "Just coffee"
  // }
  // { 
  //   "_id" : 1.0, 
  //   "name" : "Java Hut", 
  //   "description" : "Coffee and cakes"
  // }
  // { 
  //   "_id" : 5.0, 
  //   "name" : "Java Shopping", 
  //   "description" : "Indonesian goods"
  // }

  db.stores.find( { $text: { $search: "ja" } } ) // partial text
  /* Output Empty */

  db.stores.find( { $text: { $search: "coff" } } ) // partial text
  /* Output Empty */
  {% endhighlight %}

As we followed through the examples, it is clear that partial text search does not work in MongoDB.

### n-grams
Enter `n-grams`, whose [Wikipedia][n_gram_wiki] definition states:
> an *n-gram* is a contiguous sequence of n items from a given sample of text or speech. The items can be phonemes, syllables, letters, words or base pairs according to the application.

Furthermore,
> Using Latin numerical prefixes, an n-gram of size 1 is referred to as a "unigram"; size 2 is a "bigram" (or, less commonly, a "digram"); size 3 is a "trigram". English cardinal numbers are sometimes used, e.g., "four-gram", "five-gram", and so on. In computational biology, a polymer or oligomer of a known size is called a k-mer instead of an n-gram, with specific names using Greek numerical prefixes such as "monomer", "dimer", "trimer", "tetramer", "pentamer", etc., or English cardinal numbers, "one-mer", "two-mer", "three-mer", etc.

For example,
- 1-grams (unigrams) of `coffee` would be `["c", "o", "f", "f", "e", "e"]`.
- 2-grams (bigrams) of `coffee` would be `["co", "of", "ff", "fe", "ee"]`.
- 3-grams (trigrams) of `coffee` would be `["cof", "off", "ffe", "fee"]`.

You can play with the concept in the embedded notebook or at this [NodeJS playground][runkit_endpoint]

I hope you are with me till yet. If not, scroll back up, and go through it again. We are going to jump into some code now.

### Building our N-gram generator ##

So, now we are trying to break the text itself into pieces so that the search algorithm will match these *n-grams* itself. Let's write a function to that:
- takes an array of objects as an argument
- takes an array of keys whose values will be the search space, *e.g.* `['name', 'description']` in our stores database
- takes a number as an argument for the minimum number of searchable characters, (we will set this as 2 by default)
- return the array with additional `searchText` field containing the n-grams

**Pause for a moment and think why the third argument is necessary here**
{% highlight javascript %}
function generateNGrams(arr, keys, nMinimum = 2) {
  let cache = {};
  for (const elem of arr) {
    let searchWords = new Set();
    for (const key of keys) {
      elem[key].toLowerCase().split(/[^\w]/).forEach(searchWords.add, searchWords);
    }
    gramSet = new Set();
    for (const word of searchWords) {
      if (typeof word === 'string' && word.length <= nMinimum) {
        gramSet.add(word);
      } else if (cache[word]) {
        cache[word].forEach(gramSet.add, gramSet);
      } else {
        wordGrams = new Set();
        for (let i = nMinimum; i < word.length; i += 1) {
          ngram(i)(word).forEach(wordGrams.add, wordGrams);
        }
        wordGrams.add(word);
        cache[word] = [...wordGrams];
        cache[word].forEach(gramSet.add, gramSet);
      }
    }
    elem.searchText = [...gramSet].join(' ');
  }
  cache = null;
  return arr;
}
{% endhighlight %}

Play around with the function [here][generator_playground].

### Code Snippet Explanation
1. We start looping through the array of objects `arr`
2. Then we loop through the array of keys which are to be included in the `searchText`
3. We add all the unique words of the search space to a Set
4. Next, we loop on the contents of the set to generate n-grams for each word in the Set and add them to another Set, *(say S2)* so as to avoid duplicates and save space
5. We now extract all the words from S2 in an array and concatenate into a string
6. End Loop in Step *2.*
7. End Loop in Step *1.*

### Testing our search implementation
Now let us update the documents in our database using the following queries so that they contain the `searchText`:
{% highlight javascript %}
db.stores.updateOne({ name: "Java Hut" }, { $set: { searchText: 'ja av va jav ava java hu ut hut co of ff fe ee cof off ffe fee coff offe ffee coffe offee coffee an nd and ca ak ke es cak ake kes cake akes cakes' } });

db.stores.updateOne({ name: "Burger Buns" }, { $set: { searchText: 'bu ur rg ge er bur urg rge ger burg urge rger burge urger burger un ns bun uns buns go ou rm me et gou our urm rme met gour ourm urme rmet gourm ourme urmet gourme ourmet gourmet ha am mb rs ham amb mbu ers hamb ambu mbur gers hambu ambur mburg rgers hambur amburg mburge urgers hamburg amburge mburger burgers hamburge amburger mburgers hamburger amburgers hamburgers' } });

db.stores.updateOne({ name: "Coffee Shop" }, { $set: { searchText: 'co of ff fe ee cof off ffe fee coff offe ffee coffe offee coffee sh ho op sho hop shop ju us st jus ust just' } });

db.stores.updateOne({ name: "Clothes Clothes Clothes" }, { $set: { searchText: 'cl lo ot th he es clo lot oth the hes clot loth othe thes cloth lothe othes clothe lothes clothes di is sc co ou un nt dis isc sco cou oun unt disc isco scou coun ount disco iscou scoun count discou iscoun scount discoun iscount discount hi in ng thi hin ing othi thin hing lothi othin thing clothi lothin othing clothin lothing clothing' } });

db.stores.updateOne({ name: "Java Shopping" }, { $set: { searchText: 'ja av va jav ava java sh ho op pp pi in ng sho hop opp ppi pin ing shop hopp oppi ppin ping shopp hoppi oppin pping shoppi hoppin opping shoppin hopping shopping nd do on ne es si ia an ind ndo don one nes esi sia ian indo ndon done ones nesi esia sian indon ndone dones onesi nesia esian indone ndones donesi onesia nesian indones ndonesi donesia onesian indonesi ndonesia donesian indonesia ndonesian indonesian go oo od ds goo ood ods good oods goods' } });
{% endhighlight %}

Before we go and try to search through the documents, it is important to drop the previous index that was created and create a new one on the `searchText` field.
1. Get all indexes information by executing `db.stores.getIndexes()`, look for the index which has weights for `name` and `description` which we created initially and copy its name, in my case it was **name_text_description_text**.
2. Drop the index and create a new index on the field `searchText`:
{% highlight javascript %}
db.stores.dropIndex('name_text_description_text'); // Remember to change the name of the index as per your results

/* Output */
// { 
//     "nIndexesWas" : 2.0, 
//     "ok" : 1.0
// }

db.stores.createIndex({ searchText: "text" });

/* Output */
// { 
//     "createdCollectionAutomatically" : false, 
//     "numIndexesBefore" : 1.0, 
//     "numIndexesAfter" : 2.0, 
//     "ok" : 1.0
// }
{% endhighlight %}

Let us try the partial text search queries we attempted earlier, also add the sweet projection `{ searchText: 0 }` for a tidier output:
{% highlight javascript %}
db.stores.find({ $text: { $search: "ja" }}, { searchText: 0 });

/* Output */
// { 
//     "_id" : 1.0, 
//     "name" : "Java Hut", 
//     "description" : "Coffee and cakes"
// }
// { 
//     "_id" : 5.0, 
//     "name" : "Java Shopping", 
//     "description" : "Indonesian goods"
// }

db.stores.find({ $text: { $search: "coff" } }, { searchText: 0 });

/* Output */
// { 
//     "_id" : 3.0, 
//     "name" : "Coffee Shop", 
//     "description" : "Just coffee"
// }
// { 
//     "_id" : 1.0, 
//     "name" : "Java Hut", 
//     "description" : "Coffee and cakes"
// }
{% endhighlight %}

That's it! We have a working partial text search implementation.

### Epilogue
Following through, you must have realised that:
- this will be heavy on the storage requirements
- the n-gram generation and processing logic can be implemented as a middleware (pre-save and pre-update hooks) in your ORM / driver code
- will introduce additional latencies in write operations if implemented as a middleware
- will be better than setting up another search server though

Another gotcha is the order of the search results, which can be improved on playing with the n-gram generation logic in [the code snippet](#building-our-n-gram-generator) that we constructed earlier.


[mongo_docs_search_link]: https://docs.mongodb.com/manual/text-search/
[n_gram_wiki]: https://en.wikipedia.org/wiki/N-gram
[runkit_endpoint]: https://runkit.com/abhishek-alai/n-gram-playground
[generator_playground]: https://runkit.com/abhishek-alai/n-gram-generator-for-array-of-objects
