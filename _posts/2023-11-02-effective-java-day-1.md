---
title: Effective Java 스터디 1일차
author: honghyunshik
date: 2023-11-02 14:30:00 +0800
categories: [JAVA]
tags: [effective,java]
---

이펙티브 자바(Joshua Bloch) 책을 읽고 정리 및 느낀점을 기술하는 포스트입니다.

# 1장 - 들어가기

추구하는 가장 큰 원칙은 2가지이다.
    
    1. 명료성(clarity)
    2. 단순성(simplicity)

이에 의해 파생되는 규칙은 다음과 같다.

    1. 컴포넌트는 정해진 동작이나 예측할 수 있는 동작만 수행해야 한다.
    2. 컴포넌트는 가능한 한 작되, 너무 작아도 안 된다.
    3. 코드는 복사되는 게 아니라 재사용되어야 한다.
    4. 컴포넌트 사이의 의존성을 최소로 유지해야 한다.