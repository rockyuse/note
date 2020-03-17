---
title: 'Decision Tree'
tags: 'ML'
---

# Decision Tree

[sample_submission.csv](/note/ml/supervised/decision-tree/sample_submission.csv)
[X_test.csv](/note/ml/supervised/decision-tree/X_test.csv)
[X_train.csv](/note/ml/supervised/decision-tree/X_train.csv)
[y_train.csv](/note/ml/supervised/decision-tree/y_train.csv)

## 目錄架構

```
.
├── census
│   ├── data_preprocess.py
│   ├── __init__.py
├── census_dt.py
├── data
│   ├── sample_submission.csv
│   ├── X_test.csv
│   ├── X_train.csv
│   └── y_train.csv
└── model
    ├── decision_tree.py
    └── __init__.py
```

## 主程式

census_dt.py

```py
import os
import numpy as np
import pandas as pd
from census.data_preprocess import CensusData
from model.decision_tree import DecisionTree

trees = []
accuracies = []

def validate(tree, df):
    ...

def holdoutValidation(df):
    ...

def kFoldCrossValidation(df, K):
    ...

def predictLabel(tree, df):
    ...

def main():
    project_dir = os.path.dirname(os.path.realpath(__file__))
    train_fp = project_dir + '/data/X_train.csv'
    label_fp = project_dir + '/data/y_train.csv'
    test_fp = project_dir + '/data/X_test.csv'
    # preprocess data
    census_data = CensusData(train_fp, label_fp, test_fp)
    census_data.handle_missing_data()
    census_data.shuffle_data()
    # validation
    print("1) Holdout validation with train/test:7/3")
    holdoutValidation(census_data.train_df)
    print("\n2) K-fold cross-validation with K=3")
    kFoldCrossValidation(census_data.train_df, 3)
    # choose best tree to predict test set
    global trees, accuracies
    best_tree_idx = accuracies.index(max(accuracies))
    best_tree = trees[best_tree_idx]
    predictLabel(best_tree, census_data.test_df)


if __name__ == '__main__':
    main()
```

1. 將 data 的路徑定義好
2. 丟進 CensusData 做預處理
3. 處理 Data 中缺失的欄位
4. 打亂 Train Data
5. 透過 holdout validation 以及 k-fold cross validation 檢驗正確性
6. 選出正確率最高的 Tree 對 test set 猜測答案

holdoutValidation:

```py
def holdoutValidation(df):
    # split train / validation data to 7 : 3
    dataSize = len(df)
    train_df = df.head(dataSize * 7 // 10)
    validation_df = df.tail(dataSize - dataSize * 7 // 10)
    # train model
    tree = DecisionTree(train_df)
    # validation
    cm = validate(tree, validation_df)
    accuracy = (cm[0][0] + cm[1][1]) / cm.values.sum()
    print(cm)
    print("accuracy:", accuracy)
    # store result
    global trees, accuracies
    trees.append(tree)
    accuracies.append(accuracy)
```

1. 切割 Data 為 train:validation = 7:3
2. Train Model
3. 透過 Validation Data 驗證正確性

kFoldCrossValidation:

```py
def kFoldCrossValidation(df, K):
    partSize = len(df) // K
    for i in range(K):
        # split train / validation data to K part
        validation_df = df[i * partSize: (i + 1) * partSize]
        train_df = df.drop(validation_df.index)
        validation_df.reset_index(drop=True)
        train_df.reset_index(drop=True)
        # train model
        tree = DecisionTree(train_df)
        # test model
        cm = validate(tree, validation_df)
        accuracy = (cm[0][0] + cm[1][1]) / cm.values.sum()
        print('-- fold', i)
        print(cm)
        print("accuracy:", accuracy)
        # store result
        global trees, accuracies
        trees.append(tree)
        accuracies.append(accuracy)
```

1. 做 K 次分割不同區段的 train 及 validation data
2. Train Model
3. 透過不同區段的 Validation Data 驗證正確性

validate:

