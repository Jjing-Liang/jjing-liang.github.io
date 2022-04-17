---
title:  "The Patchers in Python"
layout: post
---

```
# 只有单行注释
# ---表示一份内容的开始
---
string-example: cake
int-example: 666
float-example: 0.5
nulls-example:
  - null
  - Null
  - ~
  -
booleans-example:
  - true
  - false
canonical-date-example: 2001-12-15T02:59:43.1Z
iso8601-date-example: 2001-12-14t21:59:43.10-05:00
# !!表示类型转换
type-trans-example: !!float "666"


# 通过anchor锚点简化
setting-anchor-example: &number-category # 设置锚点
  one: 1
  two: 2
  three: 3
using-anchor-example: *number-category # 引用锚点
extending-anchor-example: # 合并锚点
  <<: *number-category
  four: 4


# literal-style表达多行内容: 文本中的换行都被保留，即cake与下一句之间有换行
literal-style-example: |
  JavaScript was initially created to “make web pages alive”.
  The programs in this language are called scripts. 
  They can be written right in a web page’s HTML and run automatically as the page loads.

# folded-style表达多行内容: 文本中的换行被替换为空格，即cake与下一句被空格连接为一句。只有空白行才表示换行
folded-style-example: >
  JavaScript was initially created to “make web pages alive”.
  The programs in this language are called scripts. 

  They can be written right in a web page’s HTML and run automatically as the page loads.


# Mapping
mapping-example-1:
  programming-languages:
    javascript: JavaScript often abbreviated JS, is a programming language that is one of the core technologies of the World Wide Web, alongside HTML and CSS
    java: Java is a high-level, class-based, object-oriented programming language that is designed to have as few implementation dependencies as possible

mapping-example-2:
  {
    programming-languages:
      {
        javascript: JavaScript often abbreviated JS, is a programming language that is one of the core technologies of the World Wide Web, alongside HTML and CSS,
        java: Java is a high-level, class-based, object-oriented programming language that is designed to have as few implementation dependencies as possible,
      },
  }
  

# 数组
array-example-option-1:
  - javascript
  - java
  - go
array-example-option-2: [javascript, java, go]
# 多维数组
two-dimensional-array-example:
  - - JavaScript Fundamentals
    - Objects
    - Data Types
  - - Java Introduction
    - Java Variables
    - Java Operators
...
# ...表示一份内容的结束（可选）
```