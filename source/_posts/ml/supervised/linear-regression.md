---
title: 'Linear Regression'
tags: 'ML'
---

# Linear Regression

[linear_data.txt](/note/ml/supervised/linear-regression/linear_data.txt)

## 反矩陣

```py
def mat_inv(mat):
    n = mat.shape[0]
    m = mat.shape[1]
    if n != m:
        return None
    iden = np.identity(n)
    argu = np.concatenate((mat, iden), axis=1)
    
    # partial pivoting
    for i in range(1, n):
        for row in reversed(range(i, n)):
            if (argu[row][i - 1] > argu[row - 1][i - 1]):
                argu[[row - 1, row]] = argu[[row, row - 1]]
    
    # reduce to diagonal  matrix
    for i in range(n): # iterate all diagonal idx
        for row in range(n):
            if row != i:
                k = argu[row][i] / argu[i][i]
                argu[row] -= argu[i] * k
    
    # reducing to unit matrix
    for i in range(n):
        argu[i] /= argu[i][i]
        
    return argu[:, n:n*2]
```

1. 建立增廣矩陣
2. 透過交換 rows 讓 pivot 不要為 0
3. 將所有非對角的 elements 減為 0 (變成對角矩陣)
4. 將對角矩陣化為單位矩陣
5. 取增廣矩陣右方當作反矩陣

## 主程式

讀入資料

```py
x = np.array([] ,dtype=float)
y = np.array([] ,dtype=float)

with open('data/linear_data.txt', 'r') as f:
    for line in f:
        xy = line.strip().split(',')
        x = np.append(x, float(xy[0]))
        y = np.append(y, float(xy[1]))
```

定義 dimension，並建立矩陣

```
N = 3
X_T = np.array([x ** i for i in range(N)])
X = X_T.transpose()
```

beta 為係數，可以透過線性代數求解：

```py
beta = mat_inv(X_T.dot(X)).dot(X_T).dot(y)
```

推得係數後，可以得到 fitting line 與 error：

```py
ey = np.zeros(len(x))
for i in range(N):
    ey += pow(x, i) * beta[i]

beta_str = ["{} X^{}".format(i, idx) for idx, i in enumerate(beta)]
fitting_line = " + ".join(beta_str)
error = sum((y - ey) ** 2)
print("Fitting line:", fitting_line)
print("Total error:", error)

plt.scatter(x, y, color='red')
plt.plot(x, ey)
```

當 N = 2

![](https://i.imgur.com/hTpsDof.png)

當 N = 3

![](https://i.imgur.com/b3pUcPf.png)