```py
def validate(tree, df):
    cm = np.zeros(shape=(2, 2))
    for index, row in df.iterrows():
        ans = row.Category
        predict = tree.predict(row)
        cm[ans][predict] += 1
    return pd.DataFrame(data=cm)
```

1. 紀錄並回傳 Confusion matrix

## Data Preprocessing

census / data_preprocess.py

```py
import os
import numpy as np
import pandas as pd


class CensusData():
    train_df = None
    test_df = None

    def __init__(self, train_fp, label_fp, test_fp):
        train_df = pd.read_csv(train_fp)
        train_df.set_index('Id', inplace=True)
        # add label to end of train data
        label_df = pd.read_csv(label_fp)
        label_df.set_index('Id', inplace=True)
        df = pd.merge(train_df, label_df, on='Id')
        # strip all string data
        df_obj = df.select_dtypes(['object'])
        df[df_obj.columns] = df_obj.apply(lambda x: x.str.strip())
        self.train_df = df

        test_df = pd.read_csv(test_fp)
        test_df.set_index('Id', inplace=True)
        # strip all string data
        df_obj = test_df.select_dtypes(['object'])
        test_df[df_obj.columns] = df_obj.apply(lambda x: x.str.strip())
        self.test_df = test_df

    # fill missing data to most frequently value (because features of missing data are all categories)
    def handle_missing_data(self):
        self.train_df = self.train_df.replace({'?': np.nan})
        self.train_df = self.train_df.fillna(self.train_df.mode().iloc[0])
        self.test_df = self.test_df.replace({'?': np.nan})
        self.test_df = self.test_df.fillna(self.test_df.mode().iloc[0])

    def shuffle_data(self):
        self.train_df = self.train_df.sample(frac=1).reset_index(drop=True)
```

建構式：

1. 把 Label 放在 Train Dataframe 的最後一個 Column
2. 同一 string 的 data，必免空格導致誤判
3. 儲存 Test Data

handle_missing_data:

1. 把 train data 與 test data 中的缺失項找出 (先換成 nan，再全部 replace)
2. 把缺失項換成出現頻率最高的值 (因為缺失項都為非連續的欄位，所以不考慮連續情況)

## 資料結構設計

Decision Tree 的框架設計為：

```
tree = {
  "feature": {
      "question": {
          "feature": {
              ...
          }
      }
      "question": {
          ...
      }
  }
}
```

舉例來說：

```
tree = {
  "age": {
      ">10": {
          "relationship": {
              "== Husband": {

              },
              ...
          }
      }
      "<=10": 0
  }
}
```

同時這個 Tree 支援 multiple branch，在可分類(非連續)的欄位會建立所有可支援的 branch，在連續的欄位會有 >, <= 區分兩種範圍

## Model

model / decision_tree.py

```py
import numpy as np
import random

class DecisionTree():
    COLUMNS = None
    tree = {}

    def __init__(self, df, max_depth=5, random_feature=None):
        data = df.values  # numpy n-d array
        self.COLUMNS = df.columns
        self.tree = self.buildTree(
            data=data, max_depth=max_depth, random_feature=random_feature)

    def guess(self, tree, entry):
        ...

    def predict(self, entry):
        ...

    def buildTree(self, data, depth=0, max_depth=5, random_feature=None):
        ...
    
    # find feature of highest Information Gain (lowest Remainder)
    def detBestFeature(self, data, levels):
        ...

    def calcCategoryRem(self, data, feature_idx, level):
        ...

    def calcContinousRem(self, data, feature_idx, split_value):
        ...

    def calcEntropy(self, data):
        ...

    # utility functions
    def splitContinuous(self, data, col_idx, value):
        ...

    def getUniqueLevels(self, data, random_feature=None):
        ...

    def classifyData(self, data):
        ...

    def checkPurity(self, data):
        ...
```

建構式：

1. 把 pandas dataframe 轉為 numpy nd-array
2. 取出 COLUMNS 的名字
3. 遞迴建立 Tree

