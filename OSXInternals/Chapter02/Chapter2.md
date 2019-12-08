# <Chapter2> E Pluribus Unum : OSX 및 IOS의 아키텍처

*E Pluribus Unum = 많은 것 중에 하나라는 의미이다.


- OSX Architectural Overview
OS X는 이전 버전인 OS 9와 비교 할 때 많은 기술적 발전하였고 전체 운영체제의 핵심부터 새롭게 디자인하여 가장 혁신적인 운영체제로 완전히 개편되었습니다. OSX는 GUI와 프로그래머 API 측면에서 새로운 그냥을 제공하고 있으며 윈도우와 리눅스로 빠르게 포팅되고 있다.

- The User Experience Layer         : Aqua, Dashboard, Spotlight와 같이 사용자의 편의를 지원하기 위한 계층
- The Application Frameworks Layer  : Cocoa, Carbon, Java와 같은 API 계층
- The Core Framworks                : 핵심프레임워크, Open GL, QuickTime 등등, 그래픽과 미디어와 관련된 계층
- Darwin                            : OS 코어, 커널과 UNIX 쉘 환경

Darwin은 오픈 소스이며 나머지 시스템의 기초 및 low-level의 API 역할을 한다. 하지만 Darwin의 최상위 계층은 비공개 소스이며, 이는 Apple의 소유이다.

(그림2) (그림1)

그림2는 단순하게 표현한 OSX의 구조이다. 하지만 각각의 층(레이어)의 구성요소는 더 세분화 될 수 있다. (그림1)과 같은 세분화된 Darwin의 구조에 대해 중점을 두고자 한다.

그림(1)은 Darwin의 실제구조로 XNU 커널로 구성되어있다. XNU는 Mach과 BSD, I/O Kit 3가지 요소로 구성되어있다.

대부분의 사용자 모드 응용프로그램은 특히 오브젝트C로 코딩된 프레임워크(Cocoa)와 인터페이스만 있으면 된다. 따라서 대부분의 OSX 및 IOS 개발자는 하위 계층, Darwin 및 커널에 대해 잘 몰라도 된다. 유저 모드 계층은 애플리케이션 별로 개별적으로 액세스할 수 있다. 커널에서는 기기 드라이버 개발자가 사용할 수 있는 구성 요소가 상당히 많다.



## The User Experience Layer
OSX에서 user interface는 사용자 경험입니다. OSX는 혁신적인 기능을 가지고 있고 발전한 인터페이스는 모방의 대상이 되었습니다. 다른 GUI 기반 운영체제의 많은 영향을 미쳤습니다.

- Aqua
- Quick Look
- Spotlight
- Accessibility options


### Aqua
아쿠아는 OSX의 친숙하고 독창적인 GUI입니다. 시스템의 유저모드에서 가장 먼저 실행되는 프로세스이며 GUI를 시작하는 역할을 가지고 있습니다.

### Quick Look
Quicklook 스페이스바를 통해 Finder에서 파일 유형을 빠르게 미리보는 기능입니다. 

### Spotlight
Spotlight는 빠른 검색 기술입니다. Finder에 완벽하게 통합되었고 빠르게 파일을 찾을 수 있습니다.

## Darwin - The UNIX CORE
OSX의 Darwin란 이름으로 UNIX 기반입니다. Apple은 Darwin의 UNIX 기능을 강조한 특별 문서를 유지 관리합니다. 그러나 대부분의 사용자에게는 UNIX 인터페이스가 완전히 숨겨져 있습니다. GUI 환경은 기본 UNIX 디렉토리를 잘 숨 깁니다. 

## The Shell
command line에 접근하는 것은 간단합니다. 터미널 응용프로그램은 UNIX 셸을 사용해서 터미널 에뮬레이터를 실행합니다. 기본적으로 bash 셸을 제공합니다. 이외에도 sh, bash, csh, tcsh, zsh, ksh 셸을 지원합니다.

OSX의 Command Line은 텔넷 또는 SSH를 통해 원격으로 접근이 가능합니다. 둘 다 기본적으로 비활성화 되었으며 텔넷은 안전하지 않고 암호화되지 않으므로 사용하지 않는 것이 좋습니다. .plist 파일(telnet.plist, ssh.plist)을 편집하여 텔넷이나 SSH를 손쉽게 사용할 수 있습니다. 이를 위해서는 sudo를 사용하여 먼저 권한을 취득해야 합니다.

