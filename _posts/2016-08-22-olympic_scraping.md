---
layout: post
title: Web Scraping Olympic Track and Field Results
date:   2016-08-22 17:30:14
categories: others
---

I am writing this post one day after the closing ceremonies of the 2016 Rio Olympic Games. This year's Olympics were truly incredible. My personal highlights were watching Michael Phelps and Allyson Felix win multiple medals for the USA. Of course, watching the media tear apart Ryan Lochte after his gas station gaffe was pretty entertaining to watch from afar as well.

Track and Field is probably my favorite sport to watch at the Olympics. Watching the superstars compete inspired me to work on a new project: ranking the performances of all Track and Field Olympic medalists over the years. To do this, I needed to collect data. However, I found it difficult to find a single data source that had all the results I wanted. It was time for my to compile my own!

Luckily for me, the [official olympics website](https://www.olympic.org/olympic-results) has results for all Athletics events going back to the original Olympic games in 1896. They even provide a nice interface to search for a particular event. However, there was not a convienient listing of what events were contested at each of the Olympic games on this page, so I had to generate my own. One thing I noticed was that the web pages that contained Olympic results had the following URL structure:

<br/>

<p align="center">
https://www.olympic.org/{olympic_year}/athletics/{event}.
</p>

<br/>

I started by playing around with the drop-down menues on the Olympic Results search page. I discovered that by selecting the 'London 2012' for the year and 'Athletics' for the sport, the HTML in the page changed to include all the possible Olympic years and modern olympic events.

<br/>

<p align="center">
  <img src="http://joeledger.com/assets/scrape.PNG" />
</p>

<br/>

I downloaded this page to perform more analysis on it offline. I noticed that the HTML contained a list of all the summer and winter games and their corresponding URL fragments. I used the following Python code snippet and a regular expression to get all the relevant URL fragments.

{% highlight python %}

olympic_html_file = "Data/scrape/Olympic Results - Official Records.html"

def get_olympic_year_strings():
    reader = open(olympic_html_file, "rb")
    text = str(reader.read())
    matches = re.findall('summergames"> <a href="https://www.olympic.org/.{1,20}"', text)
    return [x[x.rfind("/") + 1:-1] for x in matches]

{% endhighlight %}
Calling this function returns a list of all the URL fragments for all the Summer Olympic Games going back to 1896. The result looks something like this: ['rio-2016', 'london-2012', 'beijing-2008', ..., 'athens-1896'].
 I am aware that using regular expressions to parse HTML is often frowned apon, but this was a quick and easy way to get it done.

<br/>

 I used a similar function to get the event URL fragments.
{% highlight python %}

def get_olympic_event_strings():
    reader = open(olympic_html_file, "rb")
    text = str(reader.read())
    matches = re.findall('<option value="/.{1,30}/athletics/.{1,30}men"', text)
    return [x[x.rfind("/") + 1:-1] for x in matches]

{% endhighlight %}
Calling this function results in a list that looks like this: ['10000m-men', '10000m-women', '100m-hurdles-women', ..., 'triple-jump-women'].

<br/>

Now comes the task of combining our URL fragments into a list of URLs corresponding to the result pages. The list of URLs should contain an entry for each combination of Olympic year and Olympic event. To do this, we can simply use the cartesian product of the two URL fragment lists we just generated and the URL scheme discussed above.

{% highlight python %}

def get_all_result_urls():
    years = get_olympic_year_strings()
    events = get_olympic_event_strings()

    return ["https://www.olympic.org/%s/athletics/%s" % (year, event)
            for year, event in itertools.product(*[years, events])]

{% endhighlight %}
Of course, there are some limitations in the method we used to get the URL list. Since our events list only contains events that are contested at the modern Olympic games, any older events that are no longer contested will not be included in our results. In addition, our list will include a lot of events that were not actually contested: for example the Women's Olympic Marathon was not contested until 1984, so all URLs corresponding to Women's Marathons before 1984 will not resolve. Despite these shortcomings, this method is good enough for me as all results of events still contested at the Olympics will be included and URLs that don't resolve to actual pages can simply be ignored.

<br/>

Now that we have the URLs of all the results pages, we need a way to parse the results from the pages. The following method uses the Python library BeautifulSoup to do just that.
{% highlight python %}

def parse_html_results(html):
    has_place = lambda row: len(row) > 0 and (row[0] in ['G', 'S', 'B'] or row[0][:-1].isdigit())
    trim_place = lambda place: place if not place[0].isdigit() else place[:-1]
    bs = bs4.BeautifulSoup(html, "html.parser")
    table = bs.find(lambda tag: tag.name == 'table')
    table_body = table.find('tbody')
    rows = table_body.find_all('tr')
    data = []
    for row in rows:
        cols = row.find_all('td')
        cols = [ele.text.strip() for ele in cols]
        cols = [ele for ele in cols if ele]
        if(has_place(cols)):
            cols[0] = trim_place(cols[0])
            data.append(cols)

    return [x for x in map(separate_athlete_county, data)]

{% endhighlight %}
This method takes the html from a results page as an argument, and finds the results table in the page. It then converts the table into a list of lists, with each inner list corresponding to a row in the table. Rows without places and empty columns are discarded. Since athlete and country data are stored in the same column, we apply the function separate_athlete_country to each row using the Python map operator. 
{% highlight python %}

def separate_athlete_county(table_columns):
    a_and_c = table_columns[1]
    athlete = a_and_c[:a_and_c.find("\n")]
    country = a_and_c[a_and_c.rfind("\n") + 1:]
    table_columns[1] = athlete
    table_columns.insert(2, country)
    return table_columns

{% endhighlight %}

<br/>

We also want the name of each Olympic event, which is conveniently stored in the title tag of each results page. We can recover this data using the following function. 
{% highlight python %}

def get_race_title(html):
    bs = bs4.BeautifulSoup(html, "html.parser")
    title = bs.find(lambda tag: tag.name == 'title')
    return title.text

{% endhighlight %}

<br/>

Now that we have a list of URLs and a way to parse the relavent data from them, all we have left is to request data from each of the URLs and save their data to a text file. The following code snippet uses the Python requests library to accomplish this.
{% highlight python %}

def save_race_results(result_urls):
    writer = open(race_results_file, "w", encoding='utf-8')
    for url in result_urls:
        response = requests.get(url)
        if(response.status_code == 200):
            try:
                html = response.text
                title = get_race_title(html)
                data = parse_html_results(html)
                writer.write(title + "\n")
                for line in data:
                    writer.write(",".join(x for x in line) + "\n")
                writer.write("\n")
            except:
                print(title)
                traceback.print_exc()
        else:
            print(url, response.status_code)


urls = get_all_result_urls()
save_race_results(urls)

{% endhighlight %}

<br/>

And we are done! The resulting text file can be found [here](/assets/race_results.txt), and the github repo containing the script used to generate it can be found [here](https://github.com/joewledger/Olympic_Outliers). Thanks for reading!