分為 4 個部份：

* 建立樹
* 選擇最佳分支
* helper function
* 預測答案

### 建立樹

```py
def buildTree(self, data, depth=0, max_depth=5, random_feature=None):
    if self.checkPurity(data) or len(data) < 2 or (depth == max_depth):
        return self.classifyData(data)

    else:
        depth += 1
        # levels = { feature_idx: unique_level }
        levels = self.getUniqueLevels(data, random_feature)
        best_feature = self.detBestFeature(data, levels)
        best_feature_data = data[:, best_feature['idx']]

        # build tree
        best_feature_name = self.COLUMNS[best_feature['idx']]
        tree = {best_feature_name: {}}
        qnode = tree[best_feature_name]

        if best_feature['type'] == int:
            best_split = best_feature['split']
            data_above = data[best_feature_data > best_split]
            data_below = data[best_feature_data <= best_split]

            if len(data_below) == 0 or len(data_above) == 0:
                return self.classifyData(data)

            question_above = "> {}".format(best_split)
            question_below = "<= {}".format(best_split)
            ans_above = self.buildTree(
                data=data_above, depth=depth, random_feature=random_feature)
            ans_below = self.buildTree(
                data=data_below, depth=depth, random_feature=random_feature)

            if ans_above == ans_below:
                tree = ans_above
            else:
                qnode[question_above] = ans_above
                qnode[question_below] = ans_below

        else:
            best_catories = best_feature['categories']
            for category in best_catories:
                level_data = data[best_feature_data == category]
                question = "== {}".format(category)
                qnode[question] = self.buildTree(
                    data=level_data, depth=depth, random_feature=random_feature)

            # for other possible branch not exist in train data set
            question = "!= possible"
            qnode[question] = self.classifyData(data)

        return tree
```

1. 檢查 Base case
   * data 的 Purity，若 data 為單一的 label 就直接 classify Data
   * 若資料量剩一筆，直接 classify Data
   * 若遞迴深度到達最大深度，classify Data
2. 算出 data 中各個 feature 的 unique levels
3. 選出最佳的 feature 當作 root (Information Gain 最大的 feature)
4. best feature 分兩種情況
   * 連續 (data 的 type 為 int)
     * 對最佳的分割值做分割
     * 若分割後 data 為空則直接 return classify Data
     * 非空則建立 >, <= 的 branch
     * 若分割後 above 與 below 的 data 相同則直接 return 其中一個，不再分割 branch
   * 非連續 (data 的 type 為 str)
     * 對所有 category 都建一條 branch
     * 再建一條如果所有 branch 都沒有 match 到的情況 (有可能 test data 的 category 不包含在 train data 中)

### 選擇最佳分支

```py
# find feature of highest Information Gain (lowest Remainder)
def detBestFeature(self, data, levels):
    ret_val = {
        'idx': 7,
        'type': str,
        'split': 0,
        'categories': []
    }

    min_remainder = 2147483647
    for feature_idx, level in levels.items():
        column = data[:, feature_idx]
        column_type = type(column[0])

        if column_type == int:
            for split_value in level:
                remainder = self.calcContinousRem(
                    data, feature_idx, split_value)
                if remainder < min_remainder:
                    min_remainder = remainder
                    ret_val['idx'] = feature_idx
                    ret_val['type'] = int
                    ret_val['split'] = split_value

        else:
            remainder = self.calcCategoryRem(data, feature_idx, level)
            if remainder < min_remainder:
                min_remainder = remainder
                ret_val['idx'] = feature_idx
                ret_val['type'] = str
                ret_val['categories'] = level

    return ret_val
```

1. 最佳的 feature 為 remainder 最小的 feature
2. 兩種可能情況
    * 連續
      * 對所有可能的 split value (unique levels) 分割 data
      * 計算這個分割的 remainder 若為最小就更新最佳的 feature
    * 非連續
      * 對所有可能的 category 分割 data
      * 計算 category 的 remainder 若為最小就更新最佳的 feature

