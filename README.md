# Date Conversion Transformer: Tensorflow 2.x vs Pytorch vs Keras vs Tensorflow 1.x

## Requirements
- torchtext
- hparams
- pandas
- pytorch
- tensorflow 2.2


## 구현 목록
	- Pytorch: torch.nn.Transformer API를 사용.
	- Tensorflow 2.x
	- Keras
	- Tensorflow 1.x

## Data 만들기

## Padding
- 2가지 방식의 padding을 살펴보자.
![padding](./padding.png)
- 가변 길이 방식: mini batch 내에서 가장 긴 sequence를 기준으로 padding이 된다.
- 고정 길이 방식: 고정된 길이가 될 수 있도록 padding을 한다.
- 2가지 방식의 padding이 가능하도록 tensorflow, torchtext에서 api를 제공하고 있다.

## Data Preprocessing: Tensorflow Tokenizer vs torchtext
# Tensorflow-Tokenizer
```
# coding: utf-8

import tensorflow as tf
from konlpy.tag import Okt
okt = Okt()

from tensorflow.keras import preprocessing
samples = ['너 오늘 아주 이뻐 보인다', 
           '나는 오늘 기분이 더러워', 
           '끝내주는데, 좋은 일이 있나봐', 
           '나 좋은 일이 생겼어', 
           '아 오늘 진짜 너무 많이 정말로 짜증나', 
           '환상적인데, 정말 좋은거 같아']

label = [[1], [0], [1], [1], [0], [1]]

morph_sample = [okt.morphs(x) for x in samples]

tokenizer = preprocessing.text.Tokenizer(oov_token="<UKN>")   # oov: out of vocabulary
tokenizer.fit_on_texts(samples+['SOS','EOS']) 
print(tokenizer.word_index)  # 0에는 아무것도 할당되어 있지 않다.  --> pad를 할당하면 된다.

word_to_index = tokenizer.word_index
word_to_index['PAD'] = 0
index_to_word = dict(map(reversed, word_to_index.items()))

print('word_to_index(pad): ', word_to_index)
print('index_to_word', index_to_word)

sequences = tokenizer.texts_to_sequences(morph_sample)  # 역변환: tokenizer.sequences_to_texts(sequences)
print(sequences)
[[5, 2, 6, 7, 8],
[14, 1, 2, 1, 1, 11],
[12, 1, 3, 4, 1, 1],
[14, 3, 4, 15],
[16, 2, 17, 18, 19, 20, 21],
[1, 1, 1, 1, 23, 3, 1, 25]]

```
- 위에서 얻은 `sequences`에 padding을 붙혀보자.
```
padded_sequence = preprocessing.sequence.pad_sequences(sequences, maxlen=15, padding='post',truncating='post')

[[ 5  2  6  7  8  0  0  0  0  0  0  0  0  0  0]
 [14  1  2  1  1 11  0  0  0  0  0  0  0  0  0]
 [12  1  3  4  1  1  0  0  0  0  0  0  0  0  0]
 [14  3  4 15  0  0  0  0  0  0  0  0  0  0  0]
 [16  2 17 18 19 20 21  0  0  0  0  0  0  0  0]
 [ 1  1  1  1 23  3  1 25  0  0  0  0  0  0  0]]


```
- 위의 결과는 전체 data를 고정 길이로 padding했다. mini-batch별로 가변 길이 padding은 아래에서 tf.data.Dataset을 통해서 만들 수 있다. 구체적인 방식은 아래에서 살펴보자.


# torchtext
```
import torchtext
from konlpy.tag import Okt
samples = ['너 오늘 아주 이뻐 보인다', 
           '나는 오늘 기분이 더러워', 
           '끝내주는데, 좋은 일이 있나봐', 
           '나 좋은 일이 생겼어', 
           '아 오늘 진짜 너무 많이 정말로 짜증나', 
           '환상적인데, 정말 좋은거 같아']

# 1. Field 정의
tokenizer = Okt()
TEXT = torchtext.data.Field(sequential=True, tokenize=tokenizer.morphs,batch_first=True,include_lengths=False)
fields = [('text', TEXT)]

# 2. torchtext.data.Example 생성
sequences=[]
for s in samples:
    sequences.append(torchtext.data.Example.fromlist([s], fields))

for s in sequences:
    print(s.text)

# 3. Dataset생성(word data)
mydataset = torchtext.data.Dataset(sequences,fields)# Example ==> Dataset생성

# 4. vocab 생성
TEXT.build_vocab(mydataset, min_freq=1, max_size=10000)
print(TEXT.vocab.stoi)

# Dataset생성(id로 변환된 data)
mydataset = torchtext.data.Iterator(dataset=mydataset, batch_size = 3)

for d in mydataset:
    print(d.text.numpy())

[[11  2 20 22 16  1  1]
 [ 5  3  6 18  1  1  1]
 [19  2 28 12 15 27 29]]
[[10  4  3  6 24 17  1  1]
 [ 5 13  2  9 21 14  1  1]
 [30 25 23  4 26  3  8  7]]

```
- 위 코드의 결과는 mini-batch중에서 가장 긴 data를 기준으로 padding(padding token =1)이 된 것을 알 수 있다. Field에서 `fix_length`를 지정하면 고정된 길이의 data를 얻을 수 있다.
```
TEXT = torchtext.data.Field(sequential=True, tokenize=tokenizer.morphs,batch_first=True,include_lengths=False,fix_length=15)

[[ 4 19  6 15  3 21  5  1  1  1  1  1  1  1  1]
 [34  3 12  9 30  8  7 14 13  2  1  1  1  1  1]
 [17  6 26 28  5 24 11  1  1  1  1  1  1  1  1]]
[[16 20  9  8  7 10  3 29  4 23  2  1  1  1  1]
 [ 4  2  8  7 10  3 25 27  5  1  1  1  1  1  1]
 [ 2  2  6 32 18 22 31 33  1  1  1  1  1  1  1]]
```

# tf.data.Dataset
- tf.data.Dataset을 이용해서, mini-batch를 효율적으로 만들 수 있다.
- tensorflow에서 가변길이 방식의 padding을 만드는 방법을 살펴보자. 위의 tensorflow 코드를 이어서 살펴보자.
```
sequences = tokenizer.texts_to_sequences(morph_sample)  # 역변환: tokenizer.sequences_to_texts(sequences)
print(sequences)
[[5, 2, 6, 7, 8],
[14, 1, 2, 1, 1, 11],
[12, 1, 3, 4, 1, 1],
[14, 3, 4, 15],
[16, 2, 17, 18, 19, 20, 21],
[1, 1, 1, 1, 23, 3, 1, 25]]
```
- `tokenizer.texts_to_sequences`로 길이가 다른 list를 얻었다. 길이가 다르기 때문에, 다음과 같이 하면 error가 발생한다.
```
 tf.data.Dataset.from_tensor_slices(sequences)
```







## References
- <https://www.tensorflow.org/tutorials/text/transformer>
- <https://pytorch.org/tutorials/beginner/transformer_tutorial.html>