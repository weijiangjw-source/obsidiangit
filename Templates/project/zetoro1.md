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

## 📄 Zotero 条目笔记与独立便签
{%- for annotation in annotations %}
{%- if annotation.comment %}

- 📌 **个人注释/便签** ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}})): {{annotation.comment}}
{%- endif %}
{%- endfor %}

## 📝 高光与划线批注
{%- for annotation in annotations %}
{%- if annotation.annotatedText %}

- {{annotation.annotatedText}} ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}}))
  {% if annotation.comment %}> 💬 **划线附注**: {{annotation.comment}}{% endif %}
{%- elif annotation.imageRelativePath %}

- ![[{{annotation.imageRelativePath}}]] ([P.{{annotation.pageLabel}}]({{annotation.desktopURI}}))
  {% if annotation.comment %}> 💬 **截图附注**: {{annotation.comment}}{% endif %}
{%- endif %}
{%- endfor %}
