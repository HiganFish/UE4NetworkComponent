本项目保存一些前端UE4 后端LinuxC++的 网络类组件 无第三方库

# TcpComponent
## 说明
简单的包装了UE4原生的Socket函数

- C++代码
生成对象后通过传入Ip地址和端口即可创建连接
使用Send和Recv函数可以收发uint8数据流
- 蓝图
也可以通过蓝图添加组建后使用 函数创建连接
SendString和RecvAsString收发FString数据

详见注释

## 使用方法

通过VS在项目的Build.cs中添加`Networking`和`Sockets` 加上四个自动生成的代码如下

`PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "Networking", "Sockets" });`

通过VS先进行一次编译, 再继续进行下面的操作

新建名为NetworkComponent继承自ActorComponent的组件c++类

将头文件和cpp文件代码全选替换掉UE4自动生成的代码, 进行编译即可  详细使用见 UE4登录项目

**详细使用见 [UE4登录项目](https://github.com/HiganFish/UE4NetworkComponentSample)**


这里是一个登录函数 简单的调用即可发送数据流
```c++
// LoginNetworkComponent 为组件NetworkComponent的指针 在BeginPlay调用CreateDefaultSubobject生成

int ULoginManager::Login(FString Username, FString Password)
{
    FString LoginIp = "192.168.80.160";
    int32 LoginPort = 8700;

    if (!LoginNetworkComponent->ConnectToServer(LoginIp, LoginPort, "LoginConnection"))
    {
        LOG_WARN("%s", "Can't connect to server");
        return -1;
    }

    LoginNetworkPackage LoginData(TCHAR_TO_ANSI(*Username), TCHAR_TO_ANSI(*Password));
    uint8 Buffer[512];
    SSIZE_T Len = LoginData.SerializeToBuffer(Buffer, sizeof Buffer);
    if (Len <= 0)
    {
        LOG_WARN("%s", "SerializeToBuffer Error");
        return -1;
    }

    SSIZE_T Result = LoginNetworkComponent->Send(Buffer, Len);
    LoginNetworkComponent->CloseConnection();

    return Result;
}
```

# NetworkPackageComponent
通过继承父类NetworkPackageBase  添加自己的body数据变量 生成自己的子类

并且代码建议在后端项目中编写 编写结束后可以简单的将全部代码复制进UE4代码中, 不用为UE4编写额外的代码 保证后端C++服务器和UE4前端
有同样的代码, 通过的是NetworkPackageBase.h中前面的宏实现  

**如果自己有扩充 注意在这里添加对应的定义 应对同一个类型不同的名称和头文件**

```c++
#define UE4 // UE4代码中需要添加此定义

#ifdef UE4
#pragma once
#include "CoreMinimal.h"
#include "string"

#define uint32_t uint32
#define uint8_t uint8
#define ssize_t SSIZE_T
#undef UE4
#else
#include <cstdint>
#include <sys/types.h>
#include <cstring>
#include <string>
#endif
```


- 序列化
重写父类的`virtual ssize_t SerializeToBuffer(uint8_t* buffer, size_t buffer_len);` 实现序列化

如果使用默认的数据包头 则只需要重写函数时调用`SerializeHeaderToBuffer`即可自动序列化默认的头部

然后继续编写 序列化自己body变量的代码

最后调用`SerializeCheckSumToBuffer`自动生成CheckSum 得到了数据流

- 反序列化
重写父类的`virtual ssize_t DeSerializeFromBuffer(const uint8_t* buffer, size_t buffer_len);`实现反序列化

调用`DeSerializeHeaderFromBuffer` 自动反序列化包头数据 并进行CheckSum校验

然后编写自己body部分的反序列化代码

**详细使用见 [UE4登录项目](https://github.com/HiganFish/UE4NetworkComponentSample)**


这里编写了一个简单的 登录数据包  自己仅需要编写下面的代码 就可以得到数据流
```c++
class LoginNetworkPackage : public NetworkPackageBase
{
public:

	LoginNetworkPackage() = default;
	LoginNetworkPackage(std::string username, std::string password);

	ssize_t SerializeToBuffer(uint8_t* buffer, size_t BufferLen) override;

	ssize_t DeSerializeFromBuffer(const uint8_t* buffer, size_t buffer_len) override;

	const std::string& GetAuthenticationKey() const;

private:
	static uint32_t login_number;

	std::string authentication_key_;
};


uint32_t LoginNetworkPackage::login_number = 0;

LoginNetworkPackage::LoginNetworkPackage(std::string username, std::string password) :
	NetworkPackageBase(LOGIN, login_number++),
	authentication_key_(username + "#" + password)
{}

ssize_t LoginNetworkPackage::SerializeToBuffer(uint8_t* buffer, size_t BufferLen)
{
	ssize_t result = 0;

    // 调用父类函数 生成默认包头
	ssize_t header_len = SerializeHeaderToBuffer(authentication_key_.length(), buffer, BufferLen);
	if (header_len <= 0)
	{
		return -1;
	}
	result += header_len;

    // 序列化自己body数据 authentication_key_
	const uint8_t* authentication_key_data = (const uint8_t*)authentication_key_.c_str();

	memcpy(buffer + result, authentication_key_data, authentication_key_.length());
	result += authentication_key_.length();

    // 序列完自己body变量authentication_key后调用父类函数生成checksum
	SerializeCheckSumToBuffer(buffer, result);
	return result;
}

ssize_t LoginNetworkPackage::DeSerializeFromBuffer(const uint8_t* buffer, size_t buffer_len)
{
	size_t body_len = 0;
    // 调用父类函数 反序列化包头 并校验
	ssize_t parsed_len = DeSerializeHeaderFromBuffer(buffer, buffer_len, &body_len);

	if (parsed_len <= 0)
	{
		return parsed_len;
	}

    // 反序列化自己body数据authentication_key_
	authentication_key_.append((char*)buffer + parsed_len, body_len);

	return parsed_len + body_len;
}

const std::string& LoginNetworkPackage::GetAuthenticationKey() const
{
	return authentication_key_;
}
```

