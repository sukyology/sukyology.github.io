---
title: "블로그 UPDATE LOG"
date: 2021-08-05T20:27:27+09:00
draft: false
---

## 2021-08-11

#### 댓글 달기 기능 추가
<br>
disqus를 적용해보려고 했는데, 무슨 이유에서인지 잘 되지를 않았다.
그래서 뭐 이런 저런 것들을 찾아봤는데... 
disqus에 대해 부정적인 얘기를 들었다. 그러다가 어느 블로그를 들렀는데,
깃헙 계정으로 댓글을 달 수 있는 걸 볼 수 있었다. 

그래서 뭔가 하고 찾아봤더니, 블로그 댓글 기능을 깃헙 이슈를 생성하고, 
거기에 달린 댓글로 블로그 댓글에 보여주는 깃헙 앱을 누가 만들었던 것이다.
사용법은 굉장히 간단하다.
https://utteranc.es/?installation_id=18785326&setup_action=install

위에 사이트를 가서, 
1. 내 블로그 리파지토리하고,
2. issue와 블로그 글의 연동 방식,
3. 댓글에 적용할 theme

을 선택하면

```html
<script src="https://utteranc.es/client.js"
repo="sukyology/sukyology.github.io"
issue-term="pathname"
theme="github-light"
crossorigin="anonymous"
async>
</script>
```
이런 식으로 script element가 생성된다. 
해당 script는 깃헙 issue 댓글들을 html로 만들어서
`<div class="utterances"></div>`의 chidren에 주입한다.
내 블로그 글에 적용되는 template html에 위치를 정해서
```html
<div class="utterances"></div>
<script src="https://utteranc.es/client.js"
        repo="sukyology/sukyology.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
```
이런 식으로 만들어서 적용하면
![img.png](img.png)
댓글 쓰기 기능이 생긴다. 

