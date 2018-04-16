# 使用 Serverless 进行 AI 预测推理

## 概览

在 AI 项目中，通常大家关注的都是怎么进行训练、怎么调优模型、怎么来达到满意的识别率。但对于一个完整项目来说，通常是需求推动的项目，同时项目最终也要落到实际业务中来满足需求。对于 AI 项目来说，落地到实际项目中，就是将训练的模型，投入到生产环境中，使用生成环境的数据，根据模型进行推理预测，满足业务需求。

而常规的部署方案，通常都是将模型部署到一台独立设备上，对外以 API 接口的形式提供服务，业务模块或前端 APP 等所需预测推理能力的位置，通过调用 API，传递原始数据，由 AI 推理服务完成预测推理后，将结果返回。这个过程，通常称为 AI Serving。

对于最常用的 AI 训练和机器学习的工具 TensorFlow 来说，它本身也提供了 AI serving 工具 TensorFlow Serving。利用此工具，可以将训练好的模型简单保存为模型文件后，并通过脚本在 TensorFlow Serving 加载模型，输入待推理数据，得到推理结果。

而对于 AI 推理来说，其调用需求会随着业务的涨落而涨落，会出现白天高、夜间低的现象，而和 AI 训练时的较固定计算周期和运行时长而有所不同。同时，如果在推理时使用了 GPU 来加速，那一样存在着白天利用率高，夜间利用率低的现象，而又得为峰值时刻准备足够的 GPU 设备来避免业务拥塞或响应速度降低，很大程度上照成了资源浪费。

而利用 Serverless 的自动扩缩容技术，在实现 AI Serving 的云函数后，即可将其放心使用在业务中。在高业务请求到来时，云函数的执行实例自动扩容，满足业务需求；而在请求低谷到来时，或者无请求到来时，云函数自动缩容甚至完全停止，节省资源使用。同时，云函数按执行时间进行计费的方式，也可以更进一步的节约费用使用，避免为长时间空闲的 GPU 设备付费。

接下来，我们就演示下如何使用腾讯云的 SCF 无服务器云函数来实现 AI Serving 能力。目前，SCF 云函数产品，已经在python 2.7 环境内内置了 TensorFlow Serving 及相关的 Python 库，可支持直接调用 Serving 组件。同时 SCF 云函数也已经灰度开放了 GPU 支持，可以使用 GPU 来进一步加快 AI 推理速度。

## 模型准备

在这里我们使用 TensorFlow 中的 MNIST 实验作为案例来进行下面的介绍。MNIST 是一个包含了 6 万训练图片和 1 万测试图片的手写数字图片集合，图片为 28x28-pixel 大小的黑白图片。此训练集通常用来训练对数字的识别能力。

