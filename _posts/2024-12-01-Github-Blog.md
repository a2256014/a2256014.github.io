---
layout: post
title: Github Blog 구축 및 배포
subtitle: Jekyll, Obsidian를 활용한 GitBlog 구축 및 자동화
author: g3rm
categories: Dev
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - Obsidian
  - Jekyll
  - Github
  - Github_Blog
sidebar:
---
## Intro
지금껏 적어둔 정보들을 정리 및 배포하기 위해 블로그를 시작 해야겠다고 생각하고, 여러 블로그를 찾아보던 중 나름 IT업계 종사자이니 커스텀이 자유로운 GitBlog를 시작하게 되었습니다. 

저의 경우 Yet Another Theme (YAT) Theme 사용하여 해당 Theme에 맞는 설치 법을 공유하지만, 대부분의 Theme 설치 방법이 비슷하기에 따라하는데 무리가 없을 것 같습니다.
## Install
### Ruby 설치
우선 전 Jekyll Theme을 사용을 할 것이기 때문에 Ruby를 설치해 줍니다.   
[Ruby 설치](https://rubyinstaller.org/downloads/) 에 들어가서 Installer를 설치해 줍니다.

### Jekyll Theme Download & Obsidian Vault 생성
1. 무료 Jekyll Theme 보는 곳에서 마음에 드는 Jekyll Theme을 골라줍니다. 
	- https://jamstackthemes.dev/
	- https://jekyll-themes.com/free
	- https://jekyllthemes.io/free
2. 해당 Theme의 Git Repo로 가서 Download Zip을 합니다.
3. 그 후 Obsidian Vault를 생성한 뒤 다운로드한 Theme 내용물을 옮겨줍니다.

### Jekyll Theme 모듈 설치
1. IDE에서 Jekyll Theme이 들어있는 Obsidian 폴더로 가기
2. Gemfile에 해당 Theme에 맞는 설정들 삽입   
   \* *설정들은 설치하고자 하는 Theme Git에 들어가면 적혀있습니다.*
```Gemfile
gem "jekyll-theme-yat"
```
3. 그 후 번들 및 실행
~~~Shell
$ cd ~/Obsidian_Vault/Git_Blog_Vault
$ bundle
$ bundle exec jekyll serve
~~~
1. http://127,0.0.1:4000 접속해보기

## Obsidian Git 연동하기
### Repo 생성
Repo 이름의 경우 자신의 Github Name에 맞도록 생성하는 것이 중요합니다.      
*아닐경우 URL이 난잡해 질 수 있습니다.*   
Repo name exam : [GithubName].github.io 
![](/assets/images/posts/2024-12-01-Github-Blog/7b5b1619653f9bba27a180dc66c1a7e8_MD5.jpg)

### Git 연동

공유하고자 하는 Vault 위치로 터미널을 이동한다

```shell
git init
git config --local user.name "깃허브 아이디 이름"
git config --local user.email "깃허브 이메일 이름"
```

### .gitignore 설정

Vault 폴더 안에 `.gitignore` 파일을 만들어 다음과 같이 작성한다  

```.gitignore
.obsidian/

.obsidian/workspace-mobile.json
.obsidian/workspace.json
.obsidian/app.json

.trash/
.DS_Store
```

### github에 올리기
공유하고자 하는 Vault 위치로 터미널을 이동한다

```shell
git add .
git commit -m "initial commit"
git branch -M main 
git remote add origin "생성한 깃허브 repo 주소"
git push -u origin main
```

## Obsidian-Git Plugin 설치
Obsidian을 켠 후 설정(톱니바퀴) 버튼을 누른다.  
그후 Community Plugins에 들어가고, 만약 거기서 Community Plugin이 꺼져있다고 나오면 켠다.  
![](/assets/images/posts/2024-12-01-Github-Blog/2d80ac10ca27ebfc1a39e28c685f99cf_MD5.jpeg)

Community plugins 에 Browse를 누른 다음, `Obsidian git`를 검색해 플러그인을 설치한다. (설치 후에 Enable를 눌러줘야 한다.)  
![](/assets/images/posts/2024-12-01-Github-Blog/8638c85f5c3584e3b2d94efa959afb5e_MD5.jpeg)

Enable과 동시에 github에 동기화된다.
### 설정
![](/assets/images/posts/2024-12-01-Github-Blog/b77f4c219e37f3a570dc605a5564d00a_MD5.jpeg)

![](/assets/images/posts/2024-12-01-Github-Blog/994befaa30a53f96389ab7879c0cf259_MD5.jpg)
## Git Blog 배포
1. [Settings > Pages] 에서 GitHub Action으로 변경
![](/assets/images/posts/2024-12-01-Github-Blog/70d77aa4c114e23f57fec4b410556ffc_MD5.jpg)
2. Jekyll Configure 선택 및 변경사항 없이 Configure 생성
![](/assets/images/posts/2024-12-01-Github-Blog/83f9a5d86ba413b5f60f499d0cf861ea_MD5.jpg)
3. 그 후 기다리면 깃 블로그가 생성 됨. https://MyGitName.github.io 접속

~~~javascript 
function syntaxHighlight(code) { var foo = 'Hello World'; var bar = 100; } ~~~