# < Chapter7 : launchd >

Mac 또는 i-Device의 전원을 켤 때, 이전 장에서 설명한 부트 로더 (OS X : EFI, iOS : iBoot)는 커널을 찾고 시작을 합니다. 7장에서는 Kernel boot에 대해서 자세히 설명하겠습니다. 커널은 서비스를 제공할뿐, 실제 애플리케이션이 아닙니다. 유저 모드 응용 프로그램은 커널을 구축하여 파일, 멀티미디어 및 사용자 상호 작용이 친숙한 사용자 환경을 제공함으로써 시스템에서 실제 작업을 수행하는 응용 프로그램입니다. 모든 것이 어딘가에서 시작해야하고 OS X와 iOS에서 시작됩니다.

## [LAUNCHD]
launchd는 OS X와 ios에서 사용되며, UNIX 시스템 콜인 init과 같습니다. 이름은 다를 수 있지만 같은 아이디어입니다. 유저 모드에서 처음으로 시작되는 프로세스이며, 직접 또는 간접적으로 시스템의 다른 모든 프로세스의 시작을 책임지고 있습니다. 또한, OS X 및 ios 특유의 기능이 있습니다. (launchd는)독점적이지만 Darwin의 한 종류이고 완전히 오픈 소스입니다.

## [Starting launchd]
launchd는 커널에 의해 직접 시작됩니다. BSD 하위 시스템을 로드하는 메인 커널 스레드는 스레드를 분리하여 bsdinit_task를 실행합니다. 이 스레드는 BSD에 사용되던 "init"을 임시 명칭인 PID 1으로 가정합니다. 그리고 스레드는 load_init_program()을 호출합니다. load_init_program()은 데몬을 실행하기 위한 시스템 콜인 execve()로 불립니다. /sbin/launchd는 init_program_name 변수로 하드 코딩됩니다. 

데몬은 이런 방식으로 시작되도록 설계 되었습니다. 사용자가 시작할 수 없습니다. 그렇게하면 목록 7-1에 표시된대로 오류코드가 표시됩니다.

<7-1>
root@Minion (/)# /sbin/launchd
launchd: This program is not meant to be run directly.

launchd를 시작할 수 없지만 엄격하게 제어 할 수 있습니다. launchctl(1) 명령은 launchd의 인터페이스로 사용되고 데몬을 시작하거나 정지할 수 있습니다. command는 대화식이고 자체 도움말이 있습니다. -s로 시작하거나 시작 중에 Option-S를 누른 경우, 이 argument는 커널에 의해 전파됩니다.

Launchd는 [/private]/var/db에 특수 도트(dot) 파일을 생성하여 몇 가지 로깅 및 디버깅 기능으로 시작할 수 있습니다. 도트(dot)파일에는 .launchd_log_debug, .launchd_log_shutdown 및 .launchd_use_gmalloc이 포함됩니다. Launchd는 Apple 내부 로깅(logging)에 대한 /AppleInternal 파일(System root에 존재)이 있는지 확인합니다.

## [System-Wide Versus Per-User launchd]
OS X에서 ps(1) 또는 이와 유사한 명령을 사용하면 여러 인스턴스가 시작된 것을 볼 수 있습니다. 첫 번째는 PID1이며 커널에 의해 시작되었습니다. 7-2에서처럼 누군가가 로그온 한 경우, 첫 번째에서 포크되고 로그온한 사용자가 소유하는 다른 Launchd가 있을 것입니다. 시스템 사용자에게 속한 다른 인스턴스도 볼 수 있습니다.

사용자별 Launchd는 사용자가 로그인할 때마다 실행됩니다. 사용자별 Launchd는 SSH(로그인된 사용자당 한 번)를 통해 원격으로라도 로그인하더라도 실행됩니다.

시스템 전체에서 사용되는 Launchd(PID 1)를 중지 할 수 없습니다. 실제로 Launchd는 시스템에서 유일하게 불멸의 프로세스입니다. Launchd는 죽일(정지 할) 수 없으며, 그런 것이 당연한 것입니다. Launchd를 종료할 이유가 없습니다. 대부분의 UNIX에서 init 프로세스가 예기치 않게 종료되면 결과적으로 커널 패닉이 발생합니다. Launchd는 시스템이 종료될 때 마지막으로 종료되는 프로세스입니다.

## [Daemons and Agents]
Launchd의 핵심 책임은, 이름에서 알 수 있듯, 스케줄링 되거나 요구로 하는 다른 프로세스나 작업을 시작하는 것입니다. Launched는 두 가지 유형의 백그라운드 작업을 구분합니다.

- 데몬은 일반적인 UNIX 개념과 마찬가지로 일반적으로 사용자와 상호 작용하지 않는 백그라운드 서비스입니다. 사용자가 로그온했는지 여부에 관계없이 시스템에 의해 자동으로 시작됩니다.
- 에이전트는 사용자가 로그온 할 때만 시작되는 특수한 데몬입니다. 데몬과 달리 사용자와 인터페이스 할 수 있으며 실제로 GUI가있을 수 있습니다.
- iOS는 사용자 로그인 개념을 지원하지 않으므로 LaunchDaemons 만 있습니다
- 데몬과 에이전트는 모두 개별 속성 목록 (.plist) 파일에 선언되어 있습니다. 이런 파일은 일반적으로 XML(OS X) 또는 바이너리(iOS)입니다. launchd.plist(5)는 유효한 plist 항목에 대한 자세한 설명있는 man page 이고 문서화 되지 않은 몇몇 키가 빠져있습니다.

