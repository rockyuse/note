---
title: 'Backpropagation'
tags: 'ML'
---

# Backpropagation

[data.txt](/note/ml/deep/backpropagation/data.txt)

## 程式架構

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-x))

def d_sigmoid(x):
    return x * (1.0 - x)

class Net:
    def __init__(self, lr=0.05):
        ...

    def forward(self, x, y):
        ...
        
    def backward(self, label):
        ...
        
class Agent:
    def __init__(self, data):
        ...
        
    def predict(self):
        ...
        
    def train(self):
        ...
    
    def plot(self):
        ...
    
    def main(self):
        ...


if __name__ == '__main__':
    ...
```

### 主程式

```python
if __name__ == '__main__':
    data = []
    with open('data.txt', 'r') as f:
        for line in f:
            line_data = line.strip().split(',')
            line_data = [float(d) for d in line_data]
            data.append(line_data)
    agent = Agent(data)

    # Train Model
    agent.main()

    # Plot Loss and Accuracy
    agent.plot()

    fig = plt.figure(figsize=(10, 8))
    ax1 = fig.add_subplot(221)
    ax2 = fig.add_subplot(222)
    ax1.set_title('Predict result')
    ax2.set_title('Ground truth')

    # line
    x = np.linspace(0,100)
    y = x

    # plot ground truth
    ground_truth = pd.DataFrame(data, columns=['x', 'y', 'label'])
    category_0 = ground_truth.loc[ground_truth['label'] == 0]
    category_1 = ground_truth.loc[ground_truth['label'] == 1]
    ax2.scatter(category_0['x'], category_0['y'], color='red')
    ax2.scatter(category_1['x'], category_1['y'], color='blue')
    ax2.plot(x, y)

    # plot predict result
    pred = agent.predict()
    predict_result = pd.DataFrame(data, columns=['x', 'y', 'label'])
    predict_result['label'] = pred

    category_0 = predict_result.loc[predict_result['label'] == 0]
    category_1 = predict_result.loc[predict_result['label'] == 1]
    ax1.scatter(category_0['x'], category_0['y'], color='red')
    ax1.scatter(category_1['x'], category_1['y'], color='blue')
    ax1.plot(x, y)

    plt.show()
```

1. 讀入資料
2. 建立 Neural Network 的 Agent
3. 訓練模型
4. evaluate 訓練結果，並畫圖

### Neural Network

```python
class Net:
    def __init__(self, lr=0.05):
        self.lr = lr
        self.weights1 = np.random.rand(2, 4)
        self.weights2 = np.random.rand(4, 4)
        self.weights3 = np.random.rand(4, 1)

    def forward(self, x, y):
        self.input = np.array([[x, y]], dtype=np.float128)
        self.hidden1 = np.dot(self.input, self.weights1)
        self.hidden1 = sigmoid(self.hidden1)
        self.hidden2 = np.dot(self.hidden1, self.weights2)
        self.hidden2 = sigmoid(self.hidden2)
        self.output = np.dot(self.hidden2, self.weights3)
        self.output = sigmoid(self.output)
        
    def backward(self, label):
        label = np.array([[label]], dtype=np.float128)
        
        d_weight3 = 2 * (label - self.output) * d_sigmoid(self.output)
        loss3 = np.dot(self.hidden2.T, d_weight3)
        
        d_weight2 = np.dot(d_weight3, self.weights3.T) * d_sigmoid(self.hidden2)
        loss2 = np.dot(self.hidden1.T, d_weight2)
        
        d_weight1 = np.dot(d_weight2, self.weights2.T) * d_sigmoid(self.hidden1)
        loss1 = np.dot(self.input.T, d_weight1)
        
        self.weights1 += self.lr * loss1
        self.weights2 += self.lr * loss2
        self.weights3 += self.lr * loss3
