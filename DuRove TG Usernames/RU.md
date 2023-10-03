# Ищем покупки Telegram Usernames сделанные Павлом Дуровым с помощью Python и индексатора dton.io

##  Зачем?

В последнее время к покупкам Павла Дурова уделяется много внимания:

Перед появлением сториз групп в телеграмме с механикой boost’ов Павел покупает Telegram Username boost
25 сентября Павел покупает fabrika и через неделю привязывает его к боту проекта фабрика друзей

Ну просто кладезь инсайдов

### Откуда известно что кошелек Павла

Кошелек вычислили по юзернейму nerd, когда его нашли на возможном аккаунте Павла Дурова onetimeusername. 

Также информацию про это можно найти в списке известных кошельков Tonkeeper

https://raw.githubusercontent.com/tonkeeper/ton-assets/main/accounts.json

### Проблема с просмотром кошелька через эксплореры

Если вы захотите посмотреть на кошелек Павла через эксплорер, то найти интересную инфу будет сложно, каждый день туда падает куча спам транзакций и для пиара своих проектов туда переводят кучу токенов и нфт.

Поэтому было бы интересно посмотреть информацию о TG usernames отдельно.

## Что такое dton.io?

Dton.io индексатор блокчейна, что значит, он собирает информацию из каждого нового блока в свою базу данных. В эту базу данных можно делать GraphQL запросы и таким образом собирать историческую информацию без парсинга всей цепочки блоков.

Две самых важных источника данных dton.io это таблица транзакций и представление последнего состояния кошельков/аккаунтов в сети. Делай запросы в данной представление и таблицу, можно собрать почти любую информацию. 

В данном туториале нам пригодятся именно состояния кошельков, будем брать баланс

## Устанавливаем зависимости

Как мы знаем GraphQL запросы отправляются в теле HTTP POST-запроса, поэтому установим requests и прочие необходимые зависимости:


```python
# install requriments
!pip install requests

import requests
import string 
```

Обозначим и эндпойнт dton.io:

```python
endpoint = 'https://dton.io/graphql/'
```

# Начинаем собирать запрос

Собирать мы будем траназакции, чтобы понять какие из траназакций продажи
воспользуемся полем `parsed_seller_is_closed`:

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

### Отсекаем покупки Павла и Telegram Usernames

Так как нас интересуют только юзернеймы, то добавим фильтр по коллекции, а так же добавим фильтр по аккаунту Павла(если покупает он, то собственно новым владельцем он и становится):

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

### Выберем нужную нам информацию

Теперь у нас есть все нужные траназакции, осталось отобрать информацию, которую  мы хотим отобразить, это:
- Дата и время блока в котором произошла покупка
- Цена
- И конечно же что за юзернейм

Здесь отмечу, что у коллекции Telegram Usernames очень удобная ссылка на в ячейке с данными, в самом урле уже содержиться название юзернейма и делать отдельный реквест для получения имени не надо.

Получаем:

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

### Последний штрих

Чтобы данные удобно было смотреть отсортируем из по времени блока. Для сортировки нужно использовать поле `order_by`

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


## Что можно улучшить

В следующих туториалах посмотрим на других китов ТОН, а также добавим инфу не только по покупкам нфт)

## Заключение

Мне нравится блокчейн TON своей технической изящностью, как минимум это не очередная копия Ethereum, которую разгоняют с помощью большого капитала без оглядки, а вообще зачем это нужно пользователю. Если вы хотите узнать больше о блокчейне TON, у меня есть опенсорсные уроки, благодаря которым вы научитесь создавать полноценные приложения на TON.

https://github.com/romanovichim/TonFunClessons_ru

Новые туториалы и дата аналитику я кидаю сюда: https://t.me/ton_learn

Посмотреть, ранжирование коллекций из данного туториала в удобном виде можно здесь:

https://tonlearn.tools/#/pavel-durov-telegram-usernames

Код: https://colab.research.google.com/drive/1fIVOM3fhYeYTr5kuRaT0kyleSMedQ0Gl?usp=sharing


