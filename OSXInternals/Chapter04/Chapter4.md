# <Chapter4> : Parts of the Process: Mach-O, Process, and Thread Internals

## Processes and Threads (프로세스와 스레드)
다른 선점형 멀티 태스킹 시스템과 마찬가지로 UNIX는 실행 프로그램의 인스턴스 인 프로세스 개념을 중심으로 구축되었습니다. 이런 인스턴스는 프로세스 ID(PID)에 의해 고유하게 정의됩니다. 동일한 실행 파일이 여러 인스턴스에서 동시에 시작될 수 있지만 가각 다른 PID를 갖습니다. 프로세스는 프로세스 그룹에 속할 수도 있습니다.

특정 프로세스가 아닌 그룹에 신호를 전송하여 사용자가 둘 이상의 프로세스를 제어할 수 있도록 합니다. 프로세는 setpgrp 를 호출하여 그룹에 참여할 수 있습니다.

최신 운영 체제는 더 이상 프로세스를 기본 작업 단위로 취급하지 않고 스레드와 함께 작동합니다. 스레드는 단지 고유 한 레지스터 상태이며 주어진 프로세스에 둘 이상이 존재할 수 있습니다. 모든 스레드는 가상 메모리 공간, 설명자(디스크립터) 및 핸들을 공유합니다. 프로세스 추상화는 하나 이상의 스레드 컨테이너로 유지됩니다. 

## The Process Lifecycle

![그림4-1](../../img/chapter4/chapter4-1.PNG)

프로세스는 SIDL 상태에서 수명을 시작합니다. SIDL 상태는 일시적으로 유휴 상태인 프로세스를 나타내며, 상위 프로세스에서 방금 생성되었습니다. 이 상태에서 프로세스는 여전히 "초기화"로 정의되며 메모리 레이아웃이 설정되고 필요한 종속성이 로드되는 동안 신호에 응답하거나 조치를 수행하지 않습니다. 모든 준비가 완료되면 프로세스가 실행을 시작할 수 있으며 SIDL로 돌아 가지 않습니다. 스레드는 나중에 생성 할 수 있으므로 SIDL의 프로세스는 항상 단일 스레드입니다.

프로세스가 실행 중이면 SRUN 상태입니다. 그러나 이 상태는 실제로 실행 가능 및 실행의 두 가지 고유 상태로 구성됩니다. CPU가 다른 프로세스로 바쁘기 때문에 프로세스가 실행 대기 상태인 경우 실행할 수 있지만 실제로 실행되지는 않습니다. CPU의 레지스터가 프로세스(기술적으로는 스레드 중 하나)에 속하는 것으로 로드 된 경우에만 프로세스는 실제로 실행 상태입니다. 그러나 스케줄링은 일시적이기 때문에 커널은 두 가지 상태를 구분하지 않아도됩니다.

실행중인 프로세스의 시간 조각이 만료되었거나 우선 순위가 높은 다른 프로세스가 멈출 경우 CPU에서 “kicked out”되어 다시 큐로 돌아올 수 있습니다.

프로세스는 리소스를 기다리지 않는 한 가능한 한 오랫동안 SRUN의 실행/실행 가능 상태에서 시간을 보냅니다. 이와 관련하여 "리소스"는 일반적으로 I/O 관련 (예 : 파일 또는 장치)입니다. 리소스에는 동기화 객체(예:뮤텍스 또는 잠금)도 포함됩니다.

프로세스가 대기 중일 때는 CPU를 점유하거나 실행 큐에서 이를 고려하는 것이 의미가 없습니다. 따라서“잠자기 상태”(SSLEEP 상태)입니다. 프로세스는 리소스를 사용할 수 있을 때까지 휴면 상태가 되며, 이 시점에서 일반적으로 현재 프로세스 실행을 위해 다시 큐에 대기됩니다.

멀티 스레딩의 주요 장점은 개별 스레드 상태가 서로 다를 수 있다는 것입니다. 따라서 한 스레드가 휴면 상태 일 수 있지만 다른 스레드는 CPU에서 예약 될 수 있습니다. 스레드는 실행 가능/실행 중과 대기 (또는 "차단") 상태 사이에서 시간을 소비합니다.

특수 신호 (TSTOP 또는 TOSTOP)를 사용하여 프로세스를 중지 할 수 있습니다. 이렇게하면 프로세스가 "동결"되어 (즉, 모든 스레드가 동시에 일시 중단됨) "딥 슬립" 상태가 됩니다. 이러한 프로세스를 재개 할 수 있는 유일한 방법은 다른 신호 (CONT)를 사용하여 프로세스를 다시 실행 가능한 상태로 되돌려 스레드의 일정을 한 번 더 예약 할 수 있게하는 것입니다.

프로세스가 main()에서 리턴되거나 exit를 호출하여 수행되면 메모리에서 지워지고 효과적으로 종료됩니다. 그렇게하면 모든 스레드가 동시에 종료됩니다. 그러나 이 작업을 수행하려면 먼저 좀비 상태에서 시간을 소비해야 합니다.

## The Zombie State
좀비 상태는 언데드 상황에도 불구하고, 완전히 정상적인 상태이다. 모든 프로세스가 완전히 종료되기 직전에 무한한 시간을 소비합니다. 부모 프로세스가 자식 프로세스보다 오래 남아있으면 자식 프로세스는 좀비 상태로 빠지게 됩니다. 자식 프로세스는 빈 쉘이며 모든 리소스를 해제했지만 여전히 PID에 집착하여 프로세스 목록에서 Z상태로 표기 됩니다. 좀비상태의 프로세스는 부모 프로세스를 기다렸다가 부모가 반환(죽으면)되면 종료될 수 있다.


