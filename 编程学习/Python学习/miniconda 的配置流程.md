---
created: 2026-08-12T10:17
---


---
>  miniconda 是一种更加高效的解释器配置方法，搭配编辑器可以无缝切换对应的环境


#### 先在VS Code 中安装Microsoft Python 扩展

#### 下载安装脚本

`curl https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-MacOSX-arm64.sh -o miniconda.sh`

#### 执行安装

	bash miniconda.sh

安装按回车/yes 默认安装即可

#### 检查是否安装好

	conda -V 
	python --version

#### 配置国内源

```
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
conda config --set show_channel_urls yes
```


---

#### 创建分支环境

	conda create -n lang python=3.11 -y

#### 激活当前环境

	conda activate lang


---

#### 安装相关依赖

```
# conda装底层torch、numpy
conda install pytorch torchvision numpy -y

# pip安装langgraph等框架
pip install langgraph langchain langchain-openai chromadb
```


#### 后续使用

创建分支然后切换当前环境即可




