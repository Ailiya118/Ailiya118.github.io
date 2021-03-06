---
layout: post
title: New York Times Article Search API to MongoDB
date: 2015-01-06
categories: articles
tags: [data science, R, api, web scraping, New York Times, mongoDB]
comments: true
share: true
---

* Table of Contents
{:toc}

## Motivation

I've learned a little about a lot of different corners of the text mining and NLP world over the last few years... which sometimes makes me feel like I know nothing for certain.  I've done a decent amount web scraping, processing HTML and parsing text recently, but never a full blown text mining project.  I decided to start with some topic modeling using Latent Dirichlet Allocation and document clustering.   Unsupervised learning techniques requiring minimal upfront work beyond the text pre-processing seemed like a good (and interesting) place to get started.

After surveying APIs for a few news sources, the New York Times seemed to be the most robust.  I wanted the flexibility to acquire new documents in the future for testing models and from far enough in the past to build a hefty corpus.  I also wanted to have some fun and scrape the documents myself, experiment with a NoSQL database pipeline and process the HTML from the rawest form.

## Accessing NYT API

##### API Documentation and keys
The [New York Times API] is well documented and user-friendly.  I didn't experiment too much with targeted querying since I was pulling all articles over a period of time.  However the `q` and `fq` parameters seem to provide a lot of functionality for filtered searches.  

Requesting an a key for the Article Search API was easy and instantaneous.  I saved my key as a global parameter using R options.  I'm using `sample-key` here simply for illustrative purposes, but I found that this key (provided by default in the [NYT API Console]) actually works.  


{% highlight r %}
options(nyt_as_key = 'sample-key')  ## copy and paste your API key here.
options()$nyt_as_key
{% endhighlight %}



{% highlight text %}
## [1] "sample-key"
{% endhighlight %}

The [NYT API Console] is also a nifty way to kick the tires and understand the parameters at your disposal for constructing queries.  It's basically a GUI around the API which gives you everything you need on one page to quickly formulate a query, submit it and inspect the response.  It also allowed me to quickly determine that this is a well developed and documented API worthy of pursuing further.  I wish all APIs were like this... sigh.



##### Generate URL

This is the easy part.  I did find an R package [rtimes] for accessing the API which worked for the few queries I tried.  However, I had already started building my own pipeline, so I stuck with it.

The NYT API returns only 10 articles per request.  Which 10 are dictated by the `page` parameter.  `page=0` returns articles 1-10, `page=1` returns articles 11-20, etc.  The tricky part is knowing how many pages to iterate through.  On my first pass, I failed to find a way to figure out the total number of articles matching my query to tell me how many pages and API requests I would need.  So instead, I simply started my requests with `page=0` and incrementally added 1 to `page` until the response stopped yielding articles.

However, after I already maxed out my database, I found a way to do this using the `facet_field` and `facet_filter` parameters.  These can be used to count the number of articles matching a filtered query by simply adding `&facet_field=source&facet_filter=true` to your query URL.

And you get something like this:

{% highlight r %}
{  
   "facets":{  
      "source":{  
         "terms":[  
            {  
               "term":"The New York Times",
               "count":159
            },
            {  
               "term":"International Herald Tribune",
               "count":7
            },
            {  
               "term":"",
               "count":1
            }
         ]
      }
   },
   "status":"OK",
   "copyright":"Copyright (c) 2013 The New York Times Company.  All Rights Reserved."
}
{% endhighlight %}

Here's an example using R and httr to get the same result[^1].  These counts almost perfectly match the counts I've calculated from my collection for the New York Times and International Herald Tribune.  However, I've collected a lot of Reuters and AP articles from the Article Search API that interestingly don't appear here. 


{% highlight r %}
library('httr')
resp <- GET('http://api.nytimes.com/svc/search/v2/articlesearch.json?facet_field=source&facet_filter=true&begin_date=20130105&end_date=20130105&api-key=sample-key')
print(content(resp, 'parsed')$response$facets)
{% endhighlight %}



{% highlight text %}
## $source
## $source$terms
## $source$terms[[1]]
## $source$terms[[1]]$term
## [1] "The New York Times"
## 
## $source$terms[[1]]$count
## [1] 159
## 
## 
## $source$terms[[2]]
## $source$terms[[2]]$term
## [1] "International Herald Tribune"
## 
## $source$terms[[2]]$count
## [1] 7
## 
## 
## $source$terms[[3]]
## $source$terms[[3]]$term
## [1] ""
## 
## $source$terms[[3]]$count
## [1] 1
{% endhighlight %}

