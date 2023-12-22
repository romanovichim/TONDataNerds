How to find NFT collection owners in TON

## Introduction 

Let's continue our acquaintance with dton.io and data collection on TON, today we will take on the task of searching for owners of NFTs in the collection:
1) Let's gather the owners
2) Let’s count how many NFTs they have in the selected collection
3) Let’s build a diagram to visually understand how many shares are owned by the top 10 owners of the collection

![Group 192](https://gist.github.com/assets/18370291/a933bffb-66a0-4e0b-8994-ea33bb0e33c4)

## Checking the contract address

I am collecting this tutorial in Google Colab for ease of use. In the first step, you are asked to enter the NFT master contract to use the code:

```python
nft_col_addr = 'UQApA79Qt8VEOeTfHu9yKRPdJ_dADvspqh5BqV87PgWD998f' # @param {type:"string"}
```

Before moving on, I suggest checking that the address we entered is the address of the master contract of the NFT collection, and also immediately get the number of NFTs in the collection.

This is easy to do through the toncenter there is a convenient method there - /getTokenData

```python
#Note api has limits
def next_item_index_count(nft_col_addr):
  API_ENDPOINT = 'https://toncenter.com/api/v2'
  payload = {'address': nft_col_addr}
  r = requests.get(API_ENDPOINT+'/getTokenData', params=payload)
  #print api response
  print(json.dumps(r.json(), indent=2))

  # last index in collection
  last = r.json()["result"]["next_item_index"]

  return last


next_item_index = next_item_index_count(nft_col_addr)
```

### Sequential collections

Collections on TON, in accordance with the standard, are Sequential and Non-Sequential, that is, whether NFTs are included in the collections in a row or not. Accordingly, for Non-Sequential, you need to parse the entire blockchain and look at which NFT minitil master contract.

In this tutorial we will focus only on Sequential. We will use the parameter from get_nft_data() the get method of the master contract to understand how many total NFTs are in the collection. 

By the way a non-sequential collection would have -1 in next_item_index

## Constructing a request

Let's start with a simple request, so we have a request framework in dton.io.

```python
endpoint = 'https://dton.io/graphql/'
query = """
query MyQuery {
  account_states(

  ) {

  }
}
"""
response = requests.post(endpoint, json={'query': query})

if response.status_code == 200:
  data = response.json()['data']
  print(data)
else:
  print(response.json())
```

Как вы можете увидеть мы будем используем представление состояний аккаунтов в сети. В ответе нам нужен адрес нфт и адрес владельца, address и parsed_nft_owner_address_address соответственно.

В самом же запросе мы проверим, что выбранный нами смарт-контракт это nft, через 
account_state_state_init_code_has_get_nft_data и также передадим адрес мастер контракта - адрес коллекции, которая нас интересует в   parsed_nft_collection_address_address .

I note that the address is needed in hex format. To do this, you need to convert the address into hex format; the easiest option is via https://ton.org/address/ . In later tutorials we will also learn how to do this with a script.

Final request:

```python
endpoint = 'https://dton.io/graphql/'
query = """
query MyQuery {
  account_states(
    account_state_state_init_code_has_get_nft_data: 1
    workchain: 0
    page:0
    page_size: 150
    parsed_nft_collection_address_address: "2903BF50B7C54439E4DF1EEF722913DD27F7400EFB29AA1E41A95F3B3E0583F7"
  ) {
    address
    parsed_nft_owner_address_address
  }
}
"""

response = requests.post(endpoint, json={'query': query})

if response.status_code == 200:
  data = response.json()['data']
  print(data)
else:
  print(response.json())
```

## Collection of all owners

Since the page is limited to 150 records, we cannot get all the owners with one query (only if the collection is very small) we need to concatenate the queries. It is important to note that if the collection is very large, then it will be necessary to take into account that it is impossible to concatenate more than 100 queries. - another request will be needed:

```python
page = 0
maxPageSize = 150
addr = '"2903BF50B7C54439E4DF1EEF722913DD27F7400EFB29AA1E41A95F3B3E0583F7"'


last = next_item_index 


formatted_query =""

while(True):
  limit = last if last < maxPageSize else maxPageSize
  #print(limit)
  template = string.Template("""
    q${page}: account_states(
      parsed_nft_collection_address_address: ${addr}
      account_state_state_init_code_has_get_nft_data: 1
      workchain: 0
      page_size: ${page_size}
      page: ${page}
      ) {
      address
      parsed_nft_owner_address_address
      }
  """)

  formatted = template.substitute(page=page,page_size=limit,addr=addr)
  formatted = formatted +"\n"
  formatted_query += formatted

  last = last - maxPageSize
  if last<=0:
    break

  page = page + 1


formatted_query = """query {"""+formatted_query+"""}"""

response = requests.post(endpoint, json={'query': formatted_query})
```

## Drawing a diagram

Сгруппируем и достанем топ 10 владельцев коллекции, в дальнейшем мы будем использовать их в диаграмме. Также для диаграммы посчитаем нфт у оставшихся владельцев:

```python
res_arr = []
for v in response.json()['data'].values():
  for item in v:
    res_arr.append(item)

topten = Counter(tok['parsed_nft_owner_address_address'] for tok in res_arr).most_common()[0:10]
topten


# since there can be more than hundred owners, we need sum for all owners we take to create field others

top_sum = sum([d[1] for d in topten])

# let's count others
other_count = next_item_index - top_sum
```
All that remains is to assemble the diagram:

```python
import matplotlib.pyplot as plt

labels = [d[0] for d in topten] +['Others']

sizes = [(d[1]/next_item_index)*100 for d in topten] + [other_count]
fig1, ax1 = plt.subplots()
ax1.pie(sizes, labels=labels, autopct='%1.1f%%')
ax1.set_title( 'by @ton_learn')

plt.rcParams['figure.figsize'] = (15,15)
plt.show()
```

Result:

![image](https://gist.github.com/assets/18370291/d8dc6795-b3e5-474a-8059-f140dc2afddb)

## Conclusion

I have open source lessons that will teach you how to create full-fledged applications on TON.I will be grateful to your stars.

https://github.com/romanovichim/TonFunClessons_Eng

I will be glad to see your stars on the repository. Also I post new tutorials and data analytics here: https://t.me/ton_learn

Code from this tutorial:
https://colab.research.google.com/drive/1nX626v9j9wsFpdUW_KBt215tFVekM_A1?usp=sharing
