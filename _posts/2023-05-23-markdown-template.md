---
layout: post
title: "markdown模板"
# subtitle: '错题集'
date: 2023-05-23
categories: markdown
author: Jason
# cover: 'assets/img/profile.png'
tags: markdown demo template
---

## table

| First Header | Second Header |
| ------------ | ------------- |
| Content Cell | Content Cell  |
| Content Cell | Content Cell  |

| Left-Aligned  | Center Aligned  | Right Aligned |
| :------------ | :-------------: | ------------: |
| col 3 is      | some wordy text |         $1600 |
| col 2 is      |    centered     |           $12 |
| zebra stripes |    are neat     |            $1 |

## graph

```mermaid
graph TB;
begin(出门)-->buy("买炸鸡")
buy-->isremaining{"还有没有炸鸡？"}
isremaining--> |有|happy["买完炸鸡很开心"] -->goBack(回家)
isremaining --没有--> sad["伤心"] --> goBack(回家)

A
B(圆角矩形节点)
C[矩形节点]
D((圆形节点))
E{菱形节点}
F>右向旗帜形节点]
```

连线

```mermaid
graph TB
  A1-->B1
  A2---B2
  A3--text---B3
  A4--text-->B4
  A5-.-B5
  A6-.->B6
  A7-.text.-B7
  A8-.text.->B8
  A9===B9
  A10==>B10
  A11==text===B11
  A12==text==>B12
```

```mermaid
pie title "动物饼图"
    "Dog": 1
    "Cat": 2
    "Rabiit": 3
```

```mermaid
mindmap
  root((mindmap))
    Origins
      Long history
      ::icon(fa fa-book)
      Popularisation
        British popular psychology author Tony Buzan
    Research
      On effectiveness<br/>and features
      On Automatic creation
        Uses
            Creative techniques
            Strategic planning
            Argument mapping
    Tools
      Pen and paper
      Mermaid
```
