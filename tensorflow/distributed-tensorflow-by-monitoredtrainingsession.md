# TensorFlow分布式训练（MonitoredTrainingSession）

## 写在前面
2017年11月初，TensorFlow官网给出了分布式训练的最新版本。主要的改变在于由之前的`tf.train.Supervisor()`变更为现在的`tf.train.MonitoredTrainingSession()`。于是，就之前分布式图像识别的代码我做了相应的改变与更新。最新代码开源在[https://github.com/hemajun815/svhn](https://github.com/hemajun815/svhn)上。

## 实践过程

### 创建集群
本次实验过程中，使用了3台设备（192.168.10.200，192.168.10.155，192.168.10.181）来搭建集群，将192.168.10.200作为Parameter Job（参数服务器），其余两台作为Worker Job（计算服务器）。构建集群的相关代码如下：
```python
# parameters of cluster
flags.DEFINE_string("ps_hosts","192.168.10.200:2222", "Comma-separated list of hostname:port pairs")
flags.DEFINE_string("worker_hosts", "192.168.10.155:2222, 192.168.10.181:2222", "Comma-separated list of hostname:port pairs")
flags.DEFINE_string("job_name", None, "Job name: worker or ps")
flags.DEFINE_integer("task_index", None, "Worker task index, should be >= 0.")

# construct the cluster
ps_spec = map(lambda str: str.strip(), FLAGS.ps_hosts.split(","))
worker_spec = map(lambda str: str.strip(), FLAGS.worker_hosts.split(","))
cluster = tf.train.ClusterSpec({ "ps": ps_spec, "worker": worker_spec})
```

### 定义Server
创建好了cluster，我们就需要在每台主机上定义各自的Server，同时指定此Task的Job_name和task_index，相关代码如下：
```python
# server
server = tf.train.Server(cluster, job_name=FLAGS.job_name, task_index=FLAGS.task_index)
```
server定义完成之后，因为Parameter Job不参与计算过程，所以它会直接结束，但是我们要让其一直处于监听状态，所以会在Parameter Job上做以下操作：
```python
if FLAGS.job_name == "ps":
    server.join()
```

### 构造模型
本次使用的模型没有修改，依然是之前SVHN的CNN模型：
```python
dnn = DNN()
dnn.define_inputs(input_samples_shape=input_samples_shape, input_labels_shape=input_labels_shape)
dnn.add_cnn_layer(name='conv1', patch_size=3, in_depth=input_samples_shape[3], out_depth=32)
dnn.add_cnn_layer(name='conv2', patch_size=3, in_depth=32, out_depth=64, pooling_scale=4, pooling_stride=4)
dnn.add_cnn_layer(name='conv3', patch_size=3, in_depth=64, out_depth=128)
dnn.add_cnn_layer(name='conv4', patch_size=3, in_depth=128, out_depth=256, pooling_scale=4, pooling_stride=4)
dnn.add_fc_layer(name='fc1', in_num_nodes=1024, out_num_nodes=128)
dnn.add_fc_layer(name='fc2', in_num_nodes=128, out_num_nodes=input_labels_shape[1], activation=None)
```

### 执行训练
与`tf.train.Supervisor()`一样，`tf.train.MonitoredTrainingSession()`也提供了一系列服务来帮助实现一个鲁棒的训练过程，并且就封装本身而言，对`tf.train.MonitoredTrainingSession()`的调用更加简单易懂，操作过程与单机调用更为相似。整个训练过程如下所示：
```python
time_start = dt.datetime.now()
print("Start training %d images at %s." % (train_samples.shape[0], time_start))
data = util.data_iterator(train_samples, train_labels, batch_size, iteration_steps + 1)
with tf.train.MonitoredTrainingSession(master=server.target, is_chief=(task_index == 0), checkpoint_dir=model_path, hooks=[tf.train.StopAtStepHook(last_step=iteration_steps)]) as mon_sess:
    while not mon_sess.should_stop():
        _, samples, labels = data.next()
        _, loss, accuracy, step = mon_sess.run([optimizer, cross_entropy, accuracy_op, global_step], feed_dict={self.input_samples: samples, self.input_labels: labels})
        if (step + 1) % display_delay == 0:
            print("Step %d: finished training %d images at %s. loss=%.9f, acc=%.6f" % ((step + 1), (step + 1) * batch_size, dt.datetime.now(), loss, accuracy))
time_end = dt.datetime.now()
elapsed = time_end - time_start
print("Finish training at %s. elapsed time %s." % (time_end, elapsed))
```

### 调用方式
- 192.168.10.200:2222: `python distributed_train.py --job_name=ps --task_index=0`
- 192.168.10.155:2222: `python distributed_train.py --job_name=worker --task_index=0`
- 192.168.10.155:2222: `python distributed_train.py --job_name=worker --task_index=1`

## 小结
- 就代码编写而言，TensorFlow给出的新的接口在调用时更加简洁易懂，编写程序更加简单易操作。
- 就运行结果而言，新借口在运行时间和最终结果都与之前的运行情况一致。
- 就运行过程而言，新接口在两台配置一样的工作机上出现了运行不均衡的情况，即其中一台的执行的任务量明显多于另一台的情况。原因还在进一步研究中。
- 就函数实现而言，函数的内部是否与之前的接口一致，还有待深究，暂不下定论。