데몬 및 에이전트 목록은 표 7-1에 명시된 위치에서 찾을 수 있습니다.
<테이블 7-1>

Launchd는 런타임 구성에 /private/var/db 경로를 사용하여, 런타임 오버라이딩 및 데몬의 비활성화를 위한 com.apple.launchd[.peruser.%d] 파일을 만듭니다.


## [The Many Faces of launchd]
Launchd는 유저모드에서 등장하는 첫 번째 프로세스입니다. 시스템이 초기 단계에 있을 때, 유일한 프로세스입니다. 이는 시스템 시작 및 기능의 거의 모든 측면을 직간접적으로 의존한다는 것을 의미합니다. OS X 및 iOS에서 Launchd는 여러 역할을 수행하며 UNIX에서는 전통적으로 여러 데몬에 담당합니다.

## [Daemons and Agents - init]
launchd가 수행하는 첫 번째 및 주요 역할은 데몬 init입니다. 후자의 작업 설명은 무수한 데몬을 생성 한 다음 백그라운드로 페이딩하고 이러한 데몬이 살아 있는지 확인하여 시스템을 설정합니다. 하나가 죽으면 launchd는 단순히 그것을 살릴 수 있습니다. 후자의 작업 설명은 무수한 데몬을 생성한 다음 백그라운드로 페이딩하고 이러한 데몬이 살아있는지 확인하여 시스템을 설정합니다.

그러나 기존 init과 달리 Launchd 구현은 표 7-2에 표시된 것처럼 다소 다르며 상당히 개선되었습니다.

<init vs launchd 테이블>

## [Daemons and Agents - Per-User Initialization]
기존 UNIX에는 사용자 로그인시 응용 프로그램을 실행할 수 있는 메커니즘이 없습니다. 사용자는 쉘 및 프로파일 스크립트에 의존해야하지만 각 쉘이 서로 다른 파일을 사용하기 때문에 혼동되기 쉽고 모든 쉘이 반드시 로그인 쉘일 필요는 없습니다. 또한, GUI 환경에서는 쉘을 시작하는 것이 전혀 허용되지 않습니다(대부분의 OS X 사용자들이 Terminal.app에 대해 모르듯이).

LaunchAgent를 사용하면 launchd는 특정 응용 프로그램을 사용자별로 시작할 수 있습니다. 에이전트는 LoginWindow, Aqua 또는 Background와 같은 값으로 LimitLoadToSessionType 키를 지정하여 기본적으로 모든 세션에서 또는 GUI 세션에서만 로드를 요청할 수 있습니다.

## [Daemons and Agents - atd/crond]
UNIX는 일반적으로 지정된 시간에 지정된 명령을 실행할 때와 같이 예약 된 작업을 실행하기 위해 두 개의 데몬 (atd 및 crond)을 정의합니다. 첫 번째 데몬인 atd는 일회성 작업에 at(1) 명령을 허용하는 엔진 역할을 하는 반면 두 번째 Crond는 반복적인 작업 지원을 제공합니다.

Apple은 atd와 crond를 점차 폐지하고 있습니다. atd는 더 이상 독립형 데몬은 아니지만, launchd에 의해 시작됩니다. com.apple.atrun.plist에서 정의되는이 서비스 (목록 7-3 참조)는 일반적으로 비활성화되어 있습니다.

<7-3>
at (1) 계열의 명령이 작동 할 수 있도록 atrun plist를 활성화해야합니다. 그렇지 않으면 작업을 예약하지만 결코 발생하지 않습니다(알람을 사용하기 위해 알람을 설정하는 것 처럼).

Launchd에는 자체 StartCalendarInterval 키 세트가 있지만 Crond 서비스는 com.vix.crond.plist에서 계속 지원됩니다. Apple은 교체를 위해 periodic(8)을 제공합니다. <7-4>은 com.apple.periodic-daily에서 cron의 대체하는 것 중에 하나를 보여줍니다.(주간 및 월간에 따라)

<7-4>

## [Daemons and Agents - inetd/xinetd:]
UNIX에서는 inetd(xinetd)를 사용하여 네트워크 서버를 시작합니다. 데몬 포트(UDP 또는 TCP)을 바인딩하고 연결 요청이 도착하면 요청에 따라 서버를 시작하고 그 입출력 기술자(descriptor), (stdin, stderr 및 stdout)를 소켓에 연결합니다.

이 방법은 네트워크 서버와 시스템 모두에 매우 유용합니다. 서비스 요청이 없는 경우 시스템이 서버를 계속 실행하지 않아도되므로 시스템 부하가 줄어 듭니다. 이러한 방식으로 관리자는 포트 번호를 서비스에 기발하게 재할당 할 수 있으며 본질적으로 네트워크 포트를 통해 CLI 명령 (쉘도 포함)을 실행할 수 있습니다.

Launchd는 데몬 및 에이전트가 특정 소켓을 요청할 수 있게하여 inetd 기능을 자체에 통합합니다. 데몬은 plist에서 소켓 키를 사용하여 묻습니다. 목록 7-5는 ssh.plist에서 TCP/IP 소켓 22를 요청하는 예를 보여줍니다.

