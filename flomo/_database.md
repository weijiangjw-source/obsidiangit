---
notion-bases: true
schema:
  - id: slug
    name: Slug
    type: text
    visible: true
    width: 200
  - id: created_at
    name: Created at
    type: date
    visible: true
    width: 140
  - id: updated_at
    name: Updated at
    type: date
    visible: true
    width: 140
  - id: tags
    name: Tags
    type: multiselect
    visible: true
    width: 140
    options:
      - value: 名人语录
      - value: 思考间隙
      - value: 人际交往
      - value: 认知思维
      - value: 研究结论
      - value: 人生感悟
      - value: 管理理念
      - value: 感情思语
      - value: 古语名句
      - value: 思维模型
      - value: 批判思考
      - value: 零售哲学
      - value: 比较政治
      - value: 数据信息
      - value: 心理学
      - value: 基本概念
      - value: 观点汇聚
      - value: 技能方法
      - value: 战略历程
      - value: 科学研究
      - value: 沟通艺术
      - value: 工具应用
      - value: 决策方法
      - value: 学习方法
      - value: 语言学习
      - value: 领导技术
      - value: 权谋观念
      - value: 个人成长
      - value: 网络搜集
      - value: AI新时代
      - value: 正念练习
      - value: 呼吸觉知
      - value: 一行禅师
      - value: 能量磁场
      - value: 亲密关系
      - value: /
  - id: source
    name: Source
    type: select
    visible: true
    width: 140
    options:
      - value: ios
views:
  - id: default
    type: table
    filters: []
    sorts: []
    hiddenColumns:
      - source
      - tags
      - slug
    columnWidths: {}
---

> [!tip] Notion Bases
> 此文件是一个数据库。打开它以查看表格视图。
