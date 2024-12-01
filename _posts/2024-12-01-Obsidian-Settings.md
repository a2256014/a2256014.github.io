---
layout: post
title: Github Blog를 위한 Obsidian Settings
subtitle: Github Blog를 편하게 작성하기 위한 사용한 Obsidian Setting 정리
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
  - Settings
  - Github_Blog
sidebar:
---
## Intro
[Gitgub_Blog 구축 및 배포](./Github-Blog.html)에서 Obsidian과 GitBlog를 연동하여 Obsidian에서 글을 쓰는 것만으로 블로그에 적용되도록 했습니다.   
더 나아가서 편하게 작성하기 위해 Obsidian 내장 설정과 다양한 Obsidian Plugin 들을 소개하려합니다.

## Plugins
### Local Images Plus
이미지를 복사 붙여넣기 했을 때, 자동으로 Local 환경에 이미지를 다운로드 받아주는 기능입니다.   
설정을 통해 Github Blog에 사용되는 이미지 경로(/assets/image/)를 지정하면 이미지도 함께 적용되는 것을 볼 수 있습니다.
   
`/assets/images/posts/${notename}` 으로 설정 시 각 페이지에 사용된 이미지를 손쉽게 정리 및 배포할 수 있습니다.
![](/assets/images/posts/2024-12-01-Obsidian-Settings/d06678e7ab0e8bf69e04fd38a4987a42_MD5.jpeg)
   
추가적으로 로컬 파일로 저장되면서 `![[img]]` 형식으로 변환되는데 이 때, GitHub Blog에서는 해당 이미지를 가져오지 못하므로 아래 사진과 같이 설정 값을 OFF 해주시면 됩니다.
![](/assets/images/posts/2024-12-01-Obsidian-Settings/70576ddf103b0bdf2ae875ea49e262d0_MD5.jpeg)
### Templater
각 Jekyll Theme에 맞는 마크다운 형식이 있을텐데, 템플릿을 생성하여 형식 그대로 가져올 수 있도록 하는 플러그인입니다.   

1. Template들을 저장할 폴더를 생성해 줍니다.
2. 그 후 Obsidian 기본 템플릿 설정에서 해당 폴더를 지정합니다.
   ![](/assets/images/posts/2024-12-01-Obsidian-Settings/26a6c98aed66e0533a98663fbf5587ed_MD5.jpeg)
3. 위에서 만든 폴더에 사용할 Template를 만들어 줍니다.   
   [Templater 문법](https://silentvoid13.github.io/Templater/)중 아래 코드를 사용하면 생성과 동시에 원하는 폴더로 이동됩니다.   
   기본적으로 템플릿 생성 단축키는 `Alt + N` 입니다.
```markdown
 <% await tp.file.move("/_posts/" + tp.date.now() + "-" + "NewPost") %>
```
4. Templater 플러그인 설정에서 만든 Template를 설정해 줍니다.
   ![](/assets/images/posts/2024-12-01-Obsidian-Settings/b70380427bbffe7ed513ef2f53289632_MD5.jpeg)
   ![](/assets/images/posts/2024-12-01-Obsidian-Settings/0594d6f949665c19836479879a9a06fe_MD5.jpeg)

### Editing Toolbar & Emoji Toolbar
블로그 작성을 위해 마크다운을 작성할 때, 유용한 플러그인들 입니다.
1. Editing Toolbar
   ![](/assets/images/posts/2024-12-01-Obsidian-Settings/d272ff594d7e0ce79460e60056cd731e_MD5.jpeg)
2. Emoji Toolbar
   설정-단축키 에서 HotKey를 등록할 수 있습니다~ 😀👍😁🎃 
   ![](/assets/images/posts/2024-12-01-Obsidian-Settings/18bd1e70ae7a7563937c4a73751f20d0_MD5.jpeg)