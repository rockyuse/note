---
title: 'Logistic Regression'
tags: 'ML'
---

# Logistic Regression

[Logistic_data1-1.txt](/note/ml/supervised/logistic-regression/Logistic_data1-1.txt)
[Logistic_data1-2.txt](/note/ml/supervised/logistic-regression/Logistic_data1-2.txt)
[Logistic_data2-1.txt](/note/ml/supervised/logistic-regression/Logistic_data2-1.txt)
[Logistic_data2-2.txt](/note/ml/supervised/logistic-regression/Logistic_data2-2.txt)

## Helper function

Sigmoid function:

```py
def sigmoid(scores):
    return 1 / ( 1 + np.exp(-scores) )
```

normalized data:

```py
def normalized(arr):
    return (arr - arr.min()) / (arr.max() - arr.min())
```

## Model

```py
def logistic_regression(features_df, target_df, alpha, thold, method='l2-norm', animation=False):
    features = features_df.to_numpy()
    target = target_df.to_numpy()

    w = np.zeros(features.shape[1])
    y = target.reshape(len(target),)
    
    while(True):
        M_w = sigmoid(features.dot(w))
        error = ((y - M_w) ** 2).sum()

        if method == 'l2-norm':
            grad = np.array([((y - M_w) * M_w * (1 - M_w) * x).sum() for x in features.T])
        else:
            grad = features.T.dot(y - M_w)
        
        w = w + alpha * grad

        if animation:
            draw(w)
            print('Total error:', error)
            time.sleep(0.01)
            display.clear_output(wait=True)

        if error < thold:
            break
            
    return w
```

1. 初始化 weight
2. 計算 $M_w$ (prediction)
3. 計算 error
4. 計算 gradient (透過 L2-norm 或是 cross-entropy)
5. 計算新的 w (alpha 為 learning rate)
6. 若 error 小於 threshold 則 return

## Data 1

### 讀入資料

```py
x1 = np.array([], dtype=float)
y1 = np.array([], dtype=float)
x2 = np.array([], dtype=float)
y2 = np.array([], dtype=float)

with open('data/Logistic_data1-1.txt', 'r') as f:
    for line in f:
        xy = line.strip().split(',')
        x1 = np.append(x1, float(xy[0]))
        y1 = np.append(y1, float(xy[1]))

with open('data/Logistic_data1-2.txt', 'r') as f:
    for line in f:
        xy = line.strip().split(',')
        x2 = np.append(x2, float(xy[0]))
        y2 = np.append(y2, float(xy[1]))

t1 = np.zeros(len(y1), dtype=int)
t2 = np.ones(len(y2), dtype=int)

plt.scatter(x1, y1, color='red')
plt.scatter(x2, y2, color='blue')
```

* Logistic_data1-1.txt 的 target 為 0
* Logistic_data1-2.txt 的 target 為 1

![](https://i.imgur.com/FDAxT6T.png)

### 正規化資料

```py
df = pd.DataFrame({'norm_x': normalized(np.concatenate((x1, x2))),
                   'norm_y': normalized(np.concatenate((y1, y2))),
                   'x': np.concatenate((x1, x2)),
                   'y': np.concatenate((y1, y2)),
                   'bias': np.ones(len(x1) + len(x2)),
                   'target': np.concatenate((t1, t2))})

category_0 = df.loc[df['target'] == 0]
category_1 = df.loc[df['target'] == 1]

plt.scatter(category_0['norm_x'], category_0['norm_y'], color='red')
plt.scatter(category_1['norm_x'], category_1['norm_y'], color='blue')
```

![](https://i.imgur.com/M13HZmi.png)

### Regression

```py
features = df[['norm_x', 'norm_y', 'bias']]
target = df[['target']]

# method = 'l2-norm' | 'cross-entropy'
w = logistic_regression(features, target, alpha=0.02, thold=3, method='cross-entropy', animation=False)
draw(w)
print("w:", w)
```

得到在梯度下降後的 weight `w: [ 3.21925838  2.67672689 -2.6629523 ]`

![](https://i.imgur.com/SE7lOCO.gif)

### 計算結果

```py
M_w = sigmoid(features.dot(w))

def classify(a):
    return 1 if a > 0.5 else 0
    
prediction = np.array([classify(e) for e in M_w])
df['prediction'] = prediction
pred_0 = df.loc[df['prediction'] == 0]
pred_1 = df.loc[df['prediction'] == 1]

plt.scatter(pred_0['x'], pred_0['y'], color='red')
plt.scatter(pred_1['x'], pred_1['y'], color='blue')
```

若 prediction > 0.5，則分類為 1，否則分類為 0

```py
cm = np.zeros(shape=(2, 2))
for index, row in df.iterrows():
    ans = int(row.target)
    pred = int(row.prediction)
    cm[ans][pred] += 1
    
labelx = ['predict 1', 'predict 2']
labely = ['target 1', 'target 2']
print("Comfusion matrix:")
print(pd.DataFrame(data=cm, columns=labelx, index=labely))

precision = cm[0][0] / (cm[0][0] + cm[1][0])
recall = cm[0][0] / (cm[0][0] + cm[0][1])
print("\nPrecision:", precision)
print("Recall", recall)
```

**L2-norm:**

![](https://i.imgur.com/lVVAfOX.png)

```
Comfusion matrix:
          predict 1  predict 2
target 1       50.0        0.0
target 2        0.0       50.0

Precision: 1.0
Recall 1.0
```

**cross-entropy:**

![](https://i.imgur.com/MQ3A4ds.png)

```
Comfusion matrix:
          predict 1  predict 2
target 1       50.0        0.0
target 2        0.0       50.0

Precision: 1.0
Recall 1.0
```

## Data 2

整體架構與 Data 1 完全一致，只有讀入不同的資料

### 讀入資料

* Logistic_data2-1.txt 的 target 為 0
* Logistic_data2-2.txt 的 target 為 1

![](https://i.imgur.com/DHxxpQ4.png)

### 正規化資料

![](https://i.imgur.com/3vdx9MG.png)

### Regression

得到在梯度下降後的 weight `w: [ 4.92765738  6.45464548 -5.52168587]`

![](https://i.imgur.com/KRAvw6z.gif)

### 計算結果

**L2-norm:**

![](https://i.imgur.com/LYueVVF.png)

```
Comfusion matrix:
          predict 1  predict 2
target 1       41.0        9.0
target 2       10.0       40.0

Precision: 0.803921568627451
Recall 0.82
```

**cross-entropy:**

![](https://i.imgur.com/UvFk5za.png)

```
Comfusion matrix:
          predict 1  predict 2
target 1       40.0       10.0
target 2        9.0       41.0

Precision: 0.8163265306122449
Recall 0.8
```