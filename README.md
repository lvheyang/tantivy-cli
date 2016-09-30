[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)


`tantivy-cli` is the project hosting the command line interface for [tantivy](https://github.com/fulmicoton/tantivy), a search engine project.


# Tutorial: Indexing Wikipedia with Tantivy CLI

## Introduction

In this tutorial, we will create a brand new index with the articles of English wikipedia in it.

## Installing the tantivy CLI.

There are a couple ways to install `tantivy-cli`.

If you are a Rust programmer, you probably have `cargo` installed and you can just
run `cargo install tantivy-cli`.

Alternatively, if you are on 64-bit Linux, you can directly download a
static binary:  [binaries/linux_x86_64/](http://fulmicoton.com/tantivy-files/binaries/linux_x86_64/tantivy),
and save it in a directory on your system's `PATH`.




## Creating the index:  `new`
 
Let's create a directory in which your index will be stored.

```bash
    # create the directory
    mkdir wikipedia-index
```


We will now initialize the index and create its schema.
The [schema](http://fulmicoton.com/tantivy/tantivy/schema/index.html) defines
the list of your fields, and for each field:
- its name 
- its type, currently `u32` or `str`
- how it should be indexed.

You can find more information about the latter on 
[tantivy's schema documentation page](http://fulmicoton.com/tantivy/tantivy/schema/index.html

In our case, our documents will contain
* a title
* a body 
* a url

We want the title and the body to be tokenized and indexed. We also want 
to add the term frequency and term positions to our index.
(To be honest, phrase queries are not yet implemented in tantivy,
so the positions won't be really useful in this tutorial.)

Running `tantivy new` will start a wizard that will help you
define the schema of the new index.

Like all the other commands of `tantivy`, you will have to 
pass it your index directory via the `-i` or `--index`
parameter as follows:


```bash
    tantivy new -i wikipedia-index
```



Answer the questions as follows:

```none

    Creating new index 
    Let's define its schema! 



    New field name  ? title
    Text or unsigned 32-bit integer (T/I) ? T
    Should the field be stored (Y/N) ? Y
    Should the field be indexed (Y/N) ? Y
    Should the field be tokenized (Y/N) ? Y
    Should the term frequencies (per doc) be in the index (Y/N) ? Y
    Should the term positions (per doc) be in the index (Y/N) ? Y
    Add another field (Y/N) ? Y



    New field name  ? body
    Text or unsigned 32-bit integer (T/I) ? T
    Should the field be stored (Y/N) ? Y
    Should the field be indexed (Y/N) ? Y
    Should the field be tokenized (Y/N) ? Y
    Should the term frequencies (per doc) be in the index (Y/N) ? Y
    Should the term positions (per doc) be in the index (Y/N) ? Y
    Add another field (Y/N) ? Y



    New field name  ? url
    Text or unsigned 32-bit integer (T/I) ? T
    Should the field be stored (Y/N) ? Y
    Should the field be indexed (Y/N) ? N
    Add another field (Y/N) ? N

    [
    {
        "name": "title",
        "type": "text",
        "options": {
            "indexing": "position",
            "stored": true
        }
    },
    {
        "name": "body",
        "type": "text",
        "options": {
            "indexing": "position",
            "stored": true
        }
    },
    {
        "name": "url",
        "type": "text",
        "options": {
            "indexing": "unindexed",
            "stored": true
        }
    }
    ]


```

After the wizard has finished, a `meta.json` should exist in `wikipedia-index/meta.json`.
It is a fairly human readable JSON, so you can check its content.

It contains two sections:
- segments (currently empty, but we will change that soon)
- schema 


 

# Indexing the document: `index`


Tantivy's `index` command offers a way to index a json file.
The file must contain one JSON object per line.
The structure of this JSON object must match that of our schema definition.

```json
    {"body": "some text", "title": "some title", "url": "http://somedomain.com"}
```

For this tutorial, you can download a corpus with the 5 million+ English Wikipedia articles in the right format here: [wiki-articles.json (2.34 GB)](https://www.dropbox.com/s/wwnfnu441w1ec9p/wiki-articles.json.bz2?dl=0).
Make sure to decompress the file

```bash
    bunzip2 wiki-articles.json.bz2
```

If you are in a rush you can [download 100 articles in the right format here](http://fulmicoton.com/tantivy-files/wiki-articles-1000.json).

The `index` command will index your document.
By default it will use as many threads as there are cores on your machine.
You can change the number of threads by passing it the `-t` parameter.

On my computer (8 core Xeon(R) CPU X3450  @ 2.67GHz), it will take around 6 minutes.

```
    cat wiki-articles.json | tantivy index -i ./wikipedia-index
```

While it is indexing, you can peek at the index directory
to check what is happening.

```bash
    ls ./wikipedia-index
```

If you indexed the 5 million articles, you should see a lot of new files, all with the following format:

The main file is `meta.json`.

Our index is in fact divided in segments. Each segment acts as an individual smaller index.
Its name is simply a uuid. 




# Serve the search index: `serve`

Tantivy's cli also embeds a search server.
You can run it with the following command.

```
    tantivy serve -i wikipedia-index
```

By default, it will serve on port `3000`.

You can search for the top 20 most relevant documents for the query `Barack Obama` by accessing
the following [url](http://localhost:3000/api/?q=barack+obama&explain=true&nhits=20) in your browser

    http://localhost:3000/api/?q=barack+obama&explain=true&nhits=20


# Optimizing the index: `merge`

Each of tantivy's indexer threads closes a new segment every 100K documents (this is completely arbitrary at the moment).
You should currently have more than 50 segments in your dictionary.

Having that many segments hurts your query performance (well, mostly the fast ones).
Tantivy merge will merge your segments into one. 

```
    tantivy merge -i ./wikipedia-index
```

(The command takes around 7 minutes on my computer)

Note that your files are still there even after having run the command.
However, `meta.json` only lists one of the segments.
You will still need to remove the files manually.