### pid_suspend and pid_resume
OSX는 프로세스 제어를 위해 pid_suspend 및 pid_resume이라는 두 개의 새로운 시스템 호출을 추가했습니다. 전자는 프로세스를 “동결” 시키고 후자는 “해동” 합니다. 프로세스 STOP / CONT 신호를 보내는 것과 비슷하지만 효과는 다릅니다. 
첫째, 프로세스 상태는 SSLEEP로 유지되며, 실제로는 “잠자기” 처럼 보이지만 훨씬 더 깊은 상태입니다. 이는 기존보다 낮은 레벨 (마하 작업)에서 수행되기 때문입니다. 
둘째, 이런 호출은 여러 번 사용되어 프로세스 일시 중단 횟수를 늘리거나 줄일 수 있습니다. 따라서 pid_suspend를 호출 할 때마다 pid_resume에 일치하는 호출이 있어야합니다. 일시 중단 횟수가 0이 아닌 프로세스는 일시 중단 상태로 유지됩니다.

시스템 호출은 Apple에게만 제공되며 프로토 타입은 헤더 파일에 게시되지 않으므로 <sys / syscall.h>에 시스템 호출 번호를 언급하십시오. 이전 시스템 호출 번호는 이제 파일 포트 메커니즘에 의해 사용됩니다.


## UNIX Signals
살아있는 동안 프로세스는 일반적으로 자체 비즈니스를 염두에 두고 순차적으로 실행하거나 병렬화된 방법으로 실행합니다. 그러나 신호(signal)가 발생할 수 있습니다. 신호는 일부 예외 또는 외부 이벤트를 나타내는 소프트웨어 인터럽트입니다. 모든 UNIX 시스템과 마찬가지로 OSX는 신호 개념을 지원합니다. 데이터가 포함되지 않은 (또는 일부는 단일 비트의 데이터를 포함하는) 프로그램에 대한 비동기 알림입니다. 신호는 운영 체제에서 프로세스로 전송되어 일부 상태가 발생했음을 나타내며 이 상태는 일반적으로 일부 유형의 하드웨어 결함 또는 프로그램 예외의 원인이됩니다.


## Process Basic Security
UNIX는 전통적으로 다중 사용자 시스템으로 여러 사용자가 여러 프로세스를 동시에 실행할 수 있습니다. 보안과 격리를 모두 제공하기 위해 각 프로세스는 두 가지 기본 자격 증명인 작성자 UID (사용자 ID)와 기본 그룹 ID (GID)를 유지합니다. 파일 시스템에서 setuid 또는 setgid로 표시된 실행 파일이 호출하지 않는 한 UID와 GID는 일반적으로 실제 UID와 같습니다.

Linux와는 달리 XNU에서는 setfsuid / setfsgid 시스템 호출을 지원하지 않습니다. 두 시스템 모두 파일 시스템 검사를 위해서만 위의 ID를 선택적으로 설정하지만 실제로는 유효한 ID를 유지합니다. 이 호출은 원래 NFS를 처리하기 위해 도입되었습니다. 여기서 UID와 GID는 호스트 경계를 가로 질러 전달되어야하고 종종 일치하지 않습니다.

또한 Linux와 달리 OSX는 기능(capabilities)을 지원하지 않습니다. 기능(capabilities)은 세분화와 루트 권한을 루트가 아닌 프로세스에 위임하는 최소 권한의 원칙을 적용는 유용한 메커니즘입니다.

기능(capabilities) 대신 OSX는 샌드박스를 구분하는 메커니즘에 사용되는 "자격(entitlements)"을 지원합니다. 이것들은 코드 서명과 함께 악성 응용 프로그램 및 악성 코드가 시스템에서 실행되지 않도록하는 강력한 메커니즘을 제공합니다.

## EXECUTABLES
특수하게 조작된 파일을 메모리에 로드하면 프로세스가 생성됩니다. 이 파일은 운영 체제에서 이해하는 형식이어야하며, 파일을 구문 분석하고 필요한 종속성 (예 : 라이브러리)을 설정하고 런타임 환경을 초기화하고 실행을 시작할 수 있습니다.

UNIX에서는 간단한 (chmod + x) 명령으로 모든 것을 실행 파일로 표시 할 수 있습니다. 그러나 파일을 실행할 수 있는 것은 아닙니다. 오히려, 커널에게 이 파일을 메모리로 읽고 정확한 실행 형식을 결정할 수 있는 몇 가지 헤더 서명 중 하나를 찾도록 지시합니다. 이 헤더 시그니처는 미리 정의 된 임의의 상수 값이므로 "Magic"이라고합니다. 파일을 읽을 때 "Magic"은 이진 형식에 대한 힌트를 제공 할 수 있으며, 지원되는 경우 적절한 로더 함수가 호출됩니다.

이러한 다양한 실행 형식 중에서 OSX는 현재 마지막 세 가지를 지원합니다. (인터프리터, 범용 바이너리, Mach-O) 인터프리터는 특별한 바이너리입니다. 인터프리터는 실행되는 "real" 바이너리를 가리키는 단순한 스크립트입니다. 다음으로 바이너리와 Mach-O의 두 가지 형식에 대해 알아보겠습니다.