<7-5>

inetd와 달리 데몬이 요청한 소켓은 UNIX 도메인 소켓 일 수도 있습니다. com.apple.syslogd.plist에서 발췌 한 Listing 7-6은 다음을 보여줍니다.

<7-6>

두 소켓 제품군 (UNIX 및 INET)은 상호 배타적이지 않으며 동일한 어구에서 지정할 수 있습니다. 예를 들어, 이전 syslogd plist는 SockServiceName : syslog 키를 추가하고 선택적으로 –udp_in 및 1을 ProgramArguments 배열에 추가하여 syslog가 UDP 514의 메시지를 수락하도록 쉽게 수정할 수 있습니다.

## [Daemons and Agents - mach_init]
NEXTStep의 기원과 마찬가지로 OS X 10.4에서 출시되기 전에 시스템 시작 프로세스를 mach_init라고했습니다. 이 데몬은 실제로 별도의 프로세스인 BSD 스타일 init을 나중에 생성하는 일을 담당했습니다. 이런 2가지는 Launchd에 통합되었으며 mach_init는 거의 문서화되지 않았지만 부트 스트랩 서비스 관리자의 주요 역할을 담당했습니다.

Mach의 IPC 서비스는 endpoint 통신 역할을 하는 "포트"(TCP 및 UDP와 유사)라는 개념에 의존합니다. 그러나 현재로서는 포트를 완전한 이름으로 참조 할 수 있는 불투명 한 숫자로 간주하면 충분합니다. 서버와 클라이언트는 모두 포트를 할당 할 수 있지만 서버는 클라이언트가 포트를 찾을 수 있도록 특정 유형의 locator 서비스가 필요하거나 그렇지 않으면 "잘 알려져 있어야합니다".

부트 스트랩 서버를 입력하십시오. 이 서버는 시스템의 모든 프로세스에 액세스 할 수 있으며 지정된 포트 (bootstrap_port)를 통해 통신 할 수 있습니다. 그런 다음 클라이언트는이 포트를 통해 서버가 지정된 서비스를 이름별로 찾아 포트와 일치하도록 요청할 수 있습니다. (UNIX는 RPC portmapper에서 sunrpc와 비슷한 기능을 가지고 있습니다. 맵퍼는 잘 알려진 포트 (TCP / UDP 111)에서 수신하고 다른 RPC 서비스를 위해 matchmaker를 사용합니다.)

Launchd를 시작하기 전에 mach_init는 bootstrap_server의 역할을 맡았습니다. Launchd는 이후, 이 역할을 맡아 시작 중에 포트(bootstrap_port라는 이름이 지정된 포트)를 청구합니다. 시스템의 모든 프로세스는 이것(Launchd?)의 자손이므로 포트에 대한 액세스를 자동으로 상속합니다. bootstrap_port는 <servers/bootstrap.h>에서 extern mach_port_t로 선언되어 있습니다.

부트 스트랩 서버에 포트를 등록하려는 서버는 <servers / bootstrap.h>에 정의 된 기능을 사용하여 포트를 사용하여 포트를 등록 할 수 있습니다. 이러한 기능 (bootstrap_create_server 및 bootstrap_create_service)은 계속 지원되지만 더 이상 추천하지 않습니다. 대신 서버의 plist에서 Launched로 서비스를 등록 할 수 있습니다. 그리고 더 간단한 함수 - bootstrap_check_in() - 는 서비스의 요청이 준비되면 서버가 Launchd에게 포트를 넘겨줄 것을 요청한다.

<코드 - pdf 참고>

Launchd는 서버의 plist를 처리 할 때 포트를 사전 등록합니다. 서버 포트는 일반적으로 일시적이지만 HostSpecialPort 키가 추가 된 경우에도 잘 알려져 있습니다. Launchd는 서버 요청을 기다리도록 지시 할 수 있습니다 (목록 7-7 참조). com.apple.windowserver.active는 WindowServer가 <launch.h> 함수를 사용하여 Launchd에 체크인 한 후에 클라이언트에 보급됩니다. 

<7-7>

특정 서비스에 연결하려는 클라이언트는 비슷한 기능을 사용하여 서버 포트를 찾을 수 있습니다.

<코드>

서버의 포트가 사용 가능하고 서버가 체크인 한 경우 클라이언트로 리턴되어 메시지를 보내고 받을 수 있습니다. 부트 스트랩 프로토콜의 Mach 메시지는 .defs 파일의 Launchd 소스에서 정의되며,이 파일은 MIG (Mach Interface Generator)에 의해 사전 처리됩니다. Launchctl(1)의 하위 명령인 bslist를 사용하여 활성 데몬 목록을 볼 수 있습니다. 이 목록은 현재 컨텍스트에서 볼 수 있는 부트 스트랩 서버의 계층 네임 스페이스를 보여줍니다. 하위 명령인 bstree는 전체 계층 네임 스페이스를 표시하지만 루트 권한이 필요합니다. Lion 이상에서 bstree는 XPC 네임 스페이스도 표시합니다.

부트 스트랩 메커니즘은 Snow Leopard에 도입 된 새로운 라이브러리 인 Launchd의 vproc를 통해 구현되며 트랜잭션 기능도 제공합니다.

