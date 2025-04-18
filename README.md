# ArXiv 图文提取

本项目旨在处理 ArXiv 论文，从 LaTeX 文件中提取图像-文本对、表格和公式。系统处理压缩的 ArXiv 源文件，提取 LaTeX 内容，识别图形、表格和公式

## 系统概述

该流程提取三种主要内容：
1. **图像-文本对**：论文中的图形及其对应的caption
2. **表格**：保留结构的结构化表格数据
3. **公式**：渲染为图像的数学表达式

提取的数据存储在 Parquet 文件中，可以合并为更大的Parquet数据

## 文件结构说明

- `src/detect_file_type.py`：检测文件类型并添加适当的扩展名
- `src/arxiv_main.py`：从 ArXiv 论文中提取图像-文本对
- `src/arxiv_main_tableandequation.py`：从 ArXiv 论文中提取表格和公式
- `src/concat_parquet.py`：将多个 Parquet 文件合并为更大的数据集
- `src/daily_tools.py`：用于文件操作和日志记录的实用函数
- `src/latex2pdf2image.py`：通过 PDF 将 LaTeX 代码转换为图像
- `src/mmdata.py`：用于多模态内容表示的数据结构

## 安装要求

依赖项：
- Python 3.x
- pdflatex（用于渲染 LaTeX）[mac安装方法](https://blog.csdn.net/eloudy/article/details/135718710) [linux安装](https://blog.csdn.net/qq_15557299/article/details/128276620)
- pdfcrop（用于裁剪 PDF）
- pdf2image（用于将 PDF 转换为图像）
- pandas（用于数据处理）
- pyarrow（用于处理 Parquet 文件）
- python-magic（用于文件类型检测）
- chardet（用于编码检测）
- PyCryptodomex（用于 SHA1 哈希）

## 流程执行

执行脚本 `arxiv_imagetext_pair.sh`

### 步骤 1：文件类型检测和准备

```bash
mkdir resource_tmp
python3 src/detect_file_type.py --dir ./data
```

该步骤处理没有后缀名的文件。由于通过爬虫爬取的文件可能没有后缀名，脚本会检测文件类型，并将其复制到另一个文件夹，压缩文件路径写入到image2caption_processed_file.txt 中待处理，存在直接解压后就是.tex文件，这些文件将保存为第二部分，tableequation_processed_file.txt,以供提取表格数据，因为只有一个tex文件，就一定不存在图caption pair数据 未处理文件保存在unprocessed_text.txt 中

参数说明：
- `--dir`：源文件目录路径
- `--image2caption_processed_file`：图文对处理文件列表（默认：`resource_tmp/image2caption_processed_file.txt`）
- `--tableequation_processed_file`：表格和公式处理文件列表（默认：`resource_tmp/tableequation_processed_file.txt`）
- `--unprocessed_text`：未处理文件列表（默认：`resource_tmp/unprocessed_text_file_list.txt`）
- `--failed_text`：处理失败文件列表（默认：`resource_tmp/failed_text_file_list.txt`）
- `--new_file`：是否创建新文件（默认：True）

### 步骤 2：处理图文对数据

```bash
python3 src/arxiv_main.py --input_file resource_tmp/image2caption_processed_file.txt
```

该步骤处理图像和说明文字对。脚本解压缩文件，提取 LaTeX 代码，识别包含图形的环境（如 `figure` 环境），提取图像路径和说明文字，并将它们保存到 Parquet 文件中。

参数说明：
- `--input_file` 或 `-i`：输入文件列表路径
- `--output_dir` 或 `-o`：输出目录（默认：`data_output/data-image-parquet`）
- `--log_dir` 或 `-l`：日志目录（默认：`logs`）

### 步骤 3：处理表格和公式数据

```bash
python3 src/arxiv_main_tableandequation.py --input_file resource_tmp/tableequation_processed_file.txt
```

该步骤处理表格和数学公式。脚本识别 LaTeX 代码中的表格（`table` 环境）和公式（`equation`、`align` 环境），将它们渲染为图像，并保存到 Parquet 文件中。

参数说明：
- `--input_file` 或 `-i`：输入文件列表路径
- `--output_dir` 或 `-o`：输出目录（默认：`data_output/data-table-parquet`）
- `--log_dir` 或 `-l`：日志目录（默认：`logs`）

### 步骤 4：合并 Parquet 文件

```bash
python3 src/concat_parquet.py --input_dir data_output/data-image-parquet --output_dir data_output/data-image-total
python3 src/concat_parquet.py --input_dir data_output/data-table-parquet --output_dir data_output/data-table-total
```

该步骤将多个 Parquet 文件合并为更大的数据集，以便于后续处理和训练。

参数说明：
- `--input_dir` 或 `-i`：输入目录，包含要合并的 Parquet 文件
- `--output_dir` 或 `-o`：输出目录，用于保存合并后的 Parquet 文件
- `--log_dir` 或 `-l`：日志目录（默认：`logs`）
- `--num_of_file` 或 `-n`：每个输出文件合并的输入文件数量（默认：10000）

## 脚本详细说明

### detect_file_type.py

该脚本用于检测文件类型并添加适当的扩展名。它使用 `python-magic` 库来识别文件类型，并将文件复制到新目录，同时添加相应的扩展名。脚本还将文件分类为图文对处理、表格/公式处理或未处理。

主要功能：
- 检测文件类型（tar、gz、pdf、tex 等）
- 将文件复制到新目录并添加扩展名
- 生成处理文件列表

### arxiv_main.py

该脚本处理 ArXiv 论文的压缩源文件，提取图像和说明文字对。

主要功能：
- 解压缩 ArXiv 源文件（.tar、.tar.gz、.gz）
- 提取 LaTeX 代码
- 识别包含图形的环境（`figure`）
- 提取图像路径和说明文字
- 验证图像文件是否存在
- 将图像和说明文字对保存到 Parquet 文件中

关键函数：
- `extract_compress_file`：解压缩文件
- `extract_tex_code`：提取 LaTeX 代码
- `extract_images_code`：提取包含图形的环境
- `extract_image_path_and_caption`：提取图像路径和说明文字
- `process_a_compressed_file`：处理单个压缩文件

### arxiv_main_tableandequation.py

该脚本处理 ArXiv 论文的压缩源文件，提取表格和数学公式。

主要功能：
- 解压缩 ArXiv 源文件
- 提取 LaTeX 代码
- 识别表格（`table` 环境）和公式（`equation`、`align` 环境）
- 将表格和公式渲染为图像
- 将表格和公式保存到 Parquet 文件中

关键函数：
- `extract_tableandequation_code`：提取表格和公式环境
- `latex_to_image`：将 LaTeX 代码渲染为图像

### concat_parquet.py

该脚本将多个小的 Parquet 文件合并为更大的数据集。

主要功能：
- 读取多个 Parquet 文件
- 将它们合并为更大的数据集
- 保存合并后的数据集

关键函数：
- `concat_data`：合并多个 Parquet 文件

### daily_tools.py

用于文件操作和日志记录。

主要功能：
- 读写 JSON 文件
- 线程安全的文件操作
- 自动删除文件

关键函数：
- `lock_wraps`：线程锁装饰器
- `save_jsonl`：保存 JSON Lines 格式的文件
- `load_jsonl`：加载 JSON Lines 格式的文件

### latex2pdf2image.py

该脚本将 LaTeX 代码转换为图像。

主要功能：
- 创建临时 LaTeX 文件
- 使用 pdflatex 编译 LaTeX 代码为 PDF
- 使用 pdfcrop 裁剪 PDF
- 使用 pdf2image 将 PDF 转换为图像

关键函数：
- `latex_to_image`：将 LaTeX 代码转换为图像


## 数据输出

系统生成的数据保存在 Parquet 文件中，包含以下字段：
- 实体ID：论文标识符
- 块ID：内容块标识符
- 时间：处理时间戳
- 扩展字段：额外元数据
- 文本：文本内容（如图像说明文字）
- 图片：二进制图像数据
- 块类型：内容类型（"图片"或"文本"）

## 日志和错误处理

系统在处理过程中生成以下日志文件：
- `log_file_image2caption.log`：图像-文本对处理日志
- `spectial_file_log_image2caption.log`：特殊矢量图处理日志
- `log_file_tableequation.log`：表格和公式处理日志
- `spectial_file_log_tableequation.log`：特殊表格和公式处理日志
- `log_file_while_concat.log`：合并 Parquet 文件日志

## 使用示例

完整的处理流程可以通过执行 `arxiv_imagetext_pair.sh` 脚本完成：

```bash
bash arxiv_imagetext_pair.sh
```

也可以单独执行各个步骤：

```bash
# 步骤 1：检测文件类型
mkdir resource_tmp
python3 src/detect_file_type.py --dir ./data

# 步骤 2：处理图文对数据
python3 src/arxiv_main.py --input_file resource_tmp/image2caption_processed_file.txt

# 步骤 3：处理表格和公式数据
python3 src/arxiv_main_tableandequation.py --input_file resource_tmp/tableequation_processed_file.txt

# 步骤 4：合并 Parquet 文件
python3 src/concat_parquet.py --input_dir data_output/data-image-parquet --output_dir data_output/data-image-total
python3 src/concat_parquet.py --input_dir data_output/data-table-parquet --output_dir data_output/data-table-total
```

## 注意事项

1. 确保已安装所有依赖项（pdflatex、pdfcrop 等）
2. 表格和公式渲染可能需要一些时间，特别是对于复杂的 LaTeX 代码