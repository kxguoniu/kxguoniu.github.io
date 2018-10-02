---
layout: post
title: Python实现多叉树
categories: Python
description: python多叉树
keywords: Python
---

## 节点数据结构

```python
class Node(object):
    '''
    节点数据信息
    '''
    def __init__(self, data = {}):
        self.id = data.get('id', '')                            #ID
        self.name = data.get('name', '')                        #姓名
        self.inter = data.get('inter', '')                      #姓名拼音
        self.sex = data.get('sex', '')                          #性别
        self.superior = data.get('superior', '')                #父节点
        self.subordinate = data.get('subordinate', [])          #子节点

    def __repr__(self):
        result = ''
        result += 'id: ' + str(self.id) + '\n'
        result += 'name: ' + self.name + '\n'
        result += 'inter: ' + self.inter + '\n'
        result += 'sex: ' + self.sex + '\n'
        result += 'superior: ' + self.superior + '\n'
        return result
```
## 树结构
```python
class Tree(object):
    def __init__(self):
        self.head = Node()
        self.node = Node()
        self.n = 0

    def append(self, data, name=None):
        '''
        添加节点到多叉树中
        '''
        pass

    def deep_find(self, name):
        '''
        多叉树深度优先查找
        非递归
        '''
        pass

    def save(self):
        '''
        把多叉树中的所有节点保存到文件
        '''
        pass

    def read(self):
        '''
        读取文件中的数据到多叉树
        '''
        pass
```
### 添加
```python
def append(self, data, name=None):
        '''
        添加节点到多叉树中
        '''
        if not isinstance(data, Node):
            return False
        if name == None:
            return 'append(data, name=None) 缺少name'
        if name == 'admin':
            data.superior = 'admin'
            self.head = data
            self.node = data
            return '添加头结点成功'
        result = self.deep_find(name)
        if not result:
            return 'name 节点不存在'
        if not data.superior:
            data.superior = name
        else:
            if data.superior != name:
                return '父节点不一致'
        result.subordinate.append(data)
        self.node = data
        return '添加成功'
```
### 查找
```python
def deep_find(self, name):
        '''
        多叉树深度优先查找
        非递归
        '''
        node = self.head
        if node.name == name:
            return node
        stack_list = []
        visited = []
        stack_list.append(node)
        visited.append(node)
        while len(stack_list) > 0:
            x = stack_list[-1]
            for w in x.subordinate:
                if not w in visited:
                    if w.name == name:
                        self.node = w
                        return w
                    visited.append(w)
                    stack_list.append(w)
                    break
            if stack_list[-1] == x:
                stack_list.pop()
        return False
```
### 保存
```python
def save(self):
        '''
        把多叉树中的所有节点保存到文件
        '''
        stack_list = [self.head]
        with open('niu.csv', 'w', encoding='utf-8', newline='') as csvfile:
            fieldnames = ['id', 'name', 'inter', 'sex', 'superior', 'subordinate']
            writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
            writer.writeheader()
            while len(stack_list) > 0:
                node = stack_list[0]
                lists1 = [i.name for i in node.subordinate]
                lists2 = [i for i in node.subordinate]
                data = {'id': node.id,
                        'name': node.name,
                        'inter': node.inter,
                        'sex': node.sex,
                        'superior': node.superior,
                        'subordinate': lists1,
                        }
                writer.writerow(data)
                stack_list.extend(lists2)
                stack_list.pop(0)
```
### 读取
```python
def read(self):
        '''
        读取文件中的数据到多叉树
        '''
        t = Tree()
        with open('database.csv', 'r', encoding='utf-8') as csvfile:
            csvfile.readline()
            reader = csv.reader(csvfile)
            print(reader)
            read_list = []
            for row in reader:
                data = {'id': row[0],
                        'name': row[1],
                        'inter': row[2],
                        'sex': row[3],
                        'superior': row[4],
                        'subordinate': [],}
                node = Node(data)
                if len(read_list) == 0:
                    t.append(node, node.superior)
                    read_list.append(node)
                else:
                    head = read_list[0]
                    while node.superior != head.name:
                        read_list.pop(0)
                        head = read_list[0]
                    head.subordinate.append(node)
                    read_list.append(node)
        return t
```