---
date: 2021-09-01
title: "html selector와 cascading"
categories: HTML
tags:
    - HTML5
    - html
    - selector
    - css
    - 선택자
    - cascading
# 목차
toc: true
toc_sticky: true
---
# 웹 표준의 구성 요소
- 구조 : 콘텐츠에 의미 부여 및 구조를 형성 -> HTML
- 표현 : 시각적인 디자인과 레이아웃 표현 -> CSS
- 행위 : 모든 Front-end 브라우저 상호 작용 담당 -> JavaScript

# **Selector의 개념 및 종류**
특정 요소들을 선택하여 스타일을 적용할 수 있게 해주는 요소.
- 전체 선택자 (Universal Selector)
    - 모든 태그에 같은 CSS 속성 적용
    - margin이나 padding 값 등 기본값을 정할 때 사용
    - 문서 안의 모든 요소를 읽어내려야 하기 때문에 페이지 로딩 속도 저하
```css
\* {
    margin: 0; text-decoration: none 
}
```

## **태그 선택자 (Type Selector)**
- HTML요소를 직접 지정하여 스타일 적용
```css
h1,
h2 {
  color: blue;
  /* 모든 제목요소에 같은 디자인 속성을 지정하려면 이렇게 묶을 수 있습니다 */
}
```

## 클래스 선택자
- '.'을 사용하여 지정
- 가장 범용적으로 사용
```css
.korea,
.japan,
.china {
  /* 클래스 선택자도 마찬가지로 묶어서 선택할 수 있습니다. */
}
div.korea p {
  /* region 클래스를 가지는 div 요소 하위에 포함된 모든 p 요소를 선택합니다.
   바로 위에 있는 부모 요소이든, 혹은 멀리 떨어져 있는 계층 구조이든 상관 없습니다. */
  background-color: green;
}
.region .korea {
  /* region 클래스를 가지는 모든 요소의 하위에 포함된 korea 클래스 요소를 선택합니다. */
  background-color: orange;
}
.region.korea {
  /* 띄어쓰기가 있는 것과 없는 것은 완전히 다릅니다.
  클래스 선택자를 띄어쓰지 않고 작성하면 *두 개가 모두 포함되어 있어야* 적용된다는 의미입니다. */
  background-color: gray;
}
.container > p {
  /* module 클래스를 가진 요소의 바로 아래의 자식요소 중 h2 요소를 선택하라는 의미입니다.
  직계자손이라는 뜻에서 더욱 구체적입니다. */
  background-color: purple;
}
.container p {
  /* 위와 비교해 보세요. */
  background-color: orange;
}
```

## id 선택자
- '#'을 사용하여 지정
- id는 전체 문서에서 unique한 값

## 가상 선택자 (Pseudo-Classes)
- 요소가 특정한 상황일 때만 선택
```css
a[title] {
  /* a 요소 중에서 title 속성이 있는 모든 요소를 선택 */
  background-color: orange;
}
a[href="https://www.google.com/maps/place/일본+가가와현+다카마쓰시"] {
  /* href 속성값이 "https://www.google.com/maps/place/일본+가가와현+다카마쓰시"와 일치하는 a 요소 */
  background-color: orange;
}
a {
  /* 모든 a 요소 */
  background-color: orange;
}
a:hover,
a:active,
a:focus {
  /* a 요소에 마우스가 올라가있거나, 활성화되어 있거나, 키보드로 선택되어 있을 때 */
  background-color: black;
}
li:first-child {
/* li 요소 중 첫 번째 li 요소 */
  background-color: orange;
}
li:nth-child(5) {
/* li 요소 중 5 번째 li 요소 */
  background-color: orange;
}
li:nth-child(2n) {
/* li 요소 중 짝수 번째 li 요소 */
  background-color: orange;
}
.container div:last-of-type {
  /* container 선택자 요소의 하위 div 요소 중 마지막 요소 */
  background-color: orange;
}
h2::before {
  content: "👉 ";
  /* h2 요소 앞에 가상 요소를 넣어 '>'라는 콘텐트를 넣어줍니다. */
}
```

- 복합 선택자
    - 두 개 이상의 선택자 요소가 모인 선택자
    - 하위 선택자와 자식 선택자로 구분
        - 하위 선택자: 부모에 포함된 모든 하위 요소에 스타일 적용
        - 자식 선택자: 부모 바로 아래 자식 요소에만 적용
```css
section ul {
    /* 하위 선택자 */
    border: 1px;
}
section > ul {
    /* 자식 선택자 */
    border: 2px;
}
```

# Cascading
HTML element는 하나 이상의 스타일에 영향을 받을 수 있기 때문에, 어떤 스타일을 적용 받을지에 대한 우선순위가 결정되어야 합니다.

이를 캐스캐이딩 이라고 하는데, CSS의 정식 명칭은 Cascading Style Sheets인 만큼 캐스캐이딩이 중요하다는 것을 알 수 있습니다.

캐스캐이딩은 다음의 3가지에 의해 결정
- CSS가 어디에서 선언되었는지 ( 중요도 )
- 대상을 명확하게 지정할수록 ( 명시도 )
- 코드 순서
- Specificity Calculator

## Cascading - 중요도
```css
<head>의 <style>
<head>의 <style> 내의 @import
<link>로 연결된 CSS 파일
<link>로 연결된 CSS 파일 내의 @import
브라우저의 기본 CSS
예를 들어, <head> 요소내에 있는 <style>이 <link>로 된 CSS 파일보다 우선순위가 높습니다.
```

## Cascading - 명시도
```css
!important
inline 스타일
아이디 selector
클래스 / 가상 선택자
태그 선택자
상속된 스타일
```
!important는 거의 우선순위 끝판왕이지만, 그렇다고 남용하는 것은 좋지 않다.
inline은 높은 우선순위를 갖지만, inline으로 스타일을 주지 말고, <style>이나 외부 CSS 파일로 빼는 것이 좋다.

## Cascading - 코드 순서
늦게 선언된 스타일이 우선 적용

## Cascading - Specificity Calculator
 구체성 점수에 따라서 스타일 적용이 선택됨
    - 구체성 값
        - id 선택자 :  100점
        - Class 선택자, 가상 클래스 : 10점
        - 태그 선택자, 가상요소 : 1점

    - https://specificity.keegan.st 에서 점수 계산 가능


- 출처
    - 선택자에 대한 더 자세한 분류 및 개념: https://www.nextree.co.kr/p8468/
    - sample code: https://codepen.io/fromhans/pen/GREZjvX
    - 캐스케이딩: https://victorydntmd.tistory.com/190
