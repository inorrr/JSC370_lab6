Lab 06 - Regular Expressions and Web Scraping
================
Yinuo Zhao
Feb 16, 2024

# Learning goals

- Use a real world API to make queries and process the data.
- Use regular expressions to parse the information.
- Practice your GitHub skills.

# Lab description

In this lab, we will be working with the [NCBI
API](https://www.ncbi.nlm.nih.gov/home/develop/api/) to make queries and
extract information using XML and regular expressions. For this lab, we
will be using the `httr`, `xml2`, and `stringr` R packages.

This markdown document should be rendered using `github_document`
document ONLY and pushed to your *JSC370-labs* repository in
`lab06/README.md`.

## Question 1: How many sars-cov-2 papers?

Build an automatic counter of sars-cov-2 papers using PubMed. You will
need to apply XPath as we did during the lecture to extract the number
of results returned by PubMed in the following web address:

    https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2

Complete the lines of code:

``` r
# Downloading the website
website <- xml2::read_html("https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2")

# Finding the counts
counts <- xml2::xml_find_first(website, "//span[@class='value']")

# Turning it into text
counts <- as.character(counts)

# Extracting the data using regex
stringr::str_extract(counts, "[0-9,]{7}")
```

- How many sars-cov-2 papers are there?

*Answer here.* 218,851 results

Don’t forget to commit your work!

## Question 2: Academic publications on COVID19 related to Toronto

Use the function `httr::GET()` to make the following query:

1.  Baseline URL:
    <https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi>

2.  Query parameters:

    - db: pubmed
    - term: covid19 toronto
    - retmax: 300

The parameters passed to the query are documented
[here](https://www.ncbi.nlm.nih.gov/books/NBK25499/).

``` r
library(httr)

baseline_url <- "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi"
query_params <- list(
  db = "pubmed",
  term = "covid19 toronto",
  retmax = 300
)
query_ids <- GET(
  url   = baseline_url,
  query = query_params
)

# Extracting the content of the response of GET
ids <- httr::content(query_ids)
char_list <- as.character(ids)
```

The query will return an XML object, we can turn it into a character
list to analyze the text directly with `as.character()`. Another way of
processing the data could be using lists with the function
`xml2::as_list()`. We will skip the latter for now.

Take a look at the data, and continue with the next question (don’t
forget to commit and push your results to your GitHub repo!).

## Question 3: Get details about the articles

The Ids are wrapped around text in the following way:
`<Id>... id number ...</Id>`. we can use a regular expression that
extract that information. Fill out the following lines of code:

``` r
# Turn the result into a character vector
ids <- as.character(ids)

# Find all the ids 
ids <- stringr::str_extract_all(ids, "<Id>.*?</Id>")[[1]]

# Remove all the leading and trailing <Id> </Id>. Make use of "|"
ids <- stringr::str_remove_all(ids, "<Id>|</Id>")

char_list <- as.character(ids)
```

With the ids in hand, we can now try to get the abstracts of the papers.
As before, we will need to coerce the contents (results) to a list
using:

1.  Baseline url:
    <https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi>

2.  Query parameters:

    - db: pubmed
    - id: A character with all the ids separated by comma, e.g.,
      “1232131,546464,13131”
    - retmax: 300
    - rettype: abstract

**Pro-tip**: If you want `GET()` to take some element literal, wrap it
around `I()` (as you would do in a formula in R). For example, the text
`"123,456"` is replaced with `"123%2C456"`. If you don’t want that
behavior, you would need to do the following `I("123,456")`.

``` r
ids_string <- paste(char_list, collapse = ",")

baseline_url <- "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi"

query_params <- list(
  db = "pubmed",
  id = I(ids_string),
  retmax = 300,
  rettype = "abstract"
)

publications <- GET(
  url   = baseline_url,
  query = query_params
)

# Turning the output into character vector
publications <- httr::content(publications)
publications_txt <- as.character(publications)
```

With this in hand, we can now analyze the data. This is also a good time
for committing and pushing your work!

## Question 4: Distribution of universities, schools, and departments

Using the function `stringr::str_extract_all()` applied on
`publications_txt`, capture all the terms of the form:

1.  University of …
2.  … Institute of …

Write a regular expression that captures all such instances

``` r
library(stringr)
institution <- str_extract_all(
  publications_txt,
  "\\b(?:University\\sof|.*?\\sInstitute\\sof)\\s[^\\s,]+"
  ) 
institution <- unlist(institution)
as.data.frame(table(institution))
```

Repeat the exercise and this time focus on schools and departments in
the form of

1.  School of …
2.  Department of …

And tabulate the results

``` r
schools_and_deps <- str_extract_all(
  publications_txt,
  "\\b(?:School\\sof|Department\\sof)\\s[^\\s,]+"
  )
as.data.frame(table(schools_and_deps))
```

## Question 5: Form a database

We want to build a dataset which includes the title and the abstract of
the paper. The title of all records is enclosed by the HTML tag
`ArticleTitle`, and the abstract by `AbstractText`.

Before applying the functions to extract text directly, it will help to
process the XML a bit. We will use the `xml2::xml_children()` function
to keep one element per id. This way, if a paper is missing the
abstract, or something else, we will be able to properly match PUBMED
IDS with their corresponding records.

``` r
pub_char_list <- xml2::xml_children(publications)
pub_char_list <- sapply(pub_char_list, as.character)
```

Now, extract the abstract and article title for each one of the elements
of `pub_char_list`. You can either use `sapply()` as we just did, or
simply take advantage of vectorization of `stringr::str_extract`

``` r
abstracts <- str_extract(pub_char_list, "<AbstractText>.*?</AbstractText>")
abstracts <- str_replace_all(abstracts, "<.*?>", "")
abstracts <- str_replace_all(abstracts, "\\s+", " ")
```

- How many of these don’t have an abstract?

``` r
missing_abstracts <- sum(is.na(abstracts))

print(paste("# missing abstract:", missing_abstracts))
```

*Answer here.* 188 articles are missing abstract

Now, the title

``` r
titles <- str_extract(pub_char_list, "<ArticleTitle>.*?</ArticleTitle>")
titles <- str_replace_all(titles, "<.*?>", "")
```

- How many of these don’t have a title ?

``` r
missing_title <- sum(is.na(titles))

print(paste("# missing title:", missing_title))
```

*Answer here.* none is missing title.

Finally, put everything together into a single `data.frame` and use
`knitr::kable` to print the results

``` r
database <- data.frame(
  ArticleTitle = titles, AbstractText = abstracts
)
knitr::kable(database)

# The table will likely be huge. How can we make the output look better?
# (one idea: kableExtra::scroll_box())
```

Done! Knit the document, commit, and push.

## Final Pro Tip (optional)

You can still share the HTML document on github. You can include a link
in your `README.md` file as the following:

``` md
View [here](https://cdn.jsdelivr.net/gh/:user/:repo@:tag/:file) 
```

For example, if we wanted to add a direct link the HTML page of lecture
6, we could do something like the following:

``` md
View Week 6 Lecture [here]()
```
