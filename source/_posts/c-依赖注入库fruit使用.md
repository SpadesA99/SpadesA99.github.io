---
title: c++依赖注入库fruit使用
date: 2022-06-02 22:50:04
tags:
---

# c++ 依赖注入库 fruit 使用

#### git 项目地址

https://github.com/google/fruit.git

#### 使用 vcpkg 安装库

```bash
vcpkg install fruit
```

#### cmake 添加引用

```cmake
link_directories("/vcpkg/installed/x64-linux/lib")
target_link_libraries(main PRIVATE fruit)
```

#### demo

```c++

#include <fruit/fruit.h>
#include <iostream>


class Writer {
public:
	virtual void write(std::string s) = 0;
};

class WriterImpl : public Writer {
public:

	INJECT(WriterImpl()) = default;

	virtual void write(std::string s) override {
		std::cout << s;
	}
};

fruit::Component<Writer> getWriterComponent() {
	return fruit::createComponent().bind<Writer, WriterImpl>();
}


class Greeter {
public:
	virtual void greet() = 0;
};

class GreeterImpl : public Greeter {
private:
	Writer* writer;

public:
	INJECT(GreeterImpl(Writer* writer))
		: writer(writer) {
	}

	virtual void greet() override {
		writer->write("Hello world!\n");
	}
};

fruit::Component<Greeter> getGreeterComponent() {
	return fruit::createComponent().install(getWriterComponent).bind<Greeter, GreeterImpl>();
}

int main() {
	fruit::Injector<Greeter> injector(getGreeterComponent);
	Greeter* greeter = injector.get<Greeter*>();
	greeter->greet();
	return 0;
}
```
