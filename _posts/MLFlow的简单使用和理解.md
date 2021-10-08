---
title: MLFlow的简单使用和理解
tags: 编程基础
categories: 编程基础
abbrlink: 413e60ef
date: 2021-01-20 20:24:42
---

### 什么是MLflow
mlflow的口号喊的很响亮，实际我们分解开看，mlflow提供的其实是一个模型开发过程的管理平台，一个很轻量化的工具。

也就是说mlflow并不限制和关心你使用的什么机器学习框架，只是要求在模型开发过程中，使用mlflow提供的sdk的api对过程中的动作进行注册，以便在mlflow中进行监控。

同时mlflow提供了过程中的标准，包括日志，模型等，这样的好处就是mlflow可以统一的负责模型的管理和部署，以统一的模型服务对模型进行发布，还能基于不同版本对模型进行管理。

### MLFlow的配置

我使用的mysql作为mlflow的tracking的存储介质：

```
mlflow server --backend-store-uri   mysql+pymysql://root:baifachuan@localhost/mlflow --default-artifact-root file:./mlruns -h 0.0.0.0 -p 5000

```
可以看到mysql数据库中有这样的表：

```
mysql> use mlflow;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-----------------------+
| Tables_in_mlflow      |
+-----------------------+
| alembic_version       |
| experiment_tags       |
| experiments           |
| latest_metrics        |
| metrics               |
| model_version_tags    |
| model_versions        |
| params                |
| registered_model_tags |
| registered_models     |
| runs                  |
| tags                  |
+-----------------------+
12 rows in set (0.00 sec)

mysql>

```

用来把tracking的数据存储在表中。


### 结合MLFlow的开发

以下代码是我使用`sklearn`构建的一个线性回归模型，并且将tracking注册到MLFlow进行跟踪。


```python
# The data set used in this example is from http://archive.ics.uci.edu/ml/datasets/Wine+Quality
# P. Cortez, A. Cerdeira, F. Almeida, T. Matos and J. Reis.
# Modeling wine preferences by data mining from physicochemical properties. In Decision Support Systems, Elsevier, 47(4):547-553, 2009.

import os
import warnings
import sys

import pandas as pd
import numpy as np
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score
from sklearn.model_selection import train_test_split
from sklearn.linear_model import ElasticNet

import mlflow
import mlflow.sklearn

import logging
logging.basicConfig(level=logging.WARN)
logger = logging.getLogger(__name__)

remote_server_tracking_uri = "http://localhost:5000"
mlflow.set_tracking_uri(remote_server_tracking_uri)
mlflow.set_experiment("AirBnb Tracking")

def eval_metrics(actual, pred):
    rmse = np.sqrt(mean_squared_error(actual, pred))
    mae = mean_absolute_error(actual, pred)
    r2 = r2_score(actual, pred)
    return rmse, mae, r2



if __name__ == "__main__":
    warnings.filterwarnings("ignore")
    np.random.seed(40)

    # Read the wine-quality csv file from the URL
    csv_url =\
        'http://archive.ics.uci.edu/ml/machine-learning-databases/wine-quality/winequality-red.csv'
    try:
        data = pd.read_csv(csv_url, sep=';')
    except Exception as e:
        logger.exception(
            "Unable to download training & test CSV, check your internet connection. Error: %s", e)

    # Split the data into training and test sets. (0.75, 0.25) split.
    train, test = train_test_split(data)

    # The predicted column is "quality" which is a scalar from [3, 9]
    train_x = train.drop(["quality"], axis=1)
    test_x = test.drop(["quality"], axis=1)
    train_y = train[["quality"]]
    test_y = test[["quality"]]

    alpha = float(sys.argv[1]) if len(sys.argv) > 1 else 0.5
    l1_ratio = float(sys.argv[2]) if len(sys.argv) > 2 else 0.5

    with mlflow.start_run():
        lr = ElasticNet(alpha=alpha, l1_ratio=l1_ratio, random_state=42)
        lr.fit(train_x, train_y)

        predicted_qualities = lr.predict(test_x)

        (rmse, mae, r2) = eval_metrics(test_y, predicted_qualities)

        print("Elasticnet model (alpha=%f, l1_ratio=%f):" % (alpha, l1_ratio))
        print("  RMSE: %s" % rmse)
        print("  MAE: %s" % mae)
        print("  R2: %s" % r2)

        mlflow.log_param("alpha", alpha)
        mlflow.log_param("l1_ratio", l1_ratio)
        mlflow.log_metric("rmse", rmse)
        mlflow.log_metric("r2", r2)
        mlflow.log_metric("mae", mae)

        mlflow.sklearn.log_model(lr, "model")
```


