---
layout: post
title:  "Helping local businesses with big data analysis"
date:   2018-06-03 19:34:57 +0800
categories: jekyll update
tags: scrapping elasticsearch googleplaces google reviews bars
author: jeshocarmel
crosspost_to_medium: true
---


# **Problem Statement**

I was recently asked by a friend of mine to come over to help them with a problem. There is this bar, let's call it **`John's bar`** (to maintain anonymity) in `Ampang` which is one of the most happening and uptown places in `Kuala Lumpur, Malaysia`. The place has good vibes all over with some fantastic art pieces, really quick and polite staff and an european aura. The bar doesn't serve much varities of food but hey its an european themed bar, so duh [ deal with it amigo!].

The owner John who is a foreigner as well came over and explained the situation the bar is facing right now. Let me put that in bullets for you.

* **Everyone who comes to the bar usually say they had a good time or provide a very positive feedback.**
* **The place has everything right and they have stayed up to date with the current trend.**
* **The place has a very random distribution of crowd which is very unpredictable ( a friday can be crowded one week and have very few customers an other week ).**
* **The audience distribution is more in the age groups of 40+.**

John asked me **`what am I missing or what could I do to get the crowd in??`**

Now that's a real time problem statement for me to work with and I said my goodbyes and promised to get back to him in 2 weeks time.


# **Picking up the right axe**

So when you are given a real time problem, the first thing you usually think is:

**`where can I get the data?`** or **`how do I know what people expect in a bar?`**

What better to way to get what people think about a bar other than reviews and when you think about reviews, the first thing that came in my mind was **GOOGLE REVIEWS**.

**`What if we can scrap google reviews of all bars within Kuala Lumpur and find out what people talk about much?`**

Let's do this.

# **Developer insights of Google Reviews**

Everyone knows google reviews, but let me give a dev insight of google reviews.

* Google Reviews has an API which let's you to legally scrap their reviews data of all the institutions listed by them as a place.
* **`Radar Search`** or **`Nearby Search`** are two API's by google places which are most commonly used to scrap information about a place
* These API's list all the details that are available in Google about them to be captured by us which includes `address, photos, phone numbers, reviews, location (latitude and longitude)` of the place

![price_ny](/assets/images/types_google_places.png){:class="img-responsive"}


* Google has categorized every location into **`Types`** which specifies which category the place belongs to e.g. `Bank, Aquarium, Bars, Restaurants` and so on.
* Due to developers abusing their API's in the past they have restricted now to 5 reviews of each place i.e. **`google reviews will now give only 5 reviews for each and every place.`** *(why guys why?)*
* To access the API you need an **`API key`** which you can obtain here in this [link](https://developers.google.com/places/web-service/get-api-key){:target="_blank"}


# **Time to get our hands dirty**
*First things first, import your libraries.*

{% highlight python %}

    from googleplaces import GooglePlaces, types, lang
    import json
    import sys
{% endhighlight %}

*Call the google places api with your api_key.*

{% highlight python %}

    YOUR_API_KEY = 'WRITE_YOUR_API_KEY_HERE'
    google_places = GooglePlaces(YOUR_API_KEY)
{% endhighlight %}

*Do your radar search and store results in a file to be loaded later to Elasticsearch.  I'm doing a radar search here with **`Kuala Lumpur, NYC and London as locations`**, **`20000m as radius`** and **`Types = Bars`***

{% highlight python %}

def radar_search(keyword,location):

    query_result = google_places.radar_search(location=location, keyword=keyword,radius=20000, types=[types.TYPE_BAR])
    
     for place in query_result.places:
     place.get_details()
     print(place.name)
     
     if "reviews" in place.details:
     for review in place.details["reviews"]:
     write_review_to_es(review = {"review": review["text"],"rating": review["rating"],"location": location, "name": place.name})
     
     if query_result.has_next_page_token:
     query_result_next_page = google_places.radar_search(
     pagetoken=query_result.next_page_token)
     

{% endhighlight %}


{% highlight python %}
def write_review_to_es(review):

    try:
        with open('/Users/jeshocarmel/Documents/jsons/bar_reviews.json',
        'a') as fp:


        fp.write("""{ "index" : { "_index" : "bar_reviews"} }""")
        fp.write("\n")
        json.dump(review, fp)
        fp.write("\n")
    except:
        print("Unexpected error:", sys.exc_info()[0])
        raise

{% endhighlight %}

 *Run it*

{% highlight python %}

if __name__ == "__main__":
     radar_search(keyword='Bar',location='Kuala Lumpur, Malaysia')
     radar_search(keyword='Bar',location='London, United Kingdom')
     radar_search(keyword='Bar',location='New York City, United States of America')
{% endhighlight %}

# **Execution**

Run the above few lines of code and i got around `500 bars` in the three cities and **`500 * 5 = 2500 reviews`** in all less than two minutes. Wonderful isn't it.

Let's load our data into elastic search and visualize in Kibana. I've done a blog post previously with steps on how to load data to elasticsearch and visualize with kibana. If you are not familiar, then do visit this [blog post](https://jeshocarmel.github.io/jekyll/update/2018/05/09/us_presidential_analysis_es.html).

# **Tag Cloud for bars in 3 major cities (London, NYC and KL)**

| ![london_bars](/assets/images/london_bars.png){:class="img-responsive"} |
|:--:|
| *Tag Cloud for **London,England** google reviews* |


As you can see, people in **London** have **cocktails,service and drink** specified more when they are handing out reviews for the bars they have been to.


| ![newyork_bars](/assets/images/newyork_bars.png){:class="img-responsive"} |
|:--:|
| *Tag Cloud for **New York City's** google reviews* |

People in the city that never sleeps **(NYC)** have **food and drink** an equal importance when they are handing out reviews for the bars they have been to.


| ![kual](/assets/images/kuala_lumpur_bars.png){:class="img-responsive"} |
|:--:|
| *Tag Cloud for **Kuala Lumpur,Malaysia** google reviews* |

People here in **Kuala Lumpur** are more interested in **food** and that clearly stands out among service or drinks.


# **What next?**

Similar to google reviews, an other interesting to checkout is [Facebook places API](https://developers.facebook.com/docs/places/overview). I've done a similar analysis with facebook places API and have posted the results here.

# **Price distribution of bars in 3 major cities (London, NYC and KL)**

| ![price_ny](/assets/images/price_london.png){:class="img-responsive"} |
|:--:|
| *Price range for bars in **London,England**  according to facebook* |

| ![price_ny](/assets/images/price_ny.png){:class="img-responsive"} |
|:--:|
| *Price range for bars in **Newyork City**  according to facebook* |

| ![price_kl](/assets/images/price_kl.png){:class="img-responsive"} |
|:--:|
| *Price range for bars in **Kuala Lumpur, Malaysia**  according to facebook* |


# **Summary**
Kuala Lumpur seems to be an interesting place for a bar to be. So based on my analysis I gave John the conclusion as below:

* People in Kuala Lumpur take their food seriously. This is a known fact but they really take their food seriously even if it is a bar. **Make food a priority in your bar. Good food is good crowd.**
* There are **more number of moderately priced bars** in Kuala Lumpur compared to cheaper bars whereas that is inverse in the case of NYC and London. The reason could be people in KL like their food and drinks with better quality and moderate price rather than cheap drinks with lesser quality. *(thats just my point of view)*


