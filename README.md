# Analysing Daily Cryptocurrency News Sentiment with Python

## Do You Trade Based on News?

If you trade cryptocurrencies, it’s likely you’ve used news as part of your strategy at some point. I certainly have, with mixed results. The main challenge with this approach is that, by the time you hear about a major news event impacting a cryptocurrency, the market has often already responded. Other traders might have opened positions and seized the opportunity. Unless you’re actively monitoring the news, it’s difficult to execute the best possible trade.

## Script Overview

### Automating News Monitoring

The script I’ll guide you through in this article is designed to perform a contextual web search, using keyword inputs to fetch relevant news results. This ensures you have access to the most up-to-date news about your chosen cryptocurrency. The API I’ve used for this is the Contextual Web Search hosted on RapidAPI. It provides 100 free requests per day, with paid plans available if you require more.

### Analysing Sentiment

The next stage of the script involves sentiment analysis of the news headlines, determining whether the sentiment is positive, neutral, or negative. In theory, this saves considerable time compared to manually reading each headline to decide whether to act on it. These sentiment scores can be used to inform automated decisions, such as generating buy or sell signals for a cryptocurrency. While the trading logic isn’t built into this script, I’ll cover that aspect in a future article.

## Limitations

Before diving into the code, it’s important to be aware of a couple of limitations:

1. The search API has a strict daily limit of 100 requests (though you can replace it with a custom web scraper if needed).
2. The sentiment analysis may not fully reflect the “magnitude” or significance of the news.

With that out of the way, let’s get started!

## The Code

### Prerequisites

To set up and run this script, you’ll need the following:

- A Python interpreter – I recommend using Atom.
- [A RapidAPI account](https://rapidapi.com/).
- An API key for the [Contextual Web Search](https://rapidapi.com/contextualwebsearch/api/web-search/).
- An API key for the [Sentiment Analysis](https://rapidapi.com/fyhao/api/text-sentiment-analysis-method).

Once you’ve created a RapidAPI account and obtained your API keys, you’re ready to begin!

### Imports

Start by importing the following modules into your Python compiler:


````
from datetime import datetime, date, timedelta
import requests, json, re, os, time
from itertools import  count
````

## User Inputs and Global Variables

Next, let's define a few variables to store our API keys and specify what the algorithm should search the web for. These variables will help us customise the search for cryptocurrency news and allow the script to function properly.

````
sentiment_key = os.getenv('sentiment_key')
websearch_key = os.getenv('websearch_key')

# user input variables
# Values are keyowords to search the web for
# Keys can be used to send a trading request if integrated with an Exchange
crypto_key_pairs = {"BTCUSD":"Bitcoin", "ETHUSD":"Ethereum", "LTCUSD":"Litecoin", "XRPUSD":"Ripple", "BATUSD":"BATUSD, basic attention token", "DSHUSD":"Dash Coin", "EOSUSD":"EOS", "ETCUSD":"ETC", "IOTUSD":"IOTA", "NEOUSD":"NEO", "OMGUSD":"OMISE Go", "TRXUSD":"Tron", "XLMUSD":"Stellar Lumens", "XMRUSD":"Monero", "ZECUSD":"Zcash"}

#define from published date
date_since = date.today() - timedelta(days=1)

#store inputs in different lists
cryptocurrencies = []
crypto_keywords = []

#Storing keys and values in separate lists
for i in range(len(crypto_key_pairs)):
    cryptocurrencies.append(list(crypto_key_pairs.keys())[i])
    crypto_keywords.append(list(crypto_key_pairs.values())[i])
````

Have a quick look at the `sentiment_key` and `websearch_key` variables. I used an environment variable that I created on my computer in order to safely integrate it into my code. If you’re not planning to share the code, you can simply define your keys as `sentiment_key = ‘YOUR_KEY_HERE’`.

The dictionary `crypto_key_pairs` contains the symbol name and keywords for each coin provided, making it easier to integrate with cryptocurrency exchanges like Binance. The values are analysed by our script, and the keys can be used by the Binance API to open trades. I will cover the integration with Binance in the next blog, so keep an eye out for that one.
Fetch the News

We are now defining the function responsible for fetching the news based on the parameters we have selected above. The code below loops through each keyword in `crypto_key_pairs` and returns today’s news for each cryptocurrency. Additional configuration is possible for the `querystring` variable if you wish to adjust the number of pages to search, as well as the size (in terms of the number of articles) for each page. Along with the `fromPublishedDate` which we have already defined, a `toPublishedDate` can also be specified.

`querystring = {“q”:str(crypto),“pageNumber”:“1”,“pageSize”:“30”,“autoCorrect”:“true”,“fromPublishedDate”:date_since,“toPublishedDate”:“null”}`

````
def get_news_headlines():
    '''Search the web for news headlines based the keywords in the global variable'''
    news_output = {}

    #TO DO - looping through keywords creates odd looking dicts. Gotta loop through keys instead.
    for crypto in crypto_keywords:

        #create empty dicts in the news output
        news_output["{0}".format(crypto)] = {'description': [], 'title': []}

        #configure the fetch request and select date range. Increase date range by adjusting timedelta(days=1)
        url = "https://contextualwebsearch-websearch-v1.p.rapidapi.com/api/search/NewsSearchAPI"
        querystring = {"q":str(crypto),"pageNumber":"1","pageSize":"30","autoCorrect":"true","fromPublishedDate":date_since,"toPublishedDate":"null"}
        headers = {
            'x-rapidapi-key': websearch_key,
            'x-rapidapi-host': "contextualwebsearch-websearch-v1.p.rapidapi.com"
            }

        #get the raw response
        response = requests.request("GET", url, headers=headers, params=querystring)

        # convert response to text format
        result = json.loads(response.text)

        #store each headline and description in the dicts above
        for news in result['value']:
            news_output[crypto]["description"].append(news['description'])
            news_output[crypto]["title"].append(news['title'])

    return news_output
````

## Sentiment Analysis for Each Article

The next function is designed to evaluate the sentiment of each article retrieved, assigning a value of 1 or 0 for each of the three sentiment categories recognised by the API: positive, neutral, and negative. These results are added to the news_output variable, which is structured in a list format. For example: news_output["Bitcoin"]["sentiment"]["positive"] = [1, 1, 1, 1, 1]. This output serves as the basis for calculating an average in our final function, enabling a deeper analysis of the overall sentiment trend.

````
def analyze_headlines():
    '''Analyse each headline pulled trhough the API for each crypto'''
    news_output = get_news_headlines()

    for crypto in crypto_keywords:

        #empty list to store sentiment value
        news_output[crypto]['sentiment'] = {'pos': [], 'mid': [], 'neg': []}

        # analyse the description sentiment for each crypto news gathered
        if len(news_output[crypto]['description']) > 0:
            for title in news_output[crypto]['title']:

                # remove all non alphanumeric characters from payload
                titles = re.sub('[^A-Za-z0-9]+', ' ', title)

                import http.client
                conn = http.client.HTTPSConnection('text-sentiment.p.rapidapi.com')

                #format and sent the request
                payload = 'text='+titles
                headers = {
                    'content-type': 'application/x-www-form-urlencoded',
                    'x-rapidapi-key': sentiment_key,
                    'x-rapidapi-host': 'text-sentiment.p.rapidapi.com'
                    }
                conn.request("POST", "/analyze", payload, headers)

                #get the response and format it
                res = conn.getresponse()
                data = res.read()
                title_sentiment = json.loads(data)

                #assign each positive, neutral and negative count to another list in the news output dict
                if not isinstance(title_sentiment, int):
                    if title_sentiment['pos'] == 1:
                        news_output[crypto]['sentiment']['pos'].append(title_sentiment['pos'])
                    elif title_sentiment['mid'] == 1:
                        news_output[crypto]['sentiment']['mid'].append(title_sentiment['mid'])
                    elif title_sentiment['neg'] == 1:
                        news_output[crypto]['sentiment']['neg'].append(title_sentiment['neg'])
                    else:
                        print(f'Sentiment not found for {crypto}')

    return news_output
````

## Determine the Overall Sentiment of All Retrieved Articles

In the final function, we will calculate the overall sentiment for each cryptocurrency as a percentage. Recall that each article is categorised as having a positive, neutral, or negative sentiment. By determining the total number of articles, we can calculate the proportion of sentiment that falls into each category. The formula is straightforward: the number of positive articles multiplied by 100, divided by the total number of articles. This method is applied in the same way to calculate percentages for neutral and negative sentiment as well.

````
def calc_sentiment():
    '''Use the sentiment returned in the previous function to calculate %'''
    news_output = analyze_headlines()

    #re-assigned the sentiment list value to a single % calc of all values in each of the 3 lists
    for crypto in crypto_keywords:

        #length of title list can't be 0 otherwise we'd be dividing by 0 below
        if len(news_output[crypto]['title']) > 0:

            news_output[crypto]['sentiment']['pos'] = len(news_output[crypto]['sentiment']['pos'])*100/len(news_output[crypto]['title'])
            news_output[crypto]['sentiment']['mid'] = len(news_output[crypto]['sentiment']['mid'])*100/len(news_output[crypto]['title'])
            news_output[crypto]['sentiment']['neg'] = len(news_output[crypto]['sentiment']['neg'])*100/len(news_output[crypto]['title'])

            #print the output  for each coin to verify the result
            print(crypto, news_output[crypto]['sentiment'])

    return news_output

# TO DO - If integrated with an exhange, add weight logic to avid placing a trade
# when only 1 article is found and the sentiment is positive.


if __name__ == '__main__':
    for i in count():
        calc_sentiment()
        print(f'Iteration {i}')
        time.sleep(900)
````

The final `if` statement ensures that the code runs every 15 minutes by using the `time.sleep(900)` function. You can adjust this interval to suit your needs, but do bear in mind that the websearch API has a limit of 100 free calls per day.