```

神經網路結構：

* input layer 為 2 個 neural (x, y)
* hidden layer 為兩層各 4 個 neural
* output layer 為 1 個 neural (0~1)

建構式：

1. 初始化 learning rate
2. 因為有 2 層 hidden layer，所以共有 3 個 weights

forward:

1. 將 input 建立成 1x2 的 vector
2. 與 weights1 做矩陣乘法，得到 hidden layer 1 (1x4)
3. 與 weights2 做矩陣乘法，得到 hidden layer 2 (1x4)
4. 與 weights2 做矩陣乘法，得到 output layer (1x1)

backward:

1. 透過 loss function (sum of square error) 的梯度計算 weights3 的 loss3
2. 反向傳播計算 loss2
3. 反向傳播計算 loss1
4. 將 loss 乘上 learning rate 調整 weights

### Agent

```python
class Agent:
    def __init__(self, data):
        self.nn = Net(lr=0.04)
        self.data = data
        self.losses = []
        self.accuracy = []
        
    def predict(self):
        preds = []
        for entry in self.data:
            x = entry[0]
            y = entry[1]
            label = entry[2]
            
            self.nn.forward(x, y)
            pred = 0 if self.nn.output[0][0] < 0.5 else 1
            preds.append(pred)
        return preds
        
    def train(self):
        loss = 0
        acc_cntr = 0
        
        for entry in self.data:
            x = entry[0]
            y = entry[1]
            label = entry[2]

            self.nn.forward(x, y)
            self.nn.backward(label)

            output = self.nn.output[0][0]
            loss += (label - output) ** 2

            pred = 1 if output > 0.5 else 0
            if pred == label:
                acc_cntr += 1

        return loss, acc_cntr
    
    def plot(self):
        fig = plt.figure(figsize=(8, 6))
        ax1 = fig.add_subplot(221)
        ax2 = fig.add_subplot(222)
        ax1.plot(self.losses)
        ax1.set_title('loss')
        ax2.plot(self.accuracy)
        ax2.set_title('accuracy')
        plt.show()
    
    def main(self):
        for i in range(10001):
            loss, acc_cntr = self.train()
            acc = acc_cntr / len(self.data)
            self.losses.append(loss)
            self.accuracy.append(acc)
            if i % 1000 == 0:
                print('epochs', i, 'loss:', loss, 'accuracy:', acc)
```

建構式：

1. 建立 nn 的 instance
2. 儲存 data 與 initialize loss 與 accuracy 的 array

main():

1. 共進行 10001 次的訓練 (0~10000)
2. 將訓練結果紀錄
3. 每 1000 次 print 出當前的 loss 與 accuracy

train():

1. 對 data 內的每個 entry 進行 forward
2. 將結果 backward 並調整神經網路
3. 累加 loss 與計算預測結果

predict():

1. 對 data 內的每個 entry 進行 forward 不 backward
2. return 整個 data 的預測結果

## 執行結果

### Train

```
epochs 0 loss: 30.277324475025709246 accuracy: 0.49
epochs 1000 loss: 0.024934396168262030844 accuracy: 1.0
epochs 2000 loss: 0.008230372203206555887 accuracy: 1.0
epochs 3000 loss: 0.0048464426050304105523 accuracy: 1.0
epochs 4000 loss: 0.0034139119408884769078 accuracy: 1.0
epochs 5000 loss: 0.002627065551047310526 accuracy: 1.0
epochs 6000 loss: 0.002131124670777076718 accuracy: 1.0
epochs 7000 loss: 0.0017905544571033081806 accuracy: 1.0
epochs 8000 loss: 0.0015425349580361289617 accuracy: 1.0
epochs 9000 loss: 0.0013540187649136039913 accuracy: 1.0
epochs 10000 loss: 0.001205983589243599169 accuracy: 1.0
```

![](https://i.imgur.com/RkWbqIe.png)

### Prediction

![](https://i.imgur.com/Kyl1LI0.png)


