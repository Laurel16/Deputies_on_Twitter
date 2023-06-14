
# Analyzing French Deputies' Tweets: Exploring Topics, Interactions, and Local Representation

In January 2021, I became intrigued by the idea of utilizing Natural Language Processing (NLP) to delve into the world of tweets posted by French deputies. I pondered whether it was possible to gain valuable insights by asking a few key questions: Which topics are most frequently addressed within each group? Do these topics significantly differ from one group to another? Who retweets whom? Moreover, I wondered whether the deputies, who are meant to represent local interests, serve as conduits for these local concerns or primarily engage in national political discussions.



<details>
  <summary><b><h2> Step 1 - Retrieve infos on active deputies</h2></b></summary>
In order to complete this task, I embarked on a thorough compilation of multiple sources.

To begin, I obtained the list of deputies from the official registry of the National Assembly. However, it quickly became apparent that this list was not comprehensive, as it failed to account for the deputies who had left their positions. To address this gap, I turned to a more meticulously maintained open data CSV file, regularly updated with the latest information: you can find it here: https://www.data.gouv.fr/fr/datasets/deputes-actifs-de-lassemblee-nationale-informations-et-statistiques/.

Although this supplementary resource had its limitations. It did not provide complete details regarding the Twitter accounts of all the deputies. 

At this time (not the case anymore) the Twitter account of the French National Assembly had compiled a list of deputy accounts. Thanks to this list I came up with a csv providing infos like name, screen name, location, twitter bio, followers count, friends count, url, date of creation of the acount. However, even this compilation was not entirely up to date, as some deputies were no longer serving as elected representatives of the nation. 

To compound matters, discrepancies in the spellings of names or the reversal of first and last names further complicated the task of matching and cross-referencing the data.
  
1. **Creation of a common column** 
  
  
  

  <img width="1053" alt="nb_1" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/415e5eda-e84e-4a00-bae3-516245ecf0dd">

2. **Examination of the correspondences between these columns:**

  <img width="1037" alt="nb_2" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/33d8c28e-934e-4b81-9e3d-420c7a35f8cb">

  
