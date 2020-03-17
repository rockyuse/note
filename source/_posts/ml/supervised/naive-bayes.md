---
title: 'Naive Bayes'
tags: 'ML'
---

# Naive Bayes

Data set: [Iris](https://archive.ics.uci.edu/ml/datasets/Iris)

## Data Preprocess

```py
import pandas as pd
import numpy as np

FEATURE = ['sepal-length', 'sepal-width', 'petal-length', 'petal-width', 'class']

class IrisData():
    df = None
    _data = []

    def __init__(self, iris_fp):
        with open(iris_fp, 'r', encoding='UTF-8') as f:
            for line in f:
                line = line.rstrip('\n')
                if line:
                    self._data.append(line.split(','))
        self.df = pd.DataFrame(data=np.array(self._data), columns=FEATURE)

    def drop_missing_feature(self):
        feature_df = self.df.drop('class', axis=1)
        for feature in feature_df:
            if ('?' in self.df[feature].values):
                self.df = self.df.drop(feature, axis=1)

    def shuffle_data(self):
        self.df = self.df.sample(frac=1).reset_index(drop=True)

# preprocess data
iris_data = IrisData('data/bezdekIris.data')
iris_data.drop_missing_feature()
iris_data.shuffle_data()
```

![](2020-03-17-22-09-02.png)

## Model

```py
from math import log, exp, sqrt, pi

class IrisModel:
    df = None
    label = []
    prob = {}  # log(P(Y))
    mu = {}
    variance = {}

    def __init__(self, df, label):
        self.df = df
        self.label = label

    def train(self, df):
        ...

    def test(self, df):
        ...

    def holdoutValidation(self):
        ...

    def kFoldCrossValidation(self, K):
        ...

    def __predict(self, q):
        ...

    def __calcLabelProb(self, Y, df):
        ...

    def __calcMu(self, Y):
        ...

    def __calcVariance(self, Y):
        ...

label = ['Iris-setosa', 'Iris-versicolor', 'Iris-virginica']
iris_model = IrisModel(iris_data.df, label)
```

Constructor:

* 把 dataframe 與 label 存起來

Member variables:

* prob:
  * 一個 dictionary 用來存 log(P(Y))
  * prob[label] = log(P(Y))
* prob_cond:
  * 一個 dictionary 用來存 log(P(Fx|Y))
  * prob_cond[label][feature][event] = log(P(Fx|Y))

### Train

```py
class IrisModel:
    def train(self, df):
        for l in self.label:
            Y = df.loc[df['class'] == l].reset_index(drop=True)
            self.prob[l] = self.__calcLabelProb(Y, df)
            self.mu[l] = self.__calcMu(Y)
            self.variance[l] = self.__calcVariance(Y)

    def __calcLabelProb(self, Y, df):
        prob = len(Y.index) / len(df.index)
        return -1e9 if prob == 0 else log(prob, 10)

    def __calcMu(self, Y):
        mu = {}
        for f in FEATURE:
            if f == 'class':
                continue
            mu[f] = Y[f].astype(float).mean()
        return mu

    def __calcVariance(self, Y):
        variance = {}
        for f in FEATURE:
            if f == 'class':
                continue
            variance[f] = Y[f].astype(float).var()
        return variance
```

train():

1. 變歷每個 label
2. 把 dataframe 包含那個 label 的資料篩選出來叫做 Y
3. 算出 prob, mu 以及 variance

__calcLabelProb():

1. 算出 label 的機率為：篩選出來的數量 / train data的總數量
2. 若算出來為 0 則 return -1e9 (因為 log(0))

__calcMu():

1. 變歷 FEATURE 這個 list: ['sepal-length', 'sepal-width', 'petal-length', 'petal-width', 'class']
2. 若 feature 不叫 'class' 就算出他的平均數

__calcVariance():

1. 變歷 FEATURE 這個 list: ['sepal-length', 'sepal-width', 'petal-length', 'petal-width', 'class']
2. 若 feature 不叫 'class' 就算出他的變異數

### Train-Test-Split

```python
class IrisModel:
    def test(self, df):
        cm = np.zeros(shape=(len(self.label), len(self.label)))
        for index, row in df.iterrows():
            ans = self.label.index(row['class'])
            predict = self.label.index(self.__predict(row.drop('class')))
            cm[ans][predict] += 1
        return pd.DataFrame(data=cm, columns=self.label, index=self.label)

    def __predict(self, q):
        # calculate each label's probability
        predict = {}
        for l in self.label:
            predict[l] = self.prob[l]
            for feature, x in q.astype(float).items():
                mu_x_y = self.mu[l][feature]
                var_x_y = self.variance[l][feature]
                numerator = exp(-((x - mu_x_y) ** 2) / (2 * var_x_y))
                dominator = sqrt(2 * pi * var_x_y)
                if numerator == 0:
                    predict[l] += -1e9
                else:
                    predict[l] += log(numerator / dominator)
        # return the maximum probability's label
        return max(predict, key=lambda l: predict[l])
```

test():

1. 先建立一個都是 0 的 confusion matrix
2. 對 dataframe 變歷 rows
3. 先存出每個 rows 的答案 (label)
4. 對剩下的 events 丟進 __predict()，會 return 猜出來的結果
5. 對 confusion matrix 存成 dataframe 並 return

__predict():

1. 變歷所有 labels
2. 這個 testcase 中的這個 label 的機率以 log(P(Y)) 作為初始值
3. 變歷這個 testcase 的 feature 與 x (值)
4. 帶入 Gaussian distribution 的公式算出條件機率並不斷累加
5. 若機率為 0 則加上 -1e9 (因為 log(0))
6. return 這個 dictionary 中最大的 label

## Validation

### Holdout validation

```python
class IrisModel:
    def holdoutValidation(self):
        # split train / test data to 7 : 3
        dataSize = len(self.df)
        train_df = self.df.head(dataSize * 7 // 10)
        test_df = self.df.tail(dataSize - dataSize * 7 // 10)
        # train model
        self.train(train_df)
        # test model
        df = self.test(test_df)
        print(df)
```

1. 把 data 分成 7:3
2. 用 7 來 train model
3. 用 3 來 test model
4. print 出 confusion matrix

```py
print("1) Holdout validation with train/test:7/3")
iris_model.holdoutValidation()
```

Result:

```
1) Holdout validation with train/test:7/3
                 Iris-setosa  Iris-versicolor  Iris-virginica
Iris-setosa             10.0              0.0             0.0
Iris-versicolor          0.0             18.0             1.0
Iris-virginica           0.0              1.0            15.0
```

### K-fold cross validation

```python
class IrisModel:
    def kFoldCrossValidation(self, K):
        partSize = len(self.df) // K
        for i in range(K):
            # split train / test data to K part
            test_df = self.df[i * partSize: (i + 1) * partSize]
            train_df = self.df.drop(test_df.index)
            test_df.reset_index(drop=True)
            train_df.reset_index(drop=True)
            # train model
            self.train(train_df)
            # test model
            df = self.test(test_df)
            print('-- fold', i)
            print(df)
```

1. 計算 fold 中的 part 的大小
2. 做 K 次 train-test
3. print 出 confusion matrix

```py
print("\n2) K-fold cross-validation with K=3")
iris_model.kFoldCrossValidation(3)
```

Result:

```
2) K-fold cross-validation with K=3
-- fold 0
                 Iris-setosa  Iris-versicolor  Iris-virginica
Iris-setosa             23.0              0.0             0.0
Iris-versicolor          0.0             13.0             2.0
Iris-virginica           0.0              1.0            11.0
-- fold 1
                 Iris-setosa  Iris-versicolor  Iris-virginica
Iris-setosa             15.0              0.0             0.0
Iris-versicolor          0.0             15.0             0.0
Iris-virginica           0.0              2.0            18.0
-- fold 2
                 Iris-setosa  Iris-versicolor  Iris-virginica
Iris-setosa             12.0              0.0             0.0
Iris-versicolor          0.0             18.0             2.0
Iris-virginica           0.0              1.0            17.0
```
