## ParlAI安装手册

### 一、安装准备

```less
使用python3.8版本
在进行安装时，关闭VPN

安装文档：https://parl.ai/docs/tutorial_quick.html
		 https://parl.ai/

git clone https://github.com/facebookresearch/ParlAI.git ~/ParlAI
pip install parlai
cd ~/ParlAI
python setup.py develop
# 相关错误可以查看下面的错误处理

开始执行：parlai display_data --task babi:task10k:1
```



### 二、安装历程  <安装时关闭VPN>

错误1：解决问题：no model named pywintypes

```less
https://www.lfd.uci.edu/~gohlke/pythonlibs/#pywin32下载对应38版本安装
pip install ***.whl

重新启动新cmd运行即可
```



错误2：error: Microsoft Visual C++ 14.0 is required

```less
根据报错下载安装vs statio下的对应c++插件
重启cmd即可
```



错误2：uth-8解码错误

![image-20230228103443782](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20230228103443782.png)

```python
# 对字符解析加入'ignore'参数
#根据目录文件对C:\Users\月久矢\AppData\Local\Programs\Python\Python38\Lib\tokenize.py修改
def find_cookie(line):
        try:
            # Decode as UTF-8. Either the line is an encoding declaration,
            # in which case it should be pure ASCII, or it must be UTF-8
            # per default encoding.
            line_string = line.decode('utf-8','ignore')

#根据目录文件对codecs.py修改
def decode(self, input, final=False):
        # decode input (taking the buffer into account)
        data = self.buffer + input
        (result, consumed) = self._buffer_decode(data, 'ignore', final)
        # keep undecoded input until the next call
        self.buffer = data[consumed:]
        return result
    

    
```

错误3：AttributeError: module 'parlai.tasks.babi.agents' has no attribute 'create_agents'

![image-20230228103827888](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20230228103827888.png)

![image-20230228105710639](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20230228105710639.png)

```less
# https://zeromq.org/languages/python/
pip install pyzmq
# 出现超时错误
需要关闭VPN后进行命令执行
# 如果持续下载超时中断，可以切换网络尝试（热点）
```





### 三、查看任务并训练模型

让我们从打印出 bAbI 任务的前几个示例

```less
# display examples from bAbI 10k task 1
parlai display_data --task babi:task10k:1
```

现在让我们尝试在其上训练一个模型（即使在您的笔记本电脑上，这也应该训练得很快）。

```less
# train MemNN using batch size 1 and for 5 epochs
parlai train_model --task babi:task10k:1 --model-file /tmp/babi_memnn --batchsize 1 --num-epochs 5 --model memnn --no-cuda
```

让我们打印它的一些预测以确保它正常工作。

```less
# display predictions for model save at specified file on bAbI task 1
parlai display_model --task babi:task10k:1 --model-file /tmp/babi_memnn --eval-candidates vocab
```

“eval_labels”和“MemNN”行应该（通常）匹配！

让我们试着自己问模型一个问题。

```less
# interact with saved model
parlai interactive --model-file /tmp/babi_memnn --eval-candidates vocab
...
Enter your message: John went to the hallway.\n Where is John?  //目前只针对该问题能答对
```

运行上面的语句时报错

![image-20230228152017089](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20230228152017089.png)

最终发现是对应的 /tmp/babi_memnn.cands-babi:task10k:1.cands 文件名在windows中是违法的。所以我们需要修改文件命名规则

修改文件命名：

```python
# path = self.opt['model_file'] + '.cands-' + self.opt['task'] + '.cands'
        path = self.opt['model_file'] + '.cands'
```

![image-20230228152441089](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20230228152441089.png)



### 四、在 Twitter 上训练Transformer 

现在让我们尝试训练一个 Transformer (Vaswani, et al 2017) 排序器模型。 *确保在安装了 PyTorch 的 GPU 上完成此部分。*

我们将在 Twitter 任务上进行训练，这是一个推文和回复的数据集。这些文档中有更多关于任务的信息，包括完整的[任务](http://parl.ai/docs/tasks.html)列表和 关于指定训练和评估参数（如此处 使用的参数）的说明。让我们重新开始打印前几个示例。

```less
# display first examples from twitter dataset
parlai display_data --task twitter
```

在开始新的训练集时，我们可以通过命令知道其下载的url。因为下载总是出错或者很慢，所以我直接vpn下载后放在目标文件下，并且通过加入以下代码以跳过下载。

![image-20230228155504199](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20230228155504199.png)

成功：

![image-20230228155644914](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20230228155644914.png)

现在，我们将训练模型。这将需要一段时间才能达到收敛。

```less
# train transformer ranker
parlai train_model --task twitter --model-file /tmp/tr_twitter --model transformer/ranker --batchsize 16 --validation-every-n-secs 3600 --candidates batch --eval-candidates batch --data-parallel True
```

您可以修改我们在这里使用的一些命令行参数——我们将批量大小设置为 10，每 3600 秒运行一次验证，并从批次中提取候选人进行训练和评估。

到目前为止，训练模型脚本将在获得最佳验证结果后默认保存模型。Twitter 任务非常大，默认情况下在每个 epoch 之后运行验证（完全通过训练数据），但我们想要更频繁地保存我们的模型，所以我们可以将验证设置为每小时运行一次，使用.

```less
-vtim 3600
```

这个训练模型脚本在训练结束时，评估有效集和测试集上的模型，但是如果我们想评估一个保存的模型——也许是将我们新训练的 Transformer 的结果与我们Model Zoo的 BlenderBot 90M 基线进行比较，我们可以执行以下操作：

比较链接：https://parl.ai/docs/zoo.html

```less
# Evaluate the tiny BlenderBot model on twitter data
parlai eval_model --task twitter --model-file zoo:blender/blender_90M/model
```

最后，让我们使用与上面相同的 display_model 脚本打印我们的一些 transformer 预测。

```less
# display predictions for model saved at specific file on twitter
parlai display_model --task twitter --model-file /tmp/tr_twitter --eval-candidates batch
```

```less
#最终因为内存不够报错
parlai interactive --model-file /tmp/tr_twitter --eval-candidates batch
```

