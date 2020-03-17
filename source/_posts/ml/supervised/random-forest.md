---
title: 'Random Forest'
tags: 'ML'
---

# Random Forest

承接 [Decision Tree](/note/ml/supervised/decision-tree)

## 主程式

架構與 decision tree 相同，只有 model 的 instance 改為 RandomForest

## Data Preprocessing

與 decision tree 完全相同

## Model

```PY
import numpy as np
from .decision_tree import DecisionTree

class RandomForest():
    forest = []

    def __init__(self, df, n_trees=5, n_bootstrap=3000, n_features=7, max_depth=5):
        for i in range(n_trees):
            df_bootstrapped = self.bootstrap(df, n_bootstrap)
            tree = DecisionTree(df=df_bootstrapped,
                                max_depth=max_depth, random_feature=n_features)
            self.forest.append(tree)

    def bootstrap(self, df, n_bootstrap):
        bootstrap_indices = np.random.randint(
            low=0, high=len(df), size=n_bootstrap)
        df_bootstrapped = df.iloc[bootstrap_indices]
        return df_bootstrapped

    def predict(self, entry):
        predictions = []
        for tree in self.forest:
            prediction = tree.predict(entry)
            predictions.append(prediction)
        # return most frequently elements in list
        return max(set(predictions), key=predictions.count)
```

建構式：

* 建立 N 個 decision tree
* 若有需要 bootstrap 則在 data 中 random 選出 n_bootstrap 筆資料
* 把 tree append 到 forest 中

predict:

* 把 tree predict 的結果 append 到一個 list 中
* 找出 list 中出現頻率最高的 element

## Result

```
1) Holdout validation with train/test:7/3
        0      1
0  4803.0  374.0
1   716.0  945.0
accuracy: 0.8405966656917228
```

```
2) K-fold cross-validation with K=3
-- fold 0
        0      1
0  5524.0  270.0
1   854.0  949.0
accuracy: 0.8520468606028696
-- fold 1
        0       1
0  5362.0   349.0
1   813.0  1073.0
accuracy: 0.8470448861392655
-- fold 2
        0       1
0  5455.0   304.0
1   816.0  1022.0
accuracy: 0.8525733842306173
```
