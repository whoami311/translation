# 1. Getting Started

本节定义了 DDS 和 RTPS 的概念。它还提供了一个逐步教程，指导如何编写一个简单的 Fast DDS（前身为 Fast RTPS）发布/订阅应用程序。

## 1.1. What is DDS?

数据分发服务（DDS）是一种以数据为中心的通信协议，用于分布式软件应用程序之间的通信。它描述了通信的应用程序编程接口（API）和通信语义，使数据提供者和数据消费者之间能够进行通信。

由于它采用的是以数据为中心的发布-订阅（DCPS）模型，因此在其实现中定义了三个关键的应用实体：发布实体，用于定义生成信息的对象及其属性；订阅实体，用于定义消费信息的对象及其属性；以及配置实体，用于定义作为主题传输的信息类型，并创建发布者和订阅者及其服务质量（QoS）属性，确保上述实体的正确性能。

DDS 使用 QoS 来定义 DDS 实体的行为特征。QoS 由单独的 QoS 策略组成（继承自 QoSPolicy 类型的对象）。这些内容在 Policy 中进行了描述。

### 1.1.1. The DCPS conceptual model

在 DCPS 模型中，定义了四个基本元素，用于开发通信应用系统：

**发布者。** 发布者是负责创建和配置其实现的 **DataWriter** 的 DCPS 实体。**DataWriter** 是负责实际发布消息的实体。每个 **DataWriter** 都会有一个分配的主题（Topic），用于发布消息。详细内容请参见 Publisher。

**订阅者。** 订阅者是负责接收在其订阅的主题下发布的数据的 DCPS 实体。它服务于一个或多个 **DataReader** 对象，后者负责将新数据的可用性传递给应用程序。详细内容请参见 Subscriber。

**主题。** 主题是连接发布和订阅的实体。在 DDS 域中，它是唯一的。通过 **TopicDescription**，它允许统一发布和订阅的数据类型。详细内容请参见 Topic。

**域。** 域是用于将属于一个或多个应用程序的所有发布者和订阅者连接起来的概念，这些应用程序在不同的主题下交换数据。参与域的独立应用程序被称为 **DomainParticipant**。DDS 域通过域 ID 进行标识。DomainParticipant 定义域 ID，以指定其所属的 DDS 域。两个具有不同 ID 的 DomainParticipant 无法意识到彼此在网络中的存在。因此，可以创建多个通信通道。这种方法适用于涉及多个 DDS 应用程序的场景，其中各自的 **DomainParticipant** 相互通信，但这些应用程序之间必须保持独立，不干扰。**DomainParticipant** 充当其他 DCPS 实体的容器，是 **Publisher**、**Subscriber** 和 **Topic** 实体的工厂，并提供域内的管理服务。详细内容请参见 Domain。

这些元素如下图所示。

![dds_domain](img/dds_domain.svg)

DDS 域中的 DCPS 模型实体。

## 1.2. What is RTPS?

实时发布-订阅（RTPS）协议是为支持 DDS 应用程序而开发的，是一种基于最佳努力传输（如 UDP/IP）的发布-订阅通信中间件。此外，Fast DDS 还支持 TCP 和共享内存（SHM）传输。

它设计支持单播和多播通信。

在 RTPS 的顶部，继承自 DDS，可以找到 **Domain**，它定义了一个独立的通信平面。多个域可以同时独立共存。一个域包含任意数量的 **RTPSParticipant**，即能够发送和接收数据的元素。为了实现这一点，RTPSParticipant 使用它们的 **Endpoints**：

- **RTPSWriter**：能够发送数据的端点。
- **RTPSReader**：能够接收数据的端点。

一个 RTPSParticipant 可以拥有任意数量的写入端点和读取端点。

![rtps_domain](img/rtps_domain.svg)

RTPS 高级架构

通信围绕 **Topics** 展开，主题定义并标记正在交换的数据。主题不属于特定的参与者。参与者通过 RTPSWriter 修改发布在主题下的数据，并通过 RTPSReader 接收与其订阅的主题相关的数据。通信单元被称为 **Change**，它表示在主题下写入的数据更新。**RTPSReader/RTPSWriter** 将这些变化注册到它们的 **History** 中，这是一个作为最近变化缓存的数据结构。

在 *eProsima Fast DDS* 的默认配置中，当通过 RTPSWriter 端点发布 *change* 时，后台会发生以下步骤：

1. *change* 被添加到 RTPSWriter 的历史缓存中。
2. RTPSWriter 将变化发送给它知道的任何 RTPSReader。
3. RTPSReader 接收到数据后，会使用新变化更新它们的历史缓存。