## The File System
OSX는 Hierarchical FiSystem Plus(계층 파일 시스템, HFS+)를 사용합니다. "Plus"는 HFS+가 OSX 이전에 일반적으로 사용되었던 파일 시스템의 후속 버전임을 나타냅니다.

대소문자 구분 : HFS+는 항상 대소문자를 구분합니다. 하지만 경우에 따라 대소문자를 구분하지 않는 경우도 있습니다.

저널링 : HFS+에서는 선택적으로 저널을 사용할 수 있다. 저널을 사용하는 경우, 파일 시스템의 트랜잭션을 기록하여 파일 시스템을 보다 강력하게 구성할 수 있다. 시스템 장애등 스토리지에 문제가 생겼을 경우, 데이터는 손실될 수 있지만 파일 시스템은 일관된 상태일 가능성이 높습니다.

HFS 파일 시스템에는 확장 속성 및 투명 압축과 같은 고유한 기능이 있습니다. 그러나 프로그래밍적으로 HFS+에 대한 인터페이스는 다른 파일 시스템과 동일합니다. 커널에 의해 노출된 API는 실제로 가상 파일 시스템 스위치(VFS)이라는 일반적인 파일 시스템 계층을 통해 제공됩니다. VFS는 UNIX 기반 및 외부 커널의 모든 파일 시스템에 대해 균일한 인터페이스이다. HFS+는 사용자에게 일반적인 UNIX와 같은 파일 시스템의 사용자 경험을 제공합니다.


## UNIX System Directories
OSX는 UNIX에서 표준으로 사용되는 디렉토리를 사용한다.
- /bin: Unix binaries. 일반적인 유닉스 명령어 ( 예_ls, rm, mv, df)가 있다.
- /sbin: System binaries. 파일시스템, 네트워크 구성 등 시스템 관리에 사용되는 명령어
- /usr: The User directory. 사용자를 위한 것이 아니라 다른 소프트웨어를 위한 것, 윈도우의 Program files와 같은 역할을 한다.
- /usr: bin, sbin, and lib. /usr/lib에는 공유 객체가 존재한다.(Windows DLLs, \windows\system32 과 같은). 이 디렉터리에는 모든 표준 c헤더가 있는 [include/] 서브 디렉터리도 포함됩니다.
- /etc: 시스템 구성 파일을 포함하는 디렉토리. 예) 비밀번호 파일, /private/etc
- /dev: BSD device files. 시스템의 하드웨어 장치를 나타내는 파일
(character and block devices).
- /tmp: 임시 디렉토리 /private/tmp.
- /var: 로그 파일, 메일 저장소, 인쇄 스풀 및 기타 데이터의 디렉토리입니다. /private/var

OSX에서 UNIX 디렉터리는 Finder에서 보이지 않습니다. BSD의 시스템콜을 사용하여 "hidden"이라는 파일 속성을 통해서 GUI에서 숨겨집니다.
(그림 - 파일시스템)


## OSX-Specific Directories(OSX 특정 디렉터리)

OSX는 System Root 아래에 자신만의 특수한 디렉터리(UNIX tree)를 추가합니다.
- /Applications: 시스템의 모든 응용프로그램이 존재하는 곳입니다.
- /Developer: Xcode를 설치하는 경우, 모든 개발자 도구의 기본 설치 지점입니다.
- /Library: 시스템 응용프로그램의 date file, help, documentation이 있습니다.
- /Network: 인접 노드 발견 및 접근을 위한 가상 경로입니다.
- /System: 시스템파일(프레임워크, 커널 모듈, 폰트)와 같은 시스템의 주요 구성요소가 존재합니다.
- /Users: 유저(사용자)가 사용하는 디렉터리 입니다.
- /Volumes: 이동식 디스크 및 네트워크 파일의 마운트 지점입니다.
- /Cores: Core Dump 디렉터리, Core Dump가 충돌 했을 때 작성되며 프로세스의 Core 가상 메모리 이미지가 포함됩니다.


