---
layout:     post
title:      "将数组改写成树结构"
subtitle:   "使用js"
date:       2023-04-19
author:     " Shallow Dreamer "
header-img: "img/post-bg-js-version.jpg"
tags:
    - js
---

首先创建了一个空的根节点`root`，并将它的名称设置为空字符串，它的子节点数组设置为空数组。然后遍历整个数组，对于每个元素，在树形结构中创建对应的节点。遍历每个元素的名称，并通过比较父节点和子节点的名称来连接它们。如果节点不存在，则创建一个新节点并将它添加到父节点的子节点数组中。最后，返回整个树形结构的根节点的子节点数组。

代码：

```js
function arrayToTree(array) {
    // 创建一个空的根节点
    const root = { name: '', children: [] };
    
    // 遍历数组中的每个元素
    array.forEach(item => {
      let currentNode = root;
      
      // 遍历每个元素的名称
      item.forEach(name => {
        let childNode = currentNode.children.find(child => child.name === name);
        
        // 如果节点不存在，则创建一个新节点
        if (!childNode) {
          childNode = { name, children: [] };
          currentNode.children.push(childNode);
        }
        
        currentNode = childNode;
      });
    });
    
    return root.children;
  }
  
  const array = [['a','b','c'],['a','b','c1'],['a','b1','c2'],['a1','b']];
  const tree = arrayToTree(array);
  console.log(tree);
```

在这个代码中，我们假设每个节点的名称都是字符串。如果节点名称不是字符串，我们需要对比较函数进行修改。另外，如果节点具有其他属性或方法，则可以将它们添加到节点对象中。

其他：

```js
function buildTree(data, rootId) {
  const tree = [];
  const lookup = {};
  data.forEach(node => {
    lookup[node.id] = node;
    node.children = [];
  });
  data.forEach(node => {
    if (node.parent_id === rootId) {
      tree.push(node);
    } else {
      const parent = lookup[node.parent_id];
      parent.children.push(node);
    }
  });
  return tree;
}
```