然而，Fast DDS 支持多种配置，允许您更改 RTPSWriter/RTPSReader 的行为。对 RTPS 实体默认配置的修改会影响 RTPSWriter 和 RTPSReader 之间的数据交换流程。此外，通过选择服务质量（QoS）策略，您可以影响历史缓存的管理方式，但通信循环保持不变。您可以继续阅读 RTPS Layer 章节，了解 Fast DDS 中 RTPS 协议的实现。

## 1.3. Writing a simple C++ publisher and subscriber application

本节详细说明如何使用 C++ API 步骤性地创建一个简单的 Fast DDS 应用程序，包含一个发布者和一个订阅者。也可以通过使用 *eProsima Fast DDS-Gen* 工具自生成一个类似于本节实现的示例。这个额外的方法在 Building a publish/subscribe application 章节中进行了说明。

### 1.3.1. Background

DDS 是一种以数据为中心的通信中间件，实现了 DCPS 模型。该模型基于发布者（数据生成元素）和订阅者（数据消费元素）的开发。这些实体通过主题（Topic）进行通信，主题是连接这两个 DDS 实体的元素。发布者在某个主题下生成信息，订阅者则订阅同一个主题以接收信息。

### 1.3.2. Prerequisites

首先，您需要按照《安装手册》中列出的步骤安装 *eProsima Fast DDS* 及其所有依赖项。同时，您还需要完成《安装手册》中关于安装 *eProsima Fast DDS-Gen* 工具的相关步骤。此外，本教程中提供的所有命令均基于 Linux 环境编写。

### 1.3.3. Create the application workspace

在项目完成时，应用程序的工作空间将具有如下结构。`build/DDSHelloWorldPublisher` 和 `build/DDSHelloWorldSubscriber` 文件分别是发布者应用程序和订阅者应用程序。

```
.
└── workspace_DDSHelloWorld
    ├── build
    │   ├── CMakeCache.txt
    │   ├── CMakeFiles
    │   ├── cmake_install.cmake
    │   ├── DDSHelloWorldPublisher
    │   ├── DDSHelloWorldSubscriber
    │   └── Makefile
    ├── CMakeLists.txt
    └── src
        ├── HelloWorld.hpp
        ├── HelloWorld.idl
        ├── HelloWorldCdrAux.hpp
        ├── HelloWorldCdrAux.ipp
        ├── HelloWorldPublisher.cpp
        ├── HelloWorldPubSubTypes.cxx
        ├── HelloWorldPubSubTypes.h
        ├── HelloWorldSubscriber.cpp
        ├── HelloWorldTypeObjectSupport.cxx
        └── HelloWorldTypeObjectSupport.hpp
```

让我们首先创建目录结构。

```shell
mkdir workspace_DDSHelloWorld && cd workspace_DDSHelloWorld
mkdir src build
```

### 1.3.4. Import linked libraries and its dependencies

DDS 应用程序需要依赖 Fast DDS 和 Fast CDR 库。根据所采用的安装方式不同，使这些库可用于我们的 DDS 应用程序的过程也会略有不同。

#### 1.3.4.1. Installation from binaries and manual installation

