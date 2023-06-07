
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
  
  
  <img width="1053" alt="nb_1" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/422c6a8f-5327-466f-a8ca-924f74a6f190">

  
2. **Examination of the correspondences between these columns:**

  
  <img width="1037" alt="nb_2" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/b618838c-6780-4a99-bff2-0089f9824321">

  
**3. Using fuzzymatcher (https://pypi.org/project/fuzzymatcher/) to reconcile columns that are almost but not quite identical.**

<img width="1014" alt="nb_3" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/591638e8-c21a-46d3-a643-f5b18cbad5ad">

I am checking the data, and there are about a dozen mismatches. Examining them individually allows me to identify some data that is not up to date in both original files.

Then I correct the dataset, by manually removing the deputies who are no longer in office or addressing the inaccuracies. Finally, I have a dataframe containing 517 active deputies with a Twitter account.

<img width="1038" alt="nb_4" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/b4868997-9eb5-433d-8ff9-40e0694cf20c">

  
  
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

<img width="1053" alt="nb_30" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/72dfd149-133c-4669-9647-f4d0fb6f4214">

  
And the final result:


<img width="933" alt="nb_31" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/1e69ac12-7dca-44e6-8eed-00466482f144">



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
    
  
<img width="893" alt="nb_8" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/17d529dd-ca49-40dd-a1f9-56e340fd57e7">

<img width="902" alt="nb_9" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/c1053229-8444-4342-8021-730aae78dcac">

 **3. Creating a column that indicates whether a tweet is a retweet or not (a categorical boolean column: true or false, and then 0 or 1).**
     
  <img width="895" alt="nb_11" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/84f892d5-a17e-4d75-86cc-280914df62b7">
<img width="895" alt="nb_12" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/02551ff9-39a6-41ec-9b4e-a71bec846bbb">

**4. Extract links from tweets in a separate column and remove links from tweet content**
     
     <img width="892" alt="nb_15" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/14be12df-5086-45c0-9fc8-78bd94b68863">
<img width="890" alt="nb_16" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/bc5bb0d3-00aa-4386-98f4-25fd5566b992">
<img width="903" alt="nb_17" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/019412d5-33e1-4940-8a05-615e13266539">

 
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
  
<img width="1011" alt="nb_34" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/a1790a89-fed7-4fb6-b0be-a4ec788afa43">
<img width="924" alt="nb_40" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/1dfdb35e-ea5f-431f-8ecd-8cca49bfeabd">

  
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


   <img width="1065" alt="nb_46" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/04c6d3e9-a82d-47c9-bcc5-0b62783769e7">


  
**5. Get the citation of other tweetos in a new column**
<img width="1042" alt="nb_45" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/e13ee189-77ba-4718-9915-92fb4d0a741d">



**6 Add a code column for groups**
  
     
<img width="895" alt="nb_14" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/1490ef94-27b2-493a-9612-53670b9156eb">


    
   <img width="908" alt="nb_7" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/12b600e7-c779-48fe-a1aa-bbfac29819cd">

**7. Convert datetime to date and add a year column**
  
<img width="910" alt="nb_38" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/48d795ad-0bbc-446a-9e56-8e718d6ae61e">


</details>
  
  <details>
  <summary><b><h2> Step 4 - Data Explorations </h2></b></summary>
  
**1. How are distributed tweets over time ?**

<img width="898" alt="nb_41" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/0172c5be-67b7-4d20-afb6-0992c5c5d6d2">

**2. Who are the 20 most followed deputies?**
    

<img width="1006" alt="nb_5" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/7c622c48-2b03-47d8-a6ef-683e00414d16">

 
 The top two, far ahead, are the leaders of the far-right and far-left parties.
    
    
<img width="824" alt="nb_6" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/5b0f43f5-9d36-4436-af59-fbabcf17b461">


    
**3. Which group retweet the most ?**
     
<img width="899" alt="nb_13" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/7f28365c-6811-47e6-bfaf-f5e6b8e93f5a">


**4. What are the main topics per group**

Bi-grams and tri-grams

  If not done before:
  
  <img width="900" alt="nb_20" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/ee11825f-6b6c-4598-a008-a07ecee643f0">
  
  <img width="850" alt="nb_21" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/86786cc3-05df-44e8-a03a-82aef452f139">
<img width="863" alt="nb_22" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/796b5489-3702-4d50-8fb8-9fd68c3d7540">
<img width="860" alt="nb_23" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/8777455c-28e7-4fb7-a8ec-7a9079e3fa42">




<img width="1012" alt="nb_24" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/01bda1e3-ba4b-4b60-9616-09908a3992d2">
<img width="1011" alt="nb_25" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/8ea9fe2b-0550-4ac5-ae7b-a0a1fda527d7">
<img width="913" alt="nb_26" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/80c0f0a5-e793-479a-8212-dbe1179aead3">
<img width="1013" alt="nb_27" src="https://github.com/Laurel16/Deputies_on_Twitter/assets/16537140/9003d426-f47d-4762-90c8-732c0aad0ead">

  
</details>
  
  

  





