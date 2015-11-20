---
layout: post
title:  "fitbit data in R!"
date:   2015-11-19 19:11:09 -0800
categories: R fitbit httr
---



# Intraday heartrate data from fitbit in R using httr

Fitbit [announced](https://community.fitbit.com/t5/Web-API/Intraday-data-now-immediately-available-to-personal-apps/m-p/1014524#U1014524) a new type of application classification, Personal, that gives you access to your own intraday data.  Finally, access to second by second heart rate data from my Charge HR!
  
Here' a quick walk through of accessing the data in R using the `httr` package.
  
## Register an app at fitbit.com 
  
You'll need to register an app at [https://dev.fitbit.com/](https://dev.fitbit.com/).  Make sure you select the "Personal" Application Type, and set the Callback URL to `http://localhost:1410/`.

Once it's set up, grab your "OAuth 2.0 Client ID" and "Client (Consumer) Secret".

## OAuth2.0 authorization

fitbit uses OAuth 2.0 for user authorization and API authentication.  The exchange process is a little different, requiring a custom header, so I modified the standard `init_oauth_2.0` function in `httr` and created a new class that uses it.  These functions can be found at: (https://gist.github.com/cwickham/81be4a3c2f6eb8caa94d).

Source these functions,

{% highlight r %}
devtools::source_gist("81be4a3c2f6eb8caa94d")
{% endhighlight %}

Then get the authorization token

{% highlight r %}
library(httr)
fitbit <- oauth_endpoint(NULL, "https://www.fitbit.com/oauth2/authorize?",
  "https://api.fitbit.com/oauth2/token")

fitbit_app <- oauth_app("fitbit", 
  key = "229K52",  # replace with your OAuth 2.0 Client ID, (NOT client_key!)
  secret = "d8d25b633b3e4d6daf651ea60d8ff4ff") # your Client (Consumer) Secret

token <- oauth2.0_token_fitbit(fitbit, fitbit_app, 
  scope = "heartrate")
{% endhighlight %}

You should be directed to the fitbit site to authorize your app's access to your data.  Don't use my client ID and key!  It only accesses my data (and you don't know my fitbit login).   I've just requested access to the heartrate data (`scope = "heartrate"`), but you can check out other scopes in the fitbit docs (https://dev.fitbit.com/docs/oauth2/#scope).

## Get your data

The API docs detail all sorts of requests you can make but I'm interested in [Heart Rate Intraday Time Series](https://dev.fitbit.com/docs/heart-rate/#get-heart-rate-intraday-time-series).  

Getting that data is as simple as submitting a URL request.  Here, I'm asking for one day's worth (`1d`) of 1 second resolution heart rate data, on 2015-11-11:

{% highlight r %}
req <- GET("https://api.fitbit.com/1/user/-/activities/heart/date/2015-11-11/1d/1sec.json", 
  config(token = token))
stop_for_status(req) # check the request was successful
{% endhighlight %}


The data returned is JSON, so it takes a little work to get it into a useable form

{% highlight r %}
library(dplyr)
library(lubridate)
library(ggplot2)

names(content(req))
date <- content(req)[["activities-heart"]][[1]][["dateTime"]]

hr <- rbind_all(content(req)[["activities-heart-intraday"]][["dataset"]])
hr <- hr %>% 
  mutate(
    datetime = paste(date, time),
    datetime = parse_date_time(datetime, "ymd hms")) %>%
  rename(heartrate = value)
{% endhighlight %}

But once done, I can plot and explore away!

{% highlight r %}
qplot(datetime, heartrate, data = hr, geom = "line")
{% endhighlight %}

![plot of chunk unnamed-chunk-7](/images/2015-11-20-fitbit-data-in-R/unnamed-chunk-7-1.png) 