如果我们是通过二进制安装或手动安装完成的，那么这些库已经可以在工作空间中访问。在 Linux 系统中，Fast DDS 和 Fast CDR 的头文件分别位于目录 */usr/include/fastdds/* 和 */usr/include/fastcdr/* 中。这两个库的已编译文件位于目录 */usr/lib/* 中。

#### 1.3.4.2. Colcon installation

如果是通过 Colcon 安装的方式，有多种方式可以导入这些库。如果这些库只需要在当前会话中可用，请运行以下命令。

```shell
source <path/to/Fast-DDS/workspace>/install/setup.bash
```

可以通过在当前用户的 shell 配置文件中添加 Fast DDS 安装目录到 `$PATH` 变量，使其在任何会话中都可访问，运行以下命令。

```shell
echo 'source <path/to/Fast-DDS/workspace>/install/setup.bash' >> ~/.bashrc
```

这将在每次该用户登录后设置好环境。

### 1.3.5. Configure the CMake project

我们将使用 CMake 工具来管理项目的构建。请使用您喜欢的文本编辑器创建一个名为 CMakeLists.txt 的新文件，并将以下代码片段复制粘贴进去。将此文件保存在工作空间的根目录下。如果您已按照这些步骤操作，该目录应为 *workspace_DDSHelloWorld*。

```cmake
cmake_minimum_required(VERSION 3.20)

project(DDSHelloWorld)

# Find requirements
if(NOT fastcdr_FOUND)
    find_package(fastcdr 2 REQUIRED)
endif()

if(NOT fastdds_FOUND)
    find_package(fastdds 3 REQUIRED)
endif()

# Set C++11
include(CheckCXXCompilerFlag)
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_COMPILER_IS_CLANG OR
        CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    check_cxx_compiler_flag(-std=c++11 SUPPORTS_CXX11)
    if(SUPPORTS_CXX11)
        add_compile_options(-std=c++11)
    else()
        message(FATAL_ERROR "Compiler doesn't support C++11")
    endif()
endif()

message(STATUS "Configuring HelloWorld publisher/subscriber example...")
file(GLOB DDS_HELLOWORLD_SOURCES_CXX "src/*.cxx")
```

在每一部分中，我们将补充此文件，以包含具体生成的文件。

### 1.3.6. Build the topic data type

*eProsima Fast DDS-Gen* 是一个 Java 应用程序，它使用在接口描述语言（IDL）文件中定义的数据类型生成源代码。此应用程序可以执行两种不同的操作：

1. 为自定义主题生成 C++ 定义。
2. 生成一个使用该主题数据的功能性示例。

本教程将采用第一种方式。如需了解第二种方式的应用示例，可参阅另一个示例。详见 Introduction 部分。在本项目中，我们将使用 Fast DDS-Gen 应用程序来定义由发布者发送、订阅者接收的消息的数据类型。

在工作空间目录中，执行以下命令：

```shell
cd src && touch HelloWorld.idl
```

这将在 *src* 目录中创建 HelloWorld.idl 文件。用文本编辑器打开该文件，并复制粘贴以下代码片段。

```c++
struct HelloWorld
{
    unsigned long index;
    string message;
};
```

通过这样做，我们定义了一个名为 `HelloWorld` 的数据类型，它包含两个元素：一个 `uint32_t` 类型的 *index* 和一个 `std::string` 类型的 *message*。接下来只需生成实现该数据类型的 C++11 源代码。为此，请在 `src` 目录中运行以下命令。

```
<path/to/Fast DDS-Gen>/scripts/fastddsgen HelloWorld.idl
```

这将会生成以下文件：

- HelloWorld.hpp：HelloWorld 类型的定义。
- HelloWorldPubSubTypes.cxx：Fast DDS 用于支持 HelloWorld 类型的接口实现。
- HelloWorldPubSubTypes.h：HelloWorldPubSubTypes.cxx 的头文件。
- HelloWorldCdrAux.ipp：HelloWorld 类型的序列化与反序列化代码。
- HelloWorldCdrAux.hpp：HelloWorldCdrAux.ipp 的头文件。
- HelloWorldTypeObjectSupport.cxx：TypeObject 表示的注册代码。
- HelloWorldTypeObjectSupport.hpp：HelloWorldTypeObjectSupport.cxx 的头文件。

### 1.3.7. Write the Fast DDS publisher

在工作空间的 *src* 目录中运行以下命令以下载 HelloWorldPublisher.cpp 文件。

```shell
wget -O HelloWorldPublisher.cpp \
    https://raw.githubusercontent.com/eProsima/Fast-RTPS-docs/master/code/Examples/C++/DDSHelloWorld/src/HelloWorldPublisher.cpp
```

这是发布者应用程序的 C++ 源代码。它将在主题 *HelloWorldTopic* 下发送 10 条消息。

```c++
// Copyright 2016 Proyectos y Sistemas de Mantenimiento SL (eProsima).
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/**
 * @file HelloWorldPublisher.cpp
 *
 */

#include "HelloWorldPubSubTypes.hpp"

#include <chrono>
#include <thread>

#include <fastdds/dds/domain/DomainParticipant.hpp>
#include <fastdds/dds/domain/DomainParticipantFactory.hpp>
#include <fastdds/dds/publisher/DataWriter.hpp>
#include <fastdds/dds/publisher/DataWriterListener.hpp>
#include <fastdds/dds/publisher/Publisher.hpp>
#include <fastdds/dds/topic/TypeSupport.hpp>

using namespace eprosima::fastdds::dds;

class HelloWorldPublisher
{
private:

    HelloWorld hello_;

    DomainParticipant* participant_;

    Publisher* publisher_;

    Topic* topic_;

    DataWriter* writer_;

    TypeSupport type_;

    class PubListener : public DataWriterListener
    {
    public:

        PubListener()
            : matched_(0)
        {
        }

        ~PubListener() override
        {
        }

        void on_publication_matched(
                DataWriter*,
                const PublicationMatchedStatus& info) override
        {
            if (info.current_count_change == 1)
            {
                matched_ = info.total_count;
                std::cout << "Publisher matched." << std::endl;
            }
            else if (info.current_count_change == -1)
            {
                matched_ = info.total_count;
                std::cout << "Publisher unmatched." << std::endl;
            }
            else
            {
                std::cout << info.current_count_change
                        << " is not a valid value for PublicationMatchedStatus current count change." << std::endl;
            }
        }

        std::atomic_int matched_;

    } listener_;

public:

    HelloWorldPublisher()
        : participant_(nullptr)
        , publisher_(nullptr)
        , topic_(nullptr)
        , writer_(nullptr)
        , type_(new HelloWorldPubSubType())
    {
    }

