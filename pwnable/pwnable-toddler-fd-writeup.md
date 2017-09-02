## 1. fd

`ssh fd@pwnable.kr -p2222 (pw:guest)`

### SSH

[출처](https://ko.wikipedia.org/wiki/%EC%8B%9C%ED%81%90%EC%96%B4_%EC%85%B8)

시큐어 셸(Secure Shell, SSH)는 네트워크 상의 다른 컴퓨터에 로그인하거나 원격 시스템에서 명령을 실행하고 다른 시스템으로 파일을 복사할 수 있도록 해 주는 응용 프로그램 또는 그 프로토콜을 가리킨다. 기존의 rsh, rlogin, 텔넷 등을 대체하기 위해 설계되었으며, 강력한 인증 방법 및 안전하지 못한 네트워크에서 안전하게 통신을 할 수 있는 기능을 제공한다.

SSH는 암호화 기법을 사용하기 때문에, 통신이 노출된다 하더라도 이해할 수 없는 암호화된 문자로 보인다.

SSH의 주요 기능은 다음과 같다.

- 보안 접속을 통한 rsh, rcp, rlogin, rexec, telnet, ftp 등을 제공
- IP spoofing (IP스푸핑, 아이피 위/변조 기법중 하나)을 방지하기 위한 기능을 제공
- X11 패킷 포워딩 및 일반적인 TCP/IP 패킷 포워딩을 제공

pwnable.kr의 문제에서는 다음과 같은 형식을 사용해서 SSH에 접속하도록 한다.

`ssh fd@pwnable.kr -p2222`

여기서 `fd`는 원격 서버의 유저명을(pwnable에서는 문제의 이름을 유저명으로 사용하고 있다.), `pwnable.kr`은 원격 서버의 도메인을 나타낸다. `-p2222`는 포트 번호이다.

### 원격 서버에 접속하기

터미널에서 `ssh fd@pwnable.kr -p2222`를 치고 Are you sure you want to continue connecting (yes/no)? 라는 질문이 나오면 `yes`를 친다. password는 문제에서 알려준 `guest`를 치면 원격 서버에 접속이 된다.

### 서버에 있는 파일 살펴보기

터미널에서 `ls -l`을 치면 파일을 읽을 수 있는 권한을 볼 수 있다. root 권한으로만 볼 수 있는 파일의 경우 permission denied라는 문구가 뜨면서 내용을 볼 수 없다.

#### fd.c

이 문제는 toddler's bottle이기 때문에 c 파일을 제공한다. 

```c
#include <stdio.h>

#include <stdlib.h>

#include <string.h>

char buf[32];

int main(int argc, char* argv[], char* envp[]){

        if(argc<2){

                printf("pass argv[1] a number\n");

                return 0;

        }

        int fd = atoi( argv[1] ) - 0x1234;

        int len = 0;

        len = read(fd, buf, 32);

        if(!strcmp("LETMEWIN\n", buf)){

                printf("good job :)\n");

                system("/bin/cat flag");

                exit(0);

        }

        printf("learn about Linux file IO\n");

        return 0;

}
```

코드를 살펴보면 fd를 실행하기 위해서는 매개 변수가 필요한 것을 알 수 있다. fd는 매개 변수를 입력받은 뒤 매개 변수가 2보다 작다면 "pass argv[1] a number" 구문을 출력한다. 2 이상일 경우에는 `atoi()`를 이용해 정수로 변환한 후에 16진수 `0x1234`(10진수 4660)을 빼서 정수형 변수 fd에 저장한다. 이 값은 `read(fd, buf, 32)`에서 file descriptor 값으로 사용 된다. 

이 다음 코드를 보면 buf에 저장된 문자열이 "LETMEWIN"과 같아야 실행되기 때문에 (strcmp는 문자열 비교 후 두 문자열이 같을 때 0을 반환한다.) buf에 "LETMEWIN"이라는 문자열을 넣어주어야 하는 것을 알 수 있다. buf에 문자열을 넣기 위해서는 file descriptor 값이 standard input인 0이어야 한다. 

file descriptor의 값이 0이기 위해서는 정수형 변수 fd의 값이 0이 되어야 하기 때문에 fd 실행파일을 실행할 때 매개변수를 4660으로 넣어주면 된다. 그러면 `if(!strcmp("LETMEWIN\n", buf))`문이 실행되어 `/bin/cat flag`를 볼 수 있다. 이 flag 값 정답이 된다.

### file descriptor

[출처](http://mintnlatte.tistory.com/266)

file descriptor는 시스템으로부터 할당받은 파일이나 소켓을 대표하는 정수이다. 표준 입력 및 표준 출력도 file descriptor로 표현되는데 이들은 프로그램이 시작되면 기본적으로 열리고, 종료시 자동으로 닫히게 된다.

- 0: Standard Input(표준 입력)
- 1: Standard Output(표준 출력)
- 2: Standard Error(표준 에러)

따라서 파일을 열거나 소켓을 생성할 때 부여되는 file descriptor는 3부터 시작된다.

### read

[출처](http://forum.falinux.com/zbxe/index.php?document_srl=466628&mid=C_LIB)

`ssize_t read (int fd, void *buf, size_t nbytes)`

* int fd : 파일 디스크립터
* void *buf : 파일을 읽어 들일 버퍼
* size_t nbytes : 버퍼의 크기

open() 함수로 열기를 한 파일의 내용을 읽는다. 정상적으로 실행되었다면 읽어들인 바이트 수를, 실패했다면 -1을 반환한다.
