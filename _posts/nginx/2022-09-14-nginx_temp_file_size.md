---
title:  "nginx에서 1GB 넘는 파일이 받아지지 않은 이유"

excerpt: "nginx"

categories:
- nginx

tags:
- nginx

toc: true

toc_sticky: true

toc_label: "nginx"

last_modified_at: 2022-09-14T20:59:00-05:00

---

## Problem?

nginx를 적용한 서버에서 1GB를 넘는 파일이 받아지지 않는 상황


## Situtaion?
nginx를 사용하는 스프링 어플리케이션

## Analysis

> 어플리케이션 세션 타임아웃인가?

  - 요청에 대한 응답이 늦어지기 때문에, 세션 타임아웃이 발생해 세션이 끊어진 현상으로 추측
  - 관련된 컨트롤러에서 Httpservletrequest를 주입해서 세션 타임아웃을 늘려줘도 동일한 현상
  - 또한, 로그에서도 별다른 이상이 없음
  - <mark style='background-color: #fff5b1'>어플리케이션단에서, 세션을 컨트롤하지 않는다고 결론</mark>

> nginx에 세션 타임아웃 문제인가?

  - nginx의 error.log를 확인
    - error.log의 위치는, nginx 로그 폴더에 위치
  - error.log에서,아래 로그 발견

~~~shell
[error] 90599#0: *1031804 upstream prematurely closed connection while reading upstream
~~~

  - nginx에 proxy_read_timeout(600s) 설정을 확인해봤으나, 종료된 시점보다 더 빠르게 종료됨
  - <mark style='background-color: #fff5b1'>nginx에 read_timeout 설정과는 무관하다고 결론</mark>

> 다운로드 파일이 1.2GB인데, 1GB에서 다운로드를 중지하고 에러 페이지를 반환하는 이유?

  - nginx에서 특정 용량 이상이 발생하면, 세션을 종료하는 현상
  - 다운로드 용량을 제어하는 nginx 설정인 proxy_max_temp_file_size은 존재하지 않음
  - nginx는 존재하지 않는 지시어에 대해서는 디폴트 설정이 적용될 수 있음
  - <mark style='background-color: #fff5b1'>proxy_max_temp_file_size의 Default 값은 1024MB이므로, 이 설정이 원인이 되는걸로 추측</mark>

## nginx directive?
  - [공식 문서](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_max_temp_file_size)
  
### proxy_max_temp_file_size

  ~~~shell
    Syntax :	proxy_max_temp_file_size size;
    Default:	proxy_max_temp_file_size 1024m;
    Context:	http, server, location
  ~~~

### proxy_buffering

  ~~~shell  
    Syntax:	proxy_buffering on | off;
    Default:	proxy_buffering on;
    Context:	http, server, location
  ~~~

### proxy_buffer_size

  ~~~shell
    Syntax:	proxy_buffer_size size;
    Default:	proxy_buffer_size 4k|8k;
    Context:	http, server, location
  ~~~

> nginx는 default로 buffering이 켜져있다(이 이유에 대해서는 아래서...), 버퍼링이 동작할 때 proxy_buffer_size 크기의 버퍼에 데이터가 들어가게 된다.

> temp_file_size는 proxy_buffer_size를 초과하는 데이터들을 추가적으로 저장할 수 있는 크기를 정의한다. 또한, 이 temp에도 proxy_buffer_size처럼
> 한번에 쓰여질 크기에 대해서 proxy_temp_file_write_size로 설정할 수 있다.

## Validation
  > 공식 문서에 의하면, 버퍼 사이즈를 초과하는 파일들은 temp에 쓰여진다고 했으니, temp 크기를 늘려주면 nginx에서 세션을 
  > 종료시키는 일도 없지 않을까?

  - proxy_max_temp_file_size 2048m;
  - 위 설정을 추가하고, nginx를 리로드
    - ps -ef | grep nginx로 포트 확인 후 sudo kill -9 {pid}
    - nginx/sbin 경로로 이동해, ./nginx -t 테스트 명령어를 결과 확인(OK가 존재해야함)
    - 테스트가 성공적으로 끝나면, ./nginx -s reload로 재기동
  
  - <mark style='background-color: #fff5b1'>1GB를 초과하는 파일 다운로드 후 성공 확인!</mark>

## Question
  > proxy_max_temp_file_size 무작정 늘려도 괜찮을까?

  - 서버의 용량에 따라서, 해당 크기를 늘리지 못할 수도 있음
  - <mark style='background-color: #fff5b1'>나머지 프록시 설정들을 모두 유지하기 때문에 안전한 방법</mark>
  - 이 방법을 사용하지 못한다면, 서비스단에서 고려가 필요(파일 나누기)

  > 'proxy_max_temp_file_size 0'은 어떨까?

  - 0으로 설정되면, proxy_buffering off인 상태로 데이터를 전송
  - synchronously하게 데이터를 전송함, 즉 버퍼링없이 클라이언트로 데이터 전송
   
### 버퍼링

  - 버퍼링이 없으면, 클라이언트 환경(속도)에 따라서 nginx와 업스트림 서버간 커넥션이 불필요하게 유지되어야 함
    - 클라이언트 환경이 빠르면, 버퍼링이 없어도 큰 영향이 없음
    - 클라이언트 환경이 느리다면, 응답을 임시로 저장할 수 없기 때문에 계속 커넥션이 유지되어야 하므로, 전체적인 리소스가 낭비
    - 버퍼링이 있으면, nginx가 응답을 임시(버퍼, temp)로 저장한 다음에 업스트림 커넥션이 더 빨리 닫아질 수 있음 
    
  - 이 설정은 클라이언트가 모두 빠르다고 가정하고 사용해야 하는 설정
  - 버퍼링으로 인한 side-effect가 발생할 수 있음

## Reference
  - [nginx 버퍼링](https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching)
  - [nginx 공식문서](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_max_temp_file_size)