## Bundles(번들)
번들은 NextStep에서 시작되었고 모바일 앱과 함께 OSX의 핵심 아이디어입니다. 번들의 개념은 애플리케이션뿐만 아니라 프레임워크, 플러그인, 위젯, 커널 확장(Kext)까지 모두 번들로 패키지화 기본 개념입니다. 따라서 애플리케이션의 특정 프레임워크를 논의하기 전에는 번들을 일시정지 해야합니다. 
Apple은 번들을 "실행가능한 코드와 해당 코드에서 사용하는 리소스를 보유하는 표준화된 계층 구조"로 정의합니다. 특정 번들의 유형은 다를 수 있으며 내용은 다양하지만, 모든 번들은 동일한 디렉토리 구조를 가집니다.


## Applications And Apps (응용프로그램)
애플리케이션은 번들로 깔끔하게 패키지됩니다. 응용 프로그램 번들에는 응용 프로그램 런타임에 필요한 대부분의 파일 (기본 바이너리, 개인 라이브러리, 아이콘, UI 요소 및 그래픽)이 포함되어 있습니다.

번들은 Finder에 단일 아이콘으로 표시되므로 사용자는 이에 대해 거의 알지 못합니다. 이를 통해 Mac OS에서 설치가 간편합니다. 응용 프로그램 아이콘을 응용 프로그램 폴더로 드래그하면됩니다. 응용 프로그램 내부를 들여다 보려면 마우스 오른쪽 버튼을 사용해야합니다.

OSX에서 응용 프로그램은 일반적으로 /Applications 폴더에 있습니다. 각 애플리케이션은 AppName.app라는 자체 디렉토리에 있습니다. 여기서 애플리케이션의 리소스는 클래스에 따라 별도의 하위 디렉토리로 그룹화됩니다.

## Info.plist
Info.plist 파일은 애플리케이션의 Contents/ 하위 디렉토리에 있으며 번들의 메타 데이터를 보유하며 필요한 정보를 제공하므로 필수 파일입니다. 번들은 OS가 dependency(종속성) 및 properties(기타 속성)을 결정할 수 있습니다. Info.plist 파일은 3가지 형식 중 하나로 저장됩니다.

- XML : 파일의 시작 부분에 XML 서명과 DTD로 사람이 쉽게 읽을 수 있습니다. OSX의 일반적인 형식이며 키/값의 사전형식의 배열로 구성되어 있습니다.
- Binary : 이 파일은 컴파일 된 plist로, 사람이 읽을 수는 없지만 복잡한 XML 구문 분석 및 처리가 필요하지 않으므로 OS에 훨씬 최적화되어 있습니다. 
- JSON : JavaScript 객체 표기법을 사용하여 키/값은 쉽게 읽고 분석 할 수있는 형식으로 저장됩니다. 이 형식은 XML 또는 이진 형식과 동일하지 않습니다.


## Resource
Resources 디렉토리에는 응용 프로그램 사용에 필요한 모든 파일이 들어 있습니다. 이것은 번들 형식의 가장 큰 장점 중 하나입니다. 리소스를 실행 파일로 컴파일해야하는 다른 운영 체제와 달리 번들을 사용하면 리소스를 별도로 유지할 수 있습니다. 따라서 실행 파일이 훨씬 얇아 질뿐만 아니라 재 컴파일 할 필요없이 리소스를 선택적으로 업데이트하거나 추가 할 수 있습니다.


## NIB File
응용 프로그램의 GUI 구성 요소의 위치 및 설정을 포함하는 이진 plist입니다. XCode의 인터페이스 빌더를 사용하여 텍스트 버전을 .xib로 편집하여 바이너리 형식으로 패키징하기 전에 (더 이상 편집 할 수없는 시점) 작성합니다.

## Internationalization with .lproj Files
번들은 국제화를 지원합니다. 이는 각 언어의 하위 디렉토리에 의해 수행됩니다. 언어 디렉토리는 확장자가 .lproj입니다.

## Icons (.icns)
응용 프로그램에는 일반적으로 시각적 표시를위한 하나 이상의 아이콘이 있습니다.

