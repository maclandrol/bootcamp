# 实验一：百万向量检索

本实验使用 SIFT1B 数据集 中的一百万条数据来测试 Milvus1.0 的性能和准确率。

## 1、准备测试数据和脚本

本实验所使用的原始数据集为 SIFT1B ，关于该数据集的详细信息请参考：[http://corpus-texmex.irisa.fr/](http://corpus-texmex.irisa.fr/)。在本次测试中，我们提取了原始数据集中的 100 万条数据。

经实测，以下配置可顺利完成实验：

| 组件          | 配置                |
| ------------------ | -------------------------- |
| 操作系统            | Ubuntu LTS 18.04 |
| CPU           | Intel Core i5-8250U           |
| GPU           | NVIDIA GeForce MX150, 2GB GDDR5  |
| GPU 驱动软件    | Driver 418.74 |
| 内存        | 8 GB DDR4          |
| 硬盘       | NVMe SSD 256 GB             |
| Milvus     |  1.0  |
| pymilvus    |   1.0.1    |

测试工具下载：
- 100 万测试数据集下载地址：链接：https://pan.baidu.com/s/1TSjGaAlQOQm3PpJfZ6dtPQ   提取码：2x5o
- 查询向量集下载地址：链接：https://pan.baidu.com/s/158QrpC71I4jhEwNG6rD-kQ   提取码： 0j3c
- 搜索结果对照 （ ground truth ）下载地址：链接：https://pan.baidu.com/s/1pyjNAUdXvxcJjvTmdzQgBw 
  提取码：1776
- 测试脚本下载路径：[/bootcamp/benchmark_test/scripts/](/benchmark_test/scripts/)

为方便存放测试数据和脚本，请创建名为 `milvus_sift1m` 的文件夹。利用前文提供的下载链接，将测试数据集下载到 `milvus_sift1m` 目录下：

- 测试数据集下载并解压完成之后，你将会看到一个名为 `bvecs_data` 的文件夹。该文件夹里面存放了 10 个 `npy` 文件，每个 `npy` 文件中存放了10 万条 的向量数据。
- 查询向量集下载并解压完成之后，你将会看到一个名为 `query_data` 的文件夹。该文件夹里面存放了一个 `query.npy` 文件，该文件里存放了10,000 条需要查询的向量。
- 对照数据下载并解压完成之后，是一个名为 `gnd` 的文件夹，该文件夹下有一个 `ground_truth_1M.txt` 的文本文件，该文件里存放的是查询向量集中的每条向量的 top 1000 相似向量的位置。
- 测试脚本会包含四个 Python 脚本 `main.py`、`milvus_toolkit.py`、`milvus_load.py`、`config.py`。



## 2、 配置 Milvus 参数

Milvus 可以根据数据分布和性能、准确性的要求灵活调整相关系统参数，以发挥产品的最佳性能。在此实验中，采用如下表所示的参数配置，就可以实现90%以上召回率。

配置文件： `/home/$USER/milvus/conf/server_config.yaml`

|         参数名称         | 推荐值 |
| ---------------------- | ---- |
| `cpu_cache_capacity` |   4   |
|    `gpu_resource_config`.`cache_capacity`    |   1    |
| `gpu_search_threshold` | 1001         |
| `search_resources`     | gpu0 |

关于参数设置的详细信息请参考[Milvus 配置](https://github.com/milvus-io/milvus/blob/1.0/core/conf/demo/server_config.yaml)。

其余参数保持默认即可。配置文件参数修改完毕后，重启 Milvus Docker 使配置生效。

```bash
$ docker restart <container id>
```

## 3、 建表并建立索引

建表之前，首先确认 Milvus 已经正常启动。（ Milvus 安装及启动方法参见：[Milvus1.0 快速上手](https://milvus.io/cn/docs/v1.0.0/milvus_docker-cpu.md) ）

>  测试之前请根据脚本[说明](/benchmark_test/scripts/README.md)修改相应参数！

进入 `milvus_sift1m` 目录，运行如下脚本在 Milvus 中创建集合并建立索引：

```shell
$ python3 main.py --collection ann_1m_sq8 --dim 128 -c
$ python3 main.py --collection ann_1m_sq8 --index sq8 --build 
```

运行上述命令后，会创建一个名为 ann_1m_sq8h 的集合，其索引类型为 IVF_SQ8。可通过如下命令查看该表的相关信息：

```shell
#查看库中有哪些表
$ python3 main.py --show
#查看表ann_1m_sq8h的行数
$ python3 main.py --collection ann_1m_sq8 --rows
#查看表ann_1m_sq8h的索引类型
$ python3 main.py --collection ann_1m_sq8 --describe_index
```

## 4、 数据导入

导入数据之前，确保已成功建立集合 ann_1m_sq8。

运行如下命令导入1,000,000行数据：

```bash
$ python3 main.py --collection=ann_1m_sq8 --load
```

数据导入过程中，可以看到本项目一次性导入一个文件的数据量。

上述过程完成之后，运行如下命令以查看 Milvus 表中的向量条数：

```bash
$ python3 main.py --collection=ann_1m_sq8 --rows
```

为了确保导入 Milvus 的数据已经全部建好索引，请进入  `/home/$USER/milvus/db` 目录，在终端输入如下命令：

```bash
$ sqlite3 meta.sqlite
```

进入交互式命令行之后，输入如下命令，检查向量数据表当前的状态：

```sqlite
sqlite> select * from TableFiles where table_id='ann_1m_sq8';
```

Milvus 会将一个向量数据表分成若干数据分片进行存储，因此查询命令会返回多条记录。其中第三列数字代表数据表采用的索引类型，数字 4 代表采用的是IVF_SQ8H索引。第五列数字代表索引构建的情况，当这列数字为 3 时，代表相应的数据表分片上的索引已构建完毕。如果某个分片上的索引还没有构建完成，可以再次手动为这个数据分片建立索引。

退出 sqlite 交互式命令行:

```
sqlite> .quit
```

由于本项目数据较小，未达到自动建立索引的阈值，所以需要再次建立索引。进入 `milvus_sift1m` 目录，运行如下脚本：

```bash
$ python3 main.py --collection=ann_1m_sq8 --index=sq8 --build 
```

手动建立索引后，再次进入 sqlite 交互界面，确认所有数据分片都已经建好索引。如果想了解其他列数据代表的含义，请进入  `/home/$USER/milvus/db` 目录，在 sqlite 交互界面输入如下命令进行查看。

```sqlite
$ sqlite3 meta.sqlite
sqlite>.schema
```

## 5、准确性测试

SIFT1m 提供了10,000条向量的查询向量集，并且对于每条查询向量都给出了该向量在不同规模数据集上的 top 1000 ground truth。因此，可以方便地对 Milvus 查询结果的准确率进行计算。准确率计算公式为：

准确率＝ ( Milvus 查询结果与 ground truth 一致的向量个数 ) / ( query_records 的向量个数 * top_k )

执行查询脚本

准确率测试之前，需要手动建立目录`recall_result/recall_compare_out`来保存测试结果，从 10,000 条查询向量中随机取出500条向量，查询这500条向量各自的 top1, top10, top100, top200。执行如下命令：

```bash
$ python3 main.py --collection=ann_1m_sq8 --search_param 128 --recall
```

(注：search_param代表nprobe值，nprobe 值影响这查询结果准确率和查询性能，nprobe 越大，准确率越高，性能越差。)

执行上述命令后，在`recall_result`文件夹下会生成一个`ann_sift1m_sq8_128_500_recall.txt`的文本文件，该文本文件中记录了500条向量各自对应的 top 200的id和distance, 第一列的序号表示查询的那一条向量。文本文件中每200行为一组，对应一个 query 的查询结果。同时在`recall_compare_out`文件下会生成多个文本，以`ann_sift1m_sq8_128_500_100`为例，该文本中记录了topk=100时，查询的500条向量的各自对应的准确率及总的平均准确率。

Milvus 查询准确率与搜索子空间（ nprobe 参数）有很大关系。本次测试中 nprobe 设置为64，Milvus 查询准确率可以达到 90% 以上。可以通过增大 nprobe 值来实现更高的准确率但同时也会降低 Milvus 的查询性能。

因此，需要结合实际数据分布和业务 SLA，调整搜索子空间大小以达到性能和准确性的平衡。

## 6、性能测试

为评估 Milvus 的查询性能，进入 `milvus_sift1m` 目录，运行如下脚本：

```bash
$ python3 main.py --collection=ann_1m_sq8 --search_param 128 --performance
```

运行结束后，将会生成一个名为 `performance` 的文件夹，在该文件夹下会有一个名为 `ann_1m_sq8h_128_output.csv` 的文件，该文件保存了各个 nq 值不同的topk值运行所耗时间。

- nq: 代表要查询的向量数
- topk: 代表的是查询某个向量的前 k 个相似的向量
- total_time: 代表的是查询 nq 个向量的前 k 个相似向量一共花费的时间，单位：秒
- avg_time: 代表的是查询一个向量的 topk 个相似向量的平均时间，单位：秒

**注意**

1. `milvus_toolkit.py` 中设置的待测试的 nq 为：1、50、100、150、200、250、300、350、400、450、500、550、600、650、700、750、800。对于每一个 nq，`milvus_toolkit.py` 设置的 topk 为：1、20、50、100、300、500、800、1000。
2. Milvus 启动后，进行第一次向量检索时，需要花部分时间加载数据到内存。
3. 如果两次测试间隔一段时间，Intel CPU可能降频至基础频率。性能测试时尽量连续运行测试案例。第一个测试案例可以运行两次，取第二次的运行时间。
