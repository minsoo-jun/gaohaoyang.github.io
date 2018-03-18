---
layout: post
title:  "MongoDB $addToSet 이해하기"
date:   2018-03-04 10:47:00
categories: mongodb
tags: mongodb
excerpt: mongodb의 배열을 갱신할 때 사용하는 $addToSet의 삽질기
mathjax: true
---

MongoDB > $addToSet
===================
어떤 작업을 몽고디비 스크립트에서 하고 있었는데 중복된 값이 있을때는 그 값을 제외 시켜야 하는 필요가 있었습니다만 
몽고디비 스크립트에는 map기능이 없다는!! 정확하게는 Javascript가 map을 지원하지 않는 것입니다. 일부 라이브러리나 별도 map기능을 추가하는 방법도 있지만
몽고디비 내부에서 사용해야하기 때문에 적용하기가 부담스러워 하면서 몽고디비의 러퍼런스를 살펴 보던중 $addToSet을 찾았고 사용해 보기로 했습니다.    
### 상세한 내용은 아래에서 확인 가능합니다.
[Mongodb -> Operators](https://docs.mongodb.com/manual/reference/operator/)

[Mongodb -> Operators -> update -> addToSet](https://docs.mongodb.com/manual/reference/operator/update/addToSet/)

### 기본 문법
```javascript
{ $addToSet: { <field1>: <value1>,....}}
```
$addToSet 기능은 배열에 값이 없으면 추가를 같은 값이 있으면 갱신을 하지 않습니다. 
즉 hashmap과 비슷한 기능을 하고 있습니다.(머가 비슷하냐;)

### Input data 예제
```javascript
{
	"tagKey": "tag001",
	"popularity": 3010,
	"specPage": [
		{"id": "sp1111"},
		{"id": "sp2222"}
	],
	"normalPage": [
		{"id": "np1111"},
		{"id": "np2222"},
		{"id": "np3333"}
	],
	"labels": [
		{"en_US": "JSON"},
		{"ko_KR": "제이슨"},
		{"ja_JP": "ジェイソン"}
	]
}

{
	"tagKey": "tag002",
	"popularity": 356,
	"specPage": [
		{"id": "sp1111"},
		{"id": "sp2999"}
	],
	"normalPage": [
		{"id": "np1111"},
		{"id": "sp2999"},
		{"id": "np3333"}
	],
	"labels": [
		{"en_US": "Array"},
		{"ko_KR": "배열"},
		{"ja_JP": "配列"}
	]
}
```
tagKey값을 기준으로 하나의 JSON 문서로 구성이 되어 있고 
복수의 카테고리(spechPage, normalPage)의 복수의 페이지(id별로 다른 페이지)에서 tagKey가 사용되고 있다.
하나의 tagKey에는 여러 언어로 번역이 되어 있습니다.

출력은 각 페이지 별로 포함되어 있는 "tagKey"로 변환 하는 것입니다.
```javascript
{
	"id": "sp1111",
	"category": "specPage",
	"tagKeys":[
		{
			"tagKey": "tag001",
			"labels": [
				{"en_US": "JSON"},
				{"ko_KR": "제이슨"},
				{"ja_JP": "ジェイソン"}
			]
		},
		{
			"tagKey": "tag002",
			"labels": [
				{"en_US": "Array"},
				{"ko_KR": "배열"},
				{"ja_JP": "配列"}
			]
		}
	]
}
{
	"id": "sp2222",
	"category": "specPage",
	"tagKeys":[
		{
			"tagKey": "tag001",
			"labels": [
				{"en_US": "JSON"},
				{"ko_KR": "제이슨"},
				{"ja_JP": "ジェイソン"}
			]
		}
	]
}
{
	"id": "np1111",
	"category": "specPage",
	"tagKeys":[
		{
			"tagKey": "tag001",
			"labels": [
				{"en_US": "JSON"},
				{"ko_KR": "제이슨"},
				{"ja_JP": "ジェイソン"}
			]
		},
		{
			"tagKey": "tag002",
			"labels": [
				{"en_US": "Array"},
				{"ko_KR": "배열"},
				{"ja_JP": "配列"}
			]
		}
	]
}
```