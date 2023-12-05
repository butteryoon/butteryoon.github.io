---
layout: post
comments: true
title: "bash Array"
description: "bash 스크립트로 array 사용하기."
img: command-title.webp
date: 2023-12-06 01:00:00 +0900
last_modified_at: 2023-12-06 01:00:00 +0900
tags: [command-line, bash, script, array, chatgpt, 배열] # add tag
related: command-line
categories: tools
---

쉘 스크립트는 간간히 쓰려면 항상 문법이 헷갈려서 chatGPT에 물어보면서 원하는 답을 얻어보자. 

<!--more-->

## 첫번째 질문

**"bash script로 배열을 사용하는 예제, 그리고 데이터는 , 로 구분된 csv 형식"** 이라고 질문을 던졌다.

답변으로 아래와 같은 스크립트를 보여준다. 

아래의 스크립트는 한 라인씩 읽어서 **data_array** 에 저장하고 변수의 개수만큼 **for loop**을 돌면서 행을 출력한다.

```bash
#!/bin/bash

# CSV 파일 경로
csv_file="example.csv"

# 배열 선언
declare -a data_array

# CSV 파일 읽기
while IFS=',' read -ra line; do
    # 배열에 줄 추가
    data_array+=("${line[@]}")
done < "$csv_file"

# 배열 출력
echo "CSV 파일 내용:"
for ((i=0; i<${#data_array[@]}; i++)); do
    echo "행 $((i+1)): ${data_array[i]}"
done

# 특정 열 출력 (예: 첫 번째 열)
column_index=0
echo "첫 번째 열:"
for ((i=0; i<${#data_array[@]}; i++)); do
    echo "${data_array[i]}" | cut -d',' -f$((column_index+1))
done
```

## 두번째 질문은 좀 더 구체적으로.. 

csv 파일의 형식을 아래와 같이 정의해 주니 좀 더 구체적으로 수정해 준다. 

```text
이름,나이,직업
John,30,Engineer
Jane,25,Designer
Mike,35,Manager
```

다시 알려준 스크립트는 파일을 항목별로 읽어서 배열에 추가하고 역시 cut으로 잘라서 각 컬럼의 값을 가져온다. 

```bash
#!/bin/bash

# CSV 파일 경로
csv_file="example.csv"

# 배열 선언
declare -a data_array

# CSV 파일 읽기
while IFS=',' read -r name age job; do
    data_array+=("$name,$age,$job") # 배열에 데이터 추가
done < "$csv_file"

# 배열 출력
echo "CSV 파일 내용:"
for ((i=0; i<${#data_array[@]}; i++)); do
    echo "행 $((i+1)): ${data_array[i]}"
done

# 특정 열 출력 (예: 첫 번째 열)
column_index=0
echo "첫 번째 열:"
for ((i=0; i<${#data_array[@]}; i++)); do
    echo "${data_array[i]}" | cut -d',' -f$((column_index+1))
done
```

## 수정.. 

대략 어떻게 쓰는지 알았는데 특정 열을 출력할 때 **cut** 명령어로 잘라서 쓰는게 맘에 들지 않아 좀 수정해본다. 

**data_array**를 줄 단위로 읽어 **entry** 변수에 배열 형식으로 저장하면 각 변수의 컬럼 인덱스로 값을 가져올 수 있다. 

```bash
#!/bin/bash

# CSV 파일 경로
csv_file="example.csv"

# 배열 선언
declare -a data_array

# CSV 파일 읽기
while IFS=',' read -r name age job; do
    data_array+=("$seq $appid $devcode $chacode $custid") # 배열에 줄 추가
done < "$csv_file"

# 배열 출력
echo "CSV 파일 내용:"
for element in "${data_array[@]}";do        
    entry=($element)    # 각 라인의 값을 다시 배열로 변경
    echo ${entry[0]} ${entry[1]} ${entry[2]} ${entry[3]} ${entry[4]}    # 베열 인덱스로 
done

exit
```

## tldr 

웬만 한 쉘 명령어 사용법은 **chatGPT**가 다 알고 있다.  
bash 스크립트 문법은 항상 헷갈리니 GPT를 잘 활용하자.. 


## 참고 URL
- [고급 Bash 스크립팅 가이드](https://wiki.kldp.org/HOWTO/html/Adv-Bash-Scr-HOWTO/)
- [bash - LinuxOPsys](https://linuxopsys.com/?s=bash)