    virtual ~HelloWorldPublisher()
    {
        if (writer_ != nullptr)
        {
            publisher_->delete_datawriter(writer_);
        }
        if (publisher_ != nullptr)
        {
            participant_->delete_publisher(publisher_);
        }
        if (topic_ != nullptr)
        {
            participant_->delete_topic(topic_);
        }
        DomainParticipantFactory::get_instance()->delete_participant(participant_);
    }

    //!Initialize the publisher
    bool init()
    {
        hello_.index(0);
        hello_.message("HelloWorld");

        DomainParticipantQos participantQos;
        participantQos.name("Participant_publisher");
        participant_ = DomainParticipantFactory::get_instance()->create_participant(0, participantQos);

        if (participant_ == nullptr)
        {
            return false;
        }

        // Register the Type
        type_.register_type(participant_);

        // Create the publications Topic
        topic_ = participant_->create_topic("HelloWorldTopic", "HelloWorld", TOPIC_QOS_DEFAULT);

        if (topic_ == nullptr)
        {
            return false;
        }

        // Create the Publisher
        publisher_ = participant_->create_publisher(PUBLISHER_QOS_DEFAULT, nullptr);

        if (publisher_ == nullptr)
        {
            return false;
        }

        // Create the DataWriter
        writer_ = publisher_->create_datawriter(topic_, DATAWRITER_QOS_DEFAULT, &listener_);

        if (writer_ == nullptr)
        {
            return false;
        }
        return true;
    }

    //!Send a publication
    bool publish()
    {
        if (listener_.matched_ > 0)
        {
            hello_.index(hello_.index() + 1);
            writer_->write(&hello_);
            return true;
        }
        return false;
    }

    //!Run the Publisher
    void run(
            uint32_t samples)
    {
        uint32_t samples_sent = 0;
        while (samples_sent < samples)
        {
            if (publish())
            {
                samples_sent++;
                std::cout << "Message: " << hello_.message() << " with index: " << hello_.index()
                            << " SENT" << std::endl;
            }
            std::this_thread::sleep_for(std::chrono::milliseconds(1000));
        }
    }
};

int main(
        int argc,
        char** argv)
{
    std::cout << "Starting publisher." << std::endl;
    uint32_t samples = 10;

    HelloWorldPublisher* mypub = new HelloWorldPublisher();
    if(mypub->init())
    {
        mypub->run(samples);
    }

    delete mypub;
    return 0;
}
```

#### 1.3.7.1. Examining the code

在文件开头，我们有一个 Doxygen 风格的注释块，其中的 `@file` 字段告诉我们该文件的名称。

```c++
/**
 * @file HelloWorldPublisher.cpp
 *
 */
```

下面是 C++ 头文件的包含部分。第一个包含的是 HelloWorldPubSubTypes.h 文件，其中包含了我们在上一节中定义的数据类型的序列化与反序列化函数。

```c++
#include "HelloWorldPubSubTypes.hpp"
```

下一个代码块包含了允许使用 Fast DDS API 的 C++ 头文件：

- `DomainParticipantFactory`：`用于创建和销毁 DomainParticipant` 对象。  
- `DomainParticipant`：作为所有其他 Entity 对象的容器，并作为创建 Publisher、Subscriber 和 Topic 对象的工厂。  
- `TypeSupport`：为 participant 提供特定数据类型的序列化、反序列化以及获取键值的函数。  
- `Publisher`：负责创建 `DataWriter` 的对象。  
- `DataWriter`：允许应用程序设置要在指定 Topic 下发布的数据值。  
- `DataWriterListener`：允许重定义 `DataWriterListener` 的函数。

```c++
#include <chrono>
#include <thread>

#include <fastdds/dds/domain/DomainParticipant.hpp>
#include <fastdds/dds/domain/DomainParticipantFactory.hpp>
#include <fastdds/dds/publisher/DataWriter.hpp>
#include <fastdds/dds/publisher/DataWriterListener.hpp>
#include <fastdds/dds/publisher/Publisher.hpp>
#include <fastdds/dds/topic/TypeSupport.hpp>
```

接下来，我们定义命名空间，该命名空间包含了我们将在应用程序中使用的 eProsima Fast DDS 类和函数。

```c++
using namespace eprosima::fastdds::dds;
```

下一行创建了一个名为 HelloWorldPublisher 的类，该类实现了一个发布者。

```c++
class HelloWorldPublisher
```

接着类的私有数据成员定义，`hello_` 数据成员被定义为 `HelloWorld` 类的对象，该类表示我们通过 IDL 文件创建的数据类型。接下来定义了与 participant、publisher、topic、DataWriter 和数据类型相关的私有数据成员。`TypeSupport` 类的 `type_` 对象将用于在 `DomainParticipant` 中注册主题数据类型。

```c++
private:

    HelloWorld hello_;

    DomainParticipant* participant_;

    Publisher* publisher_;

    Topic* topic_;

    DataWriter* writer_;

    TypeSupport type_;
