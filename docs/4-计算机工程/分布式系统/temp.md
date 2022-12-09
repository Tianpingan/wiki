## 死亡预测率程序报告
采用随机森林模型，`model`函数具体实现如下：

```python

def model(_data):
    _iloc_values = _data.iloc[:, -1].values
    _data = _data.drop(["Adult Mortality"], axis=1)
    data_norm, imputer, scaler = preprocess_data(_data)

    _norm_values = data_norm.values
    ETregressor = ExtraTreesRegressor(max_depth=15, max_samples=0.90, min_samples_leaf=5, min_samples_split=2,
                                    n_estimators=300)
    ETregressor.fit(_norm_values, _iloc_values)

    joblib.dump(ETregressor, model_filename)
    joblib.dump(imputer, imputer_filename)
    joblib.dump(scaler, scaler_filename)

    return ETregressor

```
除此之外，在数据预处理部分舍弃`Country`和`Status`变量。
```python
data = data.drop(["Country", "Status"], axis=1)
```
系统测试结果如下：
![](../../img/Pasted%20image%2020221015192714.png)