**3. Using fuzzymatcher (https://pypi.org/project/fuzzymatcher/) to reconcile columns that are almost but not quite identical.**
<img width="1014" alt="nb_3" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/fd301531-74ee-425f-b231-299da86bb65c">


I am checking the data, and there are about a dozen mismatches. Examining them individually allows me to identify some data that is not up to date in both original files.

Then I correct the dataset, by manually removing the deputies who are no longer in office or addressing the inaccuracies. Finally, I have a dataframe containing 517 active deputies with a Twitter account.

<img width="1038" alt="nb_4" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/2b0e312c-44dd-4041-a331-19c44073e7bf">

  
  
To finish, I dropped a dozen columns from my final dataset that I won't be using, such as the seat number in the hemicycle and renaming certain columns with more friendly names.
  
  
  ```python
 data.drop(columns=['placeHemicycle'], inplace=True) 
  
 data = data.rename(columns={'created_at_x': 'account_created_at'})
 ```



</details>

<details>
  <summary><b><h2> Step 2 - Retrieve tweets for deputies</h2></b></summary>

I requested an API key from Twitter.
  
Due to the limitations on free requests, it was not possible to retrieve the tweets of 517 deputies in one go. I had to plan for moments when the requests would be paused, as well as network interruptions or times when I had to shut down my computer to go home, without losing the information I had already retrieved.

It took me almost an entire week to go through my list of screen names, little by little.
  
Here is the code I used: 
  
data.py 

```python
  
import os
import os.path
import pandas as pd
import tweepy #https://github.com/tweepy/tweepy
import csv

#Twitter API credentials
consumer_key = "your key"
consumer_secret = "your key"
access_key = "your key"
access_secret = "your key"
  
def get_dep_info():
        """
        This function returns a Python dict.
        Its keys should be 'sellers', 'orders', 'order_items' etc...
        Its values should be pandas.DataFrame loaded from csv files
        """

        root_dir = os.path.dirname(__file__)
        csv_path = os.path.join(root_dir, "data", "dep_info.csv")
        dep_info = pd.read_csv(os.path.join(csv_path))
        return dep_info


def tweets_from_deputy(deputy, count):

    #Twitter only allows access to a users most recent 3240 tweets with this method

    #authorize twitter, initialize tweepy
    auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
    auth.set_access_token(access_key, access_secret)
    api = tweepy.API(auth, wait_on_rate_limit=True,wait_on_rate_limit_notify=True, retry_count = 5, #retry 5 times
                   retry_delay = 5 #seconds to wait for retry
                )


    deputy_tweets = []

    # get all the tweets and retweets of the deputy
    for status in tweepy.Cursor(api.user_timeline, screen_name=deputy, tweet_mode="extended").items(count):

        # create a list of tweets
        deputy_tweets.append(status)

    # fill full text for retweets
    for tweet in deputy_tweets:

        # get tweet type
        status = api.get_status(tweet.id, tweet_mode="extended")

        # check if this is a tweet or a retweet
        if hasattr(status, "retweeted_status"):
            tweet.full_text = f"RT => {status.retweeted_status.full_text}"
            tweet.favorite_count = status.retweeted_status.favorite_count  # likes

    # create the structure to store for CSV
    tweets_list = []

    for tweet in deputy_tweets:

        # transform the tweepy tweets into a 2D array that will populate the csv
        # outtweets = [[tweet.user.name, tweet.user.id, tweet.id_str, tweet.created_at, tweet.full_text, [text['text'] for text in tweet.entities["hashtags"]], tweet.retweet_count, tweet.favorite_count ] for tweet in alltweets]

        # create a list for each observation
        tweets = [tweet.user.name, tweet.user.id, tweet.id_str, tweet.created_at, tweet.full_text]
        tweets.append([text['text'] for text in tweet.entities["hashtags"]])
        tweets += [tweet.retweet_count, tweet.favorite_count]

        tweets_list.append(tweets)

    return tweets_list


def write_tweet_csv(tweets_list):
    root_dir = os.path.dirname(__file__)
    csv_path = os.path.join(root_dir, "data", "all_deputy_tweets.csv")
    file_exists = os.path.isfile(csv_path)

    #write the csv
    with open(csv_path, 'a', encoding='utf-8') as f:
          writer = csv.writer(f)
          if not file_exists:
              writer.writerow(["name", "user_id", "tweet_id","created_at","text", "hashtags", 'retweet_count', 'like_count'])
          writer.writerows(tweets_list)


def get_all_tweets(tweet_per_deputy, deput_list):

    # iterate through all the deputies
    all_tweets = []
    for deputy in deput_list:
        print(f"get tweets for {deputy}")
        all_tweets = []

        # get deputy tweets
        dep_tweets = tweets_from_deputy(deputy, tweet_per_deputy)
        all_tweets += dep_tweets

        # write csv for all deputies
        print(f"write tweets for {deputy}")

        write_tweet_csv(all_tweets)
        print(f"CSV writed for {deputy}")


```
  

I called the get_all_tweet function in a jupyter notebook. 
I set the tweet_per_deputy to 800.
I changed the content of the `deput_list` by updating it with new screen names each time the program broke.

The output looked like this:
<img width="1053" alt="nb_30" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/77e9df5e-c3d7-4c75-aed6-814b5d324317">

  
And the final result:
<img width="933" alt="nb_31" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/248ea223-4064-4b4b-a632-5c11460de3fd">



</details>

<details>
  <summary><b><h2> Step 3 - Prepare data </h2></b></summary>
  
**1. Verify and remove null values**

```Python
data['text'].isnull().sum()
null_rows = data[data['text'].isnull()]
null_rows
data['text'].fillna('N/A', inplace=True)
# Be sure you will manipulate strings
data['text'] = data['text'].astype(str)
```
 **2. Picking emoji in a separate column (and removing them from the array they were in).**
    
  <img width="893" alt="nb_8" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/7d2a8189-5fee-4945-9728-2972fc4500e2">
<img width="902" alt="nb_9" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/1d86640a-ea78-4ec2-85d1-25de55063475">


 **3. Creating a column that indicates whether a tweet is a retweet or not (a categorical boolean column: true or false, and then 0 or 1).**
     <img width="895" alt="nb_11" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/81cfefb6-a721-4aa1-b015-a8ab98619cec">
<img width="895" alt="nb_12" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/67e19b7c-8b2f-45bd-bd67-9fc5b2ab536f">

  
**4. Extract links from tweets in a separate column and remove links from tweet content**
  <img width="892" alt="nb_15" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/bc218bd5-4c50-4892-a84c-fe26f1b7d8b6">
<img width="890" alt="nb_16" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/e225aab3-d60e-4c0b-be92-c20336472afc">
<img width="903" alt="nb_17" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/66597da1-2b12-48b6-8e75-260a2632cf45">

 
**5.clean and tokenize tweet content**
     
A generic function for this process would look like this:

```Python
     
import pandas as pd
import re
import nltk
from nltk.corpus import stopwords 
from nltk import word_tokenize
from nltk.stem import WordNetLemmatizer

# define a string of punctuation symbols
punctuation = '!"$%&\'()*+,-./:;<=>?[\\]^_`{|}~•@'


# functions to clean tweets
def remove_links(tweet):
    """Takes a string and removes web links from it"""
    tweet = re.sub(r'http\S+', '', tweet)   # remove http links
    tweet = re.sub(r'bit.ly/\S+', '', tweet)  # remove bitly links
    tweet = tweet.strip('[link]')   # remove [links]
    tweet = re.sub(r'pic.twitter\S+','', tweet)
    return tweet


def remove_users(tweet):
    """Takes a string and removes retweet and @user information"""
    tweet = re.sub('(RT\s@[A-Za-z]+[A-Za-z0-9-_]+)', '', tweet)  # remove re-tweet
    tweet = re.sub('(@[A-Za-z]+[A-Za-z0-9-_]+)', '', tweet)  # remove tweeted at
    return tweet


def remove_hashtags(tweet):
    """Takes a string and removes any hash tags"""
    tweet = re.sub('(#[A-Za-z]+[A-Za-z0-9-_]+)', '', tweet)  # remove hash tags
    return tweet


def remove_av(tweet):
    """Takes a string and removes AUDIO/VIDEO tags or labels"""
    tweet = re.sub('VIDEO:', '', tweet)  # remove 'VIDEO:' from start of tweet
    tweet = re.sub('AUDIO:', '', tweet)  # remove 'AUDIO:' from start of tweet
    return tweet

     

def tokenize(tweet):
    """Returns tokenized representation of words in lemma form excluding stopwords"""
    tokenized = word_tokenize(tweet) # Tokenize
    words_only = [word for word in tokenized if word.isalpha()] # Remove numbers
    stop_words = set(stopwords.words('french')) # Make stopword list
    without_stopwords = [word for word in words_only if not word in stop_words] # Remove Stop Words
    lemma=WordNetLemmatizer() # Initiate Lemmatizer
    lemmatized = [lemma.lemmatize(word) for word in without_stopwords] # Lemmatize
    return lemmatized
 

def preprocess_tweet(tweet):
    """Main master function to clean tweets, stripping noisy characters, and tokenizing use lemmatization"""
    tweet = remove_users(tweet)
    tweet = remove_links(tweet)
    tweet = remove_hashtags(tweet)
    tweet = remove_av(tweet)
    tweet = tweet.lower()  # lower case
    tweet = re.sub('[' + punctuation + ']+', ' ', tweet)  # strip punctuation
    tweet = re.sub('\s+', ' ', tweet)  # remove double spacing
    tweet = re.sub('([0-9]+)', '', tweet)  # remove numbers
    tweet_token_list = tokenize(tweet)  # apply lemmatization and tokenization
    tweet = ' '.join(tweet_token_list)
    return tweet


def basic_clean(tweet):
    """Main master function to clean tweets only without tokenization or removal of stopwords"""
    tweet = remove_users(tweet)
    tweet = remove_links(tweet)
    tweet = remove_hashtags(tweet)
    tweet = remove_av(tweet)
    tweet = tweet.lower()  # lower case
    tweet = re.sub('[' + punctuation + ']+', ' ', tweet)  # strip punctuation
    tweet = re.sub('\s+', ' ', tweet)  # remove double spacing
    tweet = re.sub('([0-9]+)', '', tweet)  # remove numbers
    tweet = re.sub('📝 …', '', tweet)
    return tweet


def tokenize_tweets(df):
    """Main function to read in and return cleaned and preprocessed dataframe.
    This can be used in Jupyter notebooks by importing this module and calling the tokenize_tweets() function

    Args:
        df = data frame object to apply cleaning to

    Returns:
        pandas data frame with cleaned tokens
    """

    df['tokens'] = df.tweet.apply(preprocess_tweet)
    num_tweets = len(df)
    print('Complete. Number of Tweets that have been cleaned and tokenized : {}'.format(num_tweets))
    return df

 ```
  
Then apply the function:
  <img width="1011" alt="nb_34" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/94ac3514-fe86-4acb-80f2-dc912ad4cb7c">
<img width="924" alt="nb_40" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/c5b74cb3-cd6d-43b0-a610-35de6c1ccab3">

  
Depending on the process, it could be necessary to convert the list to a string after using some function (avoiding error "TypeError: expected string or bytes-like object"):

```Python
# Apply to all texts
tweets_df['text_no_stop_word'] = tweets_df['text'].apply(tokenize)
# Since the "tokenize" function returns a list, convert this list into a string.
tweets_df['text_no_stop_word'] = tweets_df['text_no_stop_word'].apply(lambda x: ' '.join(map(str, x)))


```
  
When needed, it woul be usefull to add customs stop words to the generic one: 

```Python
custom_stopwords = ['mot1', 'mot2', 'mot3']  
stopwords_list = stop_words('french').union(custom_stopwords)
print(sorted(stop_words))

```
  
To identify the most frequent words to be removed in order to minimize the noise:

<img width="1065" alt="nb_46" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/73c3f9f9-7322-4a1d-9783-fabc6224038d">

  
**5. Get the citation of other tweetos in a new column**

<img width="1042" alt="nb_45" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/8a323377-9438-4e49-a0a0-3084540971ec">


**6 Add a code column for groups**
  
    
<img width="895" alt="nb_14" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/db5c04a9-e210-4d5e-b4cc-a2353d4ba8ab">

<img width="908" alt="nb_7" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/b5a0cfbe-ab3a-49b4-adcf-3b4de0ce675a">

   
**7. Convert datetime to date and add a year column**
  <img width="910" alt="nb_38" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/76dea9f0-b934-4422-8a69-f854af06da3a">


</details>
  
  <details>
  <summary><b><h2> Step 4 - Data Explorations </h2></b></summary>
  
**1. How are distributed tweets over time ?**
<img width="898" alt="nb_41" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/714930e0-b880-43f6-ba32-ffed276c73d3">

**2. Who are the 20 most followed deputies?**
    
<img width="1006" alt="nb_5" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/8ea0286e-7eee-4621-aa6e-8073da559baf">
 
 The top two, far ahead, are the leaders of the far-right and far-left parties.
    
    <img width="824" alt="nb_6" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/5416f375-99f9-40ac-a2d5-c23ea18f532f">


    
**3. Which group retweet the most ?**
     

<img width="899" alt="nb_13" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/2e60cd47-e575-4eda-91d0-3f176a1b504d">

**4. What are the main topics per group**

Bi-grams and tri-grams

  If not done before:
  <img width="900" alt="nb_20" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/62469e6a-b831-4682-a970-2b53e3ac465a">
<img width="850" alt="nb_21" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/16d40d18-6525-4408-889f-9b79cb33b2d1">
<img width="863" alt="nb_22" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/d30b9c6e-c496-4d62-938f-b0e25b226007">
<img width="860" alt="nb_23" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/0b032023-b1be-4d9d-9eb8-7330fd730f5e">

  


<img width="1050" alt="1" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/385af8de-439b-4a3c-854a-7da876a665a4">
  <img width="984" alt="3" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/0ebd2e8b-ddc9-4d2a-b495-8e4f29324360">
<img width="1023" alt="4" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/c608148a-fc5c-44e1-a9e1-74596a805c91">
  
<img width="929" alt="5" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/36bbfdd8-eb75-49c8-aefd-c33b7453274f">


  
</details>
  
  

  





