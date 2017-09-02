## 2. collision

`ssh col@pwnable.kr -p2222 (pw:guest)`

### MD5

[출처1](https://ko.wikipedia.org/wiki/MD5) [출처2](https://namu.wiki/w/MD5)

MD5(Message-Digest algorithm 5)는 128비트 암호화 해시 함수로 주로 프로그램이나 파일이 원본 그대로인지를 확인하는 무결성 검사 등에 사용된다. 

MD5는 설계상 오류가 발견되어 현재는 MD5 알고리즘을 보안 관련 용도로 쓰는 것은 권장하지 않으며, 심각한 보안 문제를 야기할 수도 있다. 다만 고속 연산이 가능하고 (정수 연산 및 비트 시프트 연산만 사용한다.) 임의로 변경된 패턴에 대해서는 충돌 가능성이 충분히 낮기 때문에, 현재는 주로 네트워크로 전송된 큰 파일의 무결성을 확인하는데 사용한다.

단방향 암호화이기 때문에 MD5 hash 값에서 원래 데이터를 찾아내는 것은 불가능하지만 같은 MD5를 갖는 문자열, 즉 충돌(collision)이 발생할 수 있다. MD5 값이 같으면 데이터가 다르더라도 같은 문자열이라고 판단하기 때문에 문제가 될 수 있다. 이는 모든 단방향 암호화에 통용되는 기법이다.

### 서버에 있는 파일 살펴보기

#### col.c

```c
#include <stdio.h>

#include <string.h>

unsigned long hashcode = 0x21DD09EC;

unsigned long check_password(const char* p){

        int* ip = (int*)p;

        int i;

        int res=0;

        for(i=0; i<5; i++){

                res += ip[i];

        }

        return res;

}

int main(int argc, char* argv[]){

        if(argc<2){

                printf("usage : %s [passcode]\n", argv[0]);

                return 0;

        }

        if(strlen(argv[1]) != 20){

                printf("passcode length should be 20 bytes\n");

                return 0;

        }

        if(hashcode == check_password( argv[1] )){

                system("/bin/cat flag");

                return 0;

        }

        else

                printf("wrong passcode.\n");

        return 0;

}

```

매개변수없이 코드를 실행할 경우 "usage : [passcode]"라는 문장이 출력되고 매개변수 문자열의 길이가 20이 아닐 경우 "passcode length should be 20 bytes"라는 문장이 출력된다. 만약 우리가 입력한 매개변수 문자열을 `check_password`함수에 넣은 결과값이 `hashcode`값인 0x21DD09EC와 같으면 `/bin/cat flag`를 볼 수 있다.

`check_password` 함수를 살펴보면 char형을 포인터인 `p`를 정수형 포인터로 형변환한다. `passcode`의 길이는 20byte이고 정수형 포인터`ip`는 4byte이기 때문에 `ip`에 저장되어있는 값 5개를 더해서 `hascode` 값인 0x21DD09EC로 만들어주면 된다.

#### res값 넣어주기

`res` 값은 정수형 다섯개를 더해서 구해진다. `res` 에 들어가야할 0x21DD09EC를 5로 나누면 0x6C5CEC8가 나온다. 나머지가 발생하기 때문에 0x6C5CEC8를 네 번 더한 후 0x6C5CECC를 한 번 더해주면 0x21DD09EC가 나오게 된다.

:arrow_forward: 0x21DD09EC = 0x6C5CEC8 * 4 + 0x6C5CECC

그런데 `ip` 에 넣어야 할 값은 아스키 코드 범위를 넘기 때문에 값 대입으로 넣어줄 수 없다. 그래서 코드로 직접 넣어주되 시스템은 little-endian을 따르기 때문에 little-endian방식으로 넣어주어야 한다.

결과적으로 다음과 같은 방법으로 넣어주면 `/bin/cat flag`를 볼 수 있다.

```bash
./col $(perl -e 'print "\xc8\xce\xc5\x06"x4 . "\xcc\xce\xc5\x06"')
```

```bash
./col python -c 'print "\xc8\xce\xc5\x06"*4+"\xcc\xce\xc5\x06"'
```

### 아스키 코드

[출처1](https://ko.wikipedia.org/wiki/%EB%AF%B8%EA%B5%AD%EC%A0%95%EB%B3%B4%EA%B5%90%ED%99%98%ED%91%9C%EC%A4%80%EB%B6%80%ED%98%B8) [출처2](https://namu.wiki/w/%EC%95%84%EC%8A%A4%ED%82%A4%20%EC%BD%94%EB%93%9C)

미국정보교환표준부호(American Standard Code for Information Interchange; ASCII)는 영문 알파벳을 사용하는 대표적인 문자 인코딩이다. 아스키는 컴퓨터와 통신 장비를 비롯한 문자를 사용하는 많은 장치에서 사용되며, 대부분의 문자 인코딩이 아스키에 기초를 두고 있다.

아스키 코드는 미국에서 표준화한 정보교환용 7비트 부호체계이다. 000(0x00)부터 127(0x7F)까지 총 128개의 부호가 사용된다. 이는 영문 키보드로 입력할 수 있는 모든 기호들이 할당되어 있는 부호 체계이며, 매우 단순하고 간단하기 때문에 어느 시스템에서도 적용가능하다는 장점이 있으나, 2바이트 이상의 코드를 표현할 수 없다.