## CodeResources
응용 프로그램에 포함 된 마지막으로 중요한 파일은 CodeResources이며 이는 _CodeSignature/ 경로에 있습니다. 이 파일은 번들의 다른 모든 파일 목록을 포함하는 속성 목록입니다. 속성 목록은 키가 파일 이름이고 값이 일반적으로 해시인 Base64 형식의 사전형식의 파일 단일 항목입니다.
또한, CodeResources 파일은 응용 프로그램이 손상되지 않았는지 또는 손상되었는지 확인하고 우발적 인 수정 또는 리소스 손상을 방지합니다.

## Application default settings(응용 프로그램 기본 설정)
다른 운영체제와 달리 OSX는 응용프로그램을 위한 레지스트리를 유지하지 않습니다. 즉, 응용프로그램은 사용자 기본 설정 및 다양한 기본 설정을 저장하기 위해 다른 방법을 이용합니다. 각 응용 프로그램은 고유 한 네임 스페이스를 수신하며, 여기에서 설정을 추가, 수정 또는 제거 할 수 있습니다. 이 네임 스페이스를 응용 프로그램 도메인이라고합니다.
응용 프로그램 기본값은 (일반적으로) 속성 목록에 저장됩니다. Apple은 plist에 대한 역 DNS 명명 규칙을 /Library/Preferences에서 사용자별로 유지 관리 할 것을 권장합니다. 

## Launching Default Applications
-skip-

## FRAMEWORKS
OSX 환경의 또 다른 주요 구성 요소는 프레임 워크입니다. 프레임 워크는 하나 이상의 공유 라이브러리와 관련 지원 파일로 구성된 번들입니다. 프레임 워크는 라이브러리와 매우 유사하지만 Apple 시스템에 고유하므로 이식성이 없습니다. 프레임워크는 운영체제(다윈)의 일부로 간주되지 않습니다. 모두 오픈 소스 인 다윈의 구성 요소와 달리, 애플은 대부분의 프레임 워크를 완전히 폐쇄된 소스로 유지합니다.

그 이유는 프레임워크가 Apple의 운영체제에서만 제공하는 고급 기능뿐만 아니라 Apple이 포팅하고 싶지 않은 다른 고급 기능도 제공하기 때문입니다. "전통적인" 라이브러리는 Apple 시스템에서 프레임워크가 구현되는 기반을 제공합니다. 프레임워크는 런타임 인터페이스를 제공하며 특히 Objective-C를 이용해 기본 시스템 및 라이브러리 API를 숨기는 역할을합니다.

## Framework Bundle Format
응용 프로그램과 프레임워크는 번들입니다. 그러나 프레임워크 번들은 응용 프로그램과 약간 다릅니다. 주요 차이점은 내장 버전 관리 메커니즘입니다. 프레임 워크에는 하나 이상의 코드 버전이 포함되어 있습니다.

## Finding Frameworks
프레임워크는 파일 시스템의 여러 위치에 저장됩니다
- /System/Library/Frameworks : 애플에서 제공하는 프레임워크의 위치
- /Network/Library/Frameworks : 네트워크 프레임워크 위치
- /Library/Frameworks : 3rd 파티 프레임워크 및 사용자가 제공한 프레임워크 위치

이외에도 응용 프로그램 자체에 프레임워크를 포함할 수 있습니다. 


## Top Level Frameworks
OSX에서 가장 중요한 두 가지 프레임 워크는 카본과 코코아로 알려져 있습니다.

### Carbon
OS9의 레거시 프로그래밍 인터페이스의 이름입니다. 더 이상 사용하지 않습니다.

### Cocoa
OSX에서 응용프로그램을 제작할 때 사용되는 프레임워크입니다. 주로 Objective C를 사용하지만 Java, Swift도 사용 가능합니다. 많고 유용한 클래스와 UI 쪽의 지원이 많아 탄탄한 프레임워크 입니다.

## List of OS X and iOS Public Frameworks
OSX에서 지원되는 프레임 워크와 지원되는 버전을 확인할 수 있습니다. -> 표는 애플 dev에서 확인 가능

## LIBRARIES
프레임 워크는 특별한 유형의 라이브러리입니다. 실제로, 프레임 워크 바이너리는 라이브러리이며 Command Line으로 확인할 수 있습니다. 애플은 여전히 두 용어를 구별하며, 프레임워크는 모든 UNIX 시스템에서 공통적으로 사용되는 라이브러리와는 달리 OSX에 더 특정한 경향이 있다.

