# MNBVC_ARXIV_IMAGE2CAPTION

## 处理图文对
### 最终代码
```shell
# step1 处理不带后缀的文件
# 由于通过爬虫爬取下来的文件不带有后缀名，所以需要预处理为带有后缀名的文件，具体做法为检测文件类型，然后将其复制到另一份文件夹，同时将
# 压缩文件路径写入到success_text_file_list.txt 中待处理，失败文件写入 failed_text_file_list.txt

python src/detect_file_type.py --dir xxx 下载目录

# step2 处理后缀文件
python src/arxiv_main.py --input_file success_text_file_list.txt # 批量处理file.txt
# 代码效果，会将多种图片路径和写法提取出来、校验图片路径，读取图片，转为二进制写入parquet，一个arxiv一个parquet
```
```python
# 主要传入参数
parser.add_argument("--input_file", "-i", type=str, default="list.txt", help="Input file")
parser.add_argument("--workers_num", "-w", type=int, default=2, help="multi process workers num")
parser.add_argument("--output_dir", "-o", type=str, default=r"data_output/data-image-parquet", help="Output directory")
parser.add_argument("--log_dir", "-l", type=str, default="logs", help="Log path")

# 处理单个文件的函数
process_list_of_arxiv_files(file_list: List)
```

### Other Notes
1. 文件存储路径需要定义，默认存储在 data/tex_source_parquet中
2. arxiv_main.py中需要传入file_list


## 处理公式和表格

### 最终代码
```shell
python src/arxiv_main_tableandequation.py # 批量处理file.txt
# 代码效果：会将每个arxiv的公式和图片提取出来，并尝试转为图片，形成latex公式与图文对
```


## 批量离散parquet合并为一个parquet
```shell
python src/concat_parquet.py # 批量处理parquet 文件
# 代码效果，会将n个parquet 合并为一个parquet 减少文件碎片
```



## TODO

更好的注释文件和README