Another rub is that "pagination beyond page 100 is not allowed at this time." So if you're extracting large quantities of articles, best do it chunks.  I chunked my queries into single days -- usually returning about 700-800 articles.  Without using any facets or query terms, I ran the risk of only collecting 1000 articles for a day when there were more than 1000.  This occurred 12 days out of 120 for me.

`makeURL` is a pretty trivial function that generates the URL to make the GET request from a collection of NYT API parameters.  However, I found encapsulating this step in a function which gets called in subsequent functions kept things clean.


{% highlight r %}
makeURL <- function(q=NULL, fq=NULL, begin_date=NULL, end_date=NULL, key=getOption("nyt_as_key"), page=0, 
                    sort=NULL, fl=NULL, hl=NULL, facet_field=NULL, facet_filter=NULL){
  arglist <- list(q=q, fq=fq, begin_date=begin_date, end_date=end_date, 'api-key'=key, page=page,
                  sort=sort, fl=fl, hl=hl, facet_field=facet_field, facet_filter=facet_filter)
  url <- 'http://api.nytimes.com/svc/search/v2/articlesearch.json?'
  for(i in 1:length(arglist)){
    if(is.null(unlist(arglist[i]))==F){
      url <- paste0(url, '&', names(arglist[i]), '=', arglist[i])
    }
  }
  return(url)
}

## example: generate query which will return all articles for one day.
library('httr')
url <- makeURL(begin_date='20130101', end_date='20130102', key='sample-key', page=100)
print(url)
{% endhighlight %}



{% highlight text %}
## [1] "http://api.nytimes.com/svc/search/v2/articlesearch.json?&begin_date=20130101&end_date=20130102&api-key=sample-key&page=100"
{% endhighlight %}

##### Scrape NYT metadata

Here in `getMeta` we're actually collecting the NYT article metadata and structuring it as a list object.  Like most things, the meat of it is pretty simple.  The rest is parsing, exception and error handling... which I find are worth it with web scraping, especially if you're running a job overnight and want it to work by the time you wake up.

`getMeta`:

*  DOES iterate through `pages` until no new articles are returned.
*  DOES sleep for `sleep` seconds after each request.  I had good luck with `sleep=0.1`.
*  DOES re-attempt failed requests `tryn` times.  Failed requests were almost always successful after the second try, so I set `tryn=3` just to be safe.
*  DOES NOT cache responses to disc.  Everything is saved in memory and returned as a list in R memory after the function is complete.  I opted not to bother with caching with this step as it only takes ~20 seconds to run through 100 API calls (the max per query).


{% highlight r %}
library('httr')

getMeta <- function(url, pages=Inf, sleep=0.1, tryn=3) {
  art <- list()
  i <- 1
  e <- seq(-tryn, -tryn/2, length.out=tryn) # initialize list of failed pages with arbitrary negative numbers
  while(i<=pages){
    if(length(unique(e[(length(e)-(tryn-1)):length(e)]))==1) i <- i+1 ## attempt tryn times before moving on
    tryget <- try({
      urlp <- gsub('page=\\d+', paste0('page=', i), url) ## get the next page
      p <- GET(urlp) ## actually make GET request
      pt <- content(p, 'parsed') ## retrieve contents of response formatted in a nested R list
      if(length(pt$response$docs)>0) art <- append(art, pt$response$docs) ## add articles to list
      else {print(paste0(i, ' pages collected')); break}
      if(i %% 10 ==0) print(paste0(i, ' pages of metadata collected'))
      i <- i+1
    })
    
    ## if there was  a fail...
    if(class(tryget)=='try-error') {
      print(paste0(i, ' error - metadata page not scraped'))
      e <- c(e, i) ## add page i to failed list
      e <- e[(length(e)-(tryn-1)):length(e)] ## keep the failed list to just tryn elements
      Sys.sleep(0.5) ## probably scraping too fast -- slowing down
    }
    Sys.sleep(sleep)
  }
  return(art)
}

## example: collect metadata for 3 pages (30 articles)
meta <- getMeta(url, pages=3)
str(meta[[1]]) ## look at just one article 
{% endhighlight %}