OSX는 라이브러리를 /usr/lib에 저장합니다. 라이브러리는 UNIX와 다르게 .so 가 아니라 .dylib 확장자가 붙습니다. .dylib 형식이지만 다른 UNIX에서 자주 사용하는 라이브러리를 찾을 수 있습니다. 핵심 라이브러리인 libc는 Apple 자체의 libSystem.B.dylib에 일부이며, 수학 라이브러리 (libm) 및 PThreads (libpthread)에서 제공하는 기능뿐만 아니라 libSystem의 다른 여러 기능도 제공합니다.

## OTHER APPLICATION TYPES
OSX는 다른 여러 유형의 응용 프로그램도 지원합니다.

### JAVA
OSX에는 Java 가상머신이 포함되어 있어 다른 시스템과 마찬가지로 Java 응용 프로그램도 .class 파일로 제공됩니다.

### Widgets
대시 보드 위젯(또는 간단히 위젯)은 대시 보드로 표시 할 수있는 HTML/자바 스크립트 미니 페이지입니다.

### BSD/Mach Native
OSX에서 선호되는 언어는 Objective-C이지만 기본 응용 프로그램은 C/C ++로 코딩 될 수 있으며, 시스템 라이브러리, BSD, Mach의 low-level 인터페이스를 대신하여 프레임워크를 선택할 수 있습니다. 따라서 PHP, Apache, SSH 및 기타 수많은 오픈 소스 제품과 UNIX 코드베이스를 비교적 간단하게 이식 할 수 있습니다. OSX의 POSIX 호환은 표준 시스템 호출과 라이브러리에 의존하여 응용 프로그램을 매우 쉽게 이식 할 수있게합니다.

## SYSTEM CALLS
사용자 프로그램은 시스템 리소스에 직접 액세스 할 수 없습니다. 프로그램은 범용 레지스터를 조작하고 간단한 계산을 수행 할 수 있지만 파일이나 소켓을 열거나 간단한 메시지를 출력하는 등의 중요한 기능을 달성하려면 시스템 호출을 사용해야합니다. 
시스템 호출은 커널에 의해 익스포트(export)되고 /usr/lib/libSystem.B.dylib에 연결하여 사용자 모드에서 액세스 할 수 있습니다. OSX 시스템 호출은 Mach와 POSIX의 두 가지를 이용한 다는 것이 특이합니다.

## POSIX
OSX는 UNIX 사용하며 POSIX라고하는 휴대용 운영 체제 인터페이스를 완벽하게 준수합니다. POSIX 호환성은 XNU의 BSD 계층에서 제공합니다. 
- System call prototypes : 기본 구현에 관계없이 모든 POSIX 시스템 호출은 동일한 프로토 타입을 갖습니다. 이를 통해 POSIX 호환 코드를 POSIX 호환 운영 체제간에 소스 레벨로 포팅 할 수 있습니다. POSIX 호출 및 C/C++ 표준 라이브러리에 의존하지 않는 한 OSX의 코드를 Linux, FreeBSD 및 Solaris로 이식 할 수 있습니다.
- System call numbers : 주요 POSIX 기능에는 정의된 시스템 호출 번호가 있습니다. 이것은 바이너리 이식성을 가능하게합니다. 즉, POSIX 컴파일 바이너리는 동일한 기본 아키텍처의 POSIX 시스템간에 이식 될 수 있습니다. 그러나 객체 형식 Mach-O가 ELF와 호환되지 않기 때문에 OSX는 이를 지원하지 않습니다.

## Mach System Calls
BSD 계층은 Mach 커널을 래핑하지만 기본 시스템 호출은 여전히 사용자 모드에서 액세스 할 수 있습니다.

32 비트 시스템에서 Mach 시스템 호출은 음수입니다. 이런 방법으로 POSIX과 Mach 시스템 호출이 나란히 존재할 수 있습니다. POSIX은 양수공간에서 시스템 호출을 정의하고 Mach은 음수 공간에서 시스템 호출을 사용합니다.

64 비트 시스템에서 Mach 시스템 호출은 양수이지만 0x2000000으로 고정되어 있습니다. POSIX 시스템 호출은 0x1000000으로 고정되어 명확하게 구분됩니다.

