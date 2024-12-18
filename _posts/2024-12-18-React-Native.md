---
layout: post
title: "[Dev] React Native"
subtitle: React Native 분석을 위한 개발
author: g3rm
categories: Dev
banner:
  image: /assets/images/banners/home.gif
  opacity: 0.618
  background: "#000"
  heading_style: "font-size: 4.25em; font-weight: bold;"
  subheading_style: "color: gold"
tags:
  - React-Native
  - Hermes
  - React-Native-CLI
sidebar:
---


## Intro
최근 React Native 앱을 진단하게 되었는데, Java나 Kt(코틀린)으로 개발한 앱과는 분석 방법이 다른 것 같아 스스로 개발해서 분석하기 위해 작성하게 되었습니다 👋

## Env
- Node.js 설치
- Python 설치
- JDK 설치
- Android Studio 설치

위 4가지는 준비가 되어 있어서 생략하겠습니다 ㅎㅎ...😅   

- React Native CLI 설치   
```CMD
npm install -g @react-native-community/cli
```   

## Start
### 프로젝트 생성 및 실행
```cmd
npx @react-native-community/cli init Sectest_App
```


## References
