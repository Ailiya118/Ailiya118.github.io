---
layout: post
title:   "Scraping with Selenium"
date:   2014-12-11
categories: articles
tags: [data science, R, Selenium, RSelenium, web scraping]
comments: true
share: true
---

### If you've ever...

felt like you're playing Simon Says with mouse clicks when repeatedly extracting data in chunks from a
front-end interface to a database on the web, well, you probably are.
There's probably a better solution -- [Selenium].

ever used XML or httr in R or urllib2 in Python, you've probably encountered the situation where 
the source code you've scraped for a website doesn't contain all the information you see in your browser.
[Selenium] can probably help.


### How it works
Selenium is a web automation tool.
While not developed specifically for web scraping, Selenium does it pretty dang well.
Selenium literally "drives" your browser, so it can see anything you see when you right click and inspect element in Chrome or Firefox.
This vastly widens the universe of content that can be extracted from automation, but can be slow 
as all content must be rendered in the browser.  

There are headless (invisible browsers with no GUI) such as [phantomjs] that 
speed some of this up.  That said, I've found that Selenium works best for targeted extraction where the user knows exactly
what they want.

### Example

I set out to collect tickers for all mutual funds in the asset allocation fund type.  Fidelity provides a list
of all these funds [here](https://www.fidelity.com/fund-screener/evaluator.shtml#!&ntf=N&ft=BAL_all&msrV=advanced&sortBy=FUND_MST_MSTAR_CTGY_NM&pgNo=1).
1,586 funds as of today in 80 conveniently paginated URLs.  Each URL ends in `&pgNo=5` to indicate you want page 5 (or whatever number between 1 and 80).

In my browser, when I hover my mouse over one of the fund names in the table, I see the 5 character ticker I'm looking for.
I also see the tickers directly on the webpage when I click the link to each fund. 
[Here](https://fundresearch.fidelity.com/mutual-funds/summary/72201F433) for example, where it says PSLDX in the top left.
However, if possible I'd like to scrape the tickers from the table rather than the individual fund pages.
This would mean 80 pages to scrape rather than 1,586.

### Take 1: traditional http request
When possible, it makes sense to use the simple traditional methods.  So I first tried to extract these tickers with the popular `httr` R package 
by making standard http requests.


{% highlight r %}
library('httr')
url <- 'https://www.fidelity.com/fund-screener/evaluator.shtml#!&ft=BAL_all&ntf=N&expand=%24FundType&rsk=5'
page <- GET(url)
{% endhighlight %}

Success...

{% highlight r %}
print(http_status(page)) 
{% endhighlight %}



{% highlight text %}
## $category
## [1] "success"
## 
## $message
## [1] "success: (200) OK"
{% endhighlight %}

But did our http request return the information we want?

{% highlight r %}
page_text <- content(page, as='text')
{% endhighlight %}

Nope -- can't find the tickers (one of them anyway). 

{% highlight r %}
grepl('GMMAX', page_text, ignore.case=T)
{% endhighlight %}



{% highlight text %}
## [1] FALSE
{% endhighlight %}

Nope -- can't even find the fund name that I see in the table from the webpage in my browser. 

{% highlight r %}
grepl('Aberdeen', page_text, ignore.case=T)
{% endhighlight %}



{% highlight text %}
## [1] FALSE
{% endhighlight %}

It appears that the content of interest is being generated dynamically with Javascript and Ajax on this webpage.
So the raw HTML of this page doesn't help us much.

My plan B was to grab the url for each fund from the table, navigate to that fund's page, and extract the ticker from there.
However these links weren't in our http response.  I noticed that the URLs for each fund followed a simple consistent structure.    
[https://fundresearch.fidelity.com/mutual-funds/summary/72201F433](https://fundresearch.fidelity.com/mutual-funds/summary/72201F433) for example.
I thought maybe I could find 72201F433 which looks like some sort of fund ID in a list with all fund IDs in the http response.
No dice.  Plan C -- Selenium.

### Take 2: Selenium

I used the [RSelenium] R package for this mini project.  There are also Selenium bindings for Python, Java, C#, Javascript and Ruby
which make replicating this process in your programming language of choice relatively straightforward.


**Step 1: Fire up Selenium**


{% highlight r %}
library('RSelenium')
checkForServer() # search for and download Selenium Server java binary.  Only need to run once.
startServer() # run Selenium Server binary
remDr <- remoteDriver(browserName="firefox", port=4444) # instantiate remote driver to connect to Selenium Server
remDr$open(silent=T) # open web browser
{% endhighlight %}


**Step 2: Start scraping**

To figure which DOM elements I wanted Selenium extract, I used the Chrome Developer Tools which can be invoked by right clicking a fund in the table and selecting Inspect Element.  The HTML displayed here contains exactly what we want, what we didn't see with our http request.

Since I want to grab all the funds at once, I tell Selenium to select the whole table.  Going a few levels up from the individual cell in the table I've selected, I see that `<tbody id="tbody">` is the HTML tag that contains the entire table, so I tell Selenium to find this element.  I use the nifty `highlightElement` function to confirm graphically in the browser that this is what I think it is.

Then it's business as usual.  I parse the string output from Selenium into an HTML tree and use XPath to parse the table for just the fund name and ticker.


{% highlight r %}
library('XML')
master <- c()
n <- 5 # number of pages to scrape.  80 pages in total.  I just scraped 5 pages for this example.
for(i in 1:n) {
  site <- paste0("https://www.fidelity.com/fund-screener/evaluator.shtml#!&ntf=N&ft=BAL_all&msrV=advanced&sortBy=FUND_MST_MSTAR_CTGY_NM&pgNo=",i) # create URL for each page to scrape
  remDr$navigate(site) # navigates to webpage
  
  elem <- remDr$findElement(using="id", value="tbody") # get big table in text string
  elem$highlightElement() # just for interactive use in browser.  not necessary.
  elemtxt <- elem$getElementAttribute("outerHTML")[[1]] # gets us the HTML
  elemxml <- htmlTreeParse(elemtxt, useInternalNodes=T) # parse string into HTML tree to allow for querying with XPath
  fundList <- unlist(xpathApply(elemxml, '//input[@title]', xmlGetAttr, 'title')) # parses out just the fund name and ticker using XPath
  master <- c(master, fundList) # append fund lists from each page together
}

head(master)
{% endhighlight %}



{% highlight text %}
## [1] "FidelityAA Global Balanced Fund (FGBLX)"                    
## [2] "FidelityAA Global Strategies Fund (FDYSX)"                  
## [3] "Fidelity FreedomA 2055 Fund (FDEEX)"                       
## [4] "Aberdeen Dynamic Allocation Fund Class A (GMMAX)"           
## [5] "Aberdeen Dynamic Allocation Fund Class C (GMMCX)"           
## [6] "AllianceBernstein Real Asset Strategy Advisor Class (AMTYX)"
{% endhighlight %}

**Step 3: Extract ticker**

Nothing fancy here -- just separating the ticker from the fund name.


{% highlight r %}
master2 <- data.frame(sapply(master, function(x) substr(x, nchar(x)-5, nchar(x)-1)))
master2$name <- sapply(master, function(x) substr(x, 0, nchar(x)-8))
names(master2) <- c('ticker', 'name')

head(master2)
{% endhighlight %}



{% highlight text %}
##   ticker                                                name
## 1  FGBLX                     FidelityAA Global Balanced Fund
## 2  FDYSX                   FidelityAA Global Strategies Fund
## 3  FDEEX                        Fidelity FreedomAA 2055 Fund
## 4  GMMAX            Aberdeen Dynamic Allocation Fund Class A
## 5  GMMCX            Aberdeen Dynamic Allocation Fund Class C
## 6  AMTYX AllianceBernstein Real Asset Strategy Advisor Class
{% endhighlight %}


### What else can Selenium do?

My little example makes use of the simple functionality provided by Selenium for web scraping -- rendering HTML that is dynamically generated with Javascript or Ajax.  Since Selenium is actually a web automation tool, one can be much more sophisticated by using it to automate a human navigating a webpage with mouse clicks and writing and submitting forms.  This can be a huge time saver for researchers that rely on front-end interfaces on the web to extract data in chunks.

[Here's](http://thiagomarzagao.com/2013/11/12/webscraping-with-selenium-part-1/) a basic example using Python.  
[Here's](http://cran.r-project.org/web/packages/RSelenium/vignettes/RSelenium-basics.html) a basic example using R.

### Getting setup

On the several computers I use, I've found setup ranging from seamless to frustrating.

The most frustrating issue I encountered while setting up on my Mac was this error message:


{% highlight r %}
> remDr$open()

[1] "Connecting to remote server"
Error:    Summary: UnknownError
 	 Detail: An unknown server-side error occurred while processing the command.
 	 class: java.lang.IllegalStateException
{% endhighlight %}

I was able to resolve it by killing all processes running on port 4444 and trying again.

At the terminal:

{% highlight r %}
lsof -i :4444
{% endhighlight %}

Kill PIDs of any processes listed.  For example:

{% highlight r %}
kill 30681
{% endhighlight %}




[RSelenium]:http://www.github.com/ropensci/RSelenium/
[Selenium]:http://www.seleniumhq.org/
[phantomjs]:http://phantomjs.org/



