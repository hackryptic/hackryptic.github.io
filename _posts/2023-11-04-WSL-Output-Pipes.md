---
layout: post
title:  "WSL에서 윈도우 명령어 출력 결과에 대해 pipe(|)를 사용할 시 주의점"
date:   2023-11-04 19:05:00 +0900
categories: System
---


# Table of Contents

1.  [문제의 개요](#intro)
2.  [원인](#cause)
3.  [해결 방법](#solution)
4.  [정리](#summary)

\#-**- mode: org -**-




<a id="intro"></a>

# 문제의 개요

* Host OS: Windows 10 Pro
* Target OS: BlackArch (WSL2)


리눅스 시스템에서 X11 클라이언트를 사용하기 위해서는 **$DISPLAY** 환경변수에 대한 설정이 필요하며 **HostName:DisplayName.ScreenName** 과 같은 형태로 정의한다.

예를 들어 Hostname이 111.111.111.111에 0번 Display와 0번 Screen을 사용하면 

~~~console
export DISPLAY=111.111.111.111:0.0
~~~

이런식으로 정의한다.

HostIP가 바뀔때마다 일일히 입력하는 작업은 매우 번거로운 과정이기 때문에 공통된 명령어 한번으로 HostIP 주소를 가져올 수 있는 방법에 대해 고민했다.

그래서 다음과 같은 명령을 이용하여

~~~console
export DISPLAY=$(echo "$((echo $(ipconfig.exe | grep -a 'Wi-Fi' -A 4 | cut -d":" -f 2 | tail -n1) && echo ":0.0") | tr -d "\n")")
~~~

ipconfig의 결과 값에서 내가 필요한 IP 주소를 뽑아오려는 시도를 했다. 추가적으로 뒤쪽에 0.0을 붙이는 작업도 같이 명령어에 포함하였다. (본 포스트에서는 IP가 111.111.111.111이라 가정)

그러나 명령어 실행결과,아래와 같이 뒤쪽에 붙어야 할 **:0.0**이 앞쪽을 덮어쓰는 방식으로 출력되는 문제가 발생했다.

~~~console
echo $DISPLAY
:0.0111.111.111
~~~

<a id="cause"></a>

# 원인

윈도우에서 newline은 \r\n로 인코딩되어 있다. (이 부분에 대한 역사적 맥락은 구글에 검색하면 상세히 설명된 블로그가 많다.)

그렇기 때문에 다음과 같이 ipconfig에서 IP 주소를 그대로 가져오면



**111.111.111.111\n** 이 아니라 **111.111.111.111\\r\\n**을 출력한다. 

이 상태에서 **:0.0** 을 뒤에 붙이면 **111.111.111.111\\r\\n:0.0**와 같은 형태가 되는데 \\n을 제거하면 **111.111.111.111\\r:0.0**이 되고 결국에는 Carriage Return만 남아서 **:0.0111.111.111**이 된다.

명령어로 출력해보면 다음과 같다

~~~console
echo "$(echo $(ipconfig.exe | grep -a 'Wi-Fi' -A 4 | cut -d":" -f 2 | tail -n1))"

출력:
111.111.111.111


echo "$(echo $(ipconfig.exe | grep -a 'Wi-Fi' -A 4 | cut -d":" -f 2 | tail -n1) && echo ":0.0")"

출력:
111.111.111.111
:0.0


echo "$((echo $(ipconfig.exe | grep -a 'Wi-Fi' -A 4 | cut -d":" -f 2 | tail -n1) && echo ":0.0") | tr -d "\n")"

출력:
:0.0111.111.111

~~~

<a id="solution"></a>

# 해결 방법

**tr -d "\\r"** 명령을 사용하여 CR 부분을 미리 제거한다.
 
 ~~~console
echo "$((echo $(ipconfig.exe | grep -a 'Wi-Fi' -A 4 | cut -d":" -f 2 | tail -n1 | tr -d "\r") && echo ":0.0") | tr -d "\n")"

출력:
111.111.111.111:0.0

~~~

<a id="summary"></a>

# 정리

1. WSL에서는 윈도우 명령이 필요한 경우가 간혹 있다.
2. 윈도우 시스템 상에서는 newline이 **\\r\\n**로 인코딩된다는 사실을 잊지 말것 (당연한건데 간혹 까먹는 경우가 있다.)