## UNIVERSAL BINARIES
OSX에서 애플은 Universal Binary” 라는 새로운 개념을 강조했다. 이 아이디어는 완전히 이식 가능하고 어떤 아키텍처에서도 실행될 수있는 하나의 바이너리 형식을 제공하는 것이다. 원래 PowerPC 아키텍처를 기반으로 구축 된 OSX는 Intel 아키텍처로 포팅되었고 Universal Binary는 바이너리가 PowerPC 및 x86 프로세서 모두에서 실행될 수 있도록합니다.

그러나 실제로 "universal" binary는 지원하는 각 아키텍처의 아카이브에 지나지 않습니다. 즉, 지원되는 각 아키텍처마다 바이너리의 사본이 뒤따르는 간단한 헤더를 포함합니다. Snow Leopard의 대부분의 바이너리는 인텔 이미지만 포함하지만 32비트 및 64비트 컴파일된 코드를 모두 지원하기 위해 범용 형식을 사용합니다. 그러나 일부는 여전히 PowerPC 이미지를 포함합니다. Snow Leopard를 포함하여 OSX에는 Intel 기반 프로세서에서 PowerPC 에뮬레이션을 허용하는“Rosetta”라는 선택적 구성 요소가 포함되었습니다.

이 방법으로 동일한 바이너리의 여러 복사본을 포함하면 바이너리의 크기가 크게 증가합니다. 이를 통해 범용 바이너리(Universal Binary)의 크기는 점점 증가하며, 이로 인해 시장성이 떨어지지만 Fat binary라는 명칭을 얻게 됩니다. universal binary를 사용하는 도구의 이름은 lipo입니다. 특정 아키텍쳐를 추출, 제거, 교체하여 바이너리를 "씬 다운"(크기를 줄이는데) 사용 할 수 있습니다. 또한, fat header 세부 정보를 표시하는데에도 사용할 수 있습니다. 

universal binary는 디스크에서 많은 공간을 차지하지만 OSX은 기본 플랫폼에 가장 적합한 바이너리를 자동으로 선택할 수 있습니다. 바이너리가 호출되면 Mach 로더는 먼저 lipo 명령이 보여주는 것처럼 팻 헤더를 구문 분석하고 사용 가능한 아키텍처를 결정합니다. 그런 다음 가장 적합한 아키텍처만 로드합니다. 관련이 없는 것으로 간주되는 아키텍처는 메모리를 차지하지 않습니다. 이미지는 페이지 경계에 맞게 최적화되어 있고 커널은 헤더를 읽기 위해 바이너리의 첫 번째 페이지만 로드하면 효과적으로 목차 역할을 수행 한 다음 적절한 이미지를로드 할 수 있습니다.(OS에서 로드시에 해당 아키텍쳐에 맞는 코드블록(예, Mach-O 포맷)이 로딩됩니다.)

시스템은 프로세서와 가장 일치하는 cputype 및 cpusubtype으로 이미지를 선택합니다. 특히 바이너리를 아키텍처에 일치시키는 것은 <mach-o / arch.h>의 함수에 의해 처리됩니다. 아키텍처는 CPU 유형, cpusubtype 및 바이트 순서(텍스트 설명)를 보유하는 NXArchInfo 구조체에 저장됩니다.

*아카이브(Archive) : 참고용으로 생성한 데이터


## Mach-O Binaries

