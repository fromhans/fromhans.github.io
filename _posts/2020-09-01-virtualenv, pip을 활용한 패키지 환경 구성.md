---
date: 2021-09-01
title: "virtualenv, pip을 활용한 패키지 환경 구성"
categories: python
tags:
    - linux
    - virtualenv
    - venv
    - pip
# 목차
toc: true
toc_sticky: true
---
# virtualenv, pip을 활용한 initial setup 및 버전 업그레이드

virtualenv와 pipenv는 모두 독립된 파이썬 환경을 만드는데 가장 많이 쓰이는 라이브러리로, 해당 패키지를 활용하면 개발환경의 패키지를 배포환경으로 쉽게 이식할 수 있다.

본문에서는 virtualenv를 활용한다.

## virtualenv의 동작 원리

1. $ virtualenv XXX 명령어로 가상 환경을 생성

    → XXX 이름으로 디렉토리가 생성됨

    → 디렉토리 내 /bin/ 경로에 python 바이너리 파일이 생성됨.

    → 해당 바이너리 파일은 로컬의 것과 같지만, 가상환경 폴더 내 라이브러리를 먼저 참조하게 프로그래밍 되어있어서 로컬 라이브러리와는 별도로 작업이 가능

2. $ source ./XXX/bin/activate 명령어로 가상환경 활성화

    → 프롬프트 입력창 맨 앞에 (가상환경명)이 표시 확인

# 개발 환경의 패키지 설치 현황 확인

- 개발 환경의 터미널에서 아래와 같이 실행

```bash
pip freeze >> requirements.txt
mkdir 다운받을폴더
cd 다운받을폴더
pip download -r requirements.txt
```

- 아래 파일들을 개발환경 → initial setup 대상 서버로 이동
    1. 배포할 프로젝트 zip파일
    2. /다운받을폴더/requirements.txt
    3. /다운받을폴더/*.whl 혹은 *.tar.gz  (pip download로 다운된 모든 파일)

# virtualenv를 활용한 패키지 환경 구축

1. 프로젝트.zip 파일 압축 해제 (경로/권한/소유자는 알아서 755 맞추기)
2. 압축해제된 디렉토리 내에 가상환경 생성

    ```bash
    unzip test-Dev.zip
    cd test-Dev
    virtualenv venv    #가상환경 생성
    source ./venv/bin/activate   #가상환경 활성화
    cd /home/downloadedfiles/    #개발환경에서 가져온 파일들이 있는 경로
    pip install --no-index --find-links=./ -r requirements.txt    #패키지 한번에 설치
    ```

개발환경이 만약 가상환경에서 구축되어있다면, 개발환경의 가상환경 패키지들만을 이식할 경우, 프로젝트 수행 시 에러가 발생할 수 있다. 그 이유는 virtualenv의 python 바이너리 파일이 가상환경의 라이브러리 참조 후 로컬 라이브러리를 참조하기 때문으로, 해당 에러 방지를 위해서는 로컬 라이브러리를 포함하여 그대로 이식하거나, 에러발생하는 모듈들을 하나하나 설치해줄 필요가 있다.

FA에서는 버전업을 위해서는 전역 환경변수 설정 (/etc/bashrc 후 source /etc/bashrc 하거나, virtualenv 환경 내 activate 스크립트 수정)
django collectstatic 수행
nginx 설정
supervisord 설정 등이 추가로 필요하다