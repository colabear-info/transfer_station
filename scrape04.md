# Scraping from 0 to hero

## Scraping from 0 to hero (Part 4/5) <a href="#277d" id="277d"></a>

## Advanced Scraping <a href="#db37" id="db37"></a>

Once you have enough experience to scrape tons of websites every day, you can start to think about improving the method. With some tricks, of course. Let’s check them out.

### Hardware and Software <a href="#25fc" id="25fc"></a>

Once you have the scraper developed and you know the cloud provider to use, I highly suggest installing it into a Kubernetes (K8s) with multiple cloud server instances. The structure with K8s will be similar to the next picture:

![](https://miro.medium.com/max/700/1\*ETABCC3u1CM7IvVj7j5mqw.jpeg)

Meaning that in the structure of the K8s, you will have 3 important pieces (distributed along the server instances):

1. **Scraper Web UI** (by order, [this one](https://github.com/my8100/scrapydweb), [this other](https://github.com/DormyMo/SpiderKeeper) or a manual one created by you): The installation is pretty simple and it’s the best tool that you can use once you install multiple scrapers that run at the same time. This is not only to manage all scrapers visually; it’s also to schedule them and ensure that if something goes wrong, the system will let you know by Slack or even Telegram.
2. **Scrapy workers**: this is, by far, the key of the entire infrastructure. Every scraping worker can launch multiple scrapers at the same time. The performance of each could vary depending on the number of bytes to download and the algorithm developed to retrieve the data. With K8s, using 1, 5 or 100 Scrapy workers is stupidly easy (like in the previous diagram where N Scrapy instances were launched). The unique limit it’s the machine/s that you have rented for this purpose. And your money, of course!
3. **Link provider**: once you have to scrape millions of urls, with the idea of reading from 1 datastore and be executed by multiple instances simultaneously. These data-stores can be:

* A **Redis** (using [this module](https://github.com/rmax/scrapy-redis) for Scrapy) to store all the URLs.
* A **RabbitMQ** queue to store all the URLs ([here](https://gist.github.com/abevoelker/10606489) the script to use Scrapy to push links, and [here](https://github.com/roycehaynes/scrapy-rabbitmq) the script in Scrapy to consume it)
* A **Database**, like MySQL, PostgreSQL, MongoDB or other type of database in order to manage your own data structure that will be stored. This is very manual work, but could be the best solution to handle the specific problem that you’re tackling.

Everything said so far depends on your ability to set up the environment. With this minimal infrastructure, every professional can scale up to millions of pages scraped per day or even per hour.

Depends on your necessity, you will store the retrieved data into:

* S3 in the case you need to store it as a file
* PostgreSQL in the case you require the storage of a relational information (without limitations of scalability)
* MongoDB if the information to store is unstructured
* Another pipeline, as Redis, Kafka or even other RabbitMQ to launch another Scrapy process

### External proxy provider <a href="#13d6" id="13d6"></a>

We will review it later in depth, but let’s talk about Proxies. If you want to have access to a website several times per minute and from the same IP, it’s easy for most website hosts to detect that you’re using a machine/bot to get their pages. and they could block access to their content for a while That is why everyone in the world of scraping, is using external services to shield them behind a Proxy service.

A proxy provider is a service that helps you to confuse the website, simulating that the request is coming from a different IP and not yours. Even from a different country!

There are 2 different kind of services:

* **URL Service**: Those given an URL, they will return the content. The most popular, and f-word expensive ones, is [Crawlera](https://www.scrapinghub.com/crawlera/). Another service is [Luminati](https://luminati.io), and the last one -that I used and I can recommend- is [Smartproxy](https://smartproxy.com).
* **IP Service**: Those given a request of an IP, the service will return with a Proxy IP ready to use. The examples could be [OxyLabs](https://oxylabs.io), and one pretty new is [IPBurger](https://www.ipburger.com). There are dozens over the net.

### Database structure <a href="#1256" id="1256"></a>

This is one important thing that you have to keep in mind. The data. If you take a look into the given examples in the Scrapinghub website (leader of scraping services around the world), you can understand that the structure of the information is vital.

If you are able to scrape millions of pages per day, how can you analyze all the information at once if it’s not well structured? Or at least, in the same format?

For the sake of simplicity, before creating any scraper, you have to know, in advance, what are the fields or the information that you need. You can not retrieve everything. This could be a nightmare to maintain, and to develop. This means that once you have a very clear picture of what is needed, you can create the DB structure (even if it is in a NoSQL database!).

For me, there are 2 different types of data:

* **Url to scrape by a scraper**: normally the urls to scrape every X weeks, days, hours or seconds
* **Data extracted**: the data coming from the url already extracted

### Example of URL to scrape by a scraper <a href="#0a1b" id="0a1b"></a>

The structure of the schema in DB should be similar to a table that has these fields:

* _id_: the id of every row
* _url_: the url to scrape by the scraper
* _scraper_: the name of the scraper to execute
* _periodicity_: it means every when it has to be executed per _periodicity\_type_. For example: 1, 2, or 5
* _periodicity\_type_: this value can be: monthly, weekly, daily, hourly, minutely
* _last\_execution_: the last timestamp of execution

### Example of data extracted <a href="#546e" id="546e"></a>

If you are extracting Real Estate data, you can find several tables related:

Table with the data extracted (_real\_estate\_data_ table):

* _id_: the id of every row
* _url_: the url scraped
* _scraper_: the name of the scraper used
* _price: the price of the property_
* _discount: discount applied (if there is any)_
* _square\_meters_: square meters of the property
* _rooms_: number of rooms in the property
* _bathrooms_: number of bathrooms in the property
* _location: the place of the property (with the address if it is possible)_
* _postal\_code: the postal code of the place_
* _city: the city, town or village of the property_
* _title_: title of the ad in the website
* _description_: description of the ad in the website
* _last\_execution_: the last timestamp of execution
* creation\_time: the creation of the row (when was the first time executed

Photos of the properties (_real\_estate\_photo_ table):

* _id_: the id of every row
* _property\_id_: the id of the property related to the photo
* _path_: path of the photo in S3

Historical price of the property (_real\_estate\_historic\_price_ table)

* _id_: the id of every row
* _property\_id_: the id of the property related to the historic price
* _price: the price of the property_
* _creation\_time_: the creation of the row (when was the first time executed

This structure is an example of how I would do it in the first phase. This does not mean that it could not be improved or simplified.

Scraping could be used for different scenarios, and all of them require a different data structure.

### Proxy hosting (CDN for images, JS and static files) <a href="#3b87" id="3b87"></a>

Downloading thousands of websites teaches you some tricks, like avoiding the download of useless files to improve your performance time. This can be solved by using a custom CDN.

The process should be:

![](https://miro.medium.com/max/700/1\*fJYOmVCdt6yDiB9Db5T8mg.jpeg)

Once you download the request’s files (static files like pictures, javascript or even css), you can upload it into the CDN in order to avoid losing time to download it again. This will make the difference if the amount of pages to scrape is huge.

### Schedulers <a href="#2b69" id="2b69"></a>

Another magic trick in scraping is the possibility of automation in all services and steps. This is really helpful if you need to go over millions of pages per week, because you should not send all the requests at once, to avoid overflooding the website, instead, you have to send them in small batches. This way, you can hide your requests further.

To achieve this process, you will have to schedule the requests to be sent in a discontinuous way, also, it’s better to schedule the launch during the night, as the website probably won’t receive a lot of traffic (for instance, if you are scraping a Japan website, you can scrape it during their low peak, at their mid-night).

### Javascript rendering <a href="#a2df" id="a2df"></a>

Sometimes the websites want to avoid being scraped/crawled using javascript to build the content or even request part of the content (doing ajax). To get around this and obtain the data, Scrapy’s team has created [Splash](https://github.com/scrapy-plugins/scrapy-splash), a plugin specifically made to render the page — with it’s content/data — in background (using chromium), fooling the website as if it was a human interaction.

Something to keep in mind while Splash is the tedious implementation needed to do easy stuff (like rotating proxies in your own way). This has not been well resolved when using Scrapy with Splash. They give you the possibility of using Splash Lua Script ([Lua](https://www.lua.org) coding language) to customize the behavior, but this is not as user friendly as doing it in Python.

If you want to add generic proxies, you can follow [their documentation](https://splash.readthedocs.io/en/stable/api.html#arg-proxy), but if you want to apply more fancy code, you have to develop it in this _modern_ language called Lua.

## To be continued… <a href="#0b15" id="0b15"></a>

This article is part of the list of articles about **Scraping from 0 to hero**:

* [Link to Part 1](https://medium.com/manomano-tech/scraping-from-0-to-hero-part-1-5-9acb6548d4dc)
* [Link to Part 2](https://medium.com/manomano-tech/scraping-from-0-to-hero-part-2-5-e1d7b6065d44)
* [Link to Part 3](https://medium.com/manomano-tech/scraping-from-0-to-hero-part-3-5-bc320f966593)
* Link to Part 4 (Current)
* [Link to Part 5](https://medium.com/manomano-tech/scraping-from-0-to-hero-part-5-5-d205b0a491ce)

If you have any question, you can comment below or reach me on [Twitter](https://twitter.com/fparareda) or [LinkedIn](https://es.linkedin.com/in/ferranparareda).