```

然后，`PubListener` 类通过继承 `DataWriterListener` 类被定义。该类重写了默认的 `DataWriter` 监听器回调函数，从而允许在发生事件时执行一系列操作。被重写的回调函数 `on_publication_matched()` 允许在检测到有新的 `DataReader` 正在监听 `DataWriter` 发布的主题时定义一系列操作。`info.current_count_change` 用于检测与 `DataWriter` 匹配的 `DataReader` 的变化。该字段是 `MatchedStatus` 结构体的一个成员，用于跟踪订阅状态的变化。最后，类中的 `listener_` 对象被定义为 `PubListener` 的一个实例。

```c++
class PubListener : public DataWriterListener
{
public:

    PubListener()
        : matched_(0)
    {
    }

    ~PubListener() override
    {
    }

    void on_publication_matched(
            DataWriter*,
            const PublicationMatchedStatus& info) override
    {
        if (info.current_count_change == 1)
        {
            matched_ = info.total_count;
            std::cout << "Publisher matched." << std::endl;
        }
        else if (info.current_count_change == -1)
        {
            matched_ = info.total_count;
            std::cout << "Publisher unmatched." << std::endl;
        }
        else
        {
            std::cout << info.current_count_change
                    << " is not a valid value for PublicationMatchedStatus current count change." << std::endl;
        }
    }

    std::atomic_int matched_;

} listener_;
```

下面定义了 `HelloWorldPublisher` 类的公有构造函数和析构函数。构造函数将该类的私有数据成员初始化为 `nullptr`，但 `TypeSupport` 对象除外，它被初始化为 `HelloWorldPubSubType` 类的一个实例。类的析构函数则释放这些数据成员，从而清理系统内存。

```c++
HelloWorldPublisher()
    : participant_(nullptr)
    , publisher_(nullptr)
    , topic_(nullptr)
    , writer_(nullptr)
    , type_(new HelloWorldPubSubType())
{
}

