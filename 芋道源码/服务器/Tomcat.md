# 0 Tomcat核心内容

1. 容器启动，从server到listener、jndi、service到connector、engine到host到context到wrapper一系列对象的组装，包括Digester的使用、JMX注册、JDK5中新的线程池启动方式等
2. 一个socket连接如何转化成request
3. 一条请求响应链在容器中流转的经过
4. 容器的自定义ClassLoader机制
5. session如何实现，特别是集群环境中的session粘滞和session复制的实现
6. nio处理方式的实现
7. servlet3新规范中websocket的实现
8. 容器关闭机制

# 1 启动分析

## 1.1 启动脚本

start.bat

```bash
@echo off
rem Licensed to the Apache Software Foundation (ASF) under one or more
rem contributor license agreements.  See the NOTICE file distributed with
rem this work for additional information regarding copyright ownership.
rem The ASF licenses this file to You under the Apache License, Version 2.0
rem (the "License"); you may not use this file except in compliance with
rem the License.  You may obtain a copy of the License at
rem
rem     http://www.apache.org/licenses/LICENSE-2.0
rem
rem Unless required by applicable law or agreed to in writing, software
rem distributed under the License is distributed on an "AS IS" BASIS,
rem WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
rem See the License for the specific language governing permissions and
rem limitations under the License.

if "%OS%" == "Windows_NT" setlocal
rem ---------------------------------------------------------------------------
rem Start script for the CATALINA Server
rem
rem $Id: startup.bat 895392 2010-01-03 14:02:31Z kkolinko $
rem ---------------------------------------------------------------------------

rem Guess CATALINA_HOME if not defined
set "CURRENT_DIR=%cd%"
if not "%CATALINA_HOME%" == "" goto gotHome
set "CATALINA_HOME=%CURRENT_DIR%"
if exist "%CATALINA_HOME%\bin\catalina.bat" goto okHome
cd ..
set "CATALINA_HOME=%cd%"
cd "%CURRENT_DIR%"
:gotHome
if exist "%CATALINA_HOME%\bin\catalina.bat" goto okHome
echo The CATALINA_HOME environment variable is not defined correctly
echo This environment variable is needed to run this program
goto end
:okHome

set "EXECUTABLE=%CATALINA_HOME%\bin\catalina.bat"

rem Check that target executable exists
if exist "%EXECUTABLE%" goto okExec
echo Cannot find "%EXECUTABLE%"
echo This file is needed to run this program
goto end
:okExec

rem Get remaining unshifted command line arguments and save them in the
set CMD_LINE_ARGS=
:setArgs
if ""%1""=="""" goto doneSetArgs
set CMD_LINE_ARGS=%CMD_LINE_ARGS% %1
shift
goto setArgs
:doneSetArgs

call "%EXECUTABLE%" start %CMD_LINE_ARGS%

:end

```



## 1.2 Bootstrap类中的main方法

## 1.3 Digester的使用

Bootstrap的main方法最后会调用org.apahe.catalina.startup.Catalina对象的load和start两个方法

这两个方法做的事情就两个，一个是创建Digester对象，将当前对象压入Digester里面的对象栈顶，根据inputSource里面设置的文件xml路径及所创建的Digester对象所包含的解析规则生成相应对象，并调用相应方法将对象之间关联起来。二是调用Server接口对象的init方法。

Java解析xml文件有两种方式，一种是Dom4j之类将文件全部读取到内存中，在内存里构造一棵Dom树的方法来解析。一种是SAX的读取文件流，在流中碰到相应的xml节点触发相应的节点事件回调响应的方法，基于事件解析方式，有点是不需要先将文件全部读取到内存中



## 1.4 各组件init、start方法调用

## 1.5 Lifecycle机制和实现原理

# 2 请求分析

## 2.1 处理线程的产生

## 2.2 Socket转换成内部请求对象

## 2.3 请求与容器中具体组件的匹配

## 2.4 Tomcat7阈机制原理

# 3 Web应用加载原理

## 3.1 Context构建

![img](1616a782ea61d7b6)

Tomcat启动的时候将会解析server.xml，根据里面所配置的各个节点信息逐一初始化和启动相应组件（即分别调用它们的init和start方法），server.xml的内容对于上图中已经默认配置的各级组件包括server、service、Engine、Connector、Host、Valve。上图中的Context组件实际就是我们通常所说的一个web应用，有趣的是在server.xml中并没有配置该组件，而我们默认启动的时候实际上已经有好几个web应用可以访问了

## 3.2 web.xml解析

## 3.3 Listener、Filter、Servlet的加载和调用

# 4 自动加载类及检测文件变动原理