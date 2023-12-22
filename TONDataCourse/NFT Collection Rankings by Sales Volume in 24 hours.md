# How to find popular NFT collections in the TON blockchain - ranking collections using the dton.io indexer and Python

When trading any asset, you need to understand the current state of the market, and for NFTs this is no exception. In this tutorial, I will show you how to collect information on sales volume by collection over the last 24 hours in the TON blockchain.
# Why is this necessary?
Before diving into the code and how the dton.io indexer works, let’s talk about why we might need it. Let's say we want to purchase an NFT for further resale, which, roughly speaking, is important to us:
Increasing the price of a collection - in order to resell it for more than what we bought
Liquidity - so that if the price rises, a buyer can be found
A rough understanding that the first two points are not a consequence of fraudulent actions - for example, laundering trading (how to search for laundering transactions in collections was described in [another article](https://t.me/ton_learn/43))
If we move away from general words about liquidity and price growth, then the algorithm could be as follows:
Understand the current general state of the market and specific blockchains
After deciding on the favorable state of the market for purchasing and choosing a blockchain, find out which collections are currently popular on this blockchain
Analysis of certain collections - look at the historical floor, sales history, number of unique owners, etc.
After choosing a collection, look for undervalued items in them to purchase

Of course, this is not the only strategy - you can look at what collections are on the go or what whales are buying, but still, the step of reviewing the state of the market and searching for popular collections is now difficult to skip in order to understand the overall picture.

An example of the result we will get can be seen here: 

https://tonlearn.tools/#/ton-nft-sales-volume

## How can dton.io help you find information?

Dton.io is a blockchain indexer, which means it collects information from each new block into its database. You can make GraphQL queries into this database and thus collect historical information without parsing the entire blockchain.

The two most important sources of dton.io data are the transaction table and a view of the latest state of wallets/accounts on the network. Make queries in this view and table, you can collect almost any information.

It is important to note that the dton.io indexer collects not only raw data from the TON blockchain, but also enriches the data - this is very convenient, as we will see in the example below.

##Installing dependencies

As we know, GraphQL requests are sent in the body of an HTTP POST request, so let’s install requests and other necessary dependencies:


```python
# install requriments
!pip install requests

import requests
import string 
import time
from itertools import groupby
```

Let’s also denote the endpoint dton.io:

```python
endpoint = 'https://dton.io/graphql/'
```

## Let's calculate how many NFT sales there are in total

Since we want to count the number of transactions over a certain time, we will use the lastTransactionCountSegments view:


```python
#how many sales were there in the last 24 hours
def getDaySaleNum(endpoint):
  query = '''
    query {
    lastTransactionCountSegments(
	segmentation: "hour"
    ) { cnt segment}
  }

This view returns the number of transactions per segment, at most an hour. Since data can be received by pages and specify the number of data per page (maximum 150), you can get data for 24 hours by adding `page_size: 24` But we don’t need all transactions, but only sales…. .

To understand how to cut off sales, you need to understand the standards of smart contracts, specifically here you need to understand how the sale of NFTs in TON works. I have a separate tutorial with an analysis of smart sales contracts, and in this tutorial I will say that the main thing for us is that sales smart contracts have a special get-method `get_sale_data`, which will allow us to extract everything from sales related transactions.

Since several transactions are involved in the sale, we need one that kind of closes the sale, and here the fact that dton.io has data enrichment comes to the rescue; the `parsed_seller_is_closed: 1` field will help to understand whether a transaction closes the deal. We get:


```python
#how many sales were there in the last 24 hours
def getDaySaleNum(endpoint):
  query = '''
    query {
    lastTransactionCountSegments(
      segmentation: "hour"
      account_state_state_init_code_has_get_sale_data: 1
      parsed_seller_is_closed: 1
      page_size: 24
    ) { cnt segment}
  }
  '''

  response = requests.post(endpoint, json={'query': query})

  data = response.json()['data']['lastTransactionCountSegments']
  dayTxes = sum(item['cnt'] for item in data)
  return dayTxes
```

At the end, we sum it up and get the number of NFT sales in the TON network in 24 hours.

An experienced reader may say that this does not take into account all possible situations of the right to transfer an NFT, since there are three of them:
sale
auction
offer (offer to purchase with subsequent purchase)
And indeed, this method covers sales and auctions, but within the framework of a business task, offers should be placed in a separate dashboard, since they do not particularly reflect the picture as a whole and require detailed consideration separately.

## Collect all 24 hour sales 

Now we know how many transactions we need to get. Let's take into account that since only 150 values can be obtained on the page, we will have to send requests in batches, concatenating them into one request. I’ll say right away that extra sales will not be cut off, so as not to complicate the tutorial:

```python
day_sales_count = getDaySaleNum(endpoint)

page_size = 100

needed_queries = day_sales_count // page_size + 1


formatted_query =""
for batch_num in range(needed_queries):
  print(batch_num)
  template = string.Template("""
    q${page}: transactions(
      account_state_state_init_code_has_get_sale_data: 1
      parsed_seller_is_closed: 1
      workchain: 0
      page_size: ${page_size}
      page: ${page}
      ) {
      collection: parsed_seller_nft_collection_address_address
      price: parsed_seller_nft_price
      nft: parsed_seller_nft_address_address
      block_time:gen_utime
      }
  """)

  formatted = template.substitute(page=batch_num,page_size=page_size)
  formatted = formatted +"\n"
  formatted_query += formatted

formatted_query = """query {"""+formatted_query+"""}"""
formatted_query
response = requests.post(endpoint, json={'query': formatted_query})

res_arr = []

for v in response.json()['data'].values():
  for item in v:
    if(item['collection'] is not None):
      res_arr.append(item)

print("Collected txes: ",len(res_arr))
```

You may notice that we do not take NFTs without a collection (where None), and the NFT standard in TON implies the possibility of NFTs without a collection, since unlike EVM networks, NFT is not one contract storing a certain database of records, but one smart collection contract and separate contracts for each element.

Also, a reader well familiar with blockchain technology can say, why make two requests, first how many, and then transactions, you can get the block generation time with each transaction, and by unloading all transactions in a row, you can break out of the cycle on blocks that were 24 hours. Yes, this is true, but for ease of code readability and to show you more about dton.io, I decided to do it through two requests.

Let's move on.

# Group and count the number of transactions and sales volume

Regarding the collection, let’s sum up the sales, this will be the sales volume, and also calculate the number of transactions. Why do we need both quantity and amount? One amount does not reflect well the situation in the collection, one large sale per day is worse for us than many medium ones - liquidity is important for resale.

```python
#We group the data by collection - collection, number of sales, sales volume and of course sort
# define a function for key
def key_func(k):
    return k['collection']

# sort INFO data by 'company' key.
INFO = sorted(res_arr, key=key_func)

count_arr=[]

for key, value in groupby(INFO, key_func):
    #print(key)
    #print(list(value))
    sum=0
    count=0
    for item in list(value):
      count = count + 1
      sum = sum + int(item['price'])
      #print(item['employee'])

    #print("Txes:",count)
    #print("Sum:",sum/1000000000)
    temp_d = {'col': key, 'txes': count, 'sum': round(sum/1000000000)}
    count_arr.append(temp_d)


newlist = sorted(count_arr, key=lambda d: d['sum'], reverse=True)
print("Size of sorted Arr: ",len(newlist))
```

Let's see what we have:

```python
newlist[0:5]
```

	[{'col': 'B774D95EB20543F186C06B371AB88AD704F7E256130CAF96189368A7D0CB6CCF',
	  'txes': 59,
	  'sum': 332419},
	 {'col': 'E06664290C96CEEA5687BEF89B6976173251006041FA2E752D039275F56D7C74',
	  'txes': 2,
	  'sum': 100098},
	 {'col': '28F760D832893182129CABE0A40864A4FCC817639168D523D6DB4824BD997BE6',
	  'txes': 32,
	  'sum': 12053},
	 {'col': 'D398C94E9AB5BE730AA5E4DBCEF4754F3FB1B6805B61BEB15B31052E6C0E838C',
	  'txes': 106,
	  'sum': 11737},
	 {'col': 'B3B928E4814343EB597B8081F736F55636D674457D61C73D6B5A7989A863E666',
	  'txes': 3,
	  'sum': 10052}]

It looks like it’s not very clear what kind of collection it is, and what kind of cumbersome address it is.

# Addresses in TON and a little trick with TON blockchain explorer

Addresses of smart contracts running in TON are expressed in two main formats:
Raw hex addresses: raw full representation of smart contract addresses
User-friendly addresses: extended addresses for user convenience
More information about addresses in [documentation](https://docs.ton.org/learn/overviews/addresses). To convert an address from one type to another, we will use a script, which we will not analyze in this tutorial, but at the very end of the tutorial there will be all links to the code.

It would also be cool to get information about the collection, the name, a picture of the collection, and generally see what the collection is. You can, of course, get all the data from the blockchain, but the NFT standard in TON is not focused on quick access to the entire collection, so for the first step, I suggest taking advantage of the fact that one of the explorers allows you to view collections by the collection address. Let's take advantage of this.



```python
for row in newlist:
  row['explorer_link'] = "https://tonscan.org/nft/"+account_forms('0:'+row['col'])['bounceable']['b64url']

newlist[0:5]
```
We get:

	[{'col': 'D398C94E9AB5BE730AA5E4DBCEF4754F3FB1B6805B61BEB15B31052E6C0E838C',
	  'txes': 108,
	  'sum': 289703904,
	  'explorer_link': 'https://tonscan.org/nft/EQDTmMlOmrW-cwql5NvO9HVPP7G2gFthvrFbMQUubA6DjH7-'},
	 {'col': 'B774D95EB20543F186C06B371AB88AD704F7E256130CAF96189368A7D0CB6CCF',
	  'txes': 34,
	  'sum': 510053,
	  'explorer_link': 'https://tonscan.org/nft/EQC3dNlesgVD8YbAazcauIrXBPfiVhMMr5YYk2in0Mtsz0Bz'},
	 {'col': '80D78A35F955A14B679FAA887FF4CD5BFC0F43B4A4EEA2A7E6927F3701B273C2',
	  'txes': 20,
	  'sum': 301248,
	  'explorer_link': 'https://tonscan.org/nft/EQCA14o1-VWhS2efqoh_9M1b_A9DtKTuoqfmkn83AbJzwnPi'},
	 {'col': 'A27781584DA7136F20D0D3BC1ECEC5D00BCC82C720E08BF0A81BD3063D704628',
	  'txes': 13,
	  'sum': 30868,
	  'explorer_link': 'https://tonscan.org/nft/EQCid4FYTacTbyDQ07wezsXQC8yCxyDgi_CoG9MGPXBGKLJ_'},
	 {'col': '82AADDD0778EE241E576637F17D1DA83E85AA1CAC4D7B47D5B075D6A089CB218',
	  'txes': 264,
	  'sum': 17350,
	  'explorer_link': 'https://tonscan.org/nft/EQCCqt3Qd47iQeV2Y38X0dqD6FqhysTXtH1bB11qCJyyGG8G'}]
	  
By following the link we can view the collection, but what if we want to get the data ourselves.

# The easiest way to get data about the collection

The most convenient option is to receive data via the http protocol. Since TON nodes use their own ADNL transport protocol (more about it below), an intermediate service is required for the HTTP connection.

Toncenter is such an intermediate service that receives requests via HTTP; it accesses liteserver’s of the TON network. It has a /getTokenData method that returns token information according to the TON token data standard.

```python
API_ENDPOINT = 'https://toncenter.com/api/v2'

for row in newlist:
  try:
    payload = {'address': '0:'+row['col']}
    r = requests.get(API_ENDPOINT+'/getTokenData', params=payload)
    col_content = r.json()
    print(col_content)
    # limit because we dont use api key 
    time.sleep(1.1)
  except:
    print('some error')
```

In total, we get ranked collections, ready for viewing, an approximate result can be seen here: https://tonlearn.tools/#/ton-nft-sales-volume

# Collection data directly from the blockchain without intermediaries

It was mentioned above that TON has its own transport protocol, but what should we request from Liteserver? To understand this, you need to understand what a collection actually is; there is an analysis [here](https://t.me/ton_learn/17).

Let’s say we figure it out and understand that a smart contract is an NFT collection in TON if it meets some conditions, one of which is that there must be a get_collection_data() method that will return this data.

Thus, we need to call the GET method get_collection_data() on the collection smart contract and thus enrich the data.

Data collection through ADNL is a fairly extensive topic, which I covered [here](
https://github.com/romanovichim/TonFunClessons_Eng/blob/main/lessons/requests/ADNL/adnlgetsale.md), and after some time I will publish this on dev.to.

# Conclusion

I like the TON blockchain for its technical elegance; at least it’s not another copy of Ethereum, which is being overclocked with the help of a lot of capital without looking back, and in general why the user needs it. If you want to learn more about the TON blockchain, I have open source lessons that will teach you how to create full-fledged applications on TON.

https://github.com/romanovichim/TonFunClessons_Eng

I post new tutorials and data analytics here: https://t.me/ton_learn

You can see the ranking of collections from this tutorial in a convenient form here: https://tonlearn.tools/#/ton-nft-sales-volume

Code from this tutorial: https://colab.research.google.com/drive/1Nm6ZJa1I892dq10JKpUuyijlV-SZTSRr?usp=sharing


