---
title: 'RAG知识库相关功能文档'
tags: RAG,知识库构建,爬虫
category: /FaceMind/文档/2024-04
emoji: "☺"
grammar_cjkRuby: true
---
本文档用于说明RAG（Retrieval Augmented Generation）系统中检索知识库相关构建流程。
注意，阅读本文档前请先完成RAG功能API文档的阅读。RAG系统架构如下图所示，本工作实现的是黄色部分的QK-Match System。
!whiteboard_exported_image](./images/whiteboard_exported_image.png)![whiteboard_exported_image](./images/whiteboard_exported_image_1.png)

## 1 代码结构与文件位置

## 2 论坛知识爬取
### 2.1 主要工作：
下述工作均实现于脚本文件：[faceMindTrendsCrawlUtils.py](https://github.com/FaceMindCodeBase/FaceMind_Trends_Backend/blob/crawling_time/utils/faceMindTrendsCrawlUtils.py)
 1. 添加功能：对发帖、评论时间的进行爬取并和内容互相配对
**函数原型**`  def fill_content_into_thread(thread: Thread, url: str, page: int, page_max: int,comment_idx:int=0): `
**参数说明**
* thread (Thread)：代码中定义的class Thread类型的实例
* url (str)：要进行爬取的详细帖子页面的url
* page(int)：当前所在的帖子内容的页面索引
* page_max(int)：对每个帖子爬取的最大页面数，因为每个帖子的评论内容显示不止一个页面
* comment_idx(int)：当前页面评论的起始索引
**返回值**
* None。该方法对Thread实例进行修改，将爬取的信息填充到Thread实例中。
 2. 添加功能：指定爬取的有效帖子数量
**函数原型**`  def crawl_and_rephrase(game: str, section: str, url: str,counts:int=1000, model: str="gpt-3.5-turbo", temperature: float=1.0, thread_list_pages: int=2, rephrase_attemps: int=3): `
**参数说明**
* game(str)：要爬取的NGA论坛的模块
* section(str)：指定模块下的细分模块，若为空则表示直接爬取主板信息
* url(str)：指定模块的url
* counts(int)：指定需要爬取的有效帖子数
* Others：在爬虫功能中属于无效参数，此处无需使用
**返回值**
* None。该方法将爬取的内容存储到项目结构中/cache/filter_txt和/cache/crawl_json中。其中crawl_json存储爬取的帖子信息，filter_txt存储对帖子内容进行过滤后的信息，过滤方法仅保留点赞数排名靠前的15条评论。
### 2.2 爬虫使用方法：
将[爬虫代码](https://github.com/FaceMindCodeBase/FaceMind_Trends_Backend/tree/crawling_time)clone下来之后运行faceMindTrendsCrawler.py即可。
## 3 基于LLM的知识提取
下述工作均实现于脚本[Generate_QAK.py](https://github.com/FaceMindCodeBase/FaceMind_QKMatch/blob/main/Generate_QAK.py)
### 3.1 主要工作
1. 基于LLM对爬取的论坛文本全部进行知识提取，从每个帖子中生成Question-Answer对和Knowledge，并分别存储于./QAK_txt/QA。
**函数原型**`def batch_generate_all(data_path,prompt_QAK_path):`
**参数说明**
* data_path(str)：该路径下存储论坛信息文本
* prompt_QAK_path(str)：该路径下存储用于生成QAK的prompt，指示模型生成问答对和知识
**返回值**
None。该方法将结合论坛文本+指令作为最终prompt，利用大模型进行论坛文本知识提取并存储。
2. 测试实验数据构造：对爬取的论坛文本文件进行分离，90%用于生成Knowledge，10%的文件分离出来存于./experiment_data/txt_for_QA，用于生成Qusetion-Answer对，将生成的QA对和K均存储于./experiment_data/QAK_txt中。
**函数原型**`def batch_generate(knowledge_path,QA_path,prompt_QA_path,prompt_K_path):`
**参数说明**
* knowledge_path(str)：该路径下存储用于提取知识的论坛信息文本
* QA_path(str)：该路径下存储用于提取问答对的论坛信息文本
* prompt_QA_path(str)：用于生成QA的prompt的路径，存储指令prompt指示模型生成QA
* prompt_K_path(str)：用于生成K的prompt的路径，存储指令prompt指示模型生成K
**返回值**
None。该方法将结合论坛文本+指令作为最终prompt，利用大模型从一批论坛文本提取知识，另一批文本提取问答对。
### 3.2 使用示例
1. 使用功能1，用同一批文本提取QAK：
```python
if __name__ == '__main__':
    openai.api_base = 'https://api.aigcbest.top/v1'
    openai.api_key = get_api_key('./api_key.txt')
    # txt_split('./filter_txt', 'experiment_data/txt_for_QA')
    # batch_generate("filter_txt", 'experiment_data/txt_for_QA', './prompt_QA.txt', './prompt_K.txt')
    batch_generate_all("genshin_data", 'prompt/prompt_QAK_genshin.txt')
```
2. 使用功能2，用从不同的文本中提取QA和K：
```python
if __name__ == '__main__':
    openai.api_base = 'https://api.aigcbest.top/v1'
    openai.api_key = get_api_key('./api_key.txt')
    txt_split('./filter_txt', 'experiment_data/txt_for_QA')
    batch_generate("filter_txt", 'experiment_data/txt_for_QA', './prompt_QA.txt', './prompt_K.txt')
    # batch_generate_all("genshin_data", 'prompt/prompt_QAK_genshin.txt')
```
## 4 Milvus数据库存储知识
进行下述工作前需要现在本地部署好Milvus或者其他服务器上有部署好的Milvus数据库，部署Milvus详情见[部署Milvus文档](https://iqp7kyu4j3n.feishu.cn/docx/VLjndGJzMomWhNxKqLEcjvjknud)。
### 4.1 主要工作
1. 初始化Milvus数据库，设置建表字段，并为知识向量建立索引以快速检索。
**函数原型**`def connect_local_milvus():`根据需求自行修改连接的ip地址和端口号，连接到数据库并创建知识数据库`knowledge`。
**函数原型**`def build_table_and_index():`创建知识库后设置知识库中的表格字段，为向量字段建立索引。
上述功能实现于脚本[FaceMind_QKMatch/milvus_operator
/init_local_milvus.py](https://github.com/FaceMindCodeBase/FaceMind_QKMatch/blob/main/milvus_operator/init_local_milvus.py)。
2. 实现对Milvus数据库的连接、添加数据、向量相似度检索等操作，此部分[RAG功能API文档](https://iqp7kyu4j3n.feishu.cn/docx/EugWdA4WuoapG6xy48pcYTVKnuh)中已有详细解释。由于本人工作中的Milvus数据库字段与上述文档中的不同，因此对其中向量数据库存储和检索代码进行了修改。该功能实现于脚本[FaceMind_QKMatch/milvus_operator
/retrieval_through_milvus.py](https://github.com/FaceMindCodeBase/FaceMind_QKMatch/blob/main/milvus_operator/retrieval_through_milvus.py)。
### 4.2 使用示例
```python
if __name__=='__main__':
    embedder=init_embedder()
   # 连接到Milvus数据库
   milvus=MilvusOperator(database_name='knowledge',collection_name='knowledge_from_NGA_forum',embedder=embedder)
    milvus_connect_config={
        'host':'192.168.31.22',
        'portal':19530
    }
    milvus.connect_to_milvus(milvus_connect_config=milvus_connect_config,verbose=True)
	# 这段注释的代码用于添加批量知识到Knowledge数据库中，只需要运行一次即可注释，不然会重复添加相同数据
    # with open("QAK_txt/Knowledge.txt", "r") as file:
    #     lines = file.readlines()
    # batch_knowledge=[]
    # for idx,line in enumerate(lines):
    #     batch_knowledge.append(line)
    # milvus.add_batch_knowledge(batch_knowledge)
	
	# 获取10条question，测试向量相似度检索功能
    with open('../experiment_data/QAK_txt/QA.txt', 'r')as file:
        lines=file.readlines()
    questions=[]
    for idx,line in enumerate(lines):
        if line.startswith('Q:'):
            questions.append(line)
        if idx>10:
            break
    for question in questions:
        print(question)
        retrieval_res=milvus.retrieval_topk(question,3)
        print(retrieval_res.split('\n'))
        assembled_prompt="".join(res for res in retrieval_res)
```
## 5 检索知识增强生成
### 5.1 主要工作
1. 根据用户的query对知识库中的knowledge进行相似性匹配，返回相似度最高的top_k条作为先验知识，与query一块整合成prompt给LLM返回answer。
**函数原型**`def retrieval_augmented_generation(queries,milvus):`
**参数说明**
* queries(list)：用户的query。
* milvus(MilvusOperator)：类MilvusOperator实例，用于操作数据库。
**返回值**
None。该方法获取LLM的回答后将问答对存储到./experiment_data/QAK_txt/QA_by_RAG.txt。
2. 对比实验：直接把用户的query输入给LLM，没有先验知识，仅依据LLM本身进行回答。
**函数原型**`def generate_answer_without_knowledge(queries):`
**参数说明**
* queries(list)：用户的query
**返回值**
None。该方法获取LLM的回答后将问答对存储到./experiment_data/QAK_txt/QA_by_Generation.txt。
### 5.2 使用示例
```python
if __name__=='__main__':
	# openai连接配置
    openai.api_base = 'https://api.aigcbest.top/v1'
    openai.api_key = get_api_key('./api_key.txt')
	# 获取词嵌入模型
    embedder = init_embedder()
	# 获取数据库操作器
    milvus = get_milvus('knowledge', 'knowledge_from_NGA_forum', embedder)
	# 获取用户queries
    queries=get_queries()
	# 分别进行检索增强生成回答和直接生成回答
    retrieval_augmented_generation(queries,milvus)
    generate_answer_without_knowledge(queries)
```