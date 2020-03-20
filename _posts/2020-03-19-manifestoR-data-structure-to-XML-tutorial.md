---
layout: post
title: "Enhancing {manifestoR} data objects: A short tutorial using the Swiss manifesto corpus."
author: "Hauke Licht"
date: "2019-03-07"
tags: [data wrangling, manifesto data, Manifestos Project, R, manifestoR, manifestoEnhanceR, XML]
excerpt: "In this post, I introduce my `manifestoEnhanceR` package and apply it (i) to recover document structure from Swiss parties' manifesto as provided by the Comparative Manifesto Project throught their `manifestoR` package, and (ii) to convert manifestR obejcts to XML documents."
mathjax: true
comments: false
reading_time:       true
words_per_minute:   200
---



In this short tutorial, I show how functions provided by my `manifestoEnhanceR` package can be used to convert a large batch of manifesto documents into [XML](https://www.w3schools.com/xml/xml_whatis.asp) documents enriched with document structure (headers, sentences, quasi-sentence nesting) and meta data (e.g, manifesto title or quasi-sentence CMP code).

Before getting started, a short note on why one would want to convert `manifestoR` data object to XML files seems to be in order, but you can skip this note and fo directly to section <a href="#the-manifestoenhancer-package-converting-swiss-manifestos-to-xml-documents">2</a>.

## Where all began 

If you look at a manifesto in its print or online version, it starts with a title, 
then comes an introduction or summary of key points, following by a number of chapters that comprise many more sentences.

This document structure is not reflected in the format in which the Manifesto Project makes its data available:
The CMP project distributes its annotated manifesto data in a tabular format at the "quasi-sentence" level (full sentences or parts thereof).   

While this is not a problem if you are merely interested in their codings, it can be a problem in case you want to use the manifesto texts for other puporses, such as topic modeling or for training a word embedding model.[^1]
On a more general note, I think that data fromats should be back and forward compatible, because this allows many ways of aggregating them.
This seems to be particularly important in the case of text data, where different ways of analyzing it perform differently on different levels of analyses (the whole document, chapters, paragraph, sentences, words).

[^1]: For these latter puproses, quasi-sentences are not an ideal unit of (dis)aggregation. Sentences, paragraphs or chapters seem more appropriate.

### A ray of hope 

Now, it turns out that while paragraph structure cannot be recovered from the CMP'S tabular data, we can infer sentences and chapter structure by combining the text of quasi-sentences and their codings. 

I learned about this on Twitter, when [asking for help](https://twitter.com/hauke_licht/status/1234781054780788738) with the data wrangling problem discussed above:

My friend Anna gave me a first great hint: one could use the CMP code "H" to detect chapter headings and thereby map quasi-sentences m:1 to chapters.

{% twitter https://twitter.com/annaadendorf/status/1234831169700667394 %}

The CMP colleagues followed-up on this, suggesting that in addition to using the "H" codes, 'one could also look for entries coded as NA in between entries with a proper cmp_code.'

{% twitter 'https://twitter.com/manifesto_proj/status/1235547623244824582' %}

## The `manifestoEnhanceR` package: Converting swiss manifestos to XML documents

Hence I wrote an R package to infer document structure from quasi-sentence level features in `manifestoR` data objects, and to convert the enhanced data to XML documents: `manifestoEnhanceR`.
When I write "enhancing" `manifestoR` objects, I mean reconstructing a manifestos document structure.

In what follows, I use functions provided by the `manifestoEnhanceR` package to convert the entire Swiss manifesto corpus to XML documents.
You'll be guided through the process of 

1. querying manifestos from the Manifesto Project API, 
2. covnerting the `manifestoR` package's custom data objects into a tidy data format (i.e., a data frame), 
3. enhancing manifesto data frames with text-level information (e.g., quasi-sentence and chapter numbers), and
4. converting the enhanced manifesto data frames to XML documents.

### Step 0: Loading required pacakges 

Load all required pacakges.

The `manifestoEnhanceR` package is currently only available via GitHub: https://github.com/haukelicht/manifestoEnhanceR


{% highlight r %}
### if not installed: `install.packages("manifestoR")`
library(manifestoR)
### if not installed: `devtools::install_github("haukelicht/manifestoEnhanceR)`
library(manifestoEnhanceR)
library(magrittr)
library(dplyr)
library(tidyr)
library(purrr)
library(xml2)
library(rvest)
{% endhighlight %}

### Step 1: Load manifesto data 


{% highlight r %}
### set Manifesto Project API key (see `?mp_setapikey`)
mp_setapikey(file.path(Sys.getenv("SECRETS_PATH"), "manifesto_apikey.txt")) 
{% endhighlight %}


{% highlight r %}
### get CMP party-year data
cmp <- mp_maindataset(south_america = FALSE)
{% endhighlight %}



{% highlight text %}
## Connecting to Manifesto Project DB API... 
## Connecting to Manifesto Project DB API... corpus version: 2019-2
{% endhighlight %}



{% highlight r %}
### subset to Swiss configurations
ch_configs <- cmp %>% 
  filter(countryname == "Switzerland") %>% 
  select(1:10)

### get Swiss manifestos
ch <- mp_corpus(countryname == "Switzerland", cache = TRUE)
{% endhighlight %}



{% highlight text %}
## Connecting to Manifesto Project DB API... 
## Connecting to Manifesto Project DB API... corpus version: 2019-2 
## Connecting to Manifesto Project DB API... corpus version: 2019-2 
## Connecting to Manifesto Project DB API... corpus version: 2019-2
{% endhighlight %}

### Step 2: Conversion to tidy data

The object `ch` is a "ManifestoCorpus" object, which is essentially a list "ManifestoDocument" objects with some additional fancy attributes.



{% highlight r %}
class(ch)
{% endhighlight %}



{% highlight text %}
## [1] "ManifestoCorpus" "VCorpus"         "Corpus"
{% endhighlight %}



{% highlight r %}
str(head(ch, 2), 1)
{% endhighlight %}



{% highlight text %}
## List of 2
##  $ 43110_198710:List of 2
##  $ 43110_199110:List of 2
##  - attr(*, "class")= chr [1:3] "ManifestoCorpus" "VCorpus" "Corpus"
{% endhighlight %}



{% highlight r %}
lapply(head(ch, 2), class)
{% endhighlight %}



{% highlight text %}
## $`43110_198710`
## [1] "ManifestoDocument" "PlainTextDocument" "TextDocument"     
## 
## $`43110_199110`
## [1] "ManifestoDocument" "PlainTextDocument" "TextDocument"
{% endhighlight %}

We can tidy this up (i.e., convert it to a long tibble) by applying the first `manifestoEnhanceR` function we encounter in this tutorial: `as_tibble.ManifestoCorpus`.[^2]

[^2]: The dot separating the "as_tibble" and "ManifestoCorpus" indicates that this is the "as_tibble" method for "ManifestoCorpus" instances/object. You can read more about methods in the  [R Documentation](https://stat.ethz.ch/R-manual/R-patched/library/methods/html/Methods_Details.html "R Documentation: General Information on Methods") and the amazing [*Advanced R*](http://adv-r.had.co.nz/S3.html "Advanced R: The S3 object system")




{% highlight r %}
man_dfs <- as_tibble(ch)
glimpse(man_dfs)
{% endhighlight %}



{% highlight text %}
## Observations: 94
## Variables: 17
## $ manifesto_id                <chr> "43110_198710", "43110_199110", "43110_19…
## $ party                       <dbl> 43110, 43110, 43110, 43110, 43110, 43110,…
## $ date                        <dbl> 198710, 199110, 199510, 199910, 200310, 2…
## $ language                    <chr> "german", "german", "german", "german", "…
## $ source                      <chr> "CEMP", "CEMP", "CEMP", "MARPOR", "MARPOR…
## $ has_eu_code                 <lgl> FALSE, FALSE, FALSE, FALSE, FALSE, FALSE,…
## $ is_primary_doc              <lgl> TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, TRUE,…
## $ may_contradict_core_dataset <lgl> FALSE, FALSE, FALSE, FALSE, FALSE, FALSE,…
## $ md5sum_text                 <chr> "b847c70ef9be9933216708916f3583ff", "24b4…
## $ url_original                <chr> NA, NA, NA, "/down/originals/43110_1999.p…
## $ md5sum_original             <chr> NA, NA, NA, "CURRENTLY_UNAVAILABLE", "CUR…
## $ annotations                 <lgl> FALSE, FALSE, FALSE, TRUE, TRUE, TRUE, TR…
## $ handbook                    <chr> NA, NA, NA, "1", NA, "4", "4", "5", "4", …
## $ is_copy_of                  <lgl> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, N…
## $ title                       <chr> "Plattform des Grünen Bündnisses", "Progr…
## $ id                          <chr> "43110_198710", "43110_199110", "43110_19…
## $ data                        <list> [<tbl_df[1 x 3]>, <tbl_df[1 x 3]>, <tbl_…
{% endhighlight %}

`man_dfs` is a `tibble` (a [fancy dataframe](https://cran.r-project.org/web/packages/tibble/vignettes/tibble.html "Tibbles")) with a [list-column](https://jennybc.github.io/purrr-tutorial/ls13_list-columns.html "List colums") called "data".
For each row, column data contains a manifesto tibble. 

To this manifesto-level tibble, we can left-join manifesto- and party-level data:


{% highlight r %}
man_sents <- man_dfs %>% 
  left_join(
    ch_configs %>% 
      transmute(
        manifesto_id = paste0(party, "_", date)
        , party
        , partyname
        , partyabbrev
      ) %>% 
      unique()
  )
{% endhighlight %}

### Step 3: Define some test cases

To illustrate the other `manifestoEnhanceR` functions, we take four manifestos from the above corpus, each with some particular features (as commented below).

{% highlight r %}
test1_df <- unnest(filter(man_sents, manifesto_id == "43120_201110"), data)
test2_df <- unnest(filter(man_sents, manifesto_id == "43110_198710"), data)
test3_df <- unnest(filter(man_sents, manifesto_id == "43110_199910"), data)
test4_df <- unnest(filter(man_sents, manifesto_id == "43120_201510"), data)
{% endhighlight %}

- `test2_df` has just one row, and the single cell of column "text" contains the entire, uncoded text of the manifesto. This is typical for very old manifestos or manifestos of very small parties, where the Manifesto Project team assigned no expert to split the manifesto in quasi-sentences and code it. We say that this manifesto has *not* been "annotated".
- All other test cases have multiple rows. Each row corresponds to one coded quasi sentence. Codes are recorded in column "cmp_code".
- However, in contrast to `test3_df`, `test2_df` and `test_df` have only numerical codes, that is, no "H" code indicating document headers and titles.
- But `test3_df` and `test4_df` have both titles, but in the former case which rows hold the manifesto title needs to be inferred from the ordering of text lines.

This sounds confusing? It is. But no worries. Function `enhance_manifesto_df()` handles all these intricacies for you as shown below.

### Step 4: Enhance manifesto data frames

`enhance_manifesto_df()` returns a `manifesto.df` object that inherits from the input `tibble`, and simply adds four columns indicating document structure:

- "qs_nr" (running quasi-sentence counter),
- "sent_nr" (running sentence counter),
- "role" (indicator, here "qs" for all rows), and
- "bloc_nr" (enumerates consecutive rows by "role").
  
In addition, the returned `manifesto.df` obejct has two additional attributes:

- "annotated": indicates wehtehr or not the input o has been annotated/coded by CMP experts.
- "extra_cols": names of columns added by enhancing the ta frame.

By enhancing manifesto data frames, we add bloc, sentence and quasi-sentence (qs) counters, as well as a role indicators ("qs", "header", "title", or "meta").

As one natrual sentence may contain multiple quasi-sentences, the latter map m:1 to the former.
A bloc, in turn, is a number of consecutive rolws that all have the same "role".
The following rows are defined:

- "qs": A quasi-sentence
- "title": The manifesto title. This is/are the first row(s) with CMP code "H" or `NA`.
- "header": A chapter header. Rows after title (if any) with CMP code "H" or `NA`.
- "meta": In annotated manifestos containing "H" codes (e.g. test case 4), the row(s) between "title" and the first "header" rows.

This information is important if you want to count the number of chapters in a manifesto (= No. headers), or print only the title (if exists).
And as noted above, having at hand this information, one could easily aggregate quasi-sentences at the sentence or chapter level while omitting title and other meta text.

Let's first enhance the data of test case 1 and inspect the results

{% highlight r %}
### enhance test case 1
test1_res <- enhance_manifesto_df(test1_df)
class(test1_res)
{% endhighlight %}



{% highlight text %}
## [1] "manifesto.df" "tbl_df"       "tbl"          "data.frame"
{% endhighlight %}



{% highlight r %}
attr(test1_res, "annotated")
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
attr(test1_res, "extra_cols")
{% endhighlight %}



{% highlight text %}
## [1] "role"    "qs_nr"   "sent_nr" "bloc_nr"
{% endhighlight %}



{% highlight r %}
"title" %in% test1_res$role
{% endhighlight %}



{% highlight text %}
## [1] FALSE
{% endhighlight %}

As you can see, 

- the result is a "manifesto.df" object,
- it has both an "annotated" and an "extra_cols" attribute, 
- the "annotated" indicates that input manifesto data is annotated, but
- no title line(s) could be detected from the input manifesto. 

As already discussed above, this stands in contrast to test case 3

{% highlight r %}
### enhance test case 3
test3_res <- enhance_manifesto_df(test3_df)
"title" %in% test3_res$role
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
print(test3_res$text[test3_res$role == "title"])
{% endhighlight %}



{% highlight text %}
## [1] "Für eine zukunftsfähige Schweiz" "Wahlplattform 1999"
{% endhighlight %}

As the below table shows, we cannot detect title lines because the manifesto text starts with uncoded quasi-sentences.
If you scroll through the excerpt of the resulting data frame (only columns one and 17-25, first six rows are shown), you can also see how quasi-sentences are nested in sentences, and how they are bound my header and/or title lines.
Quasi-sentence 1 and 2, for example, are nested in the first sentence.

<table class="table" style="width: auto !important; margin-left: auto; margin-right: auto;">
<caption>Enhanced test case 1 (rows 1-6)</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> manifesto_id </th>
   <th style="text-align:left;"> text </th>
   <th style="text-align:left;"> cmp_code </th>
   <th style="text-align:left;"> role </th>
   <th style="text-align:right;"> qs_nr </th>
   <th style="text-align:right;"> sent_nr </th>
   <th style="text-align:right;"> bloc_nr </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> 43110_199910 </td>
   <td style="text-align:left;"> Für eine zukunftsfähige Schweiz </td>
   <td style="text-align:left;"> NA </td>
   <td style="text-align:left;"> title </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 43110_199910 </td>
   <td style="text-align:left;"> Wahlplattform 1999 </td>
   <td style="text-align:left;"> NA </td>
   <td style="text-align:left;"> title </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 43110_199910 </td>
   <td style="text-align:left;"> Die Grünen freuen sich, eine Wahlplattform für eine ökologische, soziale und weltoffene Schweiz </td>
   <td style="text-align:left;"> 501 </td>
   <td style="text-align:left;"> qs </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 43110_199910 </td>
   <td style="text-align:left;"> - kurz eine zukunftsfähige Schweiz zu präsentieren. </td>
   <td style="text-align:left;"> 601 </td>
   <td style="text-align:left;"> qs </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
  </tr>
</tbody>
</table>

The first header (chapter) occurs in text line 40.

<table class="table" style="width: auto !important; margin-left: auto; margin-right: auto;">
<caption>Enhanced test case 1 (rows 38-42)</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> manifesto_id </th>
   <th style="text-align:left;"> text </th>
   <th style="text-align:left;"> cmp_code </th>
   <th style="text-align:left;"> role </th>
   <th style="text-align:right;"> qs_nr </th>
   <th style="text-align:right;"> sent_nr </th>
   <th style="text-align:right;"> bloc_nr </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> 43110_199910 </td>
   <td style="text-align:left;"> Viele Reformen begannen mit einem grünen Vorstoss und endeten - oft Jahre danach - mit einem Sieg in der Volksabstimmung. </td>
   <td style="text-align:left;"> 305 </td>
   <td style="text-align:left;"> qs </td>
   <td style="text-align:right;"> 36 </td>
   <td style="text-align:right;"> 29 </td>
   <td style="text-align:right;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 43110_199910 </td>
   <td style="text-align:left;"> Wirkliche Reformen gibt es nur mit uns. </td>
   <td style="text-align:left;"> 305 </td>
   <td style="text-align:left;"> qs </td>
   <td style="text-align:right;"> 37 </td>
   <td style="text-align:right;"> 30 </td>
   <td style="text-align:right;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 43110_199910 </td>
   <td style="text-align:left;"> Mit dem Beitritt zur EU und zur UNO Verantwortung übernehmen und dort mitbestimmen, wo Entscheide gefällt werden </td>
   <td style="text-align:left;"> NA </td>
   <td style="text-align:left;"> header </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 2 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 43110_199910 </td>
   <td style="text-align:left;"> Die Grünen stehen der europäischen Integration positiv gegenüber. </td>
   <td style="text-align:left;"> 108 </td>
   <td style="text-align:left;"> qs </td>
   <td style="text-align:right;"> 38 </td>
   <td style="text-align:right;"> 31 </td>
   <td style="text-align:right;"> 3 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 43110_199910 </td>
   <td style="text-align:left;"> Europäische Zusammenarbeit ist notwendig, da Ökologie und Soziales auch grenzüberschreitende Lösungen erfordern. </td>
   <td style="text-align:left;"> 108 </td>
   <td style="text-align:left;"> qs </td>
   <td style="text-align:right;"> 39 </td>
   <td style="text-align:right;"> 32 </td>
   <td style="text-align:right;"> 3 </td>
  </tr>
</tbody>
</table>

Now, we can also enhance the other test cases data.

{% highlight r %}
### enhance test case 1
test2_res <- enhance_manifesto_df(test2_df)
### enhance test case 4
test4_res <- enhance_manifesto_df(test4_df)
{% endhighlight %}

### Step 5: Convert manifesto data frames to XML documents

Having enhanced our test case data frames, we can now use `manifesto_df_to_xml()` to convert them to XML.
You have basically two options 

- setting `parse = FALSE` returns the XML-formatted string
- setting `parse = TRUE` (the default) returns an {xml2} `xml_document` object.

We convert parse the data of the first text into and XML formatted string. 


{% highlight r %}
### test case 1: as string
test1_xml <- manifesto_df_to_xml(test1_res, parse = FALSE)
class(test1_xml)
{% endhighlight %}



{% highlight text %}
## [1] "character"
{% endhighlight %}



{% highlight r %}
length(test1_xml)
{% endhighlight %}



{% highlight text %}
## [1] 1
{% endhighlight %}



{% highlight r %}
test1_xml <- manifesto_df_to_xml(head(test1_res, 6), parse = FALSE)
cat(test1_xml)
{% endhighlight %}



{% highlight text %}
## <manifesto>
##   <head
##      manifesto-id="43120_201110"
##      party="43120"
##      date="201110"
##      language="german"
##      source="MARPOR"
##      has-eu-code="FALSE"
##      is-primary-doc="TRUE"
##      may-contradict-core-dataset="FALSE"
##      md5sum-text="477db1a44a9c3531ce50e09522bb8daa"
##      annotations="TRUE"
##      handbook="4"
##      id="43120_201110"
##      partyname="Green Liberal Party"
##      partyabbrev="GLP"
##   >
##     <title>
##       <p>Eidgenössische Volksinitiative</p>
##     </title>
##   </head>
##   <body>
##     <chapter nr="1">
##       <sentence nr="1">
##         <quasi-sentence nr="1" cmp-code="501">Mit der Einführung einer Energiesteuer auf nicht erneuerbarer Energie werden Energieeffizienz, Energiesparen und erneuerbare Energien ökonomisch interessant.</quasi-sentence>
##       </sentence>
##       <sentence nr="2">
##         <quasi-sentence nr="2" cmp-code="303">Die Abschaffung der komplizierten Mehrwertsteuer befreit Wertschöpfung von Steuern.</quasi-sentence>
##       </sentence>
##       <sentence nr="3">
##         <quasi-sentence nr="3" cmp-code="411">Insgesamt stärkt die Initiative die Innovation, reduziert die Administration bei Unternehmen und Staat.</quasi-sentence>
##       </sentence>
##       <sentence nr="4">
##         <quasi-sentence nr="4" cmp-code="501">Der Verbrauch von öl, Gas und Atomstrom wird sinken, der CO2-Ausstoss reduziert: Auch bleiben Milliarden für Wertschöpfung im Inland statt ins Ausland abzufliessen.</quasi-sentence>
##       </sentence>
##       <sentence nr="5">
##         <quasi-sentence nr="5" cmp-code="000">Die fixe Verknüpfung des Ertrages an das Bruttoinlandprodukt sichert eine staatsquotenneutrale Umsetzung.</quasi-sentence>
##       </sentence>
##     </chapter>
##     <chapter nr="2">
##       <header>
##         <p>Energie</p>
##       </header>
##     </chapter>
##   </body>
## </manifesto>
{% endhighlight %}

Note that while the title of the manifesto was not contained in the text data, it was contained in the manifesto meta data (column "title").

When we use the default setting `parse = TRUE`, the resulting object is a `xml2` `xml_document` instance.


{% highlight r %}
### test case 1: as XML document
test1_xml <- manifesto_df_to_xml(test1_res)
class(test1_xml)
{% endhighlight %}



{% highlight text %}
## [1] "xml_document" "xml_node"
{% endhighlight %}



{% highlight r %}
test1_xml
{% endhighlight %}



{% highlight text %}
## {xml_document}
## <manifesto>
## [1] <head manifesto-id="43120_201110" party="43120" date="201110" language="g ...
## [2] <body>\n  <chapter nr="1">\n    <sentence nr="1">\n      <quasi-sentence  ...
{% endhighlight %}

The results looks similar for test case 3, but now the "head" node also includes a "title" node.


{% highlight r %}
### test case 3
test3_xml <- manifesto_df_to_xml(head(test3_res, 6), parse = FALSE)
cat(test3_xml)
{% endhighlight %}



{% highlight text %}
## <manifesto>
##   <head
##      manifesto-id="43110_199910"
##      party="43110"
##      date="199910"
##      language="german"
##      source="MARPOR"
##      has-eu-code="FALSE"
##      is-primary-doc="TRUE"
##      may-contradict-core-dataset="FALSE"
##      md5sum-text="4db941bd08b626f99f75317b70235a4c"
##      url-original="/down/originals/43110_1999.pdf"
##      md5sum-original="CURRENTLY_UNAVAILABLE"
##      annotations="TRUE"
##      handbook="1"
##      title="Für eine zukunftsfähige Schweiz"
##      id="43110_199910"
##      partyname="Green Party of Switzerland"
##      partyabbrev="GPS/PES"
##   >
##     <title>
##       <p>Für eine zukunftsfähige Schweiz</p>
##       <p>Wahlplattform 1999</p>
##     </title>
##   </head>
##   <body>
##     <chapter nr="0">
##       <sentence nr="1">
##         <quasi-sentence nr="1" cmp-code="501">Die Grünen freuen sich, eine Wahlplattform für eine ökologische, soziale und weltoffene Schweiz</quasi-sentence>
##         <quasi-sentence nr="2" cmp-code="601">- kurz eine zukunftsfähige Schweiz zu präsentieren.</quasi-sentence>
##       </sentence>
##       <sentence nr="2">
##         <quasi-sentence nr="3" cmp-code="201">In Verantwortung gegenüber künftigen Generationen setzen wir uns ein für die Erhaltung der Lebensgrundlagen, für die Menschenrechte</quasi-sentence>
##         <quasi-sentence nr="4" cmp-code="503">und für die Verminderung der Unterschiede zwischen arm und reich.</quasi-sentence>
##       </sentence>
##     </chapter>
##   </body>
## </manifesto>
{% endhighlight %}

As shown in the next code block, you can access the different nodes and attributes of the returned XML documents using `rvest`'s `html_*`.


{% highlight r %}
test3_xml <- manifesto_df_to_xml(test3_res)
class(test3_xml)
{% endhighlight %}



{% highlight text %}
## [1] "xml_document" "xml_node"
{% endhighlight %}



{% highlight r %}
test3_xml
{% endhighlight %}



{% highlight text %}
## {xml_document}
## <manifesto>
## [1] <head manifesto-id="43110_199910" party="43110" date="199910" language="g ...
## [2] <body>\n  <chapter nr="1">\n    <sentence nr="1">\n      <quasi-sentence  ...
{% endhighlight %}



{% highlight r %}
html_node(test3_xml, "title")
{% endhighlight %}



{% highlight text %}
## {xml_node}
## <title>
## [1] <p>Für eine zukunftsfähige Schweiz</p>
## [2] <p>Wahlplattform 1999</p>
{% endhighlight %}

### Step 6 (optional): Apply to all manifestos in one pipeline

The below code simply combines the above steps in a single pipeline.


{% highlight r %}
man_xmls <- man_sents %>% 
  ### drop unneccessary columns (optional)
  select(
    -has_eu_code
    , -may_contradict_core_dataset
    , -md5sum_text
    , -md5sum_original
    , -annotations
    , -id
  ) %>% 
  ### unnest data
  unnest(data) %>% 
  ### split by manifesto ID (-> list od DFs)
  split(.$manifesto_id) %>% 
  ### apply enhance
  map(enhance_manifesto_df) %>% 
  ### apply converter
  map(safely(manifesto_df_to_xml)) %>% 
  ### gather all in list-column data frame 
  enframe() %>% 
  transmute(
    manifesto_id = name
    , xml = map(value, "result")
    , errors = map(map(value, "error"), "message")
    , n_errors = lengths(errors)
  )
{% endhighlight %}

Calling this code returns a tibble with list-column "xml" containing the individual manifesto XML documents.



{% highlight r %}
### the returned tibble contains xml_documents in the "xml" list-column
head(man_xmls, 3)
{% endhighlight %}



{% highlight text %}
## # A tibble: 3 x 4
##   manifesto_id xml        errors n_errors
##   <chr>        <list>     <list>    <int>
## 1 43110_198710 <xml_dcmn> <NULL>        0
## 2 43110_199110 <xml_dcmn> <NULL>        0
## 3 43110_199510 <xml_dcmn> <NULL>        0
{% endhighlight %}

To make sure that all covnersions were successful, you can look at column "n_errors":


{% highlight r %}
### there were no errors raised
man_xmls %>% 
  filter(n_errors > 0) %>% 
  unnest(errors)
{% endhighlight %}



{% highlight text %}
## # A tibble: 0 x 4
## # … with 4 variables: manifesto_id <chr>, xml <list>, errors <???>,
## #   n_errors <int>
{% endhighlight %}

### Step 7 (optional): write manifestos to disk

You can alos loop over XML documents and write them to disk.


{% highlight r %}
### write to disk
map2(
  man_xmls$xml
  , man_xmls$manifesto_id
  , function(xml, id, path = file.path("your", "local", "path")) {
  write_xml(xml, file.path(path, paste0(id, ".xml")))  
})
{% endhighlight %}

## Summing up

To reconstruct manifesto document structure from `manifestoR` obejcts, you can use functions provided by the `manifestoEnhanceR` package.

The basic data processing pipeline involves: 

1. querying manifestos from the Manifesto Project API, 
2. covnerting the `manifestoR` package's custom data objects into a tidy data format (i.e., a data frame), 
3. enhancing manifesto data frames with text-level information (e.g., quasi-sentence and chapter numbers), and
4. converting the enhanced manifesto data frames to XML documents.

The `manifestoEnhanceR` package is currently only available via GitHub: https://github.com/haukelicht/manifestoEnhanceR

Feel free to retweet this post on Twitter and comment on it!

## (Footnotes)
