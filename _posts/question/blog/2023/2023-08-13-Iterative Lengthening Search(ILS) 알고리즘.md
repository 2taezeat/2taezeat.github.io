---
layout: post
title: "Iterative Lengthening Search(ILS) 알고리즘"
date: 2023-08-13 16:54:46+0900
author: 2taezeat
toc: true
categories: 
- blog
- Algorithm

---
Iterative Lengthening Search(ILS) 알고리즘
<!--more-->

### Intro
`Uniform Cost Search` 알고리즘과 동일한 결과를 얻을 수 있도록, `Iterative deepening` 원리를 적용하여,
space complexity가 `O(bd)`인 (b: branching factor, d: depth)
`Iterative Lengthening Search(ILS)` 알고리즘을 작성하는 프로젝트를 설명합니다.


* 기본적인 탐색 알고리즘 (BFS,DFS, Blind Search) 에 대한 이해가 ILS를 이해 하는데 도움이 됩니다.
* `ILS 알고리즘` '그 자체'에 대한 자료가 많이 없기에, 제가 작성한 `ILS 알고리즘`이 100% 완벽한 효율을 내는 정답은 아닐 수 있습니다.





### 전체 수도코드
![](https://images.velog.io/images/ba159sal/post/9f21ea6e-c3f3-44f1-8d55-d04052ca6390/Iterative%20Lengthening%20Search_%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-1.jpg){: width="800" height="800"}
### ITERATIVE-LENGTHENING-SEARCH 함수 설명
![](https://images.velog.io/images/ba159sal/post/087f354b-4871-48c5-bc98-a71344675324/Iterative%20Lengthening%20Search_%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-2.jpg){: width="800" height="800"}
### MY_DLS 함수 설명
![](https://images.velog.io/images/ba159sal/post/497713e3-a977-4e7b-abd5-0653c74c13f9/Iterative%20Lengthening%20Search_%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-3.jpg){: width="800" height="800"}
### LIMITCOST_CHECK 함수 설명
![](https://images.velog.io/images/ba159sal/post/d13b7517-6998-4aba-9315-e9ccf7811320/Iterative%20Lengthening%20Search_%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-4.jpg){: width="800" height="800"}

### Reference
- [Artificial Intelligence: A Modern Approach(인공지능 수업 교과서)](http://aima.cs.berkeley.edu/)
- [[ 인공지능 강좌 ] 7. Uninformed Search(Blind Search) 설명](https://blog.naver.com/PostView.nhn?blogId=ndb796&logNo=220576227914&redirect=Dlog&widgetTypeCall=true&topReferer=https%3A%2F%2Fm.blog.naver.com%2FPostView.naver%3FblogId%3Dndb796%26logNo%3D220576227914%26proxyReferer%3Dhttps%3A%252F%252Fwww.google.com%252F&directAccess=false)
- [Iterative Deepening Search(IDS) or Iterative Deepening Depth First Search(IDDFS)](https://www.geeksforgeeks.org/iterative-deepening-searchids-iterative-deepening-depth-first-searchiddfs/)