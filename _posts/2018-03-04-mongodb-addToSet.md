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