virtual ~HelloWorldPublisher()
{
    if (writer_ != nullptr)
    {
        publisher_->delete_datawriter(writer_);
    }
    if (publisher_ != nullptr)
    {
        participant_->delete_publisher(publisher_);
    }
    if (topic_ != nullptr)
    {
        participant_->delete_topic(topic_);
    }
    DomainParticipantFactory::get_instance()->delete_participant(participant_);
}
```

继续介绍 `HelloWorldPublisher` 类的公有成员函数，下一段代码定义了该类的公有发布者初始化成员函数。该函数执行以下操作：

1. 初始化 `HelloWorld` 类型的 `hello_` 结构体成员的内容。
2. 通过 `DomainParticipant` 的 QoS 为参与者指定名称。
3. 使用 `DomainParticipantFactory` 创建参与者。
4. 注册在 IDL 中定义的数据类型。
5. 为发布创建主题。
6. 创建发布者。
7. 使用先前创建的监听器创建 `DataWriter`。

如你所见，除了参与者的名称之外，所有实体的 QoS 配置均为默认配置（`PARTICIPANT_QOS_DEFAULT`、`PUBLISHER_QOS_DEFAULT`、`TOPIC_QOS_DEFAULT`、`DATAWRITER_QOS_DEFAULT`）。每个 DDS 实体的 QoS 默认值可参考 DDS 标准。

```c++
//!Initialize the publisher
bool init()
{
    hello_.index(0);
    hello_.message("HelloWorld");

    DomainParticipantQos participantQos;
    participantQos.name("Participant_publisher");
    participant_ = DomainParticipantFactory::get_instance()->create_participant(0, participantQos);

    if (participant_ == nullptr)
    {
        return false;
    }

    // Register the Type
    type_.register_type(participant_);

    // Create the publications Topic
    topic_ = participant_->create_topic("HelloWorldTopic", "HelloWorld", TOPIC_QOS_DEFAULT);

    if (topic_ == nullptr)
    {
        return false;
    }

    // Create the Publisher
    publisher_ = participant_->create_publisher(PUBLISHER_QOS_DEFAULT, nullptr);

    if (publisher_ == nullptr)
    {
        return false;
    }

    // Create the DataWriter
    writer_ = publisher_->create_datawriter(topic_, DATAWRITER_QOS_DEFAULT, &listener_);

    if (writer_ == nullptr)
    {
        return false;
    }
    return true;
}
```

为了进行发布，实现了公有成员函数 `publish()`。在 `DataWriter` 的监听器回调中，当 `DataWriter` 与监听发布主题的 `DataReader` 匹配时，数据成员 `matched_` 会被更新。该成员包含已发现的 `DataReader` 数量。因此，当发现第一个 `DataReader` 后，应用程序就开始发布。这实际上就是通过 `DataWriter` 对象*写入*一个变更。

```c++
//!Send a publication
bool publish()
{
    if (listener_.matched_ > 0)
    {
        hello_.index(hello_.index() + 1);
        writer_->write(&hello_);
        return true;
    }
    return false;
}
```

公有的 *run* 函数执行发布操作，发布指定次数，每次发布之间等待 1 秒。

```c++
//!Run the Publisher
void run(
        uint32_t samples)
{
    uint32_t samples_sent = 0;
    while (samples_sent < samples)
    {
        if (publish())
        {
            samples_sent++;
            std::cout << "Message: " << hello_.message() << " with index: " << hello_.index()
                        << " SENT" << std::endl;
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
}
```

最后，在 `main` 函数中初始化并运行 `HelloWorldPublisher`。

```c++
int main(
        int argc,
        char** argv)
{
    std::cout << "Starting publisher." << std::endl;
    uint32_t samples = 10;

    HelloWorldPublisher* mypub = new HelloWorldPublisher();
    if(mypub->init())
    {
        mypub->run(samples);
    }

    delete mypub;
    return 0;
}
```

#### 1.3.7.2. CMakeLists.txt

在你之前创建的 CMakeLists.txt 文件末尾包含以下代码片段。此操作会添加构建可执行文件所需的所有源文件，并将可执行文件与库链接在一起。

```cmake
add_executable(DDSHelloWorldPublisher src/HelloWorldPublisher.cpp ${DDS_HELLOWORLD_SOURCES_CXX})
target_link_libraries(DDSHelloWorldPublisher fastdds fastcdr)
```

此时，项目已准备好构建、编译并运行发布者应用程序。在工作区的 build 目录中，运行以下命令。

```cmake
cmake ..
cmake --build .
./DDSHelloWorldPublisher
```

### 1.3.8. Write the Fast DDS subscriber

在工作区的 *src* 目录中，执行以下命令以下载 HelloWorldSubscriber.cpp 文件。

```shell
wget -O HelloWorldSubscriber.cpp \
    https://raw.githubusercontent.com/eProsima/Fast-RTPS-docs/master/code/Examples/C++/DDSHelloWorld/src/HelloWorldSubscriber.cpp
```

这是订阅者应用程序的 C++ 源代码。该应用程序会运行一个订阅者，直到在 *HelloWorldTopic* 主题下接收到 10 个样本为止。此时，订阅者停止运行。

```c++
// Copyright 2016 Proyectos y Sistemas de Mantenimiento SL (eProsima).
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/**
 * @file HelloWorldSubscriber.cpp
 *
 */

#include "HelloWorldPubSubTypes.hpp"

#include <chrono>
#include <thread>

#include <fastdds/dds/domain/DomainParticipant.hpp>
#include <fastdds/dds/domain/DomainParticipantFactory.hpp>
#include <fastdds/dds/subscriber/DataReader.hpp>
#include <fastdds/dds/subscriber/DataReaderListener.hpp>
#include <fastdds/dds/subscriber/qos/DataReaderQos.hpp>
#include <fastdds/dds/subscriber/SampleInfo.hpp>
#include <fastdds/dds/subscriber/Subscriber.hpp>
#include <fastdds/dds/topic/TypeSupport.hpp>

using namespace eprosima::fastdds::dds;

class HelloWorldSubscriber
{
private:

    DomainParticipant* participant_;

    Subscriber* subscriber_;

    DataReader* reader_;

    Topic* topic_;

    TypeSupport type_;

    class SubListener : public DataReaderListener
    {
    public:

        SubListener()
            : samples_(0)
        {
        }

        ~SubListener() override
        {
        }

        void on_subscription_matched(
                DataReader*,
                const SubscriptionMatchedStatus& info) override
        {
            if (info.current_count_change == 1)
            {
                std::cout << "Subscriber matched." << std::endl;
            }
            else if (info.current_count_change == -1)
            {
                std::cout << "Subscriber unmatched." << std::endl;
            }
            else
            {
                std::cout << info.current_count_change
                          << " is not a valid value for SubscriptionMatchedStatus current count change" << std::endl;
            }
        }

        void on_data_available(
                DataReader* reader) override
        {
            SampleInfo info;
            if (reader->take_next_sample(&hello_, &info) == eprosima::fastdds::dds::RETCODE_OK)
            {
                if (info.valid_data)
                {
                    samples_++;
                    std::cout << "Message: " << hello_.message() << " with index: " << hello_.index()
                              << " RECEIVED." << std::endl;
                }
            }
        }

        HelloWorld hello_;

        std::atomic_int samples_;

    }
    listener_;

public:

    HelloWorldSubscriber()
        : participant_(nullptr)
        , subscriber_(nullptr)
        , topic_(nullptr)
        , reader_(nullptr)
        , type_(new HelloWorldPubSubType())
    {
    }

    virtual ~HelloWorldSubscriber()
    {
        if (reader_ != nullptr)
        {
            subscriber_->delete_datareader(reader_);
        }
        if (topic_ != nullptr)
        {
            participant_->delete_topic(topic_);
        }
        if (subscriber_ != nullptr)
        {
            participant_->delete_subscriber(subscriber_);
        }
        DomainParticipantFactory::get_instance()->delete_participant(participant_);
    }

    //!Initialize the subscriber
    bool init()
    {
        DomainParticipantQos participantQos;
        participantQos.name("Participant_subscriber");
        participant_ = DomainParticipantFactory::get_instance()->create_participant(0, participantQos);

        if (participant_ == nullptr)
        {
            return false;
        }

        // Register the Type
        type_.register_type(participant_);

        // Create the subscriptions Topic
        topic_ = participant_->create_topic("HelloWorldTopic", "HelloWorld", TOPIC_QOS_DEFAULT);

        if (topic_ == nullptr)
        {
            return false;
        }

        // Create the Subscriber
        subscriber_ = participant_->create_subscriber(SUBSCRIBER_QOS_DEFAULT, nullptr);

        if (subscriber_ == nullptr)
        {
            return false;
        }

        // Create the DataReader
        reader_ = subscriber_->create_datareader(topic_, DATAREADER_QOS_DEFAULT, &listener_);

        if (reader_ == nullptr)
        {
            return false;
        }

        return true;
    }

    //!Run the Subscriber
    void run(
            uint32_t samples)
    {
        while (listener_.samples_ < samples)
        {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }

};

int main(
        int argc,
        char** argv)
{
    std::cout << "Starting subscriber." << std::endl;
    uint32_t samples = 10;

    HelloWorldSubscriber* mysub = new HelloWorldSubscriber();
    if (mysub->init())
    {
        mysub->run(samples);
    }

    delete mysub;
    return 0;
}
```

#### 1.3.8.1. Examining the code

由于发布者和订阅者应用程序的源代码大部分是相同的，本文档将重点说明它们之间的主要区别，省略已经解释过的代码部分。

按照与发布者说明相同的结构，第一步是包含 C++ 头文件。在这些文件中，将包含发布者类的文件替换为订阅者类，同时将数据写入器类替换为数据读取器类。

- `Subscriber`：负责创建和配置 `DataReader` 的对象。
- `DataReader`：负责实际接收数据的对象。它在应用程序中注册标识要读取数据的主题（`TopicDescription`），并访问订阅者接收到的数据。
- `DataReaderListener`：分配给数据读取器的监听器。
- `DataReaderQoS`：定义 `DataReader` 质量服务（QoS）的结构。
- `SampleInfo`：是附加在每个“读取”或“获取”的样本上的信息。

```c++
#include <fastdds/dds/domain/DomainParticipant.hpp>
#include <fastdds/dds/domain/DomainParticipantFactory.hpp>
#include <fastdds/dds/subscriber/DataReader.hpp>
#include <fastdds/dds/subscriber/DataReaderListener.hpp>
#include <fastdds/dds/subscriber/qos/DataReaderQos.hpp>
#include <fastdds/dds/subscriber/SampleInfo.hpp>
#include <fastdds/dds/subscriber/Subscriber.hpp>
#include <fastdds/dds/topic/TypeSupport.hpp>
```

下一行定义了 `HelloWorldSubscriber` 类，该类实现了一个订阅者。

```c++
class HelloWorldSubscriber
```

从类的私有数据成员开始，值得一提的是数据读取器监听器（data reader listener）的实现。该类的私有数据成员包括参与者（participant）、订阅者（subscriber）、主题（topic）、数据读取器（data reader）和数据类型（data type）。与数据写入器一样，监听器实现了在发生事件时要执行的回调函数。SubListener 的第一个重写回调函数是 `on_subscription_matched()`，它是 DataWriter 的 `on_publication_matched()` 回调函数的对应项。

```c++
void on_subscription_matched(
        DataReader*,
        const SubscriptionMatchedStatus& info) override
{
    if (info.current_count_change == 1)
    {
        std::cout << "Subscriber matched." << std::endl;
    }
    else if (info.current_count_change == -1)
    {
        std::cout << "Subscriber unmatched." << std::endl;
    }
    else
    {
        std::cout << info.current_count_change
                  << " is not a valid value for SubscriptionMatchedStatus current count change" << std::endl;
    }
}
```

第二个被重写的回调函数是 `on_data_available()`。在该函数中，会获取并处理数据读取器可访问的下一个接收样本，以显示其内容。正是在这里定义了 `SampleInfo` 类的对象，它用于判断某个样本是否已被读取或提取。每当一个样本被读取时，接收的样本计数器就会增加。

```c++
void on_data_available(
        DataReader* reader) override
{
    SampleInfo info;
    if (reader->take_next_sample(&hello_, &info) == eprosima::fastdds::dds::RETCODE_OK)
    {
        if (info.valid_data)
        {
            samples_++;
            std::cout << "Message: " << hello_.message() << " with index: " << hello_.index()
                      << " RECEIVED." << std::endl;
        }
    }
}
```

下面定义了该类的公共构造函数和析构函数。

```c++
HelloWorldSubscriber()
    : participant_(nullptr)
    , subscriber_(nullptr)
    , topic_(nullptr)
    , reader_(nullptr)
    , type_(new HelloWorldPubSubType())
{
}

virtual ~HelloWorldSubscriber()
{
    if (reader_ != nullptr)
    {
        subscriber_->delete_datareader(reader_);
    }
    if (topic_ != nullptr)
    {
        participant_->delete_topic(topic_);
    }
    if (subscriber_ != nullptr)
    {
        participant_->delete_subscriber(subscriber_);
    }
    DomainParticipantFactory::get_instance()->delete_participant(participant_);
```

接下来是订阅者的初始化公共成员函数。这与 `HelloWorldPublisher` 中定义的初始化公共成员函数相同。除参与者名称外，所有实体的 QoS 配置均为默认 QoS（`PARTICIPANT_QOS_DEFAULT`、`SUBSCRIBER_QOS_DEFAULT`、`TOPIC_QOS_DEFAULT`、`DATAREADER_QOS_DEFAULT`）。每个 DDS 实体的默认 QoS 值可在 DDS 标准中查阅。

```c++
//!Initialize the subscriber
bool init()
{
    DomainParticipantQos participantQos;
    participantQos.name("Participant_subscriber");
    participant_ = DomainParticipantFactory::get_instance()->create_participant(0, participantQos);

    if (participant_ == nullptr)
    {
        return false;
    }

    // Register the Type
    type_.register_type(participant_);

    // Create the subscriptions Topic
    topic_ = participant_->create_topic("HelloWorldTopic", "HelloWorld", TOPIC_QOS_DEFAULT);

    if (topic_ == nullptr)
    {
        return false;
    }

    // Create the Subscriber
    subscriber_ = participant_->create_subscriber(SUBSCRIBER_QOS_DEFAULT, nullptr);

    if (subscriber_ == nullptr)
    {
        return false;
    }

    // Create the DataReader
    reader_ = subscriber_->create_datareader(topic_, DATAREADER_QOS_DEFAULT, &listener_);

    if (reader_ == nullptr)
    {
        return false;
    }

    return true;
```

`run()` 公共成员函数确保订阅者在接收到所有样本之前持续运行。该成员函数通过主动等待的方式实现订阅者的运行，并设置了 100 毫秒的休眠间隔以降低 CPU 占用率。

```c++
//!Run the Subscriber
void run(
        uint32_t samples)
{
    while (listener_.samples_ < samples)
    {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
```

最后，在 `main` 函数中初始化并运行实现订阅者的参与者。

```c++
};

int main(
        int argc,
        char** argv)
{
    std::cout << "Starting subscriber." << std::endl;
    uint32_t samples = 10;

    HelloWorldSubscriber* mysub = new HelloWorldSubscriber();
    if (mysub->init())
    {
        mysub->run(samples);
    }

    delete mysub;
```

#### 1.3.8.2. CMakeLists.txt

请在你之前创建的 CMakeLists.txt 文件末尾包含以下代码片段。该片段将添加构建可执行文件所需的所有源文件，并将可执行文件和库链接在一起。

```cmake
add_executable(DDSHelloWorldSubscriber src/HelloWorldSubscriber.cpp ${DDS_HELLOWORLD_SOURCES_CXX})
target_link_libraries(DDSHelloWorldSubscriber fastdds fastcdr)
```

此时，项目已准备好构建、编译并运行订阅者应用程序。请在工作空间的 build 目录中运行以下命令。

```shell
cmake ..
cmake --build .
./DDSHelloWorldSubscriber
```

### 1.3.9. Putting all together

最后，在两个终端中，从 build 目录运行发布者和订阅者应用程序。

```shell
./DDSHelloWorldPublisher
./DDSHelloWorldSubscriber
```

### 1.3.10. Summary

在本教程中，你构建了一个发布者和一个订阅者的 DDS 应用程序。你还学习了如何构建用于源代码编译的 CMake 文件，以及如何在项目中包含并使用 Fast DDS 和 Fast CDR 库。

### 1.3.11. Next steps

在 *eProsima Fast DDS* 的 GitHub 仓库中，你可以找到更复杂的示例，这些示例针对多种使用场景实现了 DDS 通信。你可以在此处找到它们。