{% highlight text %}
## List of 19
##  $ web_url         : chr "http://thecaucus.blogs.nytimes.com/2013/01/02/obama-returns-to-hawaii-to-continue-vacation/"
##  $ snippet         : chr "President Obama eased back into the relaxing groove of his Christmas in Hawaii vacation after a brief hiatus to wrangle with Co"| __truncated__
##  $ lead_paragraph  : NULL
##  $ abstract        : chr "President Obama eased back into the relaxing groove of his Christmas in Hawaii vacation after a brief hiatus to wrangle with Co"| __truncated__
##  $ print_page      : NULL
##  $ blog            : list()
##  $ source          : chr "The New York Times"
##  $ multimedia      : list()
##  $ headline        :List of 2
##   ..$ main  : chr "Obama Returns to Hawaii to Continue Vacation"
##   ..$ kicker: chr "The Caucus"
##  $ keywords        :List of 3
##   ..$ :List of 3
##   .. ..$ rank : chr "1"
##   .. ..$ name : chr "persons"
##   .. ..$ value: chr "Obama, Barack"
##   ..$ :List of 3
##   .. ..$ rank : chr "1"
##   .. ..$ name : chr "glocations"
##   .. ..$ value: chr "Hawaii"
##   ..$ :List of 3
##   .. ..$ rank : chr "1"
##   .. ..$ name : chr "subject"
##   .. ..$ value: chr "United States Politics and Government"
##  $ pub_date        : chr "2013-01-02T18:37:03Z"
##  $ document_type   : chr "blogpost"
##  $ news_desk       : NULL
##  $ section_name    : chr "U.S."
##  $ subsection_name : NULL
##  $ byline          :List of 2
##   ..$ person  :List of 1
##   .. ..$ :List of 6
##   .. .. ..$ firstname   : chr "Jeremy"
##   .. .. ..$ middlename  : chr "W."
##   .. .. ..$ lastname    : chr "PETERS"
##   .. .. ..$ rank        : int 1
##   .. .. ..$ role        : chr "reported"
##   .. .. ..$ organization: chr ""
##   ..$ original: chr "By JEREMY W. PETERS"
##  $ type_of_material: chr "Blog"
##  $ _id             : chr "50e4c5c700315214fbb821e4"
##  $ word_count      : int 366
{% endhighlight %}

## Extracting and parsing the article body text

So now we have a lot of information about NYT articles including URL, abstract, headline, publication date, section, author, type of material, etc.  However, notably absent is the article text itself.  The process for collecting article body text is much more manual.  We've used the API road as much as we can, but now we have to go off-road a little bit and collect each article's body text from its individual URL provided from the metadata field `web_url`. 

