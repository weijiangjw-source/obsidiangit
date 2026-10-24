---
title: "{{title | replace('"', '\\"')}}"
created: "{{dateAdded | format('YYYY-MM-DD HH:mm:ss')}}"
updated: "{{dateModified | format('YYYY-MM-DD HH:mm:ss')}}"
authors: "{{authors}}"
year: {{date | default(year)}}
citekey: {{citekey}}
type: literature-note
tags:
  - zotero-note
  {% for tag in tags %}- {{tag.tag | replace(' ', '_')}}
  {% endfor %}
zotero-link: "{{selectURI}}"
---


# {{title}}

## 📖 文献元数据
- **作者**: {{authors}}
- **年份**: {{date | default(year)}}
- **条目类型**: {{itemType}}
{% if publicationTitle %}- **期刊/来源**: *{{publicationTitle}}*{% endif %}
{% if doi %}- **DOI**: [{{doi}}](https://doi.org/{{doi}}){% endif %}
{% if url %}- **原文链接**: [点击跳转]({{url}}){% endif %}
- **Zotero 库**: [在 Zotero 中打开]({{desktopURI}})

{% if abstractNote %}
## 💡 摘要
> {{abstractNote}}
{% endif %}

## 📄 批注与思考
{%- for annotation in annotations %}
{%- if annotation.comment and annotation.colorCategory != "Yellow" and annotation.colorCategory != "Red" and annotation.colorCategory != "Orange" %}

- 📌 **个人注释/便签** ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}})): <span style="color: #2ecc71;">{{annotation.comment}}</span>
{%- endif %}
{%- endfor %}

## 🟡 质疑与批判
{%- for annotation in annotations %}
{%- if annotation.colorCategory == "Yellow" %}
{%- if annotation.annotatedText %}

- <span style="color: #f1c40f;">{{annotation.annotatedText}}</span> ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}}))
  {% if annotation.comment %}> 💬 **划线附注**: {{annotation.comment}}{% endif %}
{%- elif annotation.comment %}

- 📝 **黄色笔记** ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}})): <span style="color: #f1c40f;">{{annotation.comment}}</span>
{%- endif %}
{%- endif %}
{%- endfor %}

## 🔴 震撼影响
{%- for annotation in annotations %}
{%- if annotation.colorCategory == "Red" %}
{%- if annotation.annotatedText %}

- <span style="color: #e74c3c;">{{annotation.annotatedText}}</span> ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}}))
  {% if annotation.comment %}> 💬 **划线附注**: {{annotation.comment}}{% endif %}
{%- elif annotation.comment %}

- 📝 **红色笔记** ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}})): <span style="color: #e74c3c;">{{annotation.comment}}</span>
{%- endif %}
{%- endif %}
{%- endfor %}

## 🟠 重点引用
{%- for annotation in annotations %}
{%- if annotation.colorCategory == "Orange" %}
{%- if annotation.annotatedText %}

- <span style="color: #e67e22;">{{annotation.annotatedText}}</span> ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}}))
  {% if annotation.comment %}> 💬 **划线附注**: {{annotation.comment}}{% endif %}
{%- elif annotation.comment %}

- 📝 **橙色笔记** ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}})): <span style="color: #e67e22;">{{annotation.comment}}</span>
{%- endif %}
{%- endif %}
{%- endfor %}

## 📝 高光与划线批注
{%- for annotation in annotations %}
{%- if annotation.annotatedText %}

- {{annotation.annotatedText}} ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}}))
  {% if annotation.comment %}> 💬 **划线附注**: <span style="color: #2ecc71;">{{annotation.comment}}</span>{% endif %}
{%- elif annotation.imageRelativePath %}

- ![[{{annotation.imageRelativePath}}]] ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}}))
  {% if annotation.comment %}> 💬 **截图附注**: <span style="color: #2ecc71;">{{annotation.comment}}</span>{% endif %}
{%- endif %}
{%- endfor %}