[Mach-O 설명](https://www.cocoadev.co.kr/87)

Window의 PE, UNIX의 ELF(Executable and Library Format)가 있다면 Mach 커널을 사용하는 OSX는 Mach-O라는 실행파일 포맷을 사용합니다. Mach-O는 <mach-o/loader.h> 헤더 파일을 사용합니다. 

헤더는 32 비트 (MH_MAGIC, # 0xFEEDFACE로 정의) 또는 64 비트 아키텍처 (0xFEEDFACF, # MH_MAGIC_64로 정의) 용으로 로더를 신속하게 결정할 수있는 매직 값으로 시작합니다. 매직값 다음에는 범용 이진 헤더에서와 동일한 기능을 제공하는 CPU 유형 및 하위 유형 필드가 있으며 이 아키텍처에서 이진을 실행하기에 적합합니다. 그 외에, 32 비트와 64 비트 아키텍처 사이의 헤더 구조에는 실질적인 차이가 없습니다.

![그림 Mach-O](../../img/chapter4/Mach-O.PNG)

Mach-O 파일은 헤더와 load commands, 세그먼트들로 구성된 데이터로 이루어져 있습니다. load commands는 OS가 어플리케이션 실행시에 라이브러리를 올리는 등의 실행에 필요한 명령어들의 집합입니다. 헤더에 대한 상세한 내용은 툴 사용법과 함께 알아 보겠습니다. 여러 객체 유형 (실행 파일, 라이브러리, 코어 파일 또는 커널 확장)에 동일한 이진 형식이 사용되므로 다음 파일 유형은 int이며 <mach-o / loader.h>에 정의 된 값을 갖습니다.

> fat-binary -> Mach-O 순서로 실행 ??


###  Using otool(1) to Investigate Mach-O Files
otool 명령은 Mach-O 파일을 조작하는 기본 유틸리티이며 ldd 또는 readelf를 통해 다른 UNIX에서 얻은 기능을 대체하는 역할을하며 Mach-O 파일에만 적용됩니다.

## Load Commands
Mach-O 헤더에는 바이너리가 호출 될 때 바이너리를 설정하고 로드하는 방법을 명확하게 지시하는 매우 자세한 지침이 포함되어 있습니다. 이 명령어 또는 "로드 명령"은 기본 mach_header 바로 다음에 지정됩니다.

각 명령 자체는 유형 길이 값입니다. 32비트 cmd값은 유형, 32비트 값 cmdsize(32 비트의 경우 4의 배수, 64 비트의 경우 8의 배수) 및 명령 ( cmdsize에 지정된 임의의 len)은 다음과 같습니다. 이러한 명령 중 일부는 커널 로더 (bsd/kern/mach_loader.c)에 의해 직접 해석됩니다. 다른 것들은 동적 링커에 의해 처리됩니다.

프로세스를 로딩할 때 커널 부분은 가상 메모리 할당, 메인 스레드 생성 및 잠재적 인 코드서명/암호화 처리 등 새로운 프로세스의 기본 설정을 담당합니다.

그러나 동적으로 링크 된 실행 파일의 경우 실제 라이브러리 로드는 LC_LOAD_DYLINKER 명령에 지정된 동적 링커에 의해 사용자 모드에서 처리됩니다.

## LC_SEGMENT and the Process Virtual Memory Setup
주 로드 명령은 LC_SEGMENT (또는 LC_SEGMENT64) 명령으로, 커널에 새로 실행 된 프로세스의 메모리 공간을 설정하는 방법을 지시합니다. 이 “세그먼트”는 Mach-O 바이너리에서 메모리로 직접 로드됩니다.


## DYNAMIC LIBRARIES
실행 파일은 거의 독립형이 아닙니다. 정적으로 링크 된 파일을 거의 제외하고 대부분의 실행 파일은 기존 라이브러리에 의존하여 동적으로 링크되며 운영 체제의 일부로 제공되거나 3rd-party에 제공됩니다.

### Launch-Time Loading of Libraries
OSX의 거의 모든 프로그램이 동적으로 연결되므로 프로세스 수가 거의 없습니다. 즉, Mach-O 이미지에는 "hole"(외부 라이브러리 및 기호에 대한 참조)이 채워져 프로그램이 시작될 때 해결됩니다. 이것은 동적 링커의 작업입니다. 이 프로세스를 “바인딩” 이라고도 합니다.

LC_DYLINKER 로드 명령에 따라 커널에 의해 동적 링커가 시작됩니다. 모든 프로그램을 이 명령의 인수로 지정할 수 있지만 일반적으로 /usr/lib/dyld 입니다. 커널이 프로세스의 진입점을 링커의 진입점으로 설정함에 새로운 프로세서를 링커가 제어할 수 있도록 합니다.

링커의 역할은 문자 그대로 “fill the hole(구멍을 채우는 것)” 입니다. 즉, 기호 및 라이브러리 종속성을 찾아서 해결해야합니다. 라이브러리가 여전히 다른 라이브러리에 종속되어있는 경우가 많으므로 이 작업은 재귀적으로 수행해야합니다. 링커는 관심있는 특정로드 명령에 대해 Mach-O 헤더를 스캔합니다. otool의  –L을 사용하여 라이브러리 종속성을 표시 할 수 있습니다.(OSX는 ldd가 다른 UNIX에서 제공하는 기능과 동일).


 (중략 - Viewing Symbols and Loading내용 조금 전 부터 다음 목차까지 어려워서.... 중략....)

## Shared Library Caches

![dyld cache format](../../img/chapter4/dyld_cache_format.PNG)

dyld는 공유 라이브러리 캐시를 지원합니다. 이런 라이브러리는 디스크의 한 파일에 사전 연결되어 저장되는 됩니다. OSX에서 dyld 공유 캐시는 /private/var/db/dyld 에 있습니다. 캐시는 단일 파일이고 dyld_shared_ cache_armv7 입니다. OSX 공유 캐시는 부수적으로 .map 파일이 있습니다.

OSX의 공유 캐시는 매우 커질 수 있습니다. OSX에는 200 개가 넘는 파일이 포함되어 있습니다. 이런 파일 안에 라이브러리와 프레임 워크를 추출하기 위해 다양한 캐시 "unpacker"를 작성했습니다.

~~~
개인 의견 : Shared Library cache를 사용하는 이유 -> 모근 시스템 라이브러리의 성능을 향상시키기 위해 큰 캐시 파일로 결합되었다. 
~~~

<<<<<< ! 중요 ! >>>>>>

## PROCESS ADDRESS SPACE
사용자 모드의 이점 중 하나는 격리된 가상 메모리입니다. 프로세스는 4GB(32 비트 OSX)의 개인 주소 공간을 제공합니다. 주소 공간은 다양한 LC_SEGMENT 명령을 사용하여 실행 파일 및 다양한 라이브러리의 세그먼트로 채워집니다.

## The Process Entry Point
모든 표준 C프로그램과 마찬가지로 OSX의 실행 파일에는 기본적으로 "main"이라는 표준 진입 점이 있습니다. 그러나 일반적인 세 가지 인수 인 argc, argv 및 envp 외에도 Mach-O 프로그램은 "apple"이라고하는 char **의 네 번째 인수를 기대할 수 있습니다.

Snow Leopard 이하의 “apple” 인수는 단일 문자열입니다.(프로그램의 전체 경로, 즉 execve() 시스템 호출의 첫 번째 인수). 프로세스가 로딩되는 동안 dyld(1)에 의해서 argument(인수)가 사용됩니다.

Lion을 시작으로 “apple” 인수가 벡터로 확장되었습니다. 벡터에는 stack_guard 및 malloc_entropy와 같은 내부 전용으로 사용하는 새로운 2개의 매개 변수가 포함되었습니다. 전자는 GCC의 "스택 프로텍터"기능 (-fstack-protector)에 의해 사용되며 후자는 malloc에 의해 사용되며,이를 사용하여 프로세스 주소 공간에 임의성을 추가합니다. 이러한 인수는 Mach-O 로딩 중 커널에 의해 임의의 값으로 초기화됩니다. 

## Address Space Layout Randomization
프로세스는 자체 가상 주소 공간에서 시작됩니다. 전통적으로 프로세스 시작은 매번 동일한 방식으로 수행되었습니다. 그러나 이는 초기 프로세스의 가상 메모리 이미지가 특정 아키텍처의 특정 프로그램에 대해 사실상 동일하다는 것을 의미했습니다. 프로세스 수명 동안에도 대부분의 할당이 동일한 방식으로 수행되어 메모리에서 주소를 매우 예측할 수 있다는 사실로 인해 문제가 더욱 악화되었습니다.

이것은 디버깅에 이점을 제공했지만 해커에게는 더 큰 혜택을 제공했습니다. 해커가 사용하는 주요 공격 벡터는 코드 삽입입니다. 메모리에서 함수 포인터를 오버라이팅해서 프로그램 실행을 입력의 일부로 제공하는 코드로 대체 할 수 있습니다. 가장 일반적으로 오버라이트 하는 방법은 버퍼 오버플로 (확인되지 않은 메모리 복사 작업으로 인해 스택의 배열 범위를 초과)이며 오버라이트된 포인터는 함수의 반환 주소입니다. 

그러나 해커는 printf() 형식 문자열과 힙 기반 오버플로를 파괴하는 등 훨씬 더 창의적인 기술을 보유하고 있습니다. 또한 사용자 포인터 또는 구조적 예외 처리기로 코드를 주입 할 수 있습니다. 여기서 중요한 점은 포인터를 덮어 쓸 대상을 결정하는 기능, 즉 주입 된 코드가 메모리에서 상주 할 위치를 안정적으로 결정하는 기능입니다.

일반적인 해킹 모토는 자바를 분석하여 악용하는 것입니다. 해커는 버퍼 오버플로, 형식 문자열 공격 또는 기타 취약성에 관계없이 취약한 프로그램을 해부하고 주소 레이아웃을 찾는데 많은 노력을 기울인 후 취약성을 안정적으로 재현하고 유사한 시스템에서 악용 할 수 있는 방법을 만듭니다.

현재 대부분의 운영 체제에서 사용되는 기술인 ASLR(Address Space Layout Randomization)은 해킹에 대한 중요한 보호 기능입니다. 프로세스가 시작될 때마다 주소 공간이 조금씩 셔플됩니다. 기본 레이아웃은 텍스트, 데이터, 라이브러리에서도 동일합니다. 그러나 정확한 주소는 다르기 때문에 해커의 주소 추측을 방해 할 수 있습니다. 이것은 커널이 Mach-O 세그먼트를 임의의 요소로 "슬라이드" 하도록하여 수행됩니다.

Leopard는 비록 매우 제한된 형태이지만 주소 공간 레이아웃 무작위화를 도입 한 최초의 OS X 버전이었습니다. 무작위화는 시스템 설치 또는 업데이트에서만 발생했으며 라이브러리 로드만 무작위화 했습니다. Snow Leopard는 약간 개선되었지만 힙과 스택은 모두 예측 가능했으며 할당 된 주소 공간은 재부팅 후에도 지속되었습니다.

Lion은 텍스트 세그먼트를 포함하여 사용자 공간에서 완전 무작위 화를 지원하는 최초의 OS X 버전입니다. Lion은 프로그램을 호출 할 때마다 텍스트 세그먼트에서 16비트 무작위화를 제공하고 다른 곳에서는 최대 20 비트 무작위 화를 제공합니다.

64비트 Mach-O 바이너리는 MH_PIE (0x00200000)로 플래그가 지정되며 바이너리에 임의의 주소로 로드 해야한다는 것을 커널에 지정합니다. 32비트 프로그램에는 여전히 무작위가 없습니다. ASLR은 선택적으로 비활성화 할 수 있지만 기본적으로 활성화되어 있습니다.

Mountain Lion은 이전 제품을 더욱 향상시키고 커널 공간에 ASLR을 도입했습니다. 커널 주소 공간 정보를 얻기 위해 새로운 시스템 호출 kas_info가 제공됩니다. ASLR은 현저히 개선되었지만 만병 통치약은 아닙니다. 해커는 여전히 해킹을 위한 영리한 방법을 찾고 있습니다. 공격에 대한 유일한 실제 보호는 보다 안전한 코드를 작성하고 자동 및 수동의 엄격한 코드 검토를받는 것입니다.

## 32-Bit (Intel)
더 이상 디폴트는 아니지만 이전 프로그램에서 또는 32 비트를 강제로 실행하여 (32 비트 주소 공간)을 계속 사용할 수 있습니다 (–arch i386으로 컴파일). 32 비트 주소 공간은 4GB로 제한됩니다 (2^32 = 4,294,967,296 바이트). 그러나 다른 운영 체제와 달리 4GB 모두 사용자 공간에서 액세스 할 수 있습니다. 커널 공간은 예약되어 있지 않습니다.

Windows는 전통적으로 커널 공간을 위해 2GB (0x80000000-) 및 Linux 1GB (0xC0000000-)를 예약합니다. 이 메모리는 프로세스에서 기술적으로 처리 할 수 있지만 사용자 모드에서 액세스하려고하면 일반 보호 오류가 발생하고 일반적으로 세그먼트 오류가 발생하여 프로세스가 종료됩니다. OSX(32 비트 모드)는 다른 접근 방식을 사용하여 커널에 고유 한 4GB 주소 공간을 할당하고 사용자 공간을 위해 상위 1GB를 비워둡니다. Windows 2/2 및 Linux 3/1 대신 OSX는 커널 및 사용자 공간 모두에 4GB를 제공합니다. 그러나 전체 주소 공간 스위치(TLB 플래시) 비용이 발생합니다. 하지만 64비트에서는 그렇지 않습니다.

## 64-Bit
64 비트는 최대 16 엑사 바이트 (16 기가 바이트)의 거대한 주소 공간을 허용합니다. 실제로는 필요하지 않지만 주소 공간을 허용합니다. 세그먼트가 서로 훨씬 멀리 떨어져있다는 점을 제외하고 레이아웃은 동일합니다. 64 비트조차도 64 비트가 아니라는 점에 유의해야 합니다. 가상 주소를 물리적 주소로 변환하는 것과 관련된 오버 헤드로 인해 인텔 아키텍처는 48 비트의 가상 주소만 사용합니다. 이는 윈도우와 리눅스에서도 동일합니다.

64비트 모드에서는 어쨌든 엄청난 양의 메모리를 사용할 수 있기 때문에 다른 운영 체제에서 사용되는 모델을 따르는 것이 타당하다. 즉, 커널의 주소 공간을 각 프로세스와 모든 프로세스에 매핑하는 것이다. 

이는 커널을 자체 주소 공간에 두고 있던 기존 OS X 모델과는 다른 것이지만, 훨씬 빠른 사용자/커널 전환(페이지 테이블을 포함하는 제어 레지스터 CR3 공유)을 가능하게 한다.

## General Address Space Layout
ASLR로 인해 프로세스의 주소 공간이 매우 유동적입니다. 그러나 정확한 주소는 작은 임의의 오프셋에 의해 "슬라이드"될 수 있지만 대략적인 레이아웃은 동일하게 유지됩니다. vmmap(프로세스 가상 메모리 주소)에 표시되지 않은 또 다른 세그먼트는 통신 페이지입니다. 이것은 커널이 모든 사용자 모드 프로세스로 커널에서 내 보낸 일련의 페이지로, 개념적으로 Linux의 vsyscall 및 vdso와 유사합니다.

## PROCESS MEMORY ALLOCATION (USER MODE)
프로그래밍의 가장 중요한 측면 중 하나는 메모리를 유지 관리하는 것입니다. 모든 프로그램은 작동을 위해 메모리를 사용하므로 적절한 메모리 관리를 통해 빠르고 효율적인 프로그램과 불량 및 결함 프로그램간에 차이를 만들 수 있습니다.

모든 시스템과 마찬가지로 OSX은 스택 기반 및 힙 기반의 두 가지 유형의 메모리 할당을 제공합니다. 스택 기반 할당은 일반적으로 스택을 채우는 프로그램의 자동 변수이므로 일반적으로 컴파일러에서 처리합니다. 동적 메모리는 일반적으로 힙에 할당됩니다. 이 용어는 사용자 모드에서만 적용됩니다. 커널 수준에서는 사용자 힙이나 스택이 없습니다. 모든 것이 페이지로 축소됩니다.


## Heap Allocations
Darwin의 LibC가 취한 접근 방식은 특히 가장 큰 클라이언트인 Objective-C 런타임에서 사용하기에 적합합니다. Darwin의 LibC는 할당 영역을 기반으로 힙 할당을위한 특수 알고리즘을 사용합니다.


 이것들은 그림 4-6의 vmmap (1) 출력과 출력 4-13에서 보여지는 작고, 작고, 큰 영역입니다. 



각 영역에는 서로 다른 의미를 가진 자체 할당자가 있으며 할당 크기에 최적화되어 있습니다. Snow Leopard 이전에는 확장 가능한 할당자가 사용되었으며 이제는 잡지 할당자가 대신합니다. 두 할당 자의 할당 논리는 상당히 비슷하지만 할당 매거진은 스레드별로 지정되므로 잠금이나 경합이 적습니다. 매거진 할당자는 거대한 영역을 제거합니다. Foundation.Framework는 NSZone으로 malloc 영역을 캡슐화합니다.

NSCreateZone / malloc_create_zone을 호출하거나 malloc_zone_t를 직접 초기화하고 malloc_zone_register를 호출하여 새 영역을 상당히 쉽게 추가 할 수 있으며 malloc을 특정 영역에서 할당하도록 (malloc_zone_malloc 호출) 재 지정할 수 있습니다. 영역의 메모리 관리 기능이 연결될 수 있습니다. 그러나 디버깅을 위해 내부 구조를 사용하고 사용자 정의 콜백을 제공하면됩니다. 그림 4-7에서 볼 수 있듯이, 내부 검사를 통해 영역 사용법, 통계 및 모든 포인터 표시를 포함하여 영역의 세부 디버깅이 가능합니다. <malloc /malloc.h> 헤더는 디버깅 및 진단에 유용한 많은 다른 기능을 제공합니다.이 중 가장 강력한 기능은 malloc_get_all_zones ()이며, 대부분의 다른 기능과 달리 외부 메모리 모니터링을 위해 프로세스 외부에서 호출 할 수 있습니다.

Snow Leopard 이상은 libcache 및 Cocoa의 NSPurgeableData의 기반이되는 제거 가능 영역을 지원합니다. Lion은 배출 포인터 및 VM 압력 완화 기능을 추가로 지원합니다. VM 압력은 XNU (보다 정확하게는 Mach)의 개념으로, 시스템에 RAM이 부족함 (즉, 너무 많은 페이지가 상주 함)을 사용자 모드에 알립니다. 그러면 압력 해제 메커니즘이 시작되어 제공된 바이트 목표를 자동으로 해제하려고 시도합니다. RAM은 VM 압력 메커니즘이 Linux의 OOM (Out-OfMemory) 킬러와 유사한 메커니즘 인 Jetsam에 연결된 iOS에서 특히 중요합니다. 대부분의 objective-C 개발자는 didReceiveMemoryWarning을 구현할 때이 메커니즘과 인터페이스하여 가능한 한 많은 메모리를 비우고 Jetsam에 의해 무자비하게 죽이지 않도록기도합니다.

??



## Virtual Memory — The System admin Perspective

### Page Lifecycle
물리적 메모리 페이지는 표 4-10 및 그림 4-8에 표시된 것처럼 여러 상태 중 하나로 생활을 보냅니다.

![Page Life Cycle](../../img/chapter4/virtual_memory.PNG)

#### vm_stat(1)
vm_stat 유틸리티는 커널 내부 가상 메모리 카운터를 표시합니다. Mach 코어는 이러한 통계를 vm_statistics64 구조체로 유지하므로 이 유틸리티는 단순히 커널에서 통계를 요청하여 인쇄합니다.

~~~ 
morpheus@ergo (/)$ vm_stat
Mach Virtual Memory Statistics: (page size of 4096 bytes)
Pages free: 5366.
Pages active: 440536.
Pages inactive: 267339.
Pages speculative: 19096.
c04.indd 141 10/1/2012 5:57:07 PM
142 x CHAPTER 4 PARTS OF THE PROCESS: MACH-O, PROCESS, AND THREAD INTERNALS
Pages wired down: 250407.
"Translation faults": 18696843.
Pages copy-on-write: 517083.
Pages zero filled: 9188179.
Pages reactivated: 98580.
Pageins: 799179.
Pageouts: 42569.
~~~

vm_stat 유틸리티는 다양한 수명주기 단계의 페이지 수를 나열하고 부팅 이후 누적 통계를 표시합니다. 여기에는 다음이 포함됩니다.

* Translation faults: 페이지 오류 수
* Pages copy-on-write:  COW 오류로 인해 복사 된 페이지 수
* Pages zero fi lled:  할당 및 초기화 된 페이지
* Pageins: 페이지에서 페이지 가져오기
* Pageouts: 페이지 푸시 (교환)

#### sysctl(8)
커널 변수를 보고 토글하기 위한 UNIX 표준명령인 sysctl 명령을 사용하여 가상 메모리 설정을 관리 할 수 있습니다. 구체적으로 vm 네임스페이스는 다음과 같은 변수를 보유합니다.

![](../../img/chapter4/sysctl.PNG)

#### dynamic_pager(8)
OSX은 커널 수준에서 직접 관리되지 않는다는 점에서 독특합니다. 대신 dynamic_pager라는 전용 사용자 프로세스가 모든 스와핑 요청을 처리합니다. dynamic_pager는 디스크에서 스왑 공간을 관리합니다. dynamic_pager에는 고유 한 속성 목록 파일 (Library/Preferences/com.apple.virtualMemory.plist)이 있습니다. 위에서 언급 한 sysctl명령을 사용하여 vm.swapusage를 통해 스왑 사용률을 볼 수 있습니다.

## Threads
OSX는 스레드만을 참조합니다. Apple은 다른 운영 체제보다 훨씬 더 풍부한 API를 지원하여 여러 스레드로 작업을 용이하게 합니다.

## Unraveling Threads
원래 UNIX는 다중 처리 운영 체제로 설계되었습니다. 프로세스는 기본 실행 단위이자 실행에 필요한 다양한 리소스 컨테이너(가상 메모리, 파일 설명자)였습니다. 개발자는 진입점(main)부터 시작하여 main 함수가 반환되거나 exit가 호출될 때 끝나는 순차적인 프로그램을 작성했습니다. 따라서 순차적인 프로그램을 작성했습니다. 

그러나 너무 엄격한 방식으로 접근하여 동시에 실행해야하는 작업에 대해 유연성을 거의 제공하지 못했습니다. 그 중 가장 중요한 것은 I/O 였습니다. 특히 소켓에서 작동할 때, read와 write는 무한정 차단될 수 있습니다. 예를 들어, 블로킹의 경우 소켓코드가 읽기(read)를 기다리는 동안 데이터 전송을 할 수 없습니다. select, poll 시스템콜은 프로세스가 모든 파일 디스크립터를 하나의 배열 넣을 수 있게하여 I/O 멀티플렉싱을 제공하였습니다. 그러나 이런 방식으로 코딩하는 것은 확장 가능하지 않거나 비효율적입니다. 
다른 고려 사항은 프로세스가 I/O를 느리게 차단하기 보다는 빠르게 차단하는 것입니다. 이는 프로세스 타임 슬라이스의 많은 부분이 손실된다는 것을 의미합니다. 프로세스 Context 전환 비용이 비싸기 때문에 이는 성능에 큰 영향을 미칩니다.

따라서 스레드는 주로 프로세스 타임 슬라이스를 최대화하기 위한 수단으로 도입되었습니다. 여러 스레드를 사용하면 실행이 겉보기 동시애 처리되는 것처럼 보이는 하위 작업으로 분할 될 수 있습니다. 하나의 하위 작업이 차단되면 나머지 타임 슬라이스는 다른 하위 작업에 할당 됩니다. 또한, 폴링이 더 이상 필요하지 않습니다. 한 스레드는 단순히 읽기만을 차단하고 데이터를 무한정 기다릴 수 있는 반면, 다른 스레드는 쓰기나 다른 작업과 같은 이외의 작업을 계속 할 수 있습니다.

대부분의 운영 체제가 스케줄링 정책을 프로세스가 아닌 스레드로 전환하는 것이 더 합리적이었습니다. 스레드 간 전환 비용은 최소한으로, 레지스터 상태를 저장하고 복원하는 것입니다. 이와 반대로 프로세스에는 플래시 캐시와 같은 저수준 오버 헤드 및 TLB (Translation Lookaside Buffer)를 포함하여 가상 메모리 공간 전환도 포함됩니다.

코어는 동일한 캐시와 RAM을 공유하므로 스레드간에 가상 메모리를 쉽게 공유 할 수 있기 때문에 여러 코어가 스레드에 적합합니다. 대조적으로, 다중 프로세서는 비균일 메모리 아키텍처 및 캐시 일관성 고려 사항으로 인해 어려움이 있습니다.

UNIX 시스템은 POSIX 스레드 모델을 채택했습니다. Windows는 자체 API를 선택했습니다. Mac OSX는 자연스럽게 UNIX의 발자취를 따랐지만 Objective-C와 그랜드 센트럴 디스패처 (Grand Central Dispatcher)같은 고급 API 도입으로 몇 단계 더 나아갔습니다.

## POSIX Threads

POSIX 스레드 모델은 Windows를 제외한 모든 시스템에서 효과적으로 표준 스레딩 API입니다. OSX는 다른 운영 체제보다 더 많은 pthread를 지원합니다. 간단한 man –k pthread는 <pthread.h>에서 볼 수 있듯이 지원되는 기능의 범위를 나타냅니다.
다른 시스템과 마찬가지로 pthread API는 커널이 스레드를 작성하도록 지시하는 기본 시스템 호출에 맵핑됩니다. 표는 이 매핑을 보여줍니다. 다른 운영 체제와 달리 XNU에는 pthread의 동기화 객체를 커널 모드 (통칭하여 psynch라고 함)에서 관리 할 수 있도록 하는 특정 시스템 호출도 포함되어 있습니다. 따라서 객체를 사용자 모드로 두는 것보다 스레드 관리가 더 효율적입니다. 그러나 이러한 호출이 반드시 활성화 될 필요는 없습니다 (커널에서 조건부 컴파일 됨). 
libSystem은 동적으로 검사하고 (지원되는 경우) “old” pthread 대신 new _pthread_* 함수를 사용합니다(예 : new_pthread_mutex_init, new_pthread_rwlock_rdlock 등). psynch API가 반드시 지원되는 것은 아닙니다.

## Grand Central Dispatch
Snow Leopard는 GCD (Grand Central Dispatch)라는 다중 처리를위한 새로운 API를 소개합니다. Apple은 스레드 대신이 API를 홍보합니다. 이것은 패러다임의 전환을 제시합니다. 스레드와 스레드 함수를 생각하기 보다는 기능 블록에 대해 생각하는 것이 좋습니다.
GCD는 동시 및 비동기 실행 모델을 지원하기 위해 기본 스레드 풀 구현을 유지 관리하므로 개발자는 동시성 문제 및 교착 상태와 같은 잠재적인 함정을 처리할 필요가 없습니다. 
이 메커니즘은 또한 신호 및 마하 메시지와 같은 다른 비동기 알림을 처리 할 수 있습니다. Lion은 이를 비동기 I/O를 지원하기 위해 확장합니다. GCD 사용의 또 다른 이점은 시스템이 사용 가능한 논리 프로세서 수에 따라 자동으로 확장된다는 것입니다.

개발자는 작업 단위를 기능 또는 기능 블록으로 구현합니다. C블록과 매우 유사한 기능 블록은 중괄호로 묶이지만 C함수와 같이 (별표 (*)가 아닌 캐럿 (^)이 있음)을 가리킬 수 있습니다. 디스패치 API는 어느 쪽이든 잘 작동 할 수 있습니다.

* The global dispatch queues : 글로벌 디스패치 큐는 dispatch_get_ global_queue ()를 호출하고 요청 된 우선 순위를 지정하여 애플리케이션에 사용 가능합니다 (DISPATCH_QUEUE_PRIORITY_ DEFAULT, _LOW 또는 _HIGH).
* The main dispatch queue : Cocoa 애플리케이션의 런 루프와 통합됩니다. dispatch_get_main_queue ()를 호출하여 검색 할 수 있습니다.
* Custom queues : dispatch_queue_create ()를 호출하여 수동으로 작성하면 디스패치에 대한 제어를 강화하는 데 사용할 수 있습니다. 이들은 직렬 대기열 (작업이 FIFO가 실행되는) 또는 동시 대기열 일 수 있습니다.

Grand Central Dispatch의 API는 모두 <dispatch / dispatch.h>에 선언되어 있으며 libSystem 내부의 libDispatch.dylib에 구현되어 있습니다. API 자체는 pthread_workqueue API를 통해 빌드되며 XNU는 workq 시스템 호출로 지원합니다. Objective-C는 NSOperation 관련 객체에 의해 노출 된 API로 이러한 API를 추가로 래핑합니다.
