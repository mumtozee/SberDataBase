## Домашнее задание 3. Отчет по Redis

<font color="green"><h4>Ахмаджонов Мумтозбек</h4></font>

Для максимального удобства с работы с редисом, я использую официальный клиент редиса для `Python`. Его можно установить командой:


```python
! pip install redis
```

Для запуска редис сервера использую официально поддерживаемый проект `Memurai`. Т.к. моя операционная система Windows 10, а для нее редис официально не поддерживается, пришлось воспользоваться такой оберткой. Подробная информация на сайте проекта.

Есть датасет DailyDialogs - примеры различных диалогов в виде история-ответ. Попробуем разными образами над ним поизвращаться и впихивать в базу данных.


```python
import numpy as np
import pandas as pd

import json
```


```python
dialog = pd.read_csv('./daily_dialog.tsv', sep='\t')
dialog.columns = ['history', 'response', 'DA', 'SENT']
dialog.drop(columns=['DA', 'SENT'], inplace=True)
dialog.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>history</th>
      <th>response</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Say , Jim , how about going for a few beers af...</td>
      <td>What do you mean ? It will help us to relax .</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Say , Jim , how about going for a few beers af...</td>
      <td>Do you really think so ? I don't . It will jus...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Say , Jim , how about going for a few beers af...</td>
      <td>I guess you are right.But what shall we do ? I...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Say , Jim , how about going for a few beers af...</td>
      <td>I suggest a walk over to the gym where we can ...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Say , Jim , how about going for a few beers af...</td>
      <td>That's a good idea . I hear Mary and Sally oft...</td>
    </tr>
  </tbody>
</table>
</div>



Сохраним эту таблицу в виде `json` файла:


```python
dialog.to_json('./dialogs.json', orient='records')
```

Теперь подготовим наши большие данные (для сета не нашел более креативного способа как в виде `json` файла хранить данные, поэтому загенерил в помощью питона, и позже будет видно что он достаточно большой получился):


```python
with open('./dialogs.json', 'r') as data_file:
    test_data = json.load(data_file)
    big_string = str(test_data)
    big_list = big_string.split()
    big_list = big_list[:int((len(big_list) * 0.3))]
tmp_nums = np.linspace(-1000, 6000, 700000)
big_set = set(str(x) for x in tmp_nums)
big_zset = {str(x) : x for x in tmp_nums[:350000]}
```


```python
float(next(iter(big_set)))
```




    552.5322179031684




```python
import sys

def printobjsize(obj):
    print(f"The object weighs: {sys.getsizeof(obj) / (2 ** 20) :.2f} mb")
```


```python
printobjsize(big_string)
printobjsize(big_list)
printobjsize(big_set)
printobjsize(big_zset)
```

    The object weighs: 30.27 mb
    The object weighs: 15.41 mb
    The object weighs: 32.00 mb
    The object weighs: 20.00 mb
    

Теперь подключимся собственно к БД который по умолчанию `redis-server` запустил по адресу `https://localhost:6379`


```python
import redis
from time import time
```


```python
r = redis.Redis(host='localhost', port=6379, db=0)
```

Можем проверить, пустая ли БД:


```python
# 127.0.0.1:6379> DBSIZE
r.dbsize()
```




    1



Теперь докинем туда данные:


```python
# 127.0.0.1:6379> SET test_str ...
r.set('test_str', big_string)
```




    True




```python
# 127.0.0.1:6379> RPUSH test_list ...
for i in range(0, len(big_list), 1000):
    r.rpush('test_list', *big_list[i:i+1000])
```


```python
# 127.0.0.1:6379> SADD test_set ...
r.sadd('test_set', *big_set)
```




    700000




```python
# 127.0.0.1:6379> ZADD test_zset ...
r.zadd('test_zset', big_zset)
```




    350000




```python
r.dbsize()
```




    4



После того как добавили данные, можно померить скорости сохранения и чтения данных:

Достанем первые 2000 элементов из большого списка:


```python
bt = time()
res = r.lrange('test_list', 0, 2000)
et = time()
print(f"Operation took: {(et - bt) * 1000: .2f}ms")
```

    Operation took:  11.97ms
    

Теперь докинем туда столько же элементов:


```python
bt = time()
res = r.lpush('test_list', *big_list[:2000])
et = time()
print(f"Operation took: {(et - bt) * 1000: .2f}ms")
```

    Operation took:  17.98ms
    

Добавим слово "Hello" под конец нашей большой строки:


```python
bt = time()
res = r.append('test_string', "Hello")
et = time()
print(f"Operation took: {(et - bt) * 1000: .2f}ms")
```

    Operation took:  1.00ms
    

Достанем длину строки и префикс длины 30000:


```python
bt = time()
res = r.strlen('test_string')
et = time()
print(f"Operation took: {(et - bt) * 1000: .2f}ms")
```

    Operation took:  1.00ms
    


```python
bt = time()
res = r.substr('test_string', 0, 30000)
et = time()
print(f"Operation took: {(et - bt) * 1000: .2f}ms")
```

    Operation took:  0.99ms
    

Достанем ранг элемента 6000 из осторитрованного множества:


```python
bt = time()
res = r.zscore('test_zset', '6000')
et = time()
print(f"Operation took: {(et - bt) * 1000: .3f}ms")
```

    Operation took:  0.000ms
    

Достанем первые 30000 элементов из отсортирванного сета по рангам:


```python
bt = time()
res = r.zrange('test_zset', 0, 30000)
et = time()
print(f"Operation took: {(et - bt) * 1000: .2f}ms")
```

    Operation took:  144.61ms
    

Докинем ему еще 3000 элементов:


```python
tmp_nums = np.linspace(10000, 20000, 3000).astype(float)
new_zset = {str(x) : x for x in tmp_nums}
bt = time()
res = r.zadd('test_zset', new_zset)
et = time()
print(f"Operation took: {(et - bt) * 1000: .3f}ms")
```

    Operation took:  24.935ms
    

Проверим лежит ли 700000 в множестве:


```python
bt = time()
res = r.sismember('test_set', '700000')
et = time()
print(f"Operation took: {(et - bt) * 1000: .3f}ms")
```

    Operation took:  3.001ms
    

Теперь добавим этот элемент:


```python
bt = time()
res = r.sadd('test_set', '700000')
et = time()
print(f"Operation took: {(et - bt) * 1000: .3f}ms")
```

    Operation took:  0.998ms
    

Как можно увидеть, почти все операции включающие в себя добавление одного элемента происходят максимально быстро. И в нашем датасете, zset и set примерно с одинаково скоростью добавляют элемент. Можно теперь очистить БД:


```python
# # 127.0.0.1:6379> DEL
r.delete('test_set', 'test_string', 'test_list', 'test_zset')
```




    4




```python

```
