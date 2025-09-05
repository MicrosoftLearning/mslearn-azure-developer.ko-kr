---
title: Azure 개발자를 위한 연습
permalink: index.html
layout: home
---

## 개요

다음 연습은 Microsoft Azure에 솔루션을 빌드하고 배포할 때 개발자가 수행하는 일반적인 작업을 탐색하는 실습 학습 환경을 제공하도록 설계되었습니다.

> **참고**: 연습을 완료하려면 필요한 Azure 리소스를 프로비전할 수 있는 충분한 권한과 할당량이 있는 Azure 구독이 필요합니다. 아직 계정이 없는 경우 [Azure 계정](https://azure.microsoft.com/free)에 가입합니다. 

일부 연습에는 추가적이거나 다른 별도의 요구 사항이 있을 수 있습니다. 이러한 경우 해당 연습에는 **시작하기 전에**라는 특정 섹션이 포함되어 있습니다.

## 주제 영역
{% assign exercises = site.pages | where_exp:"page", "page.url contains '/instructions'" %} {% assign grouped_exercises = exercises | group_by: "lab.topic" %}

<ul>
{% for group in grouped_exercises %}
<li><a href="#{{ group.name | slugify }}">{{ group.name }}</a></li>
{% endfor %}
</ul>

{% for group in grouped_exercises %}

## <a id="{{ group.name | slugify }}"></a>{{ group.name }} 

{% for activity in group.items %} [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }}

---

{% endfor %} <a href="#overview">맨 위로 돌아가기</a> {% endfor %}

