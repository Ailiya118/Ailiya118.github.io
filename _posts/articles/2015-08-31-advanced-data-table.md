---
layout: post
title: "Advanced tips and tricks with data.table"
date: 2015-08-31
categories: articles
tags: [data science, R, data.table, R package, data wrangling]
comments: true
share: true
---
  
* Table of Contents
{:toc}


#### Tips and tricks learned along the way 

This is mostly a running list of `data.table` tricks that took me a while to figure out either by digging into the [official documentation], adapting StackOverflow posts, or more often than not, experimenting for hours.  I'd like to persist these discoveries somewhere with more memory than my head (hello internet) so I can reuse them after my mental memory forgets them.  A less organized and concise addition to DataCamp's sweet [cheat sheet for the basics](https://s3.amazonaws.com/assets.datacamp.com/img/blog/data+table+cheat+sheet.pdf).

Most, if not all of these techniques were developed for real data science projects and provided some value to my data engineering.  I've generalized everything to the `mtcars` dataset which might not make this value immediately clear in this slightly contrived context.  This list is not intended to be comprehensive as DataCamp's data.table cheatsheet is.  OK, enough disclaimers!

Some more advanced functionality from `data.table` creator Matt Dowle [here](http://user2014.stat.ucla.edu/files/tutorial_Matt.pdf).




# 1. DATA STRUCTURES & ASSIGNMENT
---

## Columns of lists
  
##### summary table (long and narrow)
This could be useful, but is easily achievable using traditional methods.


{% highlight r %}
dt <- data.table(mtcars)[, .(cyl, gear)]
dt[,unique(gear), by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl V1
## 1:   6  4
## 2:   6  3
## 3:   6  5
## 4:   4  4
## 5:   4  3
## 6:   4  5
## 7:   8  3
## 8:   8  5
{% endhighlight %}

##### summary table (short and narrow)
Add all categories of `gear` for each `cyl` to original data.table as a list.

This is more nifty.  It's so simple, I find myself using this trick to quickly explore data ad hoc at the command line.
Can also be useful for more serious data engineering.


{% highlight r %}
dt <- data.table(mtcars)[,.(gear, cyl)]
dt[,gearsL:=list(list(unique(gear))), by=cyl] # original, ugly
dt[,gearsL:=.(list(unique(gear))), by=cyl] # improved, pretty
head(dt)
{% endhighlight %}



{% highlight text %}
##    gear cyl gearsL
## 1:    4   6  4,3,5
## 2:    4   6  4,3,5
## 3:    4   4  4,3,5
## 4:    3   6  4,3,5
## 5:    3   8    3,5
## 6:    3   6  4,3,5
{% endhighlight %}

**Update 10/29/2015:** Per [these comments](http://stackoverflow.com/questions/33113013/use-of-list-in-data-tables-j-argument) 
on StackOverlow referencing my post, `t[,gearsL:=list(list(unique(gear))), by=cyl]` can be more elegantly written as `t[,gearsL:=.(list(unique(gear))), by=cyl]`.  Thanks for pointing out my unnecessarily verbose and unusual syntax!  I think I wrote the first thing that worked when I posted this, not realizing the normal `.(` syntax was equivalent to the outer list.

### Accessing elements from a column of lists

Extract second element of each list in `gearL1` and create row `gearL1`.
This isn't that groundbreaking, but explores how to access elements of columns which are constructed of lists of lists.  `lapply` is your friend.


{% highlight r %}
dt[,gearL1:=lapply(gearsL, function(x) x[2])]
dt[,gearS1:=sapply(gearsL, function(x) x[2])] 

head(dt)
{% endhighlight %}



{% highlight text %}
##    gear cyl gearsL gearL1 gearS1
## 1:    4   6  4,3,5      3      3
## 2:    4   6  4,3,5      3      3
## 3:    4   4  4,3,5      3      3
## 4:    3   6  4,3,5      3      3
## 5:    3   8    3,5      5      5
## 6:    3   6  4,3,5      3      3
{% endhighlight %}



{% highlight r %}
str(head(dt[,gearL1])) 
{% endhighlight %}



{% highlight text %}
## List of 6
##  $ : num 3
##  $ : num 3
##  $ : num 3
##  $ : num 3
##  $ : num 5
##  $ : num 3
{% endhighlight %}



{% highlight r %}
str(head(dt[,gearS1]))
{% endhighlight %}



{% highlight text %}
##  num [1:6] 3 3 3 3 5 3
{% endhighlight %}

**Update 9/24/2015:** Per Matt Dowle's comments, a slightly more syntactically succinct way of doing this:


{% highlight r %}
dt[,gearL1:=lapply(gearsL, `[`, 2)]
dt[,gearS1:=sapply(gearsL, `[`, 2)]
{% endhighlight %}

Calculate all the `gear`s for all cars of each `cyl` (excluding the current current row).
This can be useful for comparing observations to the mean of groups, where the group mean is not biased by the observation of interest.


{% highlight r %}
dt[,other_gear:=mapply(function(x, y) setdiff(x, y), x=gearsL, y=gear)]
head(dt)
{% endhighlight %}



{% highlight text %}
##    gear cyl gearsL gearL1 gearS1 other_gear
## 1:    4   6  4,3,5      3      3        3,5
## 2:    4   6  4,3,5      3      3        3,5
## 3:    4   4  4,3,5      3      3        3,5
## 4:    3   6  4,3,5      3      3        4,5
## 5:    3   8    3,5      5      5          5
## 6:    3   6  4,3,5      3      3        4,5
{% endhighlight %}

**Update 9/24/2015:** Per Matt Dowle's comments, this achieves the same as above.


{% highlight r %}
dt[,other_gear:=mapply(setdiff, gearsL, gear)]
{% endhighlight %}
## Suppressing intermediate output with {}

This is actually a base R trick that I didn't discover until working with data.table.  See ``` ?`{` ``` for some documentation and examples.
I've only used it within the J slot of data.table, it might be more generalizable.  I find it pretty useful for generating columns
on the fly when I need to perform some multi-step vectorized operation.  It can clean up code by allowing you to reference the same temporary variable
by a concise name rather than rewriting the code to re-compute it.


{% highlight r %}
dt <- data.table(mtcars)
{% endhighlight %}

Defaults to just returning the last object defined in the braces unnamed.

{% highlight r %}
dt[,{tmp1=mean(mpg); tmp2=mean(abs(mpg-tmp1)); tmp3=round(tmp2, 2)}, by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl   V1
## 1:   6 1.19
## 2:   4 3.83
## 3:   8 1.79
{% endhighlight %}

We can be more explicit by passing a named list of what we want to keep.

{% highlight r %}
dt[,{tmp1=mean(mpg); tmp2=mean(abs(mpg-tmp1)); tmp3=round(tmp2, 2); list(tmp2=tmp2, tmp3=tmp3)}, by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl     tmp2 tmp3
## 1:   6 1.191837 1.19
## 2:   4 3.833058 3.83
## 3:   8 1.785714 1.79
{% endhighlight %}

Can also write it like this without semicolons.

{% highlight r %}
dt[,{tmp1=mean(mpg)
     tmp2=mean(abs(mpg-tmp1))
     tmp3=round(tmp2, 2)
     list(tmp2=tmp2, tmp3=tmp3)},
   by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl     tmp2 tmp3
## 1:   6 1.191837 1.19
## 2:   4 3.833058 3.83
## 3:   8 1.785714 1.79
{% endhighlight %}

This is trickier with `:=` assignments... I don't think `:=` is intended to work when wrapped in `{`.  Assigning multiple columns with `:=` at once
does not allow you to use the first columns you create to use building the ones after it, as we did with `=` inside the `{` above.  Chaining and then dropping unwanted variables is a messy workaround... still exploring this one.


{% highlight r %}
dt <- data.table(mtcars)[,.(cyl, mpg)]

dt[,tmp1:=mean(mpg), by=cyl][,tmp2:=mean(abs(mpg-tmp1)), by=cyl][,tmp1:=NULL]
head(dt)
{% endhighlight %}



{% highlight text %}
##    cyl  mpg     tmp2
## 1:   6 21.0 1.191837
## 2:   6 21.0 1.191837
## 3:   4 22.8 3.833058
## 4:   6 21.4 1.191837
## 5:   8 18.7 1.785714
## 6:   6 18.1 1.191837
{% endhighlight %}


## Fast looping with `set`

I still haven't worked much with the loop + `set` framework.  I've been able to achieve pretty much everything with `:=` which is more flexible and powerful.
However, if you must loop, `set` is orders of magnitude faster than native R assignments within loops.  Here's a snippet from data.table news a while back:


    New function set(DT,i,j,value) allows fast assignment to elements
    of DT. Similar to := but avoids the overhead of [.data.table, so is
    much faster inside a loop. Less flexible than :=, but as flexible
    as matrix sub-assignment. Similar in spirit to setnames(), setcolorder(),
    setkey() and setattr(); i.e., assigns by reference with no copy at all.

    M = matrix(1,nrow=100000,ncol=100)
    DF = as.data.frame(M)
    DT = as.data.table(M)
    system.time(for (i in 1:1000) DF[i,1L] <- i)   # 591.000s
    system.time(for (i in 1:1000) DT[i,V1:=i])     #   1.158s
    system.time(for (i in 1:1000) M[i,1L] <- i)    #   0.016s
    system.time(for (i in 1:1000) set(DT,i,1L,i))  #   0.027s


data.table creators do favor `set` for [some things](http://stackoverflow.com/questions/16846380/how-to-apply-same-function-to-every-specified-column-in-a-data-table), like this task which can also be done w/ `lapply` and `.SD`.  I was actually directed to this solution after I posed [this question](http://stackoverflow.com/questions/31326691/apply-function-across-subset-of-columns-in-data-table-with-sdcols) on StackOverflow.  I was also pleased to learn that the 
functionality I was looking for -- applying a function to a subset of columns with `.SDcols` while preserving the untouched columns -- was added as a feature request. 


{% highlight r %}
dt <- data.table(mtcars)[,1:5, with=F]
for (j in c(1L,2L,4L)) set(dt, j=j, value=-dt[[j]]) # integers using 'L' passed for efficiency
for (j in c(3L,5L)) set(dt, j=j, value=paste0(dt[[j]],'!!'))
head(dt)
{% endhighlight %}



{% highlight text %}
##      mpg cyl  disp   hp   drat
## 1: -21.0  -6 160!! -110  3.9!!
## 2: -21.0  -6 160!! -110  3.9!!
## 3: -22.8  -4 108!!  -93 3.85!!
## 4: -21.4  -6 258!! -110 3.08!!
## 5: -18.7  -8 360!! -175 3.15!!
## 6: -18.1  -6 225!! -105 2.76!!
{% endhighlight %}


## Using `shift` for to lead/lag vectors and lists

Note this feature is only available in version 1.9.5 (currently on Github, not CRAN)
Base R surprisingly does not have great tools for dealing with leads/lags of vectors that most social science
statistical software (Stata, SAS, even FAME which I used in my formative data years) come equipped with out of the box.


{% highlight r %}
dt <- data.table(mtcars)[,.(mpg, cyl)]
dt[,mpg_lag1:=shift(mpg, 1)]
dt[,mpg_forward1:=shift(mpg, 1, type='lead')]
head(dt)
{% endhighlight %}



{% highlight text %}
##     mpg cyl mpg_lag1 mpg_forward1
## 1: 21.0   6       NA         21.0
## 2: 21.0   6     21.0         22.8
## 3: 22.8   4     21.0         21.4
## 4: 21.4   6     22.8         18.7
## 5: 18.7   8     21.4         18.1
## 6: 18.1   6     18.7         14.3
{% endhighlight %}

#### `shift` with `by`


{% highlight r %}
# creating some data
n <- 30
dt <- data.table(
  date=rep(seq(as.Date('2010-01-01'), as.Date('2015-01-01'), by='year'), n/6), 
  ind=rpois(n, 5),
  entity=sort(rep(letters[1:5], n/5))
  )

setkey(dt, entity, date) # important for ordering
dt[,indpct_fast:=(ind/shift(ind, 1))-1, by=entity]

lagpad <- function(x, k) c(rep(NA, k), x)[1:length(x)] 
dt[,indpct_slow:=(ind/lagpad(ind, 1))-1, by=entity]

head(dt, 10)
{% endhighlight %}



{% highlight text %}
##  1: 2010-01-01   3      a          NA          NA
##  2: 2011-01-01   2      a  -0.3333333  -0.3333333
##  3: 2012-01-01   5      a   1.5000000   1.5000000
##  4: 2013-01-01   4      a  -0.2000000  -0.2000000
##  5: 2014-01-01   1      a  -0.7500000  -0.7500000
##  6: 2015-01-01   5      a   4.0000000   4.0000000
##  7: 2010-01-01   2      b          NA          NA
##  8: 2011-01-01   6      b   2.0000000   2.0000000
##  9: 2012-01-01   8      b   0.3333333   0.3333333
## 10: 2013-01-01   9      b   0.1250000   0.1250000
{% endhighlight %}

## Create multiple columns with `:=` in one statement

This is useful, but note that that the columns operated on must be atomic vectors or lists.  That is they must exist before running computation.  
Building columns referencing other columns in this set need to be done individually or chained.

{% highlight r %}
dt <- data.table(mtcars)[,.(mpg, cyl)]
dt[,`:=`(avg=mean(mpg), med=median(mpg), min=min(mpg)), by=cyl]
head(dt)
{% endhighlight %}



{% highlight text %}
##     mpg cyl      avg  med  min
## 1: 21.0   6 19.74286 19.7 17.8
## 2: 21.0   6 19.74286 19.7 17.8
## 3: 22.8   4 26.66364 26.0 21.4
## 4: 21.4   6 19.74286 19.7 17.8
## 5: 18.7   8 15.10000 15.2 10.4
## 6: 18.1   6 19.74286 19.7 17.8
{% endhighlight %}

## Assign a column with `:=` named with a character object

This is the advised way to assign a new column whose name you already have determined and saved as a character.  Simply surround the character object in parentheses.  


{% highlight r %}
dt <- data.table(mtcars)[, .(cyl, mpg)]

thing2 <- 'mpgx2'
dt[,(thing2):=mpg*2]

head(dt)
{% endhighlight %}



{% highlight text %}
##    cyl  mpg mpgx2
## 1:   6 21.0  42.0
## 2:   6 21.0  42.0
## 3:   4 22.8  45.6
## 4:   6 21.4  42.8
## 5:   8 18.7  37.4
## 6:   6 18.1  36.2
{% endhighlight %}

This is old (now deprecated) way which still works for now.  Not advised.

{% highlight r %}
thing3 <- 'mpgx3'
dt[,thing3:=mpg*3, with=F]

head(dt)
{% endhighlight %}



{% highlight text %}
##    cyl  mpg mpgx2 mpgx3
## 1:   6 21.0  42.0  63.0
## 2:   6 21.0  42.0  63.0
## 3:   4 22.8  45.6  68.4
## 4:   6 21.4  42.8  64.2
## 5:   8 18.7  37.4  56.1
## 6:   6 18.1  36.2  54.3
{% endhighlight %}

# 2. `BY`
---

## Calculate a function over a group (using `by`) excluding each entity in a second category.

This title probably doesn't immediately make much sense.  Let me explain what I'm going to calculate and why with an example.
We want to compare the `mpg` of each car to the average `mpg` of cars in the same class (the same # of cylinders).  However, we don't want 
to bias the group mean by including the car we want to compare to the average in that average.  

This assumption doesn't appear useful in this example, but assume that `gear`+`cyl` uniquely identify the cars.  In the real project where I faced this 
problem, I was calculating an indicator related to an appraiser relative to the average of all other appraisers in their zip3. (`cyl` was really zipcode
and `gear` was the appraiser's ID).

### METHOD 1: in-line

##### 0.a Biased mean: simple mean by `cyl`
However we want to know for each row, what is the mean among all the other cars with the same # of `cyl`s, excluding that car.


{% highlight r %}
dt <- data.table(mtcars)[,.(cyl, gear, mpg)]
dt[, mpg_biased_mean:=mean(mpg), by=cyl] 
head(dt)
{% endhighlight %}



{% highlight text %}
##    cyl gear  mpg mpg_biased_mean
## 1:   6    4 21.0        19.74286
## 2:   6    4 21.0        19.74286
## 3:   4    4 22.8        26.66364
## 4:   6    3 21.4        19.74286
## 5:   8    3 18.7        15.10000
## 6:   6    3 18.1        19.74286
{% endhighlight %}

#####  1.a `.GRP` without setting key


{% highlight r %}
dt[, dt[!gear %in% unique(dt$gear)[.GRP], mean(mpg), by=cyl], by=gear] #unbiased mean
{% endhighlight %}



{% highlight text %}
##    gear cyl       V1
## 1:    4   6 19.73333
## 2:    4   8 15.10000
## 3:    4   4 25.96667
## 4:    3   6 19.74000
## 5:    3   4 27.18000
## 6:    3   8 15.40000
## 7:    5   6 19.75000
## 8:    5   4 26.32222
## 9:    5   8 15.05000
{% endhighlight %}



{% highlight r %}
# check
dt[gear!=4 & cyl==6, mean(mpg)]
{% endhighlight %}



{% highlight text %}
## [1] 19.73333
{% endhighlight %}

**Update 9/24/2015:** Per Matt Dowle's comments, this also works with slightly less code. For my simple example, there was also a marginal speed gain.  Time savings relative to the `.GRP` method will likely increase with the complexity of the problem.


{% highlight r %}
dt[, dt[!gear %in% .BY[[1]], mean(mpg), by=cyl], by=gear] #unbiased mean
{% endhighlight %}



{% highlight text %}
##    gear cyl       V1
## 1:    4   6 19.73333
## 2:    4   8 15.10000
## 3:    4   4 25.96667
## 4:    3   6 19.74000
## 5:    3   4 27.18000
## 6:    3   8 15.40000
## 7:    5   6 19.75000
## 8:    5   4 26.32222
## 9:    5   8 15.05000
{% endhighlight %}

##### 1.b Same as 1.a, but a little faster


{% highlight r %}
uid <- unique(dt$gear)
dt[, dt[!gear %in% (uid[.GRP]), mean(mpg), by=cyl] , by=gear][order(cyl, gear)] #unbiased mean
{% endhighlight %}



{% highlight text %}
##    gear cyl       V1
## 1:    3   4 27.18000
## 2:    4   4 25.96667
## 3:    5   4 26.32222
## 4:    3   6 19.74000
## 5:    4   6 19.73333
## 6:    5   6 19.75000
## 7:    3   8 15.40000
## 8:    4   8 15.10000
## 9:    5   8 15.05000
{% endhighlight %}

##### Why does this work?


{% highlight r %}
# 1.a pulling it apart with .GRP
dt[, .GRP, by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl GRP
## 1:   6   1
## 2:   4   2
## 3:   8   3
{% endhighlight %}



{% highlight r %}
dt[, .(.GRP, unique(dt$gear)[.GRP]), by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl GRP V2
## 1:   6   1  4
## 2:   4   2  3
## 3:   8   3  5
{% endhighlight %}



{% highlight r %}
dt[,dt[, .(.GRP, unique(dt$gear)[.GRP]), by=cyl], by=gear]
{% endhighlight %}



{% highlight text %}
##    gear cyl GRP V2
## 1:    4   6   1  4
## 2:    4   4   2  3
## 3:    4   8   3  5
## 4:    3   6   1  4
## 5:    3   4   2  3
## 6:    3   8   3  5
## 7:    5   6   1  4
## 8:    5   4   2  3
## 9:    5   8   3  5
{% endhighlight %}

##### 1.b Setting key

{% highlight r %}
setkey(dt, gear)
uid <- unique(dt$gear)
dt[, dt[!.(uid[.GRP]), mean(mpg), by=cyl] , by=gear] #unbiased mean
{% endhighlight %}



{% highlight text %}
##    gear cyl       V1
## 1:    3   6 19.74000
## 2:    3   4 27.18000
## 3:    3   8 15.40000
## 4:    4   6 19.73333
## 5:    4   8 15.10000
## 6:    4   4 25.96667
## 7:    5   6 19.75000
## 8:    5   8 15.05000
## 9:    5   4 26.32222
{% endhighlight %}



{% highlight r %}
mean(dt[cyl==4 & gear!=3,mpg]) # testing
{% endhighlight %}



{% highlight text %}
## [1] 27.18
{% endhighlight %}



{% highlight r %}
mean(dt[cyl==6 & gear!=3,mpg]) # testing
{% endhighlight %}



{% highlight text %}
## [1] 19.74
{% endhighlight %}

### METHOD 2: using `{}` and `.SD`
`{}` is used for to suppress intermediate operations.

##### Building up
No surprises here.


{% highlight r %}
dt[,  .SD[, mean(mpg)], by=gear] # same as `dt[, mean(mpg), by=gear]`
{% endhighlight %}



{% highlight text %}
##    gear       V1
## 1:    3 16.10667
## 2:    4 24.53333
## 3:    5 21.38000
{% endhighlight %}



{% highlight r %}
dt[,  .SD[, mean(mpg), by=cyl], by=gear] # same as `dt[, mean(mpg), by=.(cyl, by=gear)]`
{% endhighlight %}



{% highlight text %}
##    gear cyl     V1
## 1:    3   6 19.750
## 2:    3   8 15.050
## 3:    3   4 21.500
## 4:    4   6 19.750
## 5:    4   4 26.925
## 6:    5   4 28.200
## 7:    5   8 15.400
## 8:    5   6 19.700
{% endhighlight %}

##### Nested data.tables and `by` statements
This chunk shows what happens with two `by` statements nested within two different data.tables.  Explanatory purposes only - not necessary for our task.
`n` counts the # of cars in that `cyl`.  `N` counts the number of cars by `cyl` and `gear`.


{% highlight r %}
dt[,{
  vbar = sum(mpg)
  n = .N
  .SD[,.(n, .N, sum_in_gear_cyl=sum(mpg), sum_in_cyl=vbar), by=gear]
} , by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl gear  n  N sum_in_gear_cyl sum_in_cyl
## 1:   6    3  7  2            39.5      138.2
## 2:   6    4  7  4            79.0      138.2
## 3:   6    5  7  1            19.7      138.2
## 4:   8    3 14 12           180.6      211.4
## 5:   8    5 14  2            30.8      211.4
## 6:   4    3 11  1            21.5      293.3
## 7:   4    4 11  8           215.4      293.3
## 8:   4    5 11  2            56.4      293.3
{% endhighlight %}



{% highlight r %}
dt[,sum(mpg), by=cyl] # test
{% endhighlight %}



{% highlight text %}
##    cyl    V1
## 1:   6 138.2
## 2:   8 211.4
## 3:   4 293.3
{% endhighlight %}

##### Calculating "unbiased mean"
This is in a summary table.  This would need to be merged back onto `dt` if that is desired.

{% highlight r %}
dt[,{
  vbar = mean(mpg)
  n = .N
  .SD[,(n*vbar-sum(mpg))/(n-.N),by=gear]
} , by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl gear       V1
## 1:   6    3 19.74000
## 2:   6    4 19.73333
## 3:   6    5 19.75000
## 4:   8    3 15.40000
## 5:   8    5 15.05000
## 6:   4    3 27.18000
## 7:   4    4 25.96667
## 8:   4    5 26.32222
{% endhighlight %}


### METHOD 3: Super Fast Mean calculation

##### Non-function direct way
Using a vectorized approach to calculate the unbiased mean for each combination of `gear` and `cyl`.  Mechanically,
it calculates the "biased average" for all cars by `cyl`.  Then subtract off the share of cars with the combination of `gear` and `cyl` 
that we want to exclude from the average and add that share.  Then extrapolate out this pared down mean.


{% highlight r %}
dt <- data.table(mtcars)[,.(mpg,cyl,gear)]
dt[,`:=`(avg_mpg_cyl=mean(mpg), Ncyl=.N), by=cyl]
dt[,`:=`(Ncylgear=.N, avg_mpg_cyl_gear=mean(mpg)), by=.(cyl, gear)]
dt[,unbmean:=(avg_mpg_cyl*Ncyl-(Ncylgear*avg_mpg_cyl_gear))/(Ncyl-Ncylgear)]
setkey(dt, cyl, gear)  
head(dt)
{% endhighlight %}



{% highlight text %}
##     mpg cyl gear avg_mpg_cyl Ncyl Ncylgear avg_mpg_cyl_gear  unbmean
## 1: 21.5   4    3    26.66364   11        1           21.500 27.18000
## 2: 22.8   4    4    26.66364   11        8           26.925 25.96667
## 3: 24.4   4    4    26.66364   11        8           26.925 25.96667
## 4: 22.8   4    4    26.66364   11        8           26.925 25.96667
## 5: 32.4   4    4    26.66364   11        8           26.925 25.96667
## 6: 30.4   4    4    26.66364   11        8           26.925 25.96667
{% endhighlight %}

##### Wrapping up code below into a function


{% highlight r %}
leaveOneOutMean <- function(dt, ind, bybig, bysmall) {
  dtmp <- copy(dt) # copy so as not to alter original dt object w intermediate assignments
  dtmp <- dtmp[is.na(get(ind))==F,]
  dtmp[,`:=`(avg_ind_big=mean(get(ind)), Nbig=.N), by=.(get(bybig))]
  dtmp[,`:=`(Nbigsmall=.N, avg_ind_big_small=mean(get(ind))), by=.(get(bybig), get(bysmall))]
  dtmp[,unbmean:=(avg_ind_big*Nbig-(Nbigsmall*avg_ind_big_small))/(Nbig-Nbigsmall)]
  return(dtmp[,unbmean])
}

dt <- data.table(mtcars)[,.(mpg,cyl,gear)]
dt[,unbiased_mean:=leaveOneOutMean(.SD, ind='mpg', bybig='cyl', bysmall='gear')]
dt[,biased_mean:=mean(mpg), by=cyl]
head(dt)
{% endhighlight %}



{% highlight text %}
##     mpg cyl gear unbiased_mean biased_mean
## 1: 21.0   6    4      19.73333    19.74286
## 2: 21.0   6    4      19.73333    19.74286
## 3: 22.8   4    4      25.96667    26.66364
## 4: 21.4   6    3      19.74000    19.74286
## 5: 18.7   8    3      15.40000    15.10000
## 6: 18.1   6    3      19.74000    19.74286
{% endhighlight %}

### Speed check

Method 3 is roughly 100x faster than the other two.  Great for this narrow task with the vectorization built in, 
but less generalizable; The other two methods allow any function to be passed.


{% highlight r %}
dt <- data.table(mtcars)
dt <- dt[sample(1:.N, 100000, replace=T), ] # increase # of rows in mtcars
dt$gear <- sample(1:300, nrow(dt), replace=T) # adding in more cateogries
{% endhighlight %}

##### Method 3:


{% highlight r %}
system.time(dt[,unbiased_mean_vectorized:=leaveOneOutMean(.SD, ind='mpg', bybig='cyl', bysmall='gear')])
{% endhighlight %}



{% highlight text %}
##    user  system elapsed 
##   0.033   0.003   0.035
{% endhighlight %}

##### Method 2:


{% highlight r %}
system.time(tmp <- dt[,dt[!gear %in% unique(dt$gear)[.GRP], mean(mpg), by=cyl], by=gear] )
{% endhighlight %}



{% highlight text %}
##    user  system elapsed 
##   3.709   0.359   4.069
{% endhighlight %}

##### Method 1:

{% highlight r %}
uid <- unique(dt$gear)
system.time(dt[, dt[!gear %in% (uid[.GRP]), mean(mpg), by=cyl] , by=gear][order(cyl, gear)])
{% endhighlight %}



{% highlight text %}
##    user  system elapsed 
##   3.345   0.331   3.677
{% endhighlight %}

## `keyby` to key resulting aggregate table

##### Without `keyby`
Categories are not sorted


{% highlight r %}
## devtools::install_github('brooksandrew/Rsenal')
library('Rsenal') # grabbing depthbin function
tmp <- dt[, .(N=.N, sum=sum(vs), mean=mean(vs)/.N), by=depthbin(mpg, 5, labelOrder=T)]
tmp
{% endhighlight %}



{% highlight text %}
##           depthbin     N   sum         mean
## 1: (15.2,17.8] 2/5 15372  3131 1.325020e-05
## 2:   (17.8,21] 3/5 21839  6204 1.300787e-05
## 3: [10.4,15.2] 1/5 25255     0 0.000000e+00
## 4:   (21,24.4] 4/5 18817 18817 5.314343e-05
## 5: (24.4,33.9] 5/5 18717 15581 4.447571e-05
{% endhighlight %}



{% highlight r %}
tmp[,barplot(mean, names=depthbin, las=2)]
{% endhighlight %}

![plot of chunk barplot 1](/assets/svg/2015_08_31_datatable/mean_barchart_1.svg)  

{% highlight text %}
##      [,1]
## [1,]  0.7
## [2,]  1.9
## [3,]  3.1
## [4,]  4.3
## [5,]  5.5
{% endhighlight %}

##### With `keyby`

{% highlight r %}
## devtools::install_github('brooksandrew/Rsenal')
library('Rsenal')
tmp <- dt[, .(N=.N, sum=sum(vs), mean=mean(vs)/.N), keyby=depthbin(mpg, 5, labelOrder=T)]
tmp
{% endhighlight %}



{% highlight text %}
##           depthbin     N   sum         mean
## 1: [10.4,15.2] 1/5 25255     0 0.000000e+00
## 2: (15.2,17.8] 2/5 15372  3131 1.325020e-05
## 3:   (17.8,21] 3/5 21839  6204 1.300787e-05
## 4:   (21,24.4] 4/5 18817 18817 5.314343e-05
## 5: (24.4,33.9] 5/5 18717 15581 4.447571e-05
{% endhighlight %}



{% highlight r %}
tmp[,barplot(mean, names=depthbin, las=2)]
{% endhighlight %}

![plot of chunk barplot 2](/assets/svg/2015_08_31_datatable/mean_barchart_2.svg) 

{% highlight text %}
##      [,1]
## [1,]  0.7
## [2,]  1.9
## [3,]  3.1
## [4,]  4.3
## [5,]  5.5
{% endhighlight %}

## Using `[1]`, `[.N]`, `setkey` and `by` for within group subsetting

#### take highest value of column A when column B is highest by group

Max of `qsec` for each category of `cyl`
(this is easy)


{% highlight r %}
dt <- data.table(mtcars)[, .(cyl, mpg, qsec)]
dt[, max(qsec), by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl    V1
## 1:   6 20.22
## 2:   4 22.90
## 3:   8 18.00
{% endhighlight %}

##### value of `qsec `when `mpg` is the highest per category of `cyl`
(this is trickier)


{% highlight r %}
setkey(dt, mpg)
dt[,qsec[.N],  by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl    V1
## 1:   8 17.05
## 2:   6 19.44
## 3:   4 19.90
{% endhighlight %}

##### value of `qsec` when `mpg` is the lowest per category of `cyl`

{% highlight r %}
dt[,qsec[1],  by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl    V1
## 1:   8 17.98
## 2:   6 18.90
## 3:   4 18.60
{% endhighlight %}

##### value of `qsec` when `mpg` is the median per category of `cyl`

{% highlight r %}
dt[,qsec[round(.N/2)],  by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl   V1
## 1:   8 18.0
## 2:   6 15.5
## 3:   4 16.7
{% endhighlight %}

##### subset rows within by statement 
`V1` is the standard deviation of `mpg` by `cyl`.  `V2` is the standard deviation of `mpg` for just the first half of `mpg`.


{% highlight r %}
dt <- data.table(mtcars)
setkey(dt,mpg)
dt[, .(sd(mpg), sd(mpg[1:round(.N/2)])), by=cyl]
{% endhighlight %}



{% highlight text %}
##    cyl       V1        V2
## 1:   8 2.560048 2.0926174
## 2:   6 1.453567 0.8981462
## 3:   4 4.509828 1.7728508
{% endhighlight %}

# 3. FUNCTIONS
---

## Passing `data.table` column names as function arguments 

#### Method 1: No quotes, and `deparse` + `substitute`

This way seems more data.table-ish because it maintains the practice of not using quotes on variable names in most cases.


{% highlight r %}
dt <- data.table(mtcars)[,.(cyl, mpg)]
myfunc <- function(dt, v) {
  v2=deparse(substitute(v))
  dt[,v2, with=F][[1]] # [[1]] returns a vector instead of a data.table
}

myfunc(dt, mpg)
{% endhighlight %}



{% highlight text %}
##  [1] 21.0 21.0 22.8 21.4 18.7 18.1 14.3 24.4 22.8 19.2 17.8 16.4 17.3 15.2
## [15] 10.4 10.4 14.7 32.4 30.4 33.9 21.5 15.5 15.2 13.3 19.2 27.3 26.0 30.4
## [29] 15.8 19.7 15.0 21.4
{% endhighlight %}

### Method 2: quotes and `get`

However I tend to pass through column names as characters (quoted) and use `get` each time I reference that column.  That can be annoying if you have a long function
repeatedly reference column names, but I often need to write such few lines of code with data.table, it hasn't struck me as terribly unslick, yet.


{% highlight r %}
dt <- data.table(mtcars)
myfunc <- function(dt, v) dt[,get(v)]

myfunc(dt, 'mpg')
{% endhighlight %}



{% highlight text %}
##  [1] 21.0 21.0 22.8 21.4 18.7 18.1 14.3 24.4 22.8 19.2 17.8 16.4 17.3 15.2
## [15] 10.4 10.4 14.7 32.4 30.4 33.9 21.5 15.5 15.2 13.3 19.2 27.3 26.0 30.4
## [29] 15.8 19.7 15.0 21.4
{% endhighlight %}

## Beware of scoping within data.table

### `data.frame` way
When you add something to a `data.frame` within a function that exists in the global environment, it does not affect that object in the 
global environment unless you return and reassign it as such, or you use the `<<-` operator.  


{% highlight r %}
df <- mtcars[,c('cyl', 'mpg')]
add_column_df <- function(df) {
  df$addcol1<- 'here in func!'
  df$addcol2 <<- 'in glob env!'
  return(df)
}
{% endhighlight %}

When we call the function, we see `addcol1` in the output.  But not `addcol2`.  That's because it's been added to the `df` in the global environment one level up.

{% highlight r %}
head(add_column_df(df))
{% endhighlight %}



{% highlight text %}
##                   cyl  mpg       addcol1
## Mazda RX4           6 21.0 here in func!
## Mazda RX4 Wag       6 21.0 here in func!
## Datsun 710          4 22.8 here in func!
## Hornet 4 Drive      6 21.4 here in func!
## Hornet Sportabout   8 18.7 here in func!
## Valiant             6 18.1 here in func!
{% endhighlight %}

Here is `addcol2`, but not `addcol`.

{% highlight r %}
head(df)
{% endhighlight %}



{% highlight text %}
##                   cyl  mpg      addcol2
## Mazda RX4           6 21.0 in glob env!
## Mazda RX4 Wag       6 21.0 in glob env!
## Datsun 710          4 22.8 in glob env!
## Hornet 4 Drive      6 21.4 in glob env!
## Hornet Sportabout   8 18.7 in glob env!
## Valiant             6 18.1 in glob env!
{% endhighlight %}

### `data.table` way

Unlike data.frame, the `:=` operator adds a column to both the object living in the global environment and used in the function.  I think this is because
these objects are actually the same object.  data.table shaves computation time by not making copies unless explicitly directed to.


{% highlight r %}
dt <- data.table(mtcars)
add_column_dt <- function(dat) {
  dat[,addcol:='sticking_to_dt!'] # hits dt in glob env
  return(dat)
}
head(add_column_dt(dt)) # addcol here
{% endhighlight %}



{% highlight text %}
##     mpg cyl disp  hp drat    wt  qsec vs am gear carb          addcol
## 1: 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4 sticking_to_dt!
## 2: 21.0   6  160 110 3.90 2.875 17.02  0  1    4    4 sticking_to_dt!
## 3: 22.8   4  108  93 3.85 2.320 18.61  1  1    4    1 sticking_to_dt!
## 4: 21.4   6  258 110 3.08 3.215 19.44  1  0    3    1 sticking_to_dt!
## 5: 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2 sticking_to_dt!
## 6: 18.1   6  225 105 2.76 3.460 20.22  1  0    3    1 sticking_to_dt!
{% endhighlight %}



{% highlight r %}
head(dt) # addcol also here
{% endhighlight %}



{% highlight text %}
##     mpg cyl disp  hp drat    wt  qsec vs am gear carb          addcol
## 1: 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4 sticking_to_dt!
## 2: 21.0   6  160 110 3.90 2.875 17.02  0  1    4    4 sticking_to_dt!
## 3: 22.8   4  108  93 3.85 2.320 18.61  1  1    4    1 sticking_to_dt!
## 4: 21.4   6  258 110 3.08 3.215 19.44  1  0    3    1 sticking_to_dt!
## 5: 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2 sticking_to_dt!
## 6: 18.1   6  225 105 2.76 3.460 20.22  1  0    3    1 sticking_to_dt!
{% endhighlight %}

So something like this renaming the local version using `copy` bypasses this behavior, but is likely somewhat less efficient (and elegant).  I suspect there's a cleaner and/or faster way to do this: keep some variables 
local to the function while persisting and returning other columns.


{% highlight r %}
dt <- data.table(mtcars)
add_column_dt <- function(dat) {
  datloc <- copy(dat)
  datloc[,addcol:='not sticking_to_dt!'] # hits dt in glob env
  return(datloc)
}
head(add_column_dt(dt)) # addcol here
{% endhighlight %}



{% highlight text %}
##     mpg cyl disp  hp drat    wt  qsec vs am gear carb              addcol
## 1: 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4 not sticking_to_dt!
## 2: 21.0   6  160 110 3.90 2.875 17.02  0  1    4    4 not sticking_to_dt!
## 3: 22.8   4  108  93 3.85 2.320 18.61  1  1    4    1 not sticking_to_dt!
## 4: 21.4   6  258 110 3.08 3.215 19.44  1  0    3    1 not sticking_to_dt!
## 5: 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2 not sticking_to_dt!
## 6: 18.1   6  225 105 2.76 3.460 20.22  1  0    3    1 not sticking_to_dt!
{% endhighlight %}



{% highlight r %}
head(dt) # addcol not here
{% endhighlight %}



{% highlight text %}
##     mpg cyl disp  hp drat    wt  qsec vs am gear carb
## 1: 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
## 2: 21.0   6  160 110 3.90 2.875 17.02  0  1    4    4
## 3: 22.8   4  108  93 3.85 2.320 18.61  1  1    4    1
## 4: 21.4   6  258 110 3.08 3.215 19.44  1  0    3    1
## 5: 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2
## 6: 18.1   6  225 105 2.76 3.460 20.22  1  0    3    1
{% endhighlight %}

# 4. PRINTING
---

## Print data.table with `[]`

Nothing groundbreaking here, but a small miscellaneous piece of functionality.
In `data.frame` world, wrapping an expression in `()` prints the output to the console.  This also works with data.table, but there is another way.
In `data.table` this is achieved by appending `[]` to the end of the expression.  I find this useful because when I'm exploring at the console, I 
don't usually decide to print the output until I'm almost done and I'm already at the end of the expression I've written.


{% highlight r %}
# data.frame way of printing after an assignment
df <- head(mtcars) # doesn't print
(df <- head(mtcars)) # does print
{% endhighlight %}



{% highlight text %}
##                    mpg cyl disp  hp drat    wt  qsec vs am gear carb
## Mazda RX4         21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
## Mazda RX4 Wag     21.0   6  160 110 3.90 2.875 17.02  0  1    4    4
## Datsun 710        22.8   4  108  93 3.85 2.320 18.61  1  1    4    1
## Hornet 4 Drive    21.4   6  258 110 3.08 3.215 19.44  1  0    3    1
## Hornet Sportabout 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2
## Valiant           18.1   6  225 105 2.76 3.460 20.22  1  0    3    1
{% endhighlight %}



{% highlight r %}
# data.table way of printing after an assignment
dt <- data.table(head(mtcars)) # doesn't print
dt[,hp2wt:=hp/wt][] # does print
{% endhighlight %}



{% highlight text %}
##     mpg cyl disp  hp drat    wt  qsec vs am gear carb    hp2wt
## 1: 21.0   6  160 110 3.90 2.620 16.46  0  1    4    4 41.98473
## 2: 21.0   6  160 110 3.90 2.875 17.02  0  1    4    4 38.26087
## 3: 22.8   4  108  93 3.85 2.320 18.61  1  1    4    1 40.08621
## 4: 21.4   6  258 110 3.08 3.215 19.44  1  0    3    1 34.21462
## 5: 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2 50.87209
## 6: 18.1   6  225 105 2.76 3.460 20.22  1  0    3    1 30.34682
{% endhighlight %}
 

## Hide output from `:=` with knitr

It used to be that assignments using the `:=` operator printed the object to console when knitting documents with `knitr` and `rmarkdown`.  This is actually fixed in data.table v1.9.5.  However at the time of my writing, this currently not available on CRAN... only Github.  For 1.9.4 users, [this StackOverflow post](http://stackoverflow.com/questions/15267018/knitr-gets-tricked-by-data-table-assignment) has some hacky solutions.  This least impedance approach I found was simply wrapping
the expression in `invisible`.  Other solutions alter the way you use data.table which I didn't like.


{% highlight r %}
dt <- data.table(mtcars)
dt[,mpg2qsec:=mpg/qsec] # will print with knitr
invisible(dt[,mpg2qsec:=mpg/qsec]) # won't print with knitr
{% endhighlight %}


[official documentation]:https://cran.r-project.org/web/packages/data.table/data.table.pdf