## [Daemons and Agents - Transaction Support]
Launchd는 평균적인 init보다 똑똑합니다. 데몬을 시작하거나 중지 할 수 있는 init과 달리 launchd는 launchd의 vproc에서 내 보낸 유용한 기능인 트랜잭션을 지원합니다. 데몬은 공개 <vproc.h>를 통해 액세스 할 수 있습니다. 이 API를 사용하는 데몬은 트랜잭션 완료시 트랜잭션 핸들을 생성하는 vproc_transaction_begin과 해당 핸들의 vproc_transaction_end 사이에 보류중인 트랜잭션을 캡슐화하여 표시 할 수 있습니다. 트랜잭션 가능 데몬은 plist에 EnableTransactions 키를 표시 할 수 있습니다.이를 통해 시스템 종료, 사용자 로그 아웃 또는 지정된 시간 초과 후 시작된 보류 트랜잭션을 확인할 수 있습니다. 처리되지 않은 트랜잭션이없는 경우(프로세스가 깨끗함) 데몬은 정상적으로 terminated (kill -15) 대신 shoutdown(kill -9)하여 충분히 비활성화를 한 후 종료 또는 로그 아웃 프로세스 속도를 높입니다.

## [Daemons and Agents - Resource Limits and Throttling]
launchd는 작업에 자체적으로 부과 자원 제한을 적용 할 수 있습니다. 작업(데몬 또는 에이전트)은 Launchd가 setrlimit(2)를 호출시키는 HardResourceLimits 또는 SoftResourceLimits 사전을 지정할 수 있습니다. Nice 키는 nice(1) 작업 수 만큼 작업의 nice value를 설정할 수 있습니다. 또한 작업에 LowPriorityIO 키를 표시하여 Launched가 iopolicysys를 호출하고 작업의 I/O 우선 순위를 낮출 수 있습니다.

## [Daemons and Agents - Autorun Emulation and File System Watch]
Windows에서 가장 잘 알려진 기능 중 하나는 자동 실행(auto run)으로, 이동식 미디어 (예 : CD, USB 저장소 또는 하드 디스크)가 연결되면 프로그램을 자동으로 시작할 수 있습니다. launchd는 파일 시스템이 마운트 될 때마다 데몬을 시작하도록 트리거 할 수 있는 StartOnMount 키를 제공합니다. 이것은 Windows의 기능을 에뮬레이트 할뿐만 아니라 실제로는 더 안전합니다. Windows 자동 실행 기능이 악성 코드의 전파의 매개체가 되어 있기 때문입니다. launchd 데몬은 이동식 시스템이 아닌 영구적인 파일 시스템에서 실행됩니다.

launchd는 WatchPaths 키 또는 QueueDirectories 키를 사용하여 변경을 위해 반드시 마운트 지점이 아닌 특정 경로를 모니터링 하도록 만들 수 있습니다. 이 기능은 파일 시스템 변경에 실시간으로 반응 할 수 있으므로 매우 유용합니다. 이 기능은 커널 이벤트(kqueue)를 수신하여 수행됩니다. 데몬은 com.apple.fsevents.matching dict에서 일치하는 케이스인 LaunchEvents dictionary를 지정하여 FSEvents도 지원하도록 더욱 확장 할 수 있습니다.

## [Daemons and Agents - I/O Kit Integration]
Lion의 새로운 기능은 launchd와 I/O Kit의 통합입니다. I/O 키트는 장치 드라이버의 런타임 환경입니다. Launchd 데몬 또는 에이전트는 com.apple.iokit.matching dictionary를 포함하는 LaunchEvents dictionary를 지정하여 장치 도착시 호출을 요청할 수 있습니다. I/O 키트 및 matching dictionary 예제는 Listing 7-8에서 볼 수 있다. 이 목록은 com.apple.blued.plist Launchd 데몬에서 발췌한 것으로, Bluetooth SDP 트랜잭션을 처리하기 위해 트리거된다.

<7-8>

## [Daemons and Agents - Experiment: Setting up a Custom Service]
UNIX inetd의 가장 유용한 기능 중 하나는 모든 포트에서 거의 모든 UNIX 유틸리티를 실행할 수 있다는 것입니다. 한편으로는 inetd의 소켓 로직 처리와 소켓을 다른 파일 디스크립터로 취급하는 기능이 결합되어이 강력한 기능을 제공합니다. 

Launchd로 조금 더 복잡한 경우에도 가능합니다. 먼저, 프로그램을 위해 Launchd plist를 만들어야합니다. 이것은 복사, 붙여넣기, 변경과 같은 간단한 문제입니다. Label, Program, ProgramArguments 및 Sockets 키를 원하는대로 변경하여 목록 7-5에서처럼 제대로 작동하듯이.

그러나 여기서 문제가 발생합니다. launchd는 네트워크 연결에 대한 응답으로 임의의 프로그램을 실행할 수 있지만 stdin, stdout 및 stderr를 파일로 리디렉션하는 것만 지원합니다.

launchd가 설정될 때, 애플리케이션의 stdin, stdout, stderr이 소켓에 연결되길 원합니다. 이것은 우리가 시작하는 프로그램이 Launchd를 인지해야 하고 소켓 hand-off를 요청해야 한다는 것을 의미한다.

이 문제를 해결하려면 7-9에 표시된 것처럼 generic wrapper를 만들어야합니다.

<7-9>

목록에서 알 수 있듯이 랩퍼는 launchd_ API(launch_로 선언되어 <launch.h>에 정의됨)를 사용하여 launchd와 통신하고 소켓을 요청합니다. 이것은 여러 단계로 수행됩니다
> Checking in with launchd : launch_ msg() 함수를 사용하여 특별한 메시지를 보내면 됩니다. 체킹은 표준 절차이므로 launch_data_new_string (LAUNCH_KEY_CHECKIN)을 사용하여 메시지를 작성한 다음 해당 메시지를 Launchd로 전달하는 것은 간단합니다.

> Get our plist parameters : launched가 체크인 요청에 응답하면 API를 사용하여 plist에서 다양한 설정을 가져올 수 있습니다. Launchd 데몬에 매개 변수를 전달하는 두 가지 방법이 있습니다. 하나는 명령행 인수(ProgramArguments 배열) 이고, 다른 하나는 EnvironmentVariables dictionary에 전달 된 환경 변수를 통해 표준 getenv (3) 호출을 사용하여 데몬을 읽습니다.

> Get the socket descriptor : 문자열과 다른 기본 데이터 유형처럼 프로세스간에 전달하는 것이 쉽지 않기 때문에 모든 유형의 파일 설명자를 얻는 것은 약간 까다 롭습니다. 그럼에도 불구하고 launch_data_get_fd에 의해 모든 복잡성이 숨겨져 있습니다.

파일 descriptor(우리를 위해 Launchd가 오픈한 소켓)를 가져 오면 네트워크 서버와 마찬가지로 accept()를 호출합니다. 이것은 다른쪽 끝에 클라이언트와 연결된 소켓을 생성합니다. 남은 일은 dup2() 시스템 콜을 사용하여 stdin, stdout 및 stderr를 허용된 소켓으로 바꾸고 실제 프로그램을 exec() 하는 것입니다. 왜냐하면 exec()는 파일 descriptor를 유지하기 때문에 새로운 프로그램은 이미 연결된 상태에서 이러한 descriptor를 받고 read(2) 및 write(2)는 recv(2) 와 send(2)를 호출 한 것처럼 소켓 통해 리디렉션됩니다.

wrapper를 테스트하려면 출력 7-1에 표시된대로 /System/Library/LaunchDaemons(또는 다른 LaunchDaemons 디렉토리)에 plist를 삭제하고 launchctl (1)을 사용하여 시작해야합니다. 이 예제의 wrapper는 com.technologeeks.wrapper로 레이블이 지정되었으며 eponymous plist에 배치되었습니다. 출력에서 launchctl(1)은 명령이 성공했다는 코멘트가 없습니다.

<output 7-1>

wrapper는 generic 의도이기 때문에 이 프로그램이 stdin, stdout 및 stderr을 사용한다고 가정한다면 원하는 프로그램을 지정할 수 있습니다. 원하는 모든 포트에서 루트 쉘을 쉽게 설정할 수 있으므로 백도어 기능이 우수합니다. wrapper의 명령 행 argument를 /bin/zsh -i로 설정하면 출력 7-2와 유사한 출력이 발생합니다.

<output 7-2>

쉘 명령에 세미콜론을 추가 할 필요가 있다는 점에 유의하십시오. 이것은 터미널이 아니라 쉘의 stdin에서 직접 작업하고 있기 때문에 Enter 키를 리터럴 Ctrl-M으로 전송되기 때문입니다. 추가 된 세미콜론은 명령을 종료하여 쉘이 구문 분석 할 수 있게하여 Ctrl-M을 별도의 유효하지 않은 명령으로 만듭니다.

## [LISTS OF LAUNCHDAEMONS]
OS X 및 iOS에는 엄청난 양의 LaunchDaemon이 있습니다. 실제로, 많은 사이트들은 데몬과 에이전트의 목적과 유용성에 대해 토론하기 위해 수많은 HTML 페이지와 SMTP 메시지를 바칩니다. 특히 iOS에서 불필요한 CPU 사이클이 성능에 영향을 줄뿐만 아니라 배터리 수명을 크게 단축 시킵니다.

iOS와 OS X는 일반적인 LaunchDaemon을 공유합니다. 모든 plist (및 해당 Mach 서비스 항목)에는 com.apple로 시작하며 일반적으로 /usr/libexec에서 바이너리를 실행합니다. 표 7-3에 나와있습니다.

<테이블 7-3>

OS X 고유의 LaunchDaemons (및 LaunchAgents 호스트) 목록은 위 페이지에 맞추기에 너무 많고, 웹사이트에서 확인이 가능합니다.

## [LISTS OF LAUNCHDAEMONS - iOS launchdaemons]
테이블을 살펴보면 iOS에서 SpringBoard와 lockdownd의 두 가지 특별한 데몬을 발견했을 것입니다.
IOS라 내용 정리 X

## [LISTS OF LAUNCHDAEMONS - lockdownd]
IOS라 내용 정리 X

## [GUI SHELLS]
사용자가 자동 또는 자격 증명을 지정하여 콘솔에 로그인하면 시스템이 그래픽 셸 환경을 시작합니다. OS X은 Finder를 사용하지만 iOS는 SpringBoard를 사용하지만 두 모델은 생각보다 비슷합니다. launchd의 관점에서 보면 Finder와 SpringBoard은 모두 시작에 필요한 100 이상의 디먼과 에이전트의 컬렉션 중 하나 또는 두 개 이상의 에이전트입니다. 그러나 사용자에게는 이러한 프로그램은 운영 체제와 상호 작용의 첫 번째 (그리고 마지막) 선두를 구성합니다.

## [Finder (OS X)]
Finder는 OS X의 Windows 탐색기와 동일합니다. 사용자에게 그래픽 셸을 제공합니다. com.apple.Finder.plist 속성 목록 (/System/Library/LaunchAgents)에서 성공적으로 로그인하면 Launchd Agent로 시작됩니다. Finder는 30개 이상의 라이브러리와 프레임 워크에 의존하고 있지고, 일부는 비공개이며, otool(1) -l을 사용하여 쉽게 확인할 수 있습니다. 이렇게하면 특성이 밝혀지며, Finder는 암호화 된 바이너리로 되어있는 드문 경우입니다. OS X는 코드 암호화를 지원하지만 암호화된 바이너리가 거의 없습니다. <output 4-3>은 otool –l을 사용하여 Finder의 암호화 된 부분을 보여줍니다. 따라서 strings(1)를 사용하거나 Finder를 분해하려고 시도하는 것은 헛된 노력입니다(corerupt와 같은 도구로 암호화가 무효화되지 않는다면). 또한 GDB를 사용하여 실행중인 Finder에 연결하고 (바이너리 보호의 목적 전체를 무효로 되기 전에), Finder에 연결 한 다음 해당 스레드 (일반적으로 3개만)을 추적 할 수 있습니다. 

Finder는 시스템과 매우 긴밀하게 통합되어 있기 때문에 기본 파일 시스템 인 HFS+의 설계가 그 주위(Finder 주위)에 구축되어 있습니다. 파일과 폴더의 데이터 및 실제 볼륨 데이터 자체에는 특별한 파인더 정보 필드가 포함되어 있습니다. 이 필드는 사용자가 마지막으로 배치 한 정확한 크기와 위치로 폴더 창을 다시 여는 등 많은 기능을 활성화합니다. 또한 Finder는 확장 속성을 사용하여 컬러 라벨이나 별칭 등의 정보를 저장합니다. 이러한 기능은 모두 16장에서 설명합니다(HFS + 전용입니다).

## [Finder (OS X)- With a Little Help from My Friends]
풍부한 GUI를 지원하는 모든 작업은 하나의 프로세스에 압도적 일 수 있습니다. GUI 처리는 실제로 /System/Library/CoreServices 에있는 여러 프로세스로 분할됩니다. 

Dock.app은 일반적으로 이름이 의미하는 것처럼 바탕화면 하단에 있는 친숙한 아이콘 트레이를 담당하지만 프로세스가 종료 될 때 나타날 수있는 배경 화면도 설정합니다. com.apple.dock.extra에서 지원합니다. UI 액션과 Dock 액션은 com.apple.dock.extra 의해 연결됩니다. 

SystemUIServer.app 상태 표시 줄 메뉴 엑스트라 (오른쪽) 쪽을 담당하고 /System/Library/CoreServices/Menu Extras 에서 로드합니다. 거기에 메뉴 추가도 프로그램에서 만들 수 있는 점에 유의하십시오([NSStatusBar, systemStatusBar] 및 setImage/setMenu 메소드를 사용). 이 경우 이러한 추가는 그들을 만든 응용 프로그램의 책임입니다. 중요한 역할(UI를 최대한 오래 유지하려는 Apple의 바람)로 인해 Finder의 assistants(및 다른 CoreServices 앱)도 바이너리를 보호합니다.

## [Finder (OS X)- Experiment: Figuring Out Who Owns What in the GUI]
쉘(있다면 SSH를 통해)와 UNIX의 kill(1) 명령을 사용하여 어떤 프로세스가 GUI의 어떤 부분을 소유하고 있는지를 신속하게 확인할 수 있습니다. 옵션은 프로세스를 강제로 종료(kill -9 사용)하거나 프로세스를 일시 중지(kill –STOP 및 kill -CONT 사용)하는 것입니다. 

(위와 같이 하면)Finder, Dock 및 SystemUIServer와 같은 다양한 프로세스는 UI 자원이 잠깐 동안 사라집니다.(종료된 경우, 프로세스가 Launchd 에 의해 자동으로 다시 시작될 때까지), 또는 "빨리 감기"효과 (프로세스가 재개되고 대기중인 모든 UI 메시지가 전달 될 때). 앱에 의해 생성된 추가메뉴는 SystemUIServer의 일시중단 또는 조기종료에 영향을 받지않습니다. PID 대신 이름으로 신호를 보내므로 kill 대신 killall (1)을 사용할 수 있습니다. 이 방법으로 동일한 프로세스를 반복적으로 종료하는 경우 launchd는 프로세스를 조절하며 몇 초 후에 다시 생성됩니다.

## [SpringBoard (iOS)]
Finder는 OS X에서 SpringBoard는 iOS 용입니다. iOS에서 시스템은 로그온 할 필요가 없으므로 SpringBoard가 자동으로 시작되어 시스템의 친숙한 아이콘 기반 UI를 제공합니다. 이 UI는 Lion의 LaunchPad에 영감을 주었으며 동일한 GUI 개념을 사용하며 본질적으로 SpringBoard는 OS X로 백포트 됩니다. 이는 SpringBoard라는 이름의 일부 파일이 LaunchPad 바이너리에서 발견 될 수 있다는 사실입니다(기술적으로 Dock의 일부). OS X GUI에 대응하는 (Finder)와 마찬가지로 SpringBoard는 /System/Library/CoreServices/ 에서 로드됩니다. 