### Displaying Mach and BSD system calls
시스템 호출은 직접 호출되는 것이 아니라 libSystem.B.dylib의 얇은 래퍼를 통해 호출됩니다. 
OSX에서 otool, Mach-O 핸들러, 디스어셈블러를 사용하면 바이너리 파일을 디스 어셈블하고 libSystem 내부를 들여다 볼 수 있습니다. 이를 통해 OSX의 시스템 호출 인터페이스가 Mach 및 BSD 호출 모두에서 작동하는 방식을 확인할 수 있습니다.

## A HIGH-LEVEL VIEW OF XNU
OSX의 코어는 다윈이고 커널은 XNU입니다. XNU는 여러 요소로 구성되어 있습니다.
- Mach micro kernel
- BSD Layer
- I/O kit

커널은 모듈식이며 화장커널(Kext)를 요청시 동적으로 플러그인 할 수 있습니다.

## Mach
XNU의 핵심은 Mach입니다. Mach는 Carnegie Mellon University (CMU)에서 운영 체제를 위한 가볍고 효율적인 플랫폼을 만드는 연구 프로젝트로 원래 개발된 시스템입니다. 그 결과 운영 체제의 가장 기본적인 책임만 처리하는 Mach micro kernel이 있습니다.

- Process and thread abstractions (프로세스와 스레드 추상화)
- Virtual memory management (가상메모리 관리)
- Task scheduling (Task 스케줄링)
- Interprocess communication and messaging (프로세스간 통신 및 메시징)

Mach API가 매우 제한되어 있으며 완전한 운영 체제가 아닙니다. 파일 및 장치 액세스와 같은 추가 기능은 그 위에 구현되어야합니다. 이것이 바로 BSD 계층의 기능입니다.

## The BSD Layer
Mach 위에는 있지만 여전히 XNU에서 분리 할 수없는 부분은 BSD 계층입니다. 이 계층은 앞서 논의한 POSIX 호환성을 제공하고 강력하고 최신 API를 제공합니다. BSD 계층은 다음을 포함하여 더 높은 수준의 추상화를 제공합니다.

- The UNIX Process model
- The POSIX threading model (Pthread) and its related synchronization primitives
- UNIX Users and Groups
- The Network stack (BSD Socket API)
- File system access
- Device access (through the /dev directory)

## libkern 
대부분의 커널은 C 및 하위 수준 어셈블리에만 내장되어 있습니다. 그러나 XNU는 다릅니다. 다음에 설명 할 I/O 키트 드라이버라고하는 장치 드라이버는 C++로 작성할 수 있습니다. C++ 런타임을 지원하고 기본 클래스를 제공하기 위해 XNU에는 내장된 자체 포함 C++ 라이브러리인 libkern이 포함되어 있습니다.

## I/O Kit 
XNU에 특징은 I/O 키트 장치 드라이버 프레임 워크의 도입이었습니다. 이것은 커널에서 완벽하고 독립적인 실행 환경으로, 개발자는 안정적인 장치 드라이버를 만들 수 있습니다. 언어에서 제공하는 가장 중요한 기능인 상속 및 오버로드와 함께 제한된 C++환경(libkern)을 통해 개발할 수 있습니다.

I/O 키트 드라이버는 기존 드라이버슈를 슈퍼 클래스로 사용할 수 있으며 런타임에 드라이버의 모든 기능을 상속하는 것이 매우 간단해집니다. 이는 상용구 코드 복사의 필요성을 완화시켜 안정성 버그로 이어질 수 있으며 드라이버 코드를 매우 작게 만들고 커널 공간의 엄격한 메모리 제약에서 장점을 가집니다. 드라이버에 새로운 메소드를 추가하거나 기존 메소드를 오버로드/숨겨서 기능상의 모든 수정을 도입 할 수 있습니다.

C++ 환경의 또 다른 이점은 드라이버가 객체 지향 환경에서 작동 할 수 있다는 것입니다. 이로 인해 OSX 드라이버는 C로 제한되어 있으며 무거운 코드를 가지는 다른 운영 체제의 다른 장치 드라이버와 크게 다릅니다. I/O 키트는 많은 드라이버로 구성된 풍부한 환경으로 XNU에서 자체 시스템을 구성합니다.