```py
def calcCategoryRem(self, data, feature_idx, level):
    remainder = 0
    column = data[:, feature_idx]
    for category in level:
        level_data = data[column == category]
        partition_entropy = self.calcEntropy(level_data)
        conditional_prob = len(level_data) / len(data)
        remainder += partition_entropy * conditional_prob
    return remainder

def calcContinousRem(self, data, feature_idx, split_value):
    # split 2 parts
    data_below, data_above = self.splitContinuous(
        data=data, col_idx=feature_idx, value=split_value)
    # calculate total entropy
    p_data_below = len(data_below) / len(data)
    p_data_above = len(data_above) / len(data)
    remainder = (p_data_below * self.calcEntropy(data_below) +
                    p_data_above * self.calcEntropy(data_above))
    return remainder

def calcEntropy(self, data):
    label_data = data[:, -1]
    _, counts = np.unique(label_data, return_counts=True)
    probabilities = counts / counts.sum()
    entropy = -sum(probabilities * np.log2(probabilities))
    return entropy
```

1. 算出 entropy
2. 計算條件機率

### helper function

```py
# utility functions
def splitContinuous(self, data, col_idx, value):
    column = data[:, col_idx]
    data_above = data[column > value]
    data_below = data[column <= value]
    return data_below, data_above
```

1. 分割上下兩段 data

```py
def getUniqueLevels(self, data, random_feature=None):
    levels = {}
    _, n_cols = data.shape
    column_indices = list(range(n_cols - 1))  # except label

    if random_feature and random_feature <= len(column_indices):
        column_indices = random.sample(
            population=column_indices, k=random_feature)

    for col_idx in column_indices:
        col_data = data[:, col_idx]
        col_unique = np.unique(col_data)
        levels[col_idx] = col_unique

    return levels
```

1. 算出 unique level
2. 若 random forest 有要隨機選擇 feature 則隨機選擇 index

```py
def classifyData(self, data):
    label_data = data[:, -1]
    unique_label, unique_counts = np.unique(label_data, return_counts=True)
    idx = unique_counts.argmax()
    return unique_label[idx]
```

1. 選擇 data 中出現頻率最高的 label

```py
def checkPurity(self, data):
    label_data = data[:, -1]
    label_unique = np.unique(label_data)
    if len(label_unique) == 1:
        return True
    else:
        return False
```

1. 若 data 中只存在一種 label 則為 purity

### 預測答案

```py
def predict(self, entry):
    return self.guess(self.tree, entry)
```

1. call guess 這個遞迴 function

```py
def guess(self, tree, entry):
    if not isinstance(tree, dict):
        return tree

    feature = list(tree.keys())[0]

    for question, predict_ans in tree[feature].items():
        condition, value = question.split(" ")
        answer = predict_ans

        if condition == "==" and entry[feature] == value:
            break
        elif condition == "<=" and entry[feature] <= float(value):
            break
        elif condition == ">" and entry[feature] > float(value):
            break

    if not isinstance(answer, dict):
        return answer
    else:
        return self.guess(answer, entry)
```

1. 若 tree 直接為 label 則直接 return
2. 選擇 tree 的 feature
3. 對 feature 下的 question 做 parse
4. 把 answer 設為 predict_ans (可能是一顆樹或是直接是 label)，若滿足條件則 break
5. 如果 answer 是樹則遞迴下去找答案
6. 若 answer 是 label 則直接 return

## Result

```
1) Holdout validation with train/test:7/3
        0      1
0  4779.0  390.0
1   689.0  980.0
accuracy: 0.8422053231939164
```

```
2) K-fold cross-validation with K=3
-- fold 0
        0      1
0  5372.0  403.0
1   834.0  988.0
accuracy: 0.8371725681189943
-- fold 1
        0       1
0  5334.0   397.0
1   802.0  1064.0
accuracy: 0.8421745425825984
-- fold 2
        0       1
0  5319.0   439.0
1   761.0  1078.0
accuracy: 0.8420429116756615
```