## [SpringBoard (iOS) - All by Myself (Sort of)]
Finder와 달리 SpringBoard는 거의 모든 것을 자체적으로 처리하며 CoreServices 디렉토리에는 로드 가능한 번들이 거의 없습니다. otool –l 에서 볼 수 있듯이 Finder의 30가지 종속성은 SpringBoard에 의해 약 80개로 축소되며, SpringBoard는 (놀랍게도) 보호되지 않은 바이너리입니다.
IOS 내용

## [SpringBoard (iOS) - Creating the GUI]
IOS 내용

## [SpringBoard (iOS) - Experiment: Unhiding (or Hiding) an iOS App]
IOS 내용

## [SpringBoard (iOS) - Handling the UI]
Finder와 SpringBoard는 모두 UI 발표를 담당하지만 Springboard의 책임은 그 이상입니다. SpringBoard는 iOS의 모든 유형의 작업을 담당합니다. foreground 애플리케이션이 아니더라도 (신호로) 중지되면 UI 이벤트가 활성 앱에 도달하지 않고 계속 될 때 대기중인 모든 이벤트가 앱에 전달됩니다. Springboard는 다중 스레드 응용 프로그램입니다. Finder보다 훨씬 많은 스레드가 있습니다. 애플의 개발자들은 (pthread_setname_np를 사용하여) 그들 중 일부를 명명 할만큼 친절했습니다. 이 이름은 두 개의 웹 관련 스레드 (WebCore 및 WebThreads)를 나타내며 적어도 두 개는 coremedia에 속합니다. 프로세스를 디버깅하려면 시스템 워치 독을 통과해야하는데, SpringBoard가 몇 분 이상 응답하지 않으면 시스템을 재부팅합니다.

IOS 내용 - 자세한 내용은 생략

## [XPC (LION AND IOS)]
XPC는 Lion 및 iOS 5에서 처음 소개된 가벼운 프로세스 간 통신을 다루는 set 입니다. XPC는 Apple Developer [3]에 상당히 잘 문서화되어 있습니다. XPC는 Apple Developer에 상당히 잘 문서화되어 있습니다. 또한 GCD (Grand Central Dispatcher)와 긴밀하게 통합되어 있습니다. XPC를 통해 개발자는 응용 프로그램을 별도의 컴포넌트로 세분화 할 수 있습니다. 외부에서 관리되는 XPC 서비스에 취약한(또는 불안정한) 기능을 포함 할 수 있기 때문에 응용 프로그램의 안정성과 보안을 동시에 향상시킬 수 있습니다. LaunchDaemons와 마찬가지로 launchd는 필요할 때 XPC 서비스를 시작하고, 서비스를 감시하고(충돌시 재시작), 완료되거나 유휴 상태 일 때 종료(kill -9으로 종료)하는 작업을 수행합니다. Launchd는 xpcd (8), xpchelper (8) 및 xpcproxy (8)를 사용하여 XPC 서비스를 지원합니다. XPC는 표준 Mach 서비스와 함께 XPC 서비스를 개별 XPC 도메인(사용자 별, 개인 및 싱글 톤)으로 유지 관리합니다.

이것은 출력 7-4에 표시된 것처럼 launchctl의 bstree subcommand를 통해 확인 할 수 있습니다.

<output 7-4>

