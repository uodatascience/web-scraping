-   [Introduction](#introduction)
-   [Disclaimer!](#disclaimer)
-   [rvest](#rvest)
-   [HTML and CSS selectors](#html-and-css-selectors)
-   [Scraping Sports Data](#scraping-sports-data)
-   [Examining Trustpilot reviews](#examining-trustpilot-reviews)
    -   [Thinking before doing.](#thinking-before-doing.)
    -   [Finding the last page of a review](#finding-the-last-page-of-a-review)
    -   [Extracting Review Information from One page](#extracting-review-information-from-one-page)
    -   [Combining our functions!](#combining-our-functions)
-   [Minihacks](#minihacks)
    -   [Minihack 1](#minihack-1)
    -   [Minihack 2](#minihack-2)
    -   [Minihack 3](#minihack-3)

Introduction
============

This presentation is designed to give a cursory overview of the methods involved in extracting data from the web. Web scraping, also known as web crawling or web harvesting, is a technique used to obtain information from a website that is not available in a downloadable form. This is a topic that is unlike any that we have covered so far it in that it requires the use of tools outside of R and R studio and also requires a cursory knowledge of how information on the web is stored. In order to access the text or images of a website, we must dig through the underlying html code that serves as the scaffold of the website and pull out the bits and pieces that we need.

We will first explore the tools included in the ‘rvest’ package that will allow us to navigate the messy structure of the web. This will be followed by a surface level tutorial of how html code is structured and discuss the ways in which we can use tools such as developer tools, search functions, and web browser extensions to navigate through the code and find what we need. We will then go through two working examples that show different perspectives and techniques that can both achieve similar results.

Disclaimer!
===========

Some websites don’t approve of web scraping and have systems in place that identify the difference between a user accessing the website through a point and click interface and a program that is designed to access the page solely to pull information from it. Facebook for example has been known to ban people from using facebook for scraping data. Google also has safeguards in place to prevent scraping as well. Make sure you research the guidelines that are in place for a particular site before trying to extract data from it.

rvest
=====

`html(x)` or `read_html(x)`

Takes a url as its input and parses the html code that is associated with that site. Note that `html()` is deprecated.

`html_nodes(x, css, xpath)`

Turns each HTML tag into a row in an R dataframe. Takes parsed html code as an input and requires either a css selector or an xpath as an argument. This is how you tell R what you want to pull from the website that you have identified with html() or read\_html(). Css selectors will be explained below and xpath will not be explained here because it is used for xml.

`html_text(x)`

Takes a HTML tags, derived from html\_nodes(), as a parameter and extracts text from the corresponding tag(s).

``` r
# Store web url
library(rvest)
kingdom <- read_html("http://www.imdb.com/title/tt0320661/")

#Scrape the website for the movie rating
rating <- kingdom %>% 
  html_nodes("strong span") %>%
  html_text() 
rating
```

    ## [1] "7.2"

`html_attrs()`

Takes a node or nodes as a parameter and extracts the attributes from them. Can be useful for debugging.

``` r
#html_attrs() instead of html_text()
rating <- kingdom %>% 
  html_nodes("strong span") %>%
  html_attrs() 
rating
```

    ## [[1]]
    ##      itemprop 
    ## "ratingValue"

`html_session()`

Alternative to using html() or read\_html(). This function essentially opens a live browser session that allows you to do things such as “clicking buttons” and using the back page and forward page options like you would if you were were actually browsing the internet.

`follow_link()` `jump_to()` `back()` `forward()`

`html_table()`

Scrape whole HTML tables of data as a dataframe

``` r
s <- html_session("http://hadley.nz")
s %>% jump_to("hadley-wickham.jpg") %>% back() %>% session_history()
```

    ## - http://hadley.nz/
    ##   http://hadley.nz/hadley-wickham.jpg

``` r
s %>% follow_link(css = "p a")
```

    ## <session> https://www.rstudio.com/
    ##   Status: 200
    ##   Type:   text/html; charset=UTF-8
    ##   Size:   653179

HTML and CSS selectors
======================

A css selector is a tag that is used to identify the information within the html code that you need. You rarely want to just pull one number or bit of text from a site or else you would just copy and paste it. The data that you want is often structured in repetitive ways through the use of “tags” and “class” identifiers (to name a few) that allow you to pull all data that has certain qualities in common.

This is a website that is highly recommended by Hadley Wickham for understanding how css selectors work.

flukeout.github.io (demo)

[Selector Gadget](http://selectorgadget.com/)

The developer tools in the Google Chrome browser can also be incredibly helpful to identify relevant CSS tags.

Scraping Sports Data
====================

Sports data is a good proof of concept because there are lots of numbers to work with. We will start with week 1 of the regular season and scrape all of the NFL teams that played that week and then also pull the total and quarter scores for each of those teams. Using a for loop we will navigate page by page through each week of the season until we have the teams and scores for the entire season. The data will then take a bit of wrangling to get it into a usable form so we can perform some simple descriptive statistics on it.

``` r
#Load packages
library(rvest)
library(tidyverse)

#Establish the session with the url for the first page  
link <- html_session("http://www.nfl.com/scores/2017/REG1")

#create an iteration variable that represents the amount of pages you will need to access
num_weeks <- 17

#each week has a different amount of games and so I make empty lists to put the dataframes in for each page
#I have one for the teams and total scores and one for the teams and the quarter scores 
#It makes it easier to wrangle later on to do these separate 
dfs1 <- list()
dfs2 <- list()

#Create a for loop to iterate through each page and to collect the data on each that you need
for (i in 1:num_weeks) { 
  
  #collect all of the team names for each week
  team <- link  %>% 
    #identify the css selector that selects all team names
    html_nodes(".team-name") %>%  
    #parse the html into a usable form
    html_text()
  
  #collect all of the total scores for each week
  score <- link %>% 
    #identify the css selector that selects all total scores
    html_nodes(".team-data .total-score") %>%  
    html_text() %>% 
    as.numeric()
  
  #collect the scores for each quarter for each game for each week 
  q1 <- link %>% 
    html_nodes(".first-qt") %>% 
    html_text() %>% 
    as.numeric

  q2 <- link %>% 
    html_nodes(".second-qt") %>% 
    html_text() %>% 
    as.numeric

  q3 <- link %>% 
    html_nodes(".third-qt") %>% 
    html_text() %>% 
    as.numeric

  q4 <- link %>% 
    html_nodes(".fourth-qt") %>% 
    html_text() %>% 
    as.numeric
  
  #Create a dataframe that binds togther the team and the total score for each week
  x <- as.data.frame(cbind(team, score))
  #This allows you to keep the teams variable the same in each dataframe while creating a variable that identifies which week the score came from
  colnames(x) <- c("teams", paste("scores_week", i, sep = "_"))
  
  #Same thing for a dataframe that combines teams and the quarter scores for each week
  y <- as.data.frame(cbind(team, q1, q2, q3, q4))
  #The "_" after the q is very helpful later on
  colnames(y) <- c("teams", paste("q_1_week", i, sep = "_"), paste("q_2_week", i, sep = "_"), paste("q_3_week", i, sep = "_"), paste("q_4_week", i, sep = "_"))
  
  #assign a name to each dataframe that specifies which week it came from 
  dfs1[[i]] <- x
  dfs2[[i]] <- y
  
  #specify which page to stop on
  if (i < num_weeks) {
    #follow the link for the next page with the approprite css selector 
    link <- link %>%
      follow_link(css = ".active-week+ li .week-item")
  }
}

#join all of the dataframes based on team name
total <- left_join(dfs1[[1]], dfs1[[2]])

for (i in 3:num_weeks) {
  total <- left_join(total, dfs1[[i]])
}

quarters <- left_join(dfs2[[1]], dfs2[[2]])

for (i in 3:num_weeks) {
  quarters <- left_join(quarters, dfs2[[i]])
}


#put the dataframe into long format
total_tidy <- total %>% 
  gather(week, score, -1) %>% 
  #split up the week variable so that all you have is a number
  separate(week, c("dis", "dis2", "week"), sep = "_") %>% 
  select(-starts_with("dis"))

#do the same for the quarter score dataframes 
q_tidy <- quarters %>%
  gather(q, q_score, -1) %>% 
  #this is why the "_" after the q earlier was important 
  separate(q, c("dis", "quarter", "dis2", "week"), sep = "_") %>% 
  select(-starts_with("dis"))

#join the total and quarter dataframes
full_tidy <- left_join(total_tidy, q_tidy)

full_tidy$score <- as.numeric(full_tidy$score)
full_tidy$q_score <- as.numeric(full_tidy$q_score)

week <- full_tidy %>% 
  group_by(week) %>% 
  summarise(mean = mean(score, na.rm = TRUE))

team_score <- full_tidy %>% 
  group_by(teams) %>% 
  summarise(mean = mean(score, na.rm = TRUE))

q_score <- full_tidy %>% 
  group_by(teams, quarter) %>% 
  summarise(mean = mean(q_score, na.rm = TRUE))
```

Examining Trustpilot reviews
============================

``` r
library(tidyverse)
library(rvest)
library(stringr)
library(lubridate)
library(rebus)
```

This example is adapted from a [Data Camp tutorial](https://www.datacamp.com/community/tutorials/r-web-scraping-rvest).

Trustpilot has become a popular website for customers to review businesses and services. On the site, a review consists of a short description of the service, a 5-star rating, a user name and the time the post was made. By the end of this example we will have a function in R that will extract this information for any company we choose.

Let's use Amazon as an example company to scrape reviews from.

Thinking before doing.
----------------------

First, imagine what we might need if we wanted to collect all the Amazon reviews on Trustpilot. Our approach will heavily depend on the structure of the Trustpilot website, so let's take a moment to see how it's formatted. With a general understanding of the way Trustpilot organizes their reviews, we can write functions that take advantage of the structure.

If we're going to pull reviews, here are some things we might want to think about:

-   There are 20 reviews per web page. How do we know how many pages to look through, and how are we going to go from page to page?
-   A review has multiple components to it. What information are we looking to pull?
-   How do we find the CSS tags that correspond to the desired information?

Finding the last page of a review
---------------------------------

``` r
url = 'http://www.trustpilot.com/review/www.amazon.com'

htmlPage1 = read_html(url)
```

Looking at the bottom of the first page of amazon reviews, we see that there are page numbers, and one of them is the last page of reviews! After some digging around, we found that `.pagination-page` is the corresponding tag.

``` r
(nodes = html_nodes(htmlPage1, '.pagination-page'))
```

    ## {xml_nodeset (9)}
    ## [1] <a href="/review/www.amazon.com" data-page-number="1" class="paginat ...
    ## [2] <a href="/review/www.amazon.com?page=2" data-page-number="2" class=" ...
    ## [3] <a href="/review/www.amazon.com?page=3" data-page-number="3" class=" ...
    ## [4] <a href="/review/www.amazon.com?page=4" data-page-number="4" class=" ...
    ## [5] <a href="/review/www.amazon.com?page=5" data-page-number="5" class=" ...
    ## [6] <a href="/review/www.amazon.com?page=6" data-page-number="6" class=" ...
    ## [7] <a href="/review/www.amazon.com?page=7" data-page-number="7" class=" ...
    ## [8] <a href="/review/www.amazon.com?page=167" data-page-number="167" cla ...
    ## [9] <a href="/review/www.amazon.com?page=2" data-page-number="next-page" ...

``` r
(htext = html_text(nodes))
```

    ## [1] "1"         "2"         "3"         "4"         "5"         "6"        
    ## [7] "7"         "167"       "Next page"

Notice that this isn't a list of all the pages, but it does include the last page we want. So now we can write a function "get\_last\_page" that returns the last page as a number from a given html. \* remember that this will work for Trustpilot, but not necessarily other websites \*\* Writing the function:

``` r
get_last_page = function(html){

      pages_data = html %>% 
                      # The '.' indicates the class
                      html_nodes('.pagination-page') %>% 
                      # Extract the raw text as a list
                      html_text()                   

      # The second to last of the buttons is the one
      pages_data[(length(pages_data)-1)] %>%            
        # Take the raw string
        unname() %>%                                     
        # Convert to number
        as.numeric()                                     
}


# What calling the function returns

(lastPageNumber = get_last_page(htmlPage1))
```

    ## [1] 167

Now that we have a way to get the last page number, we can generate a list of all relevant URLs for a given review site.

``` r
#note: str_c() is from the stringr package
list_of_pages = str_c(url, '?page=', 1:lastPageNumber)
tail(list_of_pages)
```

    ## [1] "http://www.trustpilot.com/review/www.amazon.com?page=162"
    ## [2] "http://www.trustpilot.com/review/www.amazon.com?page=163"
    ## [3] "http://www.trustpilot.com/review/www.amazon.com?page=164"
    ## [4] "http://www.trustpilot.com/review/www.amazon.com?page=165"
    ## [5] "http://www.trustpilot.com/review/www.amazon.com?page=166"
    ## [6] "http://www.trustpilot.com/review/www.amazon.com?page=167"

Again, this works because of the way Trustpilot is set up (taking advantage of the url structure of the given website).

Extracting Review Information from One page
-------------------------------------------

Now that we have a list of all the pages of reviews for a company, our next functions only have to scrape information for a given page. Suppose we want to extract four things: the review text, rating, name of the author of each review, and time of submission of each review for a given subpage.

The most tedious part is determining the correct CSS tags you want to extract.

### Review text body

After some digging, we find that the review tag we want is

``` r
nodes = html_nodes(htmlPage1, '.review-info__body__text')
head(nodes)
```

    ## {xml_nodeset (6)}
    ## [1] <p class="review-info__body__text">\n            customer care free  ...
    ## [2] <p class="review-info__body__text">\n            They lost my packag ...
    ## [3] <p class="review-info__body__text">\n            I ordered a TV from ...
    ## [4] <p class="review-info__body__text">\n            Great customer serv ...
    ## [5] <p class="review-info__body__text">\n            Where to start! I a ...
    ## [6] <p class="review-info__body__text">\n            I ordered 2 mobile  ...

``` r
htext = html_text(nodes)
head(htext)
```

    ## [1] "\n            customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364\n        "                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           
    ## [2] "\n            They lost my package, then found it.. then lost it and then i was double charged. It was refunded in the end but It was a right faff on.\n        "                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     
    ## [3] "\n            I ordered a TV from Amazon which was unfortunately lost when out for delivery. Got in contact with their support team and explained the situation. Got a full refund back a 1 free month of Amazon Prime which was nice of them. They have a great support team would recommend\n        "                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              
    ## [4] "\n            Great customer service.I have been buying in Amazon for two years. i had two problems with two products, in one case they replace it for free and in the other case they refunded me without even having to return the faulty item. They were also very polite and the customer service chat is quick and easy.\n        "                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              
    ## [5] "\n            Where to start! I am 35 weeks pregnant and have ordered 2 medium ticket items for the baby from amazon last Wednesday. Having had limited communication I today received an email about automatic refund- with no explanation why that is.Now the refund is not so automatic- as it takes 5-7 business days apparently- which is just poor, especially given that I would have been quite happy to be offered re- delivery/ replacement as I found out that my parcel was damaged in transit - I had to call customer service as this information was not available in the email. Now you would think- that this should be pretty straightforward conversation and straightforward resolution- except it was anything but!!!Effectively having spent 30min having circular conversation getting me absolutely nowhere. I will have to wait for the refund to be processed 5- 7 days and if I still want the items I will have to re-order them again- which amazon will charge me for again. This effectively means that I would have had £300 out of my account- rather than £150-which I cannot afford!!After 25min of conversation I was offered £40 discount- which is all well and good if you actually have spare cash to fork out to pay again for the items you have already been charged for while they are processing the refund! Mind boggles! The whole experience has caused me significant inconvenience and great deal of frustration. Computer says- NO!! it seems, and common sense is not Amazon strongest forte. Needless to say, I will not be using Amazon for future purchases. I had similar experience few years back and ceased to order from amazon for a number of years and I am disappointed to report that unfortunately in this time nothing has changed. Now having this experience for the second time- it is with regret- I will most certainly not be a returning customer and shall delete the account as soon as my refund is processed.\n        "
    ## [6] "\n            I ordered 2 mobile phones from Amazon. After a short while one broke down.  I then discovered that the phones had been mis-sold in that they had been used in the USA and the warranty had expired 4 months before I received them.  I had proof of this from HTC but Amazon refused to take any action.  I therefore have 2 useless bricks which I paid over £600 for less than 6 months ago.  What's even worse is that Amazon won't tell me why they are not upholding my complaint.  I appealed to them but their rules say \"if you don't hear within 3 days, assume your appeal has been refused\".  In those circumstances they can't even be bothered to write individually.  Amazon - trustworthy.  I think not.\n        "

``` r
get_reviews = function(html){
      html %>% 
        # The relevant tag
        html_nodes('.review-info__body__text') %>%      
        html_text() %>% 
        # Trim additional white space
        str_trim() %>%                       
        # Convert the list into a vector
        unlist()                             
}

#testing the function
reviews = get_reviews(htmlPage1)
head(reviews)
```

    ## [1] "customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364customer care free call centre. \U0001f4de91+8420326364"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           
    ## [2] "They lost my package, then found it.. then lost it and then i was double charged. It was refunded in the end but It was a right faff on."                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     
    ## [3] "I ordered a TV from Amazon which was unfortunately lost when out for delivery. Got in contact with their support team and explained the situation. Got a full refund back a 1 free month of Amazon Prime which was nice of them. They have a great support team would recommend"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              
    ## [4] "Great customer service.I have been buying in Amazon for two years. i had two problems with two products, in one case they replace it for free and in the other case they refunded me without even having to return the faulty item. They were also very polite and the customer service chat is quick and easy."                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              
    ## [5] "Where to start! I am 35 weeks pregnant and have ordered 2 medium ticket items for the baby from amazon last Wednesday. Having had limited communication I today received an email about automatic refund- with no explanation why that is.Now the refund is not so automatic- as it takes 5-7 business days apparently- which is just poor, especially given that I would have been quite happy to be offered re- delivery/ replacement as I found out that my parcel was damaged in transit - I had to call customer service as this information was not available in the email. Now you would think- that this should be pretty straightforward conversation and straightforward resolution- except it was anything but!!!Effectively having spent 30min having circular conversation getting me absolutely nowhere. I will have to wait for the refund to be processed 5- 7 days and if I still want the items I will have to re-order them again- which amazon will charge me for again. This effectively means that I would have had £300 out of my account- rather than £150-which I cannot afford!!After 25min of conversation I was offered £40 discount- which is all well and good if you actually have spare cash to fork out to pay again for the items you have already been charged for while they are processing the refund! Mind boggles! The whole experience has caused me significant inconvenience and great deal of frustration. Computer says- NO!! it seems, and common sense is not Amazon strongest forte. Needless to say, I will not be using Amazon for future purchases. I had similar experience few years back and ceased to order from amazon for a number of years and I am disappointed to report that unfortunately in this time nothing has changed. Now having this experience for the second time- it is with regret- I will most certainly not be a returning customer and shall delete the account as soon as my refund is processed."
    ## [6] "I ordered 2 mobile phones from Amazon. After a short while one broke down.  I then discovered that the phones had been mis-sold in that they had been used in the USA and the warranty had expired 4 months before I received them.  I had proof of this from HTC but Amazon refused to take any action.  I therefore have 2 useless bricks which I paid over £600 for less than 6 months ago.  What's even worse is that Amazon won't tell me why they are not upholding my complaint.  I appealed to them but their rules say \"if you don't hear within 3 days, assume your appeal has been refused\".  In those circumstances they can't even be bothered to write individually.  Amazon - trustworthy.  I think not."

### Reviewer Names

Once we determine the appropriate tag to call, this is the same function as the one above.

``` r
get_reviewer_names = function(html){
      html %>% 
        html_nodes('.consumer-info__details__name') %>% 
        html_text() %>% 
        str_trim() %>% 
        unlist()
}

#sanity check
reviewer_names = get_reviewer_names(htmlPage1)
head(reviewer_names)
```

    ## [1] "Rahul Kumar"   "John F"        "jpe"           "Ignacio"      
    ## [5] "Marta Chamska" "Terry Elliott"

### Review Date Time

Date time is going to be a bit trickier because it is stored as an attribute. We will also be using the `lubridate` package to convert some of the date information. Below is the completed function that returns the dates for reviews for a given page, then let’s walk through it to make sure we understand how it works.

``` r
get_review_dates = function(html){

      status = html %>% 
                  html_nodes('time') %>% 
                  # The status information is this time a tag attribute
                  html_attrs() %>%             
                  # Extract the second element
                  map(2) %>%                    
                  unlist() 

      dates = html %>% 
                  html_nodes('time') %>% 
                  html_attrs() %>% 
                  map(1) %>% 
                  # Parse the string into a datetime object with lubridate
                  ymd_hms() %>%                 
                  unlist()

      # Combine the status and the date information to filter one via the other
      return_dates = tibble(status = status, dates = dates) %>%   
                        # Only these are actual reviews
                        filter(status == 'ndate') %>%              
                        # Select and convert to vector
                        pull(dates) %>%                            
                        # Convert DateTimes to POSIX objects
                        as.POSIXct(origin = '1970-01-01 00:00:00') 

      # The lengths still occasionally do not line up. You then arbitrarily crop the dates to fit
      # This can cause data imperfections, however reviews on one page are generally close in time)

      length_reviews = length(get_reviews(html))

      return_reviews = if (length(return_dates) > length_reviews) {
          return_dates[1:length_reviews]
        } else{
          return_dates
        }
      return_reviews
}
```

Now let’s walk through what the function is doing, calling the data structures at each step so we know what the data looks like at each point along the way.

``` r
nodes = html_nodes(htmlPage1, 'time')
attributes = html_attrs(nodes)
head(attributes)
```

    ## [[1]]
    ##                              datetime 
    ##       "2018-05-17T15:33:01.000+00:00" 
    ##                                 class 
    ##                               "ndate" 
    ##                                 title 
    ## "Thursday, May 17, 2018 - 3:33:01 PM" 
    ## 
    ## [[2]]
    ##                               datetime 
    ##        "2018-05-17T11:47:36.000+00:00" 
    ##                                  class 
    ##                                "ndate" 
    ##                                  title 
    ## "Thursday, May 17, 2018 - 11:47:36 AM" 
    ## 
    ## [[3]]
    ##                               datetime 
    ##        "2018-05-17T10:27:50.000+00:00" 
    ##                                  class 
    ##                                "ndate" 
    ##                                  title 
    ## "Thursday, May 17, 2018 - 10:27:50 AM" 
    ## 
    ## [[4]]
    ##                                datetime 
    ##         "2018-05-16T10:06:31.000+00:00" 
    ##                                   class 
    ##                                 "ndate" 
    ##                                   title 
    ## "Wednesday, May 16, 2018 - 10:06:31 AM" 
    ## 
    ## [[5]]
    ##                             datetime                                class 
    ##      "2018-05-15T16:50:29.000+00:00"                              "ndate" 
    ##                                title 
    ## "Tuesday, May 15, 2018 - 4:50:29 PM" 
    ## 
    ## [[6]]
    ##                             datetime                                class 
    ##      "2018-05-15T15:13:11.000+00:00"                              "ndate" 
    ##                                title 
    ## "Tuesday, May 15, 2018 - 3:13:11 PM"

``` r
#status variable in getter function
(date_attribute_2 = unlist(map(attributes,2)))
```

    ##  [1] "ndate" "ndate" "ndate" "ndate" "ndate" "ndate" "ndate" "ndate"
    ##  [9] "ndate" "ndate" "ndate" "ndate" "ndate" "ndate" "ndate" "ndate"
    ## [17] "ndate" "ndate" "ndate" "ndate"

``` r
#dates variable in getter function
(date_attribute_1 = unlist(ymd_hms(map(attributes, 1))))
```

    ##  [1] "2018-05-17 15:33:01 UTC" "2018-05-17 11:47:36 UTC"
    ##  [3] "2018-05-17 10:27:50 UTC" "2018-05-16 10:06:31 UTC"
    ##  [5] "2018-05-15 16:50:29 UTC" "2018-05-15 15:13:11 UTC"
    ##  [7] "2018-05-15 13:59:19 UTC" "2018-05-15 10:10:37 UTC"
    ##  [9] "2018-05-15 08:54:02 UTC" "2018-05-15 05:16:27 UTC"
    ## [11] "2018-05-15 05:04:00 UTC" "2018-05-14 16:17:59 UTC"
    ## [13] "2018-05-13 22:35:11 UTC" "2018-05-13 16:36:54 UTC"
    ## [15] "2018-05-12 16:19:30 UTC" "2018-05-12 02:22:49 UTC"
    ## [17] "2018-05-11 19:31:11 UTC" "2018-05-11 15:40:10 UTC"
    ## [19] "2018-05-11 10:31:29 UTC" "2018-05-11 07:13:57 UTC"

Notice here that we are pulling 22 dates from 20 reviews. To help match them up, we only want the original date times (not updated ones), so we need to return the dates associated with "ndate" and not "ndate updatedDate"

``` r
return_ndates = tibble(status = date_attribute_2, dates = date_attribute_1) %>% 
                        # filter for 'ndate'
                        filter(status == 'ndate') %>%              
                        # Select and convert to vector
                        pull(dates) %>%                            
                        # Convert DateTimes to POSIX objects
                        as.POSIXct(origin = '1970-01-01 00:00:00')

return_ndates
```

    ##  [1] "2018-05-17 15:33:01 UTC" "2018-05-17 11:47:36 UTC"
    ##  [3] "2018-05-17 10:27:50 UTC" "2018-05-16 10:06:31 UTC"
    ##  [5] "2018-05-15 16:50:29 UTC" "2018-05-15 15:13:11 UTC"
    ##  [7] "2018-05-15 13:59:19 UTC" "2018-05-15 10:10:37 UTC"
    ##  [9] "2018-05-15 08:54:02 UTC" "2018-05-15 05:16:27 UTC"
    ## [11] "2018-05-15 05:04:00 UTC" "2018-05-14 16:17:59 UTC"
    ## [13] "2018-05-13 22:35:11 UTC" "2018-05-13 16:36:54 UTC"
    ## [15] "2018-05-12 16:19:30 UTC" "2018-05-12 02:22:49 UTC"
    ## [17] "2018-05-11 19:31:11 UTC" "2018-05-11 15:40:10 UTC"
    ## [19] "2018-05-11 10:31:29 UTC" "2018-05-11 07:13:57 UTC"

``` r
#testing get_review_dates 
(dates = get_review_dates(htmlPage1))
```

    ##  [1] "2018-05-17 15:33:01 UTC" "2018-05-17 11:47:36 UTC"
    ##  [3] "2018-05-17 10:27:50 UTC" "2018-05-16 10:06:31 UTC"
    ##  [5] "2018-05-15 16:50:29 UTC" "2018-05-15 15:13:11 UTC"
    ##  [7] "2018-05-15 13:59:19 UTC" "2018-05-15 10:10:37 UTC"
    ##  [9] "2018-05-15 08:54:02 UTC" "2018-05-15 05:16:27 UTC"
    ## [11] "2018-05-15 05:04:00 UTC" "2018-05-14 16:17:59 UTC"
    ## [13] "2018-05-13 22:35:11 UTC" "2018-05-13 16:36:54 UTC"
    ## [15] "2018-05-12 16:19:30 UTC" "2018-05-12 02:22:49 UTC"
    ## [17] "2018-05-11 19:31:11 UTC" "2018-05-11 15:40:10 UTC"
    ## [19] "2018-05-11 10:31:29 UTC" "2018-05-11 07:13:57 UTC"

### Reviewer Rating

The final variable we want is the reviewer rating. Looking at the source code, we see that the rating is placed as an attribute of the tag. Instead of being just a number (1,2,3,4,5), it's part of a string "count-X" where X is the desired number. For this, we will use regular expressions for pattern matching. In particular, we will use the handy piping functionality in the rebus package, written as the %R% operator, which can decompose complex patterns into simpler subpatterns. Again, here is the full function, and then we will walk through what the data structure looks like at each step of the way.

``` r
get_star_rating = function(html){

      # The pattern you look for: the first digit after `count-`
      pattern = 'count-'%R% capture(DIGIT)    

      ratings =  html %>% 
        html_nodes('.star-rating') %>% 
        html_attrs() %>% 
        # Apply the pattern match to all attributes
        map(str_match, pattern = pattern) %>%
        # str_match[1] is the fully matched string, the second entry
        # is the part you extract with the capture in your pattern  
        map(2) %>%                             
        unlist()

      # Leave out the first instance, as it is not part of a review
      ratings[2:length(ratings)]               
    }
```

Walking through the above function:

``` r
# The pattern we're looking for later on, of the form 'count-[X]'

pattern = 'count-' %R% capture(DIGIT)
pattern
```

    ## <regex> count-([:digit:])

``` r
# just like before, we get the ratings attributes

nodes = html_nodes(htmlPage1, '.star-rating')
ratings_attributes = html_attrs(nodes)

head(ratings_attributes)
```

    ## [[1]]
    ##                                      class 
    ## "star-rating count-3 size-medium clearfix" 
    ## 
    ## [[2]]
    ##                                      class 
    ## "star-rating count-5 size-medium clearfix" 
    ## 
    ## [[3]]
    ##                                      class 
    ## "star-rating count-3 size-medium clearfix" 
    ## 
    ## [[4]]
    ##                                      class 
    ## "star-rating count-5 size-medium clearfix" 
    ## 
    ## [[5]]
    ##                                      class 
    ## "star-rating count-5 size-medium clearfix" 
    ## 
    ## [[6]]
    ##                                      class 
    ## "star-rating count-1 size-medium clearfix"

``` r
#applying pattern match to each attribute

pattern_match = map(ratings_attributes, str_match, pattern = pattern) 
head(pattern_match)
```

    ## [[1]]
    ##      [,1]      [,2]
    ## [1,] "count-3" "3" 
    ## 
    ## [[2]]
    ##      [,1]      [,2]
    ## [1,] "count-5" "5" 
    ## 
    ## [[3]]
    ##      [,1]      [,2]
    ## [1,] "count-3" "3" 
    ## 
    ## [[4]]
    ##      [,1]      [,2]
    ## [1,] "count-5" "5" 
    ## 
    ## [[5]]
    ##      [,1]      [,2]
    ## [1,] "count-5" "5" 
    ## 
    ## [[6]]
    ##      [,1]      [,2]
    ## [1,] "count-1" "1"

``` r
#Only want second element of each
ratings = map(pattern_match, 2)
head(ratings)
```

    ## [[1]]
    ## [1] "3"
    ## 
    ## [[2]]
    ## [1] "5"
    ## 
    ## [[3]]
    ## [1] "3"
    ## 
    ## [[4]]
    ## [1] "5"
    ## 
    ## [[5]]
    ## [1] "5"
    ## 
    ## [[6]]
    ## [1] "1"

``` r
#turn ratings into a vector.

ratings = unlist(ratings)
head(ratings)
```

    ## [1] "3" "5" "3" "5" "5" "1"

``` r
length(ratings)
```

    ## [1] 21

``` r
#observe that the length of ratings is 21 for 20 ratings on the page.  The first star rating is not from a review, so we don't want that one.

ratings_list = ratings[2:length(ratings)]
head(ratings_list)
```

    ## [1] "5" "3" "5" "5" "1" "1"

``` r
#sanity check
getRatings = get_star_rating(htmlPage1)
head(getRatings)
```

    ## [1] "5" "3" "5" "5" "1" "1"

Combining our functions!
------------------------

Now that we have tested that the individual extractor functions work on a single URL, we can combine them to create a tibble (basically a data frame) for the whole page. Because we are likely to apply this function to more than one company, it's helpful to add a parameter for the company name. This is especially helpful in later analysis should you want to compare different companies.

``` r
get_data_table = function(html, company_name){

      # Extract the Basic information from the HTML
      reviews = get_reviews(html)
      reviewer_names = get_reviewer_names(html)
      dates = get_review_dates(html)
      ratings = get_star_rating(html)

      # Combine into a tibble
      combined_data = tibble(reviewer = reviewer_names,
                              date = dates,
                              rating = ratings,
                              review = reviews) 

      # Tag the individual data with the company name
      combined_data %>% 
        mutate(company = company_name) %>% 
        select(company, reviewer, date, rating, review)
    }
```

We then wrap this function in another basic function that takes the URL and company\_name and extracts the required HTML

``` r
get_data_from_url = function(url, company_name){
      html = read_html(url)
      get_data_table(html, company_name)
    }
```

Testing this function from our original URL for page 1 of Amazon Reviews

``` r
url = 'http://www.trustpilot.com/review/www.amazon.com'

amazon_data_page1 = get_data_from_url(url, 'amazon')
head(amazon_data_page1)
```

    ## # A tibble: 6 x 5
    ##   company reviewer      date                rating review                 
    ##   <chr>   <chr>         <dttm>              <chr>  <chr>                  
    ## 1 amazon  Rahul Kumar   2018-05-17 15:33:01 5      customer care free cal…
    ## 2 amazon  John F        2018-05-17 11:47:36 3      They lost my package, …
    ## 3 amazon  jpe           2018-05-17 10:27:50 5      I ordered a TV from Am…
    ## 4 amazon  Ignacio       2018-05-16 10:06:31 5      Great customer service…
    ## 5 amazon  Marta Chamska 2018-05-15 16:50:29 1      Where to start! I am 3…
    ## 6 amazon  Terry Elliott 2018-05-15 15:13:11 1      "I ordered 2 mobile ph…

Finally, we can write a simple function that will take as input the URL of the landing page of a company and the label we want to give the company. We want the function to extract all reviews, binding them into one tibble, and writing them to a tsv.

Aside: This is also a good starting point for optimising the code. The map function applies the get\_data\_from\_url() function in sequence, but it does not have to. One could apply parallelisation here, such that several CPUs can each get the reviews for a subset of the pages and they are only combined at the end.

``` r
scrape_write_table = function(url, company_name){

      # Read first page
      first_page = read_html(url)

      # Extract the number of pages that have to be queried
      latest_page_number = get_last_page(first_page)

      # Generate the target URLs
      list_of_pages = str_c(url, '?page=', 1:latest_page_number)

      # Apply the extraction and bind the individual results back into one table, 
      # which is then written as a tsv file into the working directory
      list_of_pages %>% 
        # Apply to all URLs
        map(get_data_from_url, company_name) %>%  
        # Combine the tibbles into one tibble
        bind_rows() %>%                           
        # Write a tab-separated file
        write_tsv(str_c(company_name,'.tsv'))     
    }
```

So now if we wanted all Amazon reviews:

``` r
scrape_write_table(url, 'amazon')

amazon_table = read_tsv('amazon.tsv')

head(amazon_table)
```

    ## # A tibble: 6 x 5
    ##   company reviewer      date                rating review                 
    ##   <chr>   <chr>         <dttm>               <int> <chr>                  
    ## 1 amazon  Rahul Kumar   2018-05-17 15:33:01      5 customer care free cal…
    ## 2 amazon  John F        2018-05-17 11:47:36      3 They lost my package, …
    ## 3 amazon  jpe           2018-05-17 10:27:50      5 I ordered a TV from Am…
    ## 4 amazon  Ignacio       2018-05-16 10:06:31      5 Great customer service…
    ## 5 amazon  Marta Chamska 2018-05-15 16:50:29      1 Where to start! I am 3…
    ## 6 amazon  Terry Elliott 2018-05-15 15:13:11      1 "I ordered 2 mobile ph…

We can now scrape reviews from any company on Trustpilot by calling one line of code!

Minihacks
=========

Minihack 1
----------

Navigate to the nfl page that was used in the first example and pull the big plays count for each game. Once you have all of the data, report the mean and the standard deviation. \*If you are feeling ambitious, create a for loop that does this for each week of the season and store each mean and sd in a list.

Minihack 2
----------

Go back to the TrustPilot website and look at the reviews. You’ll notice that there is information on each review about the number of reviews the author has posted on the website (Ex: Joey Perry, 4 reviews). Write a function that, for a given webpage, gets the number of reviews each reviewer has made.

If you are having trouble finding the corresponding CSS tag, ask a classmate! Note as well that only pulling the number of other reviews will involve some text manipulation to get rid of irrelevant text.

At the end you should have a function that takes an html object as a parameter and returns a numeric vector of length 20.

Minihack 3
----------

The web is a vast ocean of data that is waiting to be scraped. For this hack, be creative. Find a website of your choice and pull some data that you find interesting. Tidy your data and print the head of your dataframe. Perform some simple descriptive statistics and if you’re feeling ambitious create at least one visualization of the trends that you have discovered. There are no limitations to what kind of data you decide to pull (but don’t forget our initial disclaimer!). It can be numbers and it can also be text if you decide that you want to use the skills from the text processing lesson that we covered last week.

If you don’t know where to start, scraping from imdb.com, espn.go.com, news sites, bestplaces.net, is sufficient.

Helpful Sources:

[Hadley Wickham’s rvest](http://blog.rstudio.com/2014/11/24/rvest-easy-web-scraping-with-r/)

[RPubs Web Scraping Tutorial](https://rpubs.com/ryanthomas/webscraping-with-rvest)

[CRAN Selectorgadget](https://cran.r-project.org/web/packages/rvest/vignettes/selectorgadget.html)
