# Python操作知识图谱数据库
------
![](http://p20tr36iw.bkt.clouddn.com/graph.jpg)

## 1.安装Neo4J

### 官网下载Neo4J的zip包，然后解压，将neo4j_path/bin配入path中，进入bin目录运行
```python
neo4j.bat console
```
## 2.运行Neo4J
![](http://p20tr36iw.bkt.clouddn.com/neo4j.png)

### 浏览器输入:http://localhost:7474，初始用户名与密码均为neo4j

## 3.Python操作Neo4J
### 3.1 py2neo安装
```python
pip install py2neo
```
### 3.2 py2neo连接neo4j
```python
from py2neo import Graph
def __init__(self):
    # 建立连接
    link = Graph("http://localhost:7474", username="neo4j", password="0312")
    self.graph=link

```
### 3.3 py2neo清空数据库结点与边
```python
def clean_node(self):
    # 清空数据库
    self.graph.delete_all()
```
>注意：此时会发现Property Keys未删除，要想删除只有找到你的数据库data/graph.db里面全部删除掉才可以。
### 3.4 py2neo创建结点
>创建结点是会发现label需要传参，那么label到底是什么呢?在neo4j中不存在表的概念，可以把label当作表,
相当于在创建多个结点时，指定其为同一label，就类似于为这几个结点(关系型数据库中类似与字段)储存到一张表中。

为了更好的描述疾病、药物等的构建,参考以下ER图进行构建

![](http://p20tr36iw.bkt.clouddn.com/rela.png)

```python
from py2neo import Node
def create_node(self):
    # 疾病、临床表现、药物等结点定义
    for each_dis in dis_list:
        dis_node=Node(dis_label,name=each_dis)
        self.graph.create(dis_node)

    for each_cli in cli_list:
        cli_node = Node(cli_label, name=each_cli)
        self.graph.create(cli_node)

    for each_sdef in drug_list:
        drug_node = Node(dru_label, name=each_sdef)
        self.graph.create(drug_node)

    for each_sdef in sdef_list:
        sdef_node=Node(side_effect_label,name=each_sdef)
        self.graph.create(sdef_node)

    for each_zd in zd_method_list:
        zd_node=Node(diagnostic_label,name=each_zd)
        self.graph.create(zd_node)
```
### 3.5 py2neo创建关系

>一个难点：取结点操作

```python
# 取结点，使用find_one()方法，通过指定label，property_key, property_key获取相应的结点
hyp = self.graph.find_one(
  label=dis_label,
  property_key="name",
  property_key="高血压"
)

```
>结点关系方法封装

```python
from py2neo import Relationship
def create_Rel(self):
    """
    建立关系
    高血压疾病与临床表现之间的双向关系定义
    :return:
    """
    # 获取高血压与糖尿病结点，然后通过循环，建立这两个疾病与临床表现的关系
    hyp_node = self.graph.find_one(
        label=dis_label,
        property_key="name",
        property_value="高血压"
    )
    tnb_node = self.graph.find_one(
        label=dis_label,
        property_key="name",
        property_value="糖尿病"
    )
    # 建立疾病与临床表现的关系
    for cli_name in cli_list:
        cli_node = self.graph.find_one(
            label=cli_label,
            property_key="name",
            property_value=cli_name
        )
        hyp_to_cli = Relationship(hyp_node, '产生', cli_node)
        self.graph.create(hyp_to_cli)
        tnb_to_cli = Relationship(tnb_node, '产生', cli_node)
        self.graph.create(tnb_to_cli)
    # 建立疾病与诊断方法之间的关系
    for diag_name in zd_method_list:
        diag_node = self.graph.find_one(
            label=diagnostic_label,
            property_key="name",
            property_value=diag_name
        )
        if diag_name=="血糖" and diag_name=="血脂" and diag_name=="胆固醇":
            diag_to_dis = Relationship(diag_node, '辅助检查', tnb_node)
        else:
            diag_to_dis = Relationship(diag_node, '辅助检查', hyp_node)
        self.graph.create(diag_to_dis)
    # 建立疾病与药物关系
    for drug_name in drug_list:
        drug_node = self.graph.find_one(
            label=dru_label,
            property_key="name",
            property_value=drug_name
        )
        if drug_name=="胰岛素" or drug_name=="胰高血糖素":
            drug_to_disease=Relationship(drug_node,'治疗',tnb_node)
        else:
            drug_to_disease= Relationship(drug_node, '治疗', hyp_node)
        self.graph.create(drug_to_disease)

    # 建立药物与副作用之间的关系
    for drug_name in drug_list:
        drug_node = self.graph.find_one(
            label=dru_label,
            property_key="name",
            property_value=drug_name
        )
        for sdef_name in sdef_list:
            sdef_node = self.graph.find_one(
                label=side_effect_label,
                property_key="name",
                property_value=sdef_name
            )

            if drug_name == "利尿药" and sdef_name == "尿酸升高":
                drug_to_sdef = Relationship(drug_node, '引发', sdef_node)
                self.graph.create(drug_to_sdef)
            elif drug_name == "钙拮抗药" and sdef_name == "血钾降低":
                drug_to_sdef = Relationship(drug_node, '引发', sdef_node)
                self.graph.create(drug_to_sdef)
            elif drug_name == "胰岛素" and (sdef_name == "恶心" or sdef_name == "呕吐"):
                drug_to_sdef = Relationship(drug_node, '引发', sdef_node)
                self.graph.create(drug_to_sdef)
            elif drug_name == "胰高血糖素" and (sdef_name == "头晕" or sdef_name == "眼花"):
                drug_to_sdef = Relationship(drug_node, '引发', sdef_node)
                self.graph.create(drug_to_sdef)
```

### 3.6 调用

### 上述代码全部封装在createBHPData类中，需要实例化对象，然后调用相应方法。

````python
c=createBHPData()
c.clean_node()
c.create_node()
c.create_Rel()
```

>最后,刷新浏览器版neo4j，然后就可以看到自己的图了。
## 4.项目地址:[点击这里,欢迎Star](https://github.com/Light-City/PyToNeo4J)