XPC 서비스 및 클라이언트 응용 프로그램은 직접 또는 Cocoa를 통해 libxpc.dylib와 연결되어 다양한 C 수준 XPC 원형(예 : Mountain Lion의 NSXPCConnection)을 제공합니다. 이 글을 쓰는 시점에 라이브러리는 비공개 소스로 남아 있지만 Apple은 <xpc/*>에 API 공개를 통해 내부에 대해 설명을 합니다. XPC는 또한 XPCService 및 XPCObject의 private 프레임 워크에 의존합니다. 전자(XPCService)는 서비스의 런타임 측면을 처리하고 후자(XPCObject의)는 XPC 객체에 대한 인코딩 및 디코딩 서비스를 제공합니다. iOS에는 세번째 private 프레임 워크 인 XPCKit이 포함되어 있습니다.

## [XPC (LION AND IOS) - XPC Object Types]
XPC는 CoreFoundation 프레임 워크와 유사한 방식으로 다양한 데이터 유형을 래핑하고 직렬화합니다. <xpc / xpc.h>는 XPC가 지원하는 개체와 데이터 형식을 정의합니다(7-5참조). type은 XPC_TYPE_typename 매크로로 정의되어 테이블에서 해당 유형에 대한 포인터를 래핑하고 xpc_typename_create 함수로 인스턴스화 할 수 있습니다. 대부분의 경우 xpc_typename_get_value를 사용하여 메시지에서 개체를 얻을 수 있습니다. 두 가지 특수 객체 유형은 사전 및 배열이며 다른 객체 유형의 컨테이너 역할을합니다(xpc_[array|dictionary]_[get|set]_typename 을 사용하여 이 객체에서 작성하거나 액세스 할 수 있음).

<7-5>

XPC 객체는 모두 불분명한 xpc_object_t 로 처리 할 수 있으며 xpc_object(3)에 함수로 조작 할 수 있습니다. 여기에는 xpc_retain/release, xpc_get_type(표 7-5에 해당하는 XPC_TYPE 중 하나를 반환), xpc_hash(배열 인덱싱을 위해 객체의 해시 값을 제공하는 데 사용), xpc_equal(객체 비교를 위해) 및 xpc_copy가 포함됩니다.

## [XPC (LION AND IOS) - XPC Messages]
메시지로 개체를 보내거나받을 수 있습니다. 메시지는 표 7-6에 표시된 것처럼 <xpc/connection.h>의 여러 기능 중 하나를 사용하여 전송됩니다. 

<7-6>

기본적으로 메시지는 비동기식으로 전송되며 그림 7-2와 같이 디스패치 큐 (예 : GCD)에 의해 처리됩니다. 장벽을 사용함으로써, 프로그래머는 특정 연결의 모든 메시지가 전송 될 때 실행될 블록을 제공 할 수 있다. _reply_sync 함수는 메시지가 수신 될 때까지 차단하는 데 사용될 수 있지만 메시지는 비동기적으로 응답을 기대합니다.

<7-2>

XPC 메시지는 Mach 메시지를 통해 구현되며 xpc_domain 서브 시스템을 제공하는 Mach Interface Generator (MIG) 기능을 사용합니다. 이 서브 시스템에는 부트 스트랩 프로토콜과 유사하게 체크인,로드 또는 서비스 추가, 서비스의 이름을 가져오는 메시지가 들어 있습니다. (XPC는 부트 스트랩의 서브시스템으로 간주 될 수 있으며 내부적으로 사용합니다). 

## [XPC (LION AND IOS) - XPC services]
XPC 서비스는 Objective-C 또는 C/C ++에서 작성할 수 있습니다. 두 경우 모두 libxpc.dylib의 xpc_main 을 호출하여 서비스를 시작합니다. C/C++ 서비스의 main은 간단한 wrapper이며 이벤트 핸들러 함수(xpc_connection_handler_t)와 함께 xpc_main (<xpc / xpc.h>로 선언)을 호출합니다. Objective-C 서비스는 NSXPCConnection의 resume 메소드를 통해 간접적으로 xpc_main()를 호출합니다.

이벤트 핸들러 함수는 xpc_connection_t라는 단일 인수를 사용합니다.(Objective-C는 이 객체를 Foundation.framework의 NSXPCConnection으로 래핑합니다.) XPC 연결은 xpc_connection_* 기능을 사용하여 불분명한 객체로 처리됩니다. <xpc/connection.h> 에서 속성은 getter, 이벤트 처리기 및 target 큐는 setter로 사용되었습니다. 연결 이름, 유효 UID 및 GID, PID 및 Audit 세션 ID를 모두 쿼리(알 수) 할 수 있습니다.

XPC 서비스의 일반적인 아키텍처는 dispatch_queue_create를 호출하여 클라이언트에서 들어오는 메시지에 대한 큐를 작성하고 xpc_connection_set_target_queue를 사용하여 큐를 연결에 지정합니다. 또한 서비스는 핸들러 블록(함수를 감쌀 수 있음)과 함께 xpc_connection_set_event_handler를 호출하여 연결(connection)에 이벤트 핸들러를 설정합니다. 핸들러는 서비스가 메시지를 수신 할 때마다 호출됩니다. 서비스는 xpc_dictionary_create_reply를 호출하여 회신을 작성하여 보낼 수 있습니다. XPC의 잘 문서화 된 예는 SandBoxedFetch이며, Apple Developer 문서화되어있어 책의 예가 필요 없습니다.

## [XPC (LION AND IOS) - XPC Property Lists]
XPC 서비스는 자체 번들로 정의되며 상위 애플리케이션 또는 프레임 워크의 XPCServices 서브 폴더에 포함됩니다. 모든 번들과 마찬가지로 Info.plist는 다양한 서비스 속성 및 요구 사항을 선언하는데 사용됩니다.

- CFBundlePackageType 속성은“XPC!”로 정의됩니다.
- CFBundleIdentifier 특성은 XPCService의 이름을 정의합니다. 번들 이름과 동일하게 설정되어 있습니다.
- ㄴXPCService 속성은 ServiceType 속성(Application.User 또는 System)을 지정할 수 있는 dictionary와 xpc_main()이 선택한 실행 루프 스타일인 RunLoopType(dispatch_main 또는 NSRunLoop)을 정의합니다. 
- XPCService dictionary는 밑줄로 시작하는 추가 특성을 지정하는 데 사용될 수 있습니다. 여기에는 _SandboxProfile 및 _AllowedClients가 포함되어 있으며 서비스에 연결할 수 있는 응용 프로그램의 식별자를 지정할 수 있습니다.

## [Summary]
이 장에서는 OS X 및 iOS에서 기존 UNIX init를 대체하는 것에 대해 설명했습니다. launchd는 두 운영체제(UNIX 데몬의 기능과 Mach의 기능)에 많은 기능을 충족합니다. 이 장은 OS X (Finder) 및 iOS (SpringBoard)의 GUI에 대해 살펴보았습니다.