模型保存后会存储在当前目录`mlruns`下，按照不同的版本进行管理。

### MLflow部署

MLFlow在部署的时候需要一个`python`的运行隔离环境，和`virtualenv`的概念类似，mlflow默认使用的conda，但是我自己本身已经有`virtualenv`了，又不想再安装conda的全家桶污染环境，所以使用了手动的静默安装：

```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-x86_64.sh -O ~/miniconda.sh
bash ~/miniconda.sh -b -p $HOME/miniconda
```

其中的参数解释如下：

```
-b---Batch mode with no PATH modifications to ~/.bashrc. Assumes that you agree to the license agreement. Does not edit the .bashrc or .bash_profile files.
-p---Installation prefix/path.
-f---Force installation even if prefix -p already exists.
```

在运行部署的时候先设置Conda的环境变量：

```
export MLFLOW_CONDA_HOME=/Users/fcbai/software/workspace/python/mlflow_ws/miniconda
```

紧接着启动部署：

```
mlflow models serve -m file:/Users/fcbai/software/workspace/python/mlflow_ws/mlruns/1/84bad08c43f34c78a4ada11ea0e30cf6/artifacts/model/
```

也可以指定端口：
```
mlflow models serve -m file:/Users/fcbai/software/workspace/python/mlflow_ws/mlruns/1/84bad08c43f34c78a4ada11ea0e30cf6/artifacts/model/ -p 8088
```
在经过启动部署后，有如下输出：

```
2021/01/20 10:26:06 INFO mlflow.pyfunc.backend: === Running command 'source /Users/fcbai/software/workspace/python/mlflow_ws/miniconda/bin/../etc/profile.d/conda.sh && conda activate mlflow-73c2330f885e4aee44450a5d5830493e7d78ae67 1>&2 && gunicorn --timeout=60 -b 127.0.0.1:5000 -w 1 ${GUNICORN_CMD_ARGS} -- mlflow.pyfunc.scoring_server.wsgi:app'
[2021-01-20 10:26:07 +0800] [52783] [INFO] Starting gunicorn 20.0.4
[2021-01-20 10:26:07 +0800] [52783] [INFO] Listening at: http://127.0.0.1:5000 (52783)
[2021-01-20 10:26:07 +0800] [52783] [INFO] Using worker: sync
[2021-01-20 10:26:07 +0800] [52801] [INFO] Booting worker with pid: 52801
^C[2021-01-20 10:27:54 +0800] [52783] [INFO] Handling signal: int
[2021-01-20 10:27:54 +0800] [52801] [INFO] Worker exiting (pid: 52801)
```

代表模型正式启动，以htt的方式，json的协议提供模型的访问，例如：

```
curl -X POST -H "Content-Type:application/json; format=pandas-split" \
--data '{"columns":["alcohol", "chlorides", "citric acid", "density", "fixed acidity", "free sulfur dioxide", "pH", "residual sugar", "sulphates", "total sulfur dioxide", "volatile acidity"],"data":[[12.8, 0.029, 0.48, 0.98, 6.2, 29, 3.33, 1.2, 0.39, 75, 0.66]]}' \
http://127.0.0.1:8088/invocations
```
返回结果如下：

```
[6.379428821398614]
```
整个流程便跑完了。

