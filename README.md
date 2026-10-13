# LogMinify / Denum 可执行文件

x86-64 Linux 预编译文件，在 Ubuntu 20.04（gcc 9.4）上构建。

| 文件 | 说明 | 来源 |
|---|---|---|
| `bin/denum_compress` | Denum 原版压缩器（固定 4 线程） | `Denum/denum_compress.cpp` |
| `bin/logminify` | LogMinify 压缩器；ART 版和 RSSPDC 版是同一个文件，用环境变量 `LOGMINIFY_INDEX` 选择 | LogMinify 分支 `exp/rsspdc-index`，提交 `50c6250` |
| `bin/decompress_logminify` | LogMinify 解压器（ART 和 RSSPDC 的输出都能解） | 同上 |

SHA-256：

```
c68648814d7ea3630d8e91e25bee298cfb15bc2333a0fdbb6b950afeb74aa34e  denum_compress
fa52214fb07f86a19cdaecce2d52e247b3a54194535c3aaf6f8b946bca75d41a  logminify
61ce1ba77e54e00483aa5ee1029b37a5288dad3ff84ae74e2cbe8549f0df5311  decompress_logminify
```

## 运行依赖

```bash
apt install -y libpcre2-8-0 libjemalloc2 libtbb2 tar xz-utils
```

- `logminify` 还需要 Intel MKL 的 `libmkl_rt.so`（构建时位于 `/opt/intel/mkl/lib/intel64/`）。运行前设置 `export LD_LIBRARY_PATH=/opt/intel/mkl/lib/intel64:$LD_LIBRARY_PATH`。
- 每个块都调用外部的 `tar` 和 `lzma -6` 压缩，所以 `lzma` 命令必须在 `PATH` 里（`xz-utils` 提供）。
- 可用 `ldd bin/logminify` 检查是否缺库。

## 目录约定

三个程序都在**当前工作目录**下读写，`<name>` 是数据集名：

```
./Logs/<name>/<name>.log   输入日志
./output/                  压缩结果（output/<name>/）
./temp/                    临时文件
./info.txt                 追加写入的 CR、CS 等统计
```

运行前先建好目录：

```bash
mkdir -p Logs/<name> output temp
cp /path/to/<name>.log Logs/<name>/
```

> 注意：每次压缩都会先清空 `output/<name>/`。

## 执行指令

块大小统一用 100000 行。

### Denum 原版

```bash
./bin/denum_compress <name> 100000 1
```

第 3 个参数是模式：`1` 为默认 Denum；`2` 为 RQ3 生成去掉数字的日志；`3` 为 RQ4 去掉字符串处理的 Denum。

### LogMinify：ART 版

```bash
LOGMINIFY_INDEX=art LOGMINIFY_FG_THREADS=40 ./bin/logminify <name> 100000
```

### LogMinify：RSSPDC 版

```bash
LOGMINIFY_INDEX=rsspdc LOGMINIFY_FG_THREADS=40 ./bin/logminify <name> 100000
```

### LogMinify 解压

```bash
./bin/decompress_logminify <name>
# 输出：output/<name>_decompressed.log
```

解压器根据 `output/<name>/` 里的字典文件（`art.tar.xz` / `rsspdc.tar.xz`）自动判断是哪种索引，不需要再设置 `LOGMINIFY_INDEX`。

## LogMinify 环境变量

| 变量 | 取值 | 默认 | 说明 |
|---|---|---|---|
| `LOGMINIFY_INDEX` | `art`、`rsspdc`（另有 `sindex`、`hash`、`litmt` 用于对比实验） | `sindex` | 模板索引 |
| `LOGMINIFY_FG_THREADS` | 正整数 | — | worker 线程数，实验中用 40 |
| `LOGMINIFY_DICT_FORMAT` | `radix_fc_ids`、`radix_ids`、`radix` | `radix_fc_ids` | 字典输出格式，实验中都用默认值 `radix_fc_ids`（前缀压缩加独立 ID 文件） |

压缩结束后，CR 和 CS 打印在标准输出，同时追加到 `info.txt`。

## 示例：同一份数据跑三个版本

```bash
name=Apache
mkdir -p Logs/$name output temp && cp /data/$name.log Logs/$name/
./bin/denum_compress $name 100000 1
LOGMINIFY_INDEX=art    LOGMINIFY_FG_THREADS=40 ./bin/logminify $name 100000
LOGMINIFY_INDEX=rsspdc LOGMINIFY_FG_THREADS=40 ./bin/logminify $name 100000
./bin/decompress_logminify $name
# 输入末尾没有换行时，解压结果只会多一个结尾换行，其余逐字节一致
cmp <(head -c $(stat -c %s Logs/$name/$name.log) output/${name}_decompressed.log) Logs/$name/$name.log && echo OK
tail -20 info.txt
```