After a while of trial-and-error and guessing and checking, I developed a basic utility function `parseArticleBody` to strip just the article body text from the raw HTML. At least one of three simple XPath queries seemed to work reasonably well for the normal plain vanilla articles.  Most of the video and photography pages contain little to no text, so these often come up empty.  Some of the more modern pages like this [one](http://www.nytimes.com/roomfordebate/2013/01/02/should-social-security-cuts-be-considered) which utilize multimedia and spread content over several pages usually fail too.

However, there's enough NYT articles in the world to sink a small ship[^2], so I'm more concerned with precision than recall -- I'm willing to let some articles slip through the cracks if it boosts the quality of text for the articles I am able to extract it for.  In most cases the gains from nailing the XPath query were marginal as the most common cause for missing article body text was out-of-date URLs provided by the NYT API.  



{% highlight r %}
library('XML')
library('httr')

parseArticleBody <- function(artHTML) {
  xpath2try <- c('//div[@class="articleBody"]//p',
                 '//p[@class="story-body-text story-content"]',
                 '//p[@class="story-body-text"]'
                 )
  for(xp in xpath2try) {
    bodyi <- paste(xpathSApply(htmlParse(artHTML), xp, xmlValue), collapse='')
    if(nchar(bodyi)>0) break
  }
  return(bodyi)
}

## Example: extract article text for one article
p <- GET('http://www.nytimes.com/2013/01/31/garden/faketv-can-fool-intruders.html')
html <- content(p, 'text')
artBody <- parseArticleBody(html)
print(artBody)
{% endhighlight %}



{% highlight text %}
## [1] "\nYou might not be home every evening watching television, but no burglar needs to know that. The coruscating light of a television set ??? signature feature of an occupied home ??? might be enough to scare prowlers off.        \nThat???s the idea behind FakeTV ($35), a security gadget that replicates a television???s flickering light in all of its dynamic variety, from kinetic commercials to subdued news reports.        \nThe lightweight wedge, smaller than a slice of cake, is the brainchild of Blaine Readler, an engineer, who intentionally left his TV on when going out one night. The fake version, which uses bright LEDs, saves wear and tear on a real TV, as well as the cost of electricity. Energy-wise, it???s equivalent to a 2-watt night light, said Rein Teder, president of the manufacturer, Hydreon Corporation.        \nAnd unlike most TVs today, which turn on and off with buttons or remotes, FakeTV can be put on a timer. Information: faketv.com or (877) 532-5388.??        "
{% endhighlight %}

## Writing to MongoDB

##### Choosing MongoDB

Github was a logical place to store code for this project -- my blog is already hosted there and integrates well with my workflow allowing me to work from my home, office or villa on a remote island[^3].  Where to store the data was less obvious.  I wanted something more robust than a directory full of text files... some sort of NoSQL database that I could access from Python and R.  I also wanted to make the data accessible via the web. And I wanted it all for free.  

[mongolab] (MongoDB's cloud database-as-a-service) fit the bill, for 500MB at least. I don't have much experience with NoSQL databases and haven't worked with Mongo in the past, but I had a lot of fun learning and working with Mongo.  My grasp of the Mongo querying language is still tenuous, but I was able make it do everything I needed it to do.

I found the web interface intuitive and useful for testing queries or other Mongo command line operations when I was getting started.  From here you can also create database users with read-only or full access -- useful for sharing your work with the world while holding the keys to the kingdom to yourself.

##### Working with MongoDB in Python and R

I used the [RMongo] R package and the [pymongo] Python package for interacting with the database.  No complaints on either.  I like how the usage inside Python and R using these packages seems to mirror pretty closely usage at the command line.

##### When the cloud can bring you down

So high, yet so low.  I did experience a period of down-time where I was unable to access my database for an hour or so.  Although mongolab does a decent job of documenting issues on their [status page], the list doesn't appear to be short... As is life sometimes, I suppose, when you're on the free-tier of a cloud service.

##### Pulling it all together
 
`getArticles` does most of the heaviest lifting -- extracts, parses and adds the article body text to the `meta` object and inserts to MongoDB after scraping each article.

`getArticles`:

*  DOES use the list of metadata as a starting point (the `meta` argument).  Then scrapes, parses and adds the body as an attribute (text string) to the article's metadata.  Only adds to the `meta` object.
*  DOES cache -- writes to MongoDB after extracting each article.  I didn't bump into any rate-limits from MongoDB and I had to wait a decisecond or two after each article anyway, so insert speed was not an issue for me.  Had all the articles been pulled into memory first, I imagine a bulk insert would be more efficient.
*  DOES convert the R list of metadata for each article to JSON on the fly using the `toJSON` function which is then conveniently inserted to MongoDB as-is.  MongoDB likes JSON.
*  DOES re-attempt failed article extraction, parsing and MongoDB inserts -- If there was an error in one of these steps, I tried another 2 times before moving on to the next article.
*  DOES include parameters: 
    +  `meta` is created from the function `getMeta`.  
    +  `n` is the number of articles to scrape. 
    +  `overwrite` decides whether to start scraping at the beginning or the first article that does not have a body attribute.  
    +  `sleep` is the time in seconds to pause after each article.  Defaults to 0.1 seconds.
    +  `mongo` is list of credentials used to write to the database.  To return results in memory and not write to database, specify `mongo=NULL`.




{% highlight r %}
library('XML')
library('RMongo')
library('rjson')

getArticles <- function(meta, n=Inf, overwrite=F, sleep=0.1, mongo=list(dbName, collection, host='127.0.0.1', username, password, ...)) {
  metaArt <- meta
  if(is.null(mongo$dbName)==F){
    con <- mongoDbConnect(mongo$dbName, mongo$host)
    try(authenticated <-dbAuthenticate(con, username=mongo$username, password=mongo$password))
  }
  if(overwrite==T) {artIndex <- 1:(min(n, length(meta)))
  } else { 
    ii <-  which(sapply(meta, function(x) is.null(x[['bodyHTML']]))) ## get index of articles that have not been scraped yet
    artIndex <- ii[1:min(n,length(ii))] ## take first n articles
  }
  e <- c(-3,-2,-1) # initialize failed list of articles with arbitrary negative numbers
  i<-1
  while(i<=length(artIndex)){
    if(length(unique(e[(length(e)-2):length(e)]))==1) i <- i+1 ## if we tried and failed 3 times, move on
    tryget <- try({
      if(overwrite==T | (is.null(meta[[i]]$body)==T & overwrite==F)) {
        p <- GET(meta[[i]]$web_url)
        html <- content(p, 'text') ## metaArt[[i]]$bodyHTML <- content(p, 'text')
        metaArt[[i]]$body <- parseArticleBody(html)
        if(is.null(mongo$dbName)==F) dbInsertDocument(con, mongo$collection, toJSON(metaArt[[i]]))
        if(i %% 10==0) print(paste0(i, ' articles scraped'))
        i<-i+1
      }
    })
    if(class(tryget)=='try-error') {
      print(paste0(i, ' error - article not scraped'))
      e <- c(e, i)
      e <- e[(length(e)-2):length(e)]
      Sys.sleep(0.5) ## probably scraping too fast -- slowing down
    }
    Sys.sleep(sleep)
  }
  return(metaArt)
}

## Example: get article bodies for first 10 articles.  
art <- getArticles(meta, n=3, mongo=NULL) ## scrape article body text for first 3 articles.  Do not write to MongoDB
length(art) ## we only add to meta.  Still returns articles for which we did not collect body text.
{% endhighlight %}



{% highlight text %}
## [1] 30
{% endhighlight %}



{% highlight r %}
str(art[[1]]) ## we now have a "body" attribute.
{% endhighlight %}



{% highlight text %}
## List of 20
##  $ web_url         : chr "http://thecaucus.blogs.nytimes.com/2013/01/02/obama-returns-to-hawaii-to-continue-vacation/"
##  $ snippet         : chr "President Obama eased back into the relaxing groove of his Christmas in Hawaii vacation after a brief hiatus to wrangle with Co"| __truncated__
##  $ lead_paragraph  : NULL
##  $ abstract        : chr "President Obama eased back into the relaxing groove of his Christmas in Hawaii vacation after a brief hiatus to wrangle with Co"| __truncated__
##  $ print_page      : NULL
##  $ blog            : list()
##  $ source          : chr "The New York Times"
##  $ multimedia      : list()
##  $ headline        :List of 2
##   ..$ main  : chr "Obama Returns to Hawaii to Continue Vacation"
##   ..$ kicker: chr "The Caucus"
##  $ keywords        :List of 3
##   ..$ :List of 3
##   .. ..$ rank : chr "1"
##   .. ..$ name : chr "persons"
##   .. ..$ value: chr "Obama, Barack"
##   ..$ :List of 3
##   .. ..$ rank : chr "1"
##   .. ..$ name : chr "glocations"
##   .. ..$ value: chr "Hawaii"
##   ..$ :List of 3
##   .. ..$ rank : chr "1"
##   .. ..$ name : chr "subject"
##   .. ..$ value: chr "United States Politics and Government"
##  $ pub_date        : chr "2013-01-02T18:37:03Z"
##  $ document_type   : chr "blogpost"
##  $ news_desk       : NULL
##  $ section_name    : chr "U.S."
##  $ subsection_name : NULL
##  $ byline          :List of 2
##   ..$ person  :List of 1
##   .. ..$ :List of 6
##   .. .. ..$ firstname   : chr "Jeremy"
##   .. .. ..$ middlename  : chr "W."
##   .. .. ..$ lastname    : chr "PETERS"
##   .. .. ..$ rank        : int 1
##   .. .. ..$ role        : chr "reported"
##   .. .. ..$ organization: chr ""
##   ..$ original: chr "By JEREMY W. PETERS"
##  $ type_of_material: chr "Blog"
##  $ _id             : chr "50e4c5c700315214fbb821e4"
##  $ word_count      : int 366
##  $ body            : chr "\tHONOLULU ??? President Obama eased back into the relaxing groove of his Christmas in Hawaii vacation on Wednesday after a brief"| __truncated__
{% endhighlight %}



{% highlight r %}
print(art[[1]]['body']) ## full body text of the article.
{% endhighlight %}



{% highlight text %}
## $body
## [1] "\tHONOLULU ??? President Obama eased back into the relaxing groove of his Christmas in Hawaii vacation on Wednesday after a brief hiatus to wrangle with Congress over a deal to avert a fiscal crisis.\tIn most respects, Wednesday on the island of Oahu was 5,000 miles away and a world apart from the political brinksmanship of the nation???s capital. The morning included a trip to the local Marine Corps base, where the president went to the gym.\tAnd after speaking with Gov. Andrew M. Cuomo of New York and Gov. Chris Christie of New Jersey about the stalled efforts to provide tens of billions of dollars to the states to aid their recovery from Hurricane Sandy, Mr. Obama hit the golf course. \tThe White House announced Mr. Obama???s golf partners, as is standard protocol, and they once again included a friend who always creates a bit of a stir whenever he is spotted with the president. Bobby Titcomb, an old friend of the president???s from their days at the Punahou School here, was arrested in 2011 on a charge of soliciting prostitution after being swept up in an undercover sting. He later entered a plea of no contest. \tThe president???s group also included Marty Nesbitt, a Chicago friend. Mr. Nesbitt was an early backer of Mr. Obama, and he later served as treasurer for his 2008 presidential campaign. \tThe third in their foursome was Allison Davis, a Chicago developer and lawyer who was one of the founders of the firm the president once worked for and who was another financial backer of his early political career. \tThe White House also released a new taped message from the president on Wednesday in which he outlined his priorities for 2013 ??? listing them in order as winding down the war in Afghanistan, reforming immigration and gun control ??? and cautioned that many of the issues that made the tax-and-spending deal so difficult remain unresolved.\t???Obviously there???s still more to do when it comes to reducing our debt,??? he said. ???And I???m willing to do more as long as it does it in a balanced way that doesn???t put all the burden on seniors or students or middle-class families.???"
{% endhighlight %}

## Pipeline

So putting it all together, the pipeline looks like this:


{% highlight r %}
## dependencies
library('rjson')
library('httr')
library('RMongo')
library('XML')

## parameters
days <- gsub('-', '', seq(as.Date('2013-01-01'), Sys.Date()-1, by=1))
mongoCreds <- list(dbName='nyt', collection='articles1', host='ds063240.mongolab.com:63240', username='myUsername', password='myPassword')
key <- 'myKey' # replace with your personal key

## Letting it rip ... 
all <- list() ## persist all results in memory in addition to MongoDB... just in case.
for(d in days) {
  url <- makeURL(begin_date=d, end_date=d, key=key) # generate URL
  meta <- getMeta(url, pages=Inf, sleep=0.1) # extract metadata from NYT API
  artxt <- getArticles(meta, n=Inf, sleep=0.1, overwrite=T, mongo=mongoCreds) # extract article text and write to MongoDB
  print(paste0('day ',  d, ' complete at ', Sys.time()))
  all <- append(all, artxt) # persist results
}
{% endhighlight %}

## Results
It took me about 1 full day to max out the 500MB in my free mongolab database.  I ended up with 85,000 articles covering roughly 4 months of NYT articles.  Since I'm specifically interested in text mining the article text, my next step will likely be to delete the documents where the article body could not be extracted. 

A lot of the URLs provided from the NYT API were stale -- only ~35% of URLs contained a real article with text.  It wasn't until I was doing some preliminary analysis poking around on the full corpus that I realized links to historic AP and Reuters articles (of which there are many) were included and almost universally not fruitful.  These could easily be filtered out using the `fq` parameter in the NYT API: `fq=source:("The New York Times")`.


{% highlight r %}
## Crosstab of article source vs. whether article was successfully scraped 

                        **body scraped successfully**
  **Source**                   FALSE  TRUE
  The New York Times            6686 34322
  AP                           28944     8
  Reuters                      24001     0
  International Herald Tribune     9  1221
                                1007     4
  du_recipe                      273     0
  The Broadway Channel             8     0
  CNBC                             6     0
  ADAM                             1     0
{% endhighlight %}

I decided to use [gensim] and [nltk] in Python for the text pre-processing and topic modeling.  More to come on that analysis and visualization.

##### Footnotes

[rtimes]:https://github.com/ropengov/rtimes
[New York Times API]:http://developer.nytimes.com/docs/read/article_search_api_v2
[NYT API]:http://developer.nytimes.com/docs/read/article_search_api_v2
[NYT API Console]:http://developer.nytimes.com/io-docs
[mongolab]:https://mongolab.com/
[RMongo]:http://cran.r-project.org/web/packages/RMongo/index.html
[pymongo]:https://pypi.python.org/pypi/pymongo/
[status page]:http://status.mongolab.com/
[gensim]:https://radimrehurek.com/gensim/
[nltk]:http://www.nltk.org/

[^1]: Note the use of the `content` function from the httr package with `as='parsed'` is not recommended by the authors for use in other R packages or production systems.  It's provided as a convenience function... which I found, well, convenient... so I used it.
[^2]: In fact there are over 1000 articles alone matching the query parameters "ship+sink" 
[^3]: Still working on the villa ...
