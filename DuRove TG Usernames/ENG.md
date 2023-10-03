# Purchases of Telegram Usernames made by Pavel Durov using Python and the dton.io indexer

##  For what?

Recently, a lot of attention has been paid to Pavel Durov’s purchases:

- Before the appearance of group stories in telegram with boost mechanics, Pavel buys Telegram Username boost
- On September 25, Pavel buys fabrika and a week later connects it to the bot of the Friends Factory project

Well, just a treasure trove of insights

### How do we know that Pavel’s wallet

The wallet was identified by the username nerd when it was found on a possible account of Pavel Durov, onetimeusername.

Information about this can also be found in the list of known Tonkeeper wallets

https://raw.githubusercontent.com/tonkeeper/ton-assets/main/accounts.json

### Problem viewing the wallet through explorers

If you want to look at Pavel’s wallet through an explorer, it will be difficult to find interesting information; every day a lot of spam transactions fall there and a lot of tokens and NFTs are transferred there for PR of their projects.

Therefore, it would be interesting to see information about TG usernames separately.

## What is dton.io?

Dton.io is a blockchain indexer, which means it collects information from each new block into its database. You can make GraphQL queries into this database and thus collect historical information without parsing the entire blockchain.

The two most important sources of dton.io data are the transaction table and a view of the latest state of wallets/accounts on the network. Make queries in this view and table, you can collect almost any information.

In this tutorial, we will need the states of wallets; we will take the balance

## Install dependencies

As we know, GraphQL requests are sent in the body of an HTTP POST request, so let’s install requests and other necessary dependencies:


```python
# install requriments
!pip install requests

import requests
import string 
```

dton.io endpoint for our requests:

```python
endpoint = 'https://dton.io/graphql/'
```

# We begin to collect the request

We will collect transactions to understand which transactions are sales
Let's use the `parsed_seller_is_closed` field:

```python
def getDuRoveNFT(endpoint):
  query = '''
  {
    transactions(
    parsed_seller_is_closed: 1

    ) {
    block_time: gen_utime
    }
  }
  '''
  response = requests.post(endpoint, json={'query': query})

  data = response.json()['data']['transactions']
  return data
```

### We cut off Pavel's purchases and Telegram Usernames

Since we are only interested in usernames, we will add a filter by collection, and also add a filter by Pavel’s account (if he buys, then he actually becomes the new owner):

```python
def getDuRoveNFT(endpoint):
  query = '''
  {
    transactions(
    parsed_seller_is_closed: 1
    workchain: 0
    page_size: 100
    page: 0
    parsed_seller_nft_new_owner_address_address: "D8CD999FB2B1B384E6CA254C3883375E23111A8B78C015B886286C31BF11E29D"
    parsed_seller_nft_collection_address_address: "80D78A35F955A14B679FAA887FF4CD5BFC0F43B4A4EEA2A7E6927F3701B273C2"
    ) {

    block_time: gen_utime
    }
  }
  '''

  response = requests.post(endpoint, json={'query': query})

  data = response.json()['data']['transactions']
  return data
```

### Let's select the information we need

Now we have all the necessary transactions, all that remains is to select the information that we want to display, this is:
- Date and time of the block in which the purchase occurred
- Price
- And of course, what kind of username is that?

Here I will note that the Telegram Usernames collection has a very convenient link in the data cell; the URL itself already contains the name of the username and there is no need to make a separate request to get the name.

We get:

```python
def getDuRoveNFT(endpoint):
  query = '''
  {
    transactions(
    parsed_seller_is_closed: 1
    order_by: "gen_utime"
    order_desc: true
    workchain: 0
    page_size: 100
    page: 0
    parsed_seller_nft_new_owner_address_address: "D8CD999FB2B1B384E6CA254C3883375E23111A8B78C015B886286C31BF11E29D"
    parsed_seller_nft_collection_address_address: "80D78A35F955A14B679FAA887FF4CD5BFC0F43B4A4EEA2A7E6927F3701B273C2"
    ) {
    nft: parsed_seller_nft_address_address
    price: parsed_seller_nft_price
    block_time: gen_utime
    offchain_url: parsed_nft_content_offchain_url
    }
  }
  '''

  response = requests.post(endpoint, json={'query': query})

  data = response.json()['data']['transactions']
  return data
```

### The final touch

To make it easier to view, let’s sort the data by block time. To sort you need to use the `order_by` field

```python
def getDuRoveNFT(endpoint):
  query = '''
  {
    transactions(
    parsed_seller_is_closed: 1
    order_by: "gen_utime"
    order_desc: true
    workchain: 0
    page_size: 100
    page: 0
    parsed_seller_nft_new_owner_address_address: "D8CD999FB2B1B384E6CA254C3883375E23111A8B78C015B886286C31BF11E29D"
    parsed_seller_nft_collection_address_address: "80D78A35F955A14B679FAA887FF4CD5BFC0F43B4A4EEA2A7E6927F3701B273C2"
    ) {
    nft: parsed_seller_nft_address_address
    price: parsed_seller_nft_price
    block_time: gen_utime
    offchain_url: parsed_nft_content_offchain_url
    }
  }
  '''

  response = requests.post(endpoint, json={'query': query})

  data = response.json()['data']['transactions']
  return data
```


## What can be improved

In the following tutorials we will look at other TON whales, and also add information not only on NFT purchases)

## Conclusion

I like the TON blockchain for its technical elegance; at least it’s not another copy of Ethereum, which is being overclocked with the help of a lot of capital without looking back, and in general why the user needs it. If you want to learn more about the TON blockchain, I have open source lessons that will teach you how to create full-fledged applications on TON.

https://github.com/romanovichim/TonFunClessons_Eng

I post new tutorials and data analytics here: https://t.me/ton_learn

You can see the ranking of collections from this tutorial in a convenient form here:

https://tonlearn.tools/#/pavel-durov-telegram-usernames

Code from tutorial: https://colab.research.google.com/drive/1fIVOM3fhYeYTr5kuRaT0kyleSMedQ0Gl?usp=sharing