关于如何编写代码，使用 MNIST 训练集完成模型训练，可以见 [TF层指南：建立卷积神经网络](https://www.tensorflow.org/tutorials/layers?hl=zh-cn)，这篇文章详细介绍了如何通过使用 Tensorflow layer 构建卷积神经网络，并设置如何进行训练和评估。

而在进行训练和评估后，就可以进行模型的导出了。TensorFlow 的模型文件包含了深度学习模型的 Graph 和参数，也就是 checkpoint 文件。在导出模型文件后，我们可以加载模型文件继续训练或者对外提供推理服务。

这里我们可以通过 SavedModelBuilder 模块来进行模型到处保存，更具体的文档和操作方法可见 [训练和导出 TF 模型](https://www.tensorflow.org/serving/serving_basic?hl=zh-cn#train_and_export_tensorflow_model)。

导出后的文件，为 saved_model.pb 文件， variables 文件夹及包含的若干variables文件，分别是模型的图文件和参数文件。后续在提供推理能力时，就是使用这些图及变量文件，加载到 TF Serving 内。

为了便于后续的操作，我们在这里也直接提供我们导出的模型文件供后续操作，可以点击这里的[导出模型文件](https://main.qcloudimg.com/raw/c184a8b6e802192e4ff71ecb4cb07ec8.zip)来下载。

## 测试文件准备

测试文件我们可以从 MNIST 的测试集中随意抽取若干，用于验证我们最终推理 API 的工作状态。同样，我们也在这里准备了若干图片用于最终验证，可以点击这里的[测试图片文件](https://main.qcloudimg.com/raw/3353ca26b71061c7b3f3c57bc8039832.zip)来下载。或者记录以下连接，用于直接测试在线图片的推理情况。

```
https://main.qcloudimg.com/raw/84783c178cdc6d6b2302bc1b4749b91b.bmp
https://main.qcloudimg.com/raw/0f4630a815c44107a79a224d3263da2c.bmp
https://main.qcloudimg.com/raw/360a7cdd638d22d4145c94e67b3c059f.bmp
https://main.qcloudimg.com/raw/90067917b01d4b4d31f207ac78e70416.bmp
https://main.qcloudimg.com/raw/7828379e768aa5c8a6d09a3d58f64921.bmp
https://main.qcloudimg.com/raw/79e0c07766c739c880dfed8d0433ff83.bmp
https://main.qcloudimg.com/raw/b5f1c6f4ba08c3376c333c5a62f0c1dd.bmp
https://main.qcloudimg.com/raw/40adedc18205428276c3753bac51e740.bmp
https://main.qcloudimg.com/raw/4c750171b4b31772f0b923beef92c9f3.bmp
https://main.qcloudimg.com/raw/8504482825667f97b21209b9570249e6.bmp
https://main.qcloudimg.com/raw/5f22f6f83d26a3f267c823cd5437bdfc.bmp
https://main.qcloudimg.com/raw/fb1f59a3fdbcad0a140045508f28f688.bmp
```

## 函数创建及测试

我们使用如下示例来创建函数程序包。

### 准备函数文件

创建目录 mnist_demo，并在根目录下创建文件 mnist.py，文件内容如下。

```
#!/usr/bin/env python2.7

import os
import sys
import base64
import urllib
import tensorflow as tf
import numpy as np
import utils
import json

os.environ['TF_CPP_MIN_LOG_LEVEL'] = '2'

# Load model
cur_dir = os.getcwd()
model_dir = cur_dir+"/export/4"
sess = tf.Session(graph=tf.Graph())
meta_graph_def = tf.saved_model.loader.load(sess, [tf.saved_model.tag_constants.SERVING], model_dir)
x = sess.graph.get_tensor_by_name('x:0')
y = sess.graph.get_tensor_by_name('y:0')

def run(event):
    local_path = ""
    if event['image_base64'] != "":
        data=base64.b64decode(event['image_base64'])
        local_path = '/tmp/test_image.xxx'
        file=open(local_path,'wb')
        file.write(data)
        file.close()
    elif event['image_url'] != "":
        img_url = event['image_url']
        filename = urllib.unquote(img_url).decode('utf8').split('/')[-1]
        local_path = os.path.join('/tmp/', filename)
        if not utils.download_image(event['image_url'], local_path):
            return False, -1
    else:
        print('Please specify a image.')

    try:
        x_ = utils.get_image_array(local_path)
        predict = tf.argmax(y, 1)
        res = sess.run(predict, feed_dict={x: x_})[0] 
        return True, sess.run(predict, feed_dict={x: x_})[0]
    except Exception as e:
        print(e)
        return False, -1

def demo(event, context):
    res, num = run(event)
    print num
    return num
        
def apigw_interface(event, context):
    if 'requestContext' not in event.keys():
        return {"errMsg":"request not from api gw"}
    body_str = event['body']
    body_info = json.loads(body_str)
    res, num = run(body_info)
    return {"result":num}

```

从代码中我们可以看到，函数在初始化时就将目录 export 下的文件作为模型加载到了 TensorFlow 中。在实际的事件处理中，既可以从事件中抽取 base64 编码后的图片，也可以识别 url 参数，并均把图片保存至本地 /tmp 目录下。然后在使用 util 工具，对图片进行规整处理后，将处理后的数据送入 TensorFlow，获得推理结果并返回。

在根目录下同时创建 util.py，代码内容如下。此模块提供了图片下载、图片处理等辅助能力。

```
#!/usr/bin/env python2.7

import urllib2
import numpy as np
from PIL import Image

def download_image(img_url, local_file):
    ret = True
    try: 
      f = urllib2.urlopen(img_url)
      data = f.read()
      with open(local_file, "wb") as img:
          img.write(data)
    except Exception as e:
        print(e)
        ret = False
    return ret

def get_image_array(img_path):
    x_s = 28
    y_s = 28
    n0 = 0
    n255 = 0
    threshold = 100
    im = None
    im = Image.open(img_path)

    img = np.array(im.resize((x_s, y_s), Image.ANTIALIAS).convert('L'))

    for x in range(x_s):
      for y in range(y_s):
        if img[x][y] > threshold:
          n255 = n255 + 1
        else:
          n0 = n0 + 1

    if(n255 > n0) :
      for x in range(x_s):
        for y in range(y_s):
          img[x][y] = 255 - img[x][y]
          if(img[x][y] < threshold) :
            img[x][y] = 0

    arr = img.reshape((1, 784))
    arr = arr.astype(np.float32)
    arr = np.multiply(arr, 1.0 / 255.0)
    return arr
```

同时由于 util.py 中使用了 PIL 库，需要在代码根目录下提供 PIL 库，可以使用此 [PIL库文件](https://main.qcloudimg.com/raw/7e394fa1410e615da121dd5748d8724d.zip) 来下载库并解压后放置在代码根目录。

### 准备函数部署包

最终，我们得到的代码目录结构为如下结构，其中PIL文件夹下由于文件过多就不进行展开了。

```
mnist_demo
|
|-- mnist.py
|-- utils.py
| export
    | 4
        |-- saved_model.pb
        | variables
            |-- variables.data-00000-of-00001
            |-- variables.index
| PIL
    |-- ...
```

在 mnist_demo 这个目录下，我们选择所有文件然后打包为 zip 包。注意，这些文件需要在 zip 包的根目录下，而不是 mnist_demo 文件夹在zip包的根目录。

最终我们得到了一个可以上传到云函数的 zip 包。如果对于前面的操作都不想进行，也可以直接在[这里](https://main.qcloudimg.com/raw/4d3540e468b3caa757429f85cd9c0915.zip)下载已经打好的包即可。


### 创建函数

由于创建的函数部署包稍大，所以我们需要通过对象存储来上传代码包。

我们可以在腾讯云对象存储 COS 中先创建一个 bucket，例如在广州区创建名为 code 的 bucket，并将上一步获取的代码包上传 bucket，作为我们后续创建函数的代码来源。

进入腾讯云无服务器云函数 SCF 的[控制台](https://console.cloud.tencent.com/scf/list)，选择广州区以后，点击新建函数，为函数起一个比较容易记住的名字，例如 testai，选择运行环境为 Python 2.7，然后下一步到代码配置页面。

在代码配置页面，选择代码输入种类为 `通过 COS 上传 zip 包`，选择刚刚创建的bucket为 cos，并填写对象文件为 `/mnist_demo.zip`。

同时，函数执行方法需要确定为 **mnist.apigw_interface**，对应代码包中的 mnist 文件和 apigw_interface 函数。

### 测试函数

点击函数界面右上角的测试按钮，并使用如下测试模版来测试函数。

```
{
  "requestContext": {
    "serviceName": "testsvc",
    "path": "/test/{path}",
    "httpMethod": "POST",
    "requestId": "c6af9ac6-7b61-11e6-9a41-93e8deadbeef",
    "identity": {
      "secretId": "abdcdxxxxxxxsdfs"
    },
    "sourceIp": "10.0.2.14",
    "stage": "prod"
  },
  "headers": {
    "Accept-Language": "en-US,en,cn",
    "Accept": "text/html,application/xml,application/json",
    "Host": "service-3ei3tii4-251000691.ap-guangzhou.apigateway.myqloud.com",
    "User-Agent": "User Agent String"
  },
  "body": "{\"image_base64\": \"\", \"image_url\": \"https://main.qcloudimg.com/raw/84783c178cdc6d6b2302bc1b4749b91b.bmp\"}",
  "pathParameters": {
    "path": "value"
  },
  "queryStringParameters": {
    "foo": "bar"
  },
  "headerParameters":{
    "Refer": "10.0.2.14"
  },
  "stageVariables": {
    "stage": "test"
  },
  "path": "/test/value?foo=bar",
  "httpMethod": "POST"
}
```

这里可以看到，我们主要使用了body字段，并在body字段内填写的数据结构为：

```
{
  "image_base64": "",
  "image_url": "https://main.qcloudimg.com/raw/84783c178cdc6d6b2302bc1b4749b91b.bmp"
}
```

这个数据结构也是我们创建的函数所能接受和处理的结构，如果有 base64 编码的图片文件内容，则使用编码的内容，或者使用url传入的图片地址，将图片下载到本地后交由 TensorFlow 进行预测推理。在这里测试时，我们可以用上我们在一开始准备的图片地址，而在实际推理时，可以替换成其他可访问的图片地址。

点击测试运行后，如果使用的是上面的地址，使用的是第一副图片，那么函数的返回内容将是 `{"result": 0}`，标识其推理出来的图片内是数值 0。

## 使用 API 网关进行 API 封装

接下来我们通过 API 网关服务，来创建一个 API 对刚刚创建的推理函数进行封装，并对外提供 API 服务。

### 创建 API 服务及 API

首先在 API 网关的控制台，在广州区创建一个 API 服务。服务名可以起一个容易记住的名字，例如 testai。

接着进入API 管理，创建 API。同样可以起一个容易记住的名字，例如 ai。请求路径可以写为 /ai，请求方法为 POST。为了方便调试，我们这里可以勾选上免鉴权。

接着下一步，后端类型选择为 cloud function，并选择我们在前面创建的函数 testai。

最后一步填入响应内容为 JSON，描述正确示例为 `{"result":0}`，错误示例为 `{"errMsg":"error info"}` 即可。

### 调试 API

点击 API 查看界面的 API 调试，进入调试页面。确定 Content-Type 为 application/json，输入框内填入以下内容后点击发送请求。

```
{
  "image_base64": "",
  "image_url": "https://main.qcloudimg.com/raw/84783c178cdc6d6b2302bc1b4749b91b.bmp"
}
```

确认响应的 body 为 {"result": 0}，符合预期。同理，这里的 url 地址同样可以更改，并且响应内容随图片的不同而不同。

### API 发布及外网测试

在 API 调试成功后，我们可以在服务列表页面，将我们刚刚创建的 testai 服务发布到 `发布环境`。

然后根据 testai 的服务域名，我们可以得到刚才的 API 的完整路径为：`http://service-kzeyrb6x-1253970226.ap-beijing.apigateway.myqcloud.com/release/ai`。我们可以使用 http request 的发起工具，例如 Postman，或 restclient 等，向此 API 地址发起 POST请求，POST内容可以为如下内容。


```
{
  "image_base64": "",
  "image_url": "https://main.qcloudimg.com/raw/84783c178cdc6d6b2302bc1b4749b91b.bmp"
}
```

或

```
{
  "image_base64": "Qk1mCQAAAAAAADYAAAAoAAAAHAAAABwAAAABABgAAAAAADAJAAAAAAAAAAAAAAAAAAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////AAAA////////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////AAAAAAAA////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////",
  "image_url": ""
}
```

这两个不同的数据结构，分别测试了使用 base64 编码的图片内容，或者使用图片 url 地址传递图片内容的方式。同时可以根据自身需求，修改数据结构内的 image_base64 或 image_url 内容，查看测试结果。

## 总结

我们在这里，通过一段实际的代码，及 TensorFlow 训练出来的模型，实现了利用 Serverless 架构进行 AI 推理，并通过 API 网关的封装，使得 AI 能力能够以 API 的形式对外暴露。可以看到，在获取到 TensorFlow 训练出的模型后，可以通过简单的函数包装，立即开始对外提供预测推理服务，而无需准备服务器，配置 web server 等繁琐配置准备操作。同时，在训练模型调优后，可以通过简单的升级函数，或发布新的函数版本，完成模型更新，而对用户提供无感知的升级。

Serverless 以按需调用，按运行计费的方式，使得 AI 推理服务的提供成本可以大幅降低，供应速度进一步提升，可以使得更多的 AI 能力能尽快服务于客户。

同时，目前上面提供的 AI 推理，由于比较简单，并无需使用 GPU。而在模型较复杂，计算量较大的情况下，使用 GPU 将能进一步加速推理速度。GPU 的使用，可以为 AI 推理的速度带来数量级的加速，将有些需要使用 CPU 秒级的推理，降低到使用 GPU 的10ms级。目前腾讯云云函数已经灰度开放了 GPU 支持能力，欢迎大家递交工单或提供联系方式，申请试用。

