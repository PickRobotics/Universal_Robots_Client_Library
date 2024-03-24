# Universal Robots Client Library
- [Universal Robots Client Library](#universal-robots-client-library)
  - [요구사항](#요구사항)
  - [Build 방법](#build-방법)
    - [Plain cmake](#plain-cmake)
    - [ROS workspace 내부](#ros-workspace-내부)
  - [다른 projects에서 이 library를 사용하기](#다른-projects에서-이-library를-사용하기)
  - [License](#license)
  - [Library contents](#library-contents)
  - [예제 driver](#예제-driver)
  - [Architecture](#architecture)
    - [RTDEClient](#rtdeclient)
      - [RTDEWriter](#rtdewriter)
    - [ReverseInterface](#reverseinterface)
    - [ScriptSender](#scriptsender)
    - [Other public interface functions](#other-public-interface-functions)
      - [`check_calibration()`](#check_calibration)
      - [`sendScript()`](#sendscript)
    - [DashboardClient](#dashboardclient)
  - [A word on the primary / secondary interface](#a-word-on-the-primary--secondary-interface)
  - [A word on Real-Time scheduling](#a-word-on-real-time-scheduling)
  - [Producer / Consumer architecture](#producer--consumer-architecture)
  - [Logging configuration](#logging-configuration)
    - [Change logging level](#change-logging-level)
    - [Create new log handler](#create-new-log-handler)
  - [Contributor Guidelines](#contributor-guidelines)
  - [Acknowledgment](#acknowledgment)


Universal Robots 인터페이스에 접근하기 위한 C++ 라이브러리입니다. 이 라이브러리는 사용하면 C++ 기반 드라이버를 구현하여 외부 응용 프로그램을 만들 수 있으며, 이를 통해 Universal Robots 로봇 manipulators의 다양성을 활용할 수 있습니다.

## 요구사항
 * 로봇 컨트롤러에서 실행되는 **Polyscope** 소프트웨어 버전은 3.14.3(CB3-시리즈용) 또는 5.9.4(e-시리즈용) 이상이어야 합니다.
 * **POSIX threads**(`pthread` 라이브러리 등) 구현이 필요합니다.
 * 현재 Linux 소켓을 기반으로 하고 있으므로 이 라이브러리는 빌드 및 사용을 위해 Linux가 필요합니다.
 * 이 [master](https://github.com/UniversalRobots/Universal_Robots_Client_Library/tree/master)저장소의 마스터 브랜치는 C++17 호환 컴파일러가 필요합니다. C++17 요구 사항 없이 이 라이브러리를 빌드하려면 boost 라이브러리를 필요로 하는 boost 브랜치를 대신 사용하십시오. C++17 기능의 경우 다음 최소 컴파일러 버전을 사용하십시오:

   | Compiler  | min. version |
   |-----------|--------------|
   | **GCC**   | 7            |
   | **Clang** | 7            |


## Build 방법
### Plain cmake
이 독립 실행형 라이브러리를 만들어 이 라이브러리를 사용하여 어플리케이션을 빌드하려면 일반적인 cmake 절차를 따르세요.:
```bash
cd <clone of this repository>
mkdir build && cd build
cmake ..
make
sudo make install
```

이렇게 하면 해당 라이브러리가 시스템에 설치되어 다른 cmake 프로젝트에서 직접 사용할 수 있습니다.

### ROS workspace 내부
예를 들어 [Universal Robots ROS driver](https://github.com/UniversalRobots/Universal_Robots_ROS_Driver)를 소스 코드에서 빌드하고 싶기 때문에 ROS workspace 내부에 이 라이브러리를 빌드하려는 경우 이 라이브러리가 catkin 패키지가 아니므로 `catkin_make`를 직접 사용할 수 없습니다. 대신 [`catkin_make_isolated`](http://docs.ros.org/independent/api/rep/html/rep-0134.html) 또는 [catkin build](https://catkin-tools.readthedocs.io/en/latest/verbs/catkin_build.html)를 사용하여 workspace을 빌드해야 합니다.

## 다른 projects에서 이 library를 사용하기
다른 cmake 프로젝트에서 이 라이브러리를 사용하려면 다음을 확인하십시오.
 * `CMakeLists.txt` 에 `find_package(ur_client_library REQUIRED)` 추가하기
 * `ur_client_library::urcl` 를 CMakeLists.txt 파일 내부에 `target_link_libraries(...)` 목록에 추가하기

다음 "project"를 최소한의 예제로 살펴봅시다:
```c++
/*main.cpp*/

#include <iostream>
#include <ur_client_library/ur/dashboard_client.h>

int main(int argc, char* argv[])
{
  urcl::DashboardClient my_client("192.168.56.101");
  bool connected = my_client.connect();
  if (connected)
  {
    std::string answer = my_client.sendAndReceive("PolyscopeVersion\n");
    std::cout << answer << std::endl;
    my_client.disconnect();
  }
  return 0;
}

```

```cmake
# CMakeLists.txt

cmake_minimum_required(VERSION 3.0.2)
project(minimal_example)

find_package(ur_client_library REQUIRED)
add_executable(db_client main.cpp)
target_link_libraries(db_client ur_client_library::urcl)
```

## License
The majority of this library is licensed under the Apache-2.0 licensed. However, certain parts are
licensed under different licenses:
 - The queue used inside the communication structures is originally written by Cameron Desrochers
   and is released under the BSD-2-Clause license.
 - The semaphore implementation used inside the queue implementation is written by Jeff Preshing and
   licensed under the zlib license
 - The Dockerfile used for the integration tests of this repository is originally written by Arran
   Hobson Sayers and released under the MIT license

While the main `LICENSE` file in this repository contains the Apache-2.0 license used for the
majority of the work, the respective libraries of third-party components reside together with the
code imported from those third parties.

## Library contents
현재 이 라이브러리는 다음과 같은 컴포넌트로 이루어져 있습니다:
 * **Basic primary interface:** 기본 인터페이스는 현재 완전히 구현되어 있지 않은 상태이며 기본 기능만 제공합니다. 추가 정보는  [A word on the primary / secondary
   interface](#a-word-on-the-primary--secondary-interface) 를 참고합니다.
 * **RTDE interface:** 이 라이브러리는 [RTDE interface](https://www.universal-robots.com/articles/ur-articles/real-time-data-exchange-rtde-guide/)를 완전히 지원합니다. 이 라이브러리를 RTDE 클라이언트로 사용하는 방법에 대한 자세한 정보는 [RTDEClient](#rtdeclient) 를 참고하세요.
 * **Dashboard interface:** 이 라이브러리를 사용하여 C++에서 helper 함수를 통해 [Dashboard server](https://www.universal-robots.com/articles/ur-articles/dashboard-server-e-series-port-29999/)에 직접 액세스할 수 있습니다.
 * **Custom motion streaming:** 이 라이브러리는 처음에는 y was initially developed as part of the [Universal
   Robots ROS driver](https://github.com/UniversalRobots/Universal_Robots_ROS_Driver) 일부로 개발되었습니다. 따라서 이 라이브러리는 커스텀 소켓을 통한 데이터 스트리밍 메커니즘도 포함하고 있으며, 예를 들어 모션 명령 스트리밍을 수행할 수 있습니다.

## 예제 driver
"examples"라는 하위 폴더에는 driver를 실행하는 최소 예제가 있습니다. `UrDriver` 클래스의 인스턴스를 시작하고 컨트롤러에서 읽은 RTDE 값을 출력합니다. 실행하려면 다음을 확인하십시오.
 * 설정한 IP 주소에서 실행 중인 로봇 컨트롤러 / URSim 인스턴스가 있어야 합니다 (또는 필요에 따라 주소를 조정하십시오)
 * 이 README.md 파일이 저장된 package의 main 폴더에서 실행하십시오. 간단한 이유로 필요한 파일을 찾는데 정교한 방법을 사용하지 않습니다.

## Architecture
아래 이미지는 개발자가 이 라이브러리에 있는 다른 모듈을 사용하는 데 도움이 되는 대략적인 아키텍처 개요를 보여줍니다. 이는 관련된 클래스에 대한 불완전한 보기임을 유의하십시오.

[![Data flow](doc/dataflow.svg "Data flow")](doc/dataflow.svg)

이 라이브러리의 핵심은 완벽하게 작동하는 로봇 인터페이스를 생성하는 `UrDriver` 클래스입니다. 사용 방법에 대한 자세한 내용은 예제 드라이버: [Example
driver](#example-driver) 섹션을 참조하십시오.

다음 내용에서 `UrDriver`의 모듈에 대해 설명합니다.

### RTDEClient
`RTDEClient` 클래스는 독립 실행형 [RTDE](https://www.universal-robots.com/articles/ur-articles/real-time-data-exchange-rtde-guide/) 클라이언트 역할을 합니다. RTDE-Client를 사용하려면, 별도로 초기화 및 시작해야 합니다.:

```c++
rtde_interface::RTDEClient my_client(ROBOT_IP, notifier, OUTPUT_RECIPE, INPUT_RECIPE);
my_client.init();
my_client.start();
while (true)
{
  std::unique_ptr<rtde_interface::DataPackage> data_pkg = my_client.getDataPackage(READ_TIMEOUT);
  if (data_pkg)
  {
    std::cout << data_pkg->toString() << std::endl;
  }
}
```

클래스를 생성할 때 RTDE 입력에 대한 레시피 파일 하나와 RTDE 출력에 대한 레시피 파일 하나를 제공해야 합니다. 사용 가능한 요소에 대한 자세한 내용은 [RTDE
guide](https://www.universal-robots.com/articles/ur-articles/real-time-data-exchange-rtde-guide/) 참조하십시오.

`RTDEclient` 안에서는 데이터가 별도의 스레드를 통해 수신되고, `RTDEParser`에 의해 파싱되어 파이프라인 큐에 추가됩니다.

`my_client.start()` 호출 직후, `getDataPackage()` 메서드를 통해 `RTDEClient`의 버퍼를 정기적으로 읽어야 합니다. 클라이언트의 큐는 한 번에 한 개의 아이템만 담을 수 있으므로, 다음 패키지가 도착하기 전에 버퍼를 읽지 않으면 `Pipeline producer overflowed!` 오류가 발생합니다.

RTDE 인터페이스에 데이터를 쓰려면 `RTDEClient`의 `RTDEWriter` 멤버를 사용합니다. 이는 `getWriter()` 메서드를 호출하여 가져올 수 있습니다. `RTDEWriter`는 RTDE 인터페이스에서 유효한 모든 데이터를 쓰기가 편리하도록 메서드를 제공합니다. 필요한 키가 입력 레시피 내부에 설정되어 있는지 확인하십시오. 데이터 필드가 레시피에 설정되지 않은 경우 send-methods가 `false`를 반환합니다.

독립 실행형 RTDE-client 예제는 `examples` 하위 폴더에서 찾을 수 있습니다. 실행하려면 다음을 확인하십시오:
 * 설정한 IP 주소에서 로봇 컨트롤러 / URSim 인스턴스가 실행 중이거나 (필요에 따라 주소를 조정하십시오)
 * 간단한 이유로 필요한 파일을 찾는 데 정교한 방법을 사용하지 않기 때문에 package의 main 폴더(이 README.md 파일이 저장된 폴더)에서 실행하십시오.

#### RTDEWriter
`RTDEWriter` 클래스는 RTDE 인터페이스에 데이터를 쓰는 기능을 제공합니다. 데이터 필드는 앞서 언급한 대로 `INPUT_RECIPE` 내부에 정의되어야 합니다.

이 클래스는 쓰기가 가능한 모든 RTDE 입력에 대해 특정 메서드를 제공합니다.

데이터는 비동기적으로 RTDE 인터페이스로 전송됩니다.

### ReverseInterface
`ReverseInterface`는 로봇과 제어 PC간에 커스텀 프로토콜이 구현된 TCP 포트를 열게 됩니다. 포트는 클래스 생성자에서 지정할 수 있습니다.

기본 기능은 벡터 형태의 실수 데이터와 함께 모드를 전송하는 것입니다. 이는 로봇에게 조인트 위치 또는 속도와 함께 해당 값을 해석하는 방법(예: `SERVOJ`, `SPEEDJ`)을 알려주는 데 사용됩니다. 따라서 이 인터페이스는 로봇에게 모션 명령 스트리밍을 수행하는 데 사용할 수 있습니다.

응용 프로그램에서 이 클래스를 로봇과 함께 사용하려면 로봇에서 전송된 명령을 해석할 수 있는 해당 URScript가 실행되고 있어야 합니다. 참고용으로 [this example
script](resources/external_control.urscript)를 확인하세요.

또한 [ScriptSender](#scriptsender) 를 통해 제어 PC에서 해당 UR스크립트를 정의하고 요청 시 로봇에게 전송하는 방법을 확인할 수 있습니다.

### ScriptSender

`ScriptSender` 클래스는 원격 PC 상에서 TCP 소켓을 열고, "*request_program*" 요청에 대해 URScript 코드 스니펫으로 응답하는 단일 목적을 가집니다. 스크립트 코드 자체는 클래스 생성자에게 제공되어야 합니다.

이 클래스를 외부 제어 [**External Control**
URCap](https://github.com/UniversalRobots/Universal_Robots_ExternalControl_URCap) 함께 사용하면 로봇에서 **External Control** 프로그램 node를 포함하는 프로그램을 시작할 때 해당 요청이 만들어 집니다. 제대로 작동하려면 로봇에서 IP 주소와 스크립트 송신 포트가 올바르게 구성되어 있는지 확인하십시오.

### Other public interface functions
This section shall explain the public interface functions that haven't been covered above

#### `check_calibration()`
This function opens a connection to the primary interface where it will receive a calibration
information as the first message. The checksum from this calibration info is compared to the one
given to this function. Connection to the primary interface is dropped afterwards.

#### `sendScript()`
This function sends given URScript code directly to the secondary interface. The
`sendRobotProgram()` function is a special case that will send the script code given in the
`RTDEClient` constructor.

### DashboardClient
`DashboardClient` 라이브러리는 [Dashboard server](https://www.universal-robots.com/articles/ur-articles/dashboard-server-e-series-port-29999/) 프로토콜에 대한 C++ 함수 인터페이스를 제공합니다.

`connect()` 함수를 이용하여 대시보드 서버에 연결 후
`sendAndReceive()` 함수를 통해 대시보드 명령을 보낼 수 있습니다.
대시보드 서버 응답은 문자열(string) 형태로 반환되며, 응답이 없는 경우 `UrException` 예외가 발생합니다.

Note: 반환된 문자열이 예상하는 내용인지 검증하는 기능은 포함되어 있지 않습니다. 개발자는 반환값을 직접 검증하거나  [DashboardClientROS](https://github.com/UniversalRobots/Universal_Robots_ROS_Driver/blob/master/ur_robot_driver/include/ur_robot_driver/dashboard_client_ros.h) 예제 코드를 참고하여 보다 완성된 상위 클래스를 만들어 사용하는 것이 좋습니다.

## A word on the primary / secondary interface
현재 이 라이브러리는 [Universal Robots
ROS driver](https://github.com/UniversalRobots/Universal_Robots_ROS_Driver)가 주로 RTDE 인터페이스를 기반으로 구축되었기 때문에 기본 인터페이스를 아주 잘 지원하지 않습니다. 따라서 기본 인터페이스에 직접 접근할 수 있는 `PrimaryClient`도 없습니다. 하지만 이는 향후 변경될 수도 있습니다.

`comm::URStream` 클래스를 사용하여 primary / secondary 인터페이스에 연결하고 데이터를 전송할 수 있습니다. 또한 [producer/consumer](#producer--consumer-architecture) 파이프라인 구조를 primary / secondary 인터페이스와 함께 사용할 수도 있습니다. 하지만 현재 대부분의 패키지에 대한 파싱 기능은 구현되어 있지 않습니다. 이 설정 방법에 대한 자세한 내용은 [`primary_pipeline` 예제](examples/primary_pipeline.cpp)를 참고하세요. 이 예제를 실행할 때, 라이브러리에 아직 구현되지 않았기 때문에 대부분의 패키지는 16진수 표기로 된 raw 바이트 스트림으로만 출력됩니다.

## A word on Real-Time scheduling
앞서 언급했듯이, clean operation을 위해서는 도착하는 RTDE 메시지가 다음 메시지가 도착하기 전에 읽혀지는 것이 매우 중요합니다. 이 때문에 RTDE 수신 스레드와 `getDataPackage()` 함수를 호출하는 스레드 모두 실시간 우선 순위로 스케줄링되어야 합니다. 설정 방법에 대한 자세한 내용은 [이 가이드](doc/real_time.md)를 참조하십시오.

RTDE 수신 스레드는 해당되는 경우 자동으로 실시간 우선 순위로 스케줄링됩니다. 이것이 작동하지 않으면 시작 시점에 오류가 발생합니다. `getDataPackage`를 호출하는 메인 스레드는 응용 프로그램에서 실시간 우선 순위로 스케줄링되어야 합니다. 예제는 [ur_robot_driver](https://github.com/UniversalRobots/Universal_Robots_ROS_Driver/blob/master/ur_robot_driver/src/hardware_interface_node.cpp)를 참조하십시오.

## Producer / Consumer architecture
primary / secondary 인터페이스 및 RTDE 인터페이스와의 통신은 consumer/producer 패턴을 사용하도록 설계되었습니다.
Producer는 소켓에서 데이터가 들어올 때마다 이를 읽고, 내용을 파싱하여 파싱된 패키지를 파이프라인 큐에 저장합니다.
사용자는 producer로부터 제공되는 패키지를 사용하는 자신만의 consumers를 작성할 수 있습니다. 예제로  [`comm::ShellConsumer`](include/ur_client_library/comm/shell_consumer.h) 를 참조하세요.

## Logging configuration
이 라이브러리는 원래 ROS 드라이버에 포함되도록 설계되었지만 독립 실행 라이브러리로도 사용될 수 있기 때문에, 직접적인 `printf` 또는 `std::cout` 대신에 커스텀 logging 매크로를 사용합니다.

매크로 기반 인터페이스는 기본적으로 [`DefaultLogHandler`](include/ur_client_library/default_log_handler.h)를 사용하여 로깅 메시지를 `printf` 구문으로 출력합니다. 사용자 정의 로그 처리기를 정의하여 동작을 변경하는 것이 가능합니다.[새로운 로그 핸들러 생성하기](#Create-new-log-handler)를 참조하십시오.

### Change logging level
응용 프로그램에서 로깅 레벨을 설정하세요. 기본적으로 WARNING 레벨 이상의 메시지만 출력됩니다.  아래 예시를 참조하세요.:
```c++
#include "ur_client_library/log.h"

int main(int argc, char* argv[])
{
  urcl::setLogLevel(urcl::LogLevel::DEBUG);

  URCL_LOG_DEBUG("Logging debug message");
  return 0;
}
```

### Create new log handler
logger는  라이브러리에서 로깅된 메시지에 대한 사용자 정의 로그 핸들러를 구현하기 위해 사용할 수 있는 인터페이스 [`LogHandler`](include/ur_client_library/log.h)를 제공합니다. 이는 `LogHandler class`를 상속받음으로써 구현할 수 있습니다.

만약 당신의 어플리케이션에서 새로운 로그 핸들러를 생성하고 싶다면, 아래 예시를 참고할 수 있습니다.:

```c++
#include "ur_client_library/log.h"
#include <iostream>

class MyLogHandler : public urcl::LogHandler
{
public:
  MyLogHandler() = default;

  void log(const char* file, int line, urcl::LogLevel loglevel, const char* log) override
  {
    switch (loglevel)
    {
      case urcl::LogLevel::INFO:
        std::cout << "INFO " << file << " " << line << ": " << log << std::endl;
        break;
      case urcl::LogLevel::DEBUG:
        std::cout << "DEBUG " << file << " " << line << ": " << log << std::endl;
        break;
      case urcl::LogLevel::WARN:
        std::cout << "WARN " << file << " " << line << ": " << log << std::endl;
        break;
      case urcl::LogLevel::ERROR:
        std::cout << "ERROR " << file << " " << line << ": " << log << std::endl;
        break;
      case urcl::LogLevel::FATAL:
        std::cout << "ERROR " << file << " " << line << ": " << log << std::endl;
        break;
      default:
        break;
    }
  }
};

int main(int argc, char* argv[])
{
  urcl::setLogLevel(urcl::LogLevel::DEBUG);
  std::unique_ptr<MyLogHandler> log_handler(new MyLogHandler);
  urcl::registerLogHandler(std::move(log_handler));

  URCL_LOG_DEBUG("logging debug message");
  URCL_LOG_INFO("logging info message");
  return 0;
}
```

## Contributor Guidelines
* 이 repo는 자동 코드 포맷팅 등을 위한 [pre-commit](https://pre-commit.com/) 지원:
  이를 통해 잘못 포맷팅된 코드를 커밋하는 것을 방지할 수 있습니다.:
  ```
  pipx install pre-commit
  pre-commit install
  ```
* PULL 요청 시 파이프라인 성공은 필수입니다 (예를 들어, 상위 스트림 변경이 있었던 경우는 제외).
* 코드 커버리지를 높이고 유지하려고 노력하고 있으므로 새로운 기능을 포함하는 PR에도 해당 기능을 테스트하는 코드가 함께 있어야 합니다.
* 공개 메소드의 파라미터는 모두 문서화되어야 합니다.

## Acknowledgment
이 라이브러리의 많은 부분들은 [ur_modern_driver](https://github.com/ros-industrial/ur_modern_driver) 에서 fork되었습니다.

Developed in collaboration between:

[<img height="60" alt="Universal Robots A/S" src="doc/resources/ur_logo.jpg">](https://www.universal-robots.com/) &nbsp; and &nbsp;
[<img height="60" alt="FZI Research Center for Information Technology" src="doc/resources/fzi-logo_transparenz.png">](https://www.fzi.de).

<!--
    ROSIN acknowledgement from the ROSIN press kit
    @ https://github.com/rosin-project/press_kit
-->

<a href="http://rosin-project.eu">
  <img src="http://rosin-project.eu/wp-content/uploads/rosin_ack_logo_wide.png"
       alt="rosin_logo" height="60" >
</a>

Supported by ROSIN - ROS-Industrial Quality-Assured Robot Software Components.
More information: <a href="http://rosin-project.eu">rosin-project.eu</a>

<img src="http://rosin-project.eu/wp-content/uploads/rosin_eu_flag.jpg"
     alt="eu_flag" height="45" align="left" >

This project has received funding from the European Union’s Horizon 2020
research and innovation programme under grant agreement no. 732287.
