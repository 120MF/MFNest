+++
title = '借助QWindowKit实现Qt多平台无边框窗口'
date = 2024-07-20T13:03:23+08:00
draft = false
description = ""
slug = "借助QWindowKit实现Qt多平台无边框窗口"
image = "/media/QQ20240811-150611.png"
categories = ["编程相关"]
tags = ["Qt","QWindowKit","无边框","CMake"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["Qt","QWindowKit","无边框","CMake","git submodule"]
readingTime = true
+++

目前市面上的Qt无边框窗口方案多结合Windows DirectX API实现，抹除了Qt的多平台优势。

[QWindowKit](https://github.com/stdware/qwindowkit)是一个开源跨平台的Qt无边框窗口组件库，支持Qt5.12以上版本。

借助QWindowKit项目，我们可以轻松在自己的Qt项目中实现无边框窗口效果。

本文将介绍使用Qt Quick并结合CMake调用QWindowKit库。QWidget应用类似，可参考官方帮助文档。

## 初始化工程

使用Qt Creator新建一个Qt Quick应用工程。若已有项目，可以跳过该步。

<img src="/media/QQ20240811-141325.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<img src="/media/QQ20240811-141604.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>默认初始工程</em></p>

- 若仅需快速使用，可以参考官方教程，单独下载QWindowKit库并对其进行编译安装后直接在CMake和源文件中应用；

- 若要将QWindowKit作为第三方库嵌入到工程当中，则要将其作为git submodule加入到工程当中。

```shell
mkdir 3rdparty
git submodule add https://github.com/stdware/qwindowkit.git 3rdparty/qwindowkit
git submodule update --init --recursive
```

添加完毕后，项目结构如下：

```text
├─3rdparty
│  └─qwindowkit
│      ......
├─build
│      ......  
│CMakeLists.txt
│main.cpp
│Main.qml
```

## 使用CMake将QWindowKit作为子目录加入到工程中

- 编辑项目目录下的CMakeLists.txt，加入QWindowKit子目录并设置相关参数，再将我们自己的主程序和QWindowKit的库链接起来；

```cmake
cmake_minimum_required(VERSION 3.16)

project(framelessQML VERSION 0.1 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD_REQUIRED ON)

#[1]
set(QWINDOWKIT_BUILD_STATIC ON)
set(QWINDOWKIT_BUILD_EXAMPLES OFF)
set(QWINDOWKIT_BUILD_QUICK ON)
set(QWINDOWKIT_BUILD_WIDGETS OFF)
set(QWINDOWKIT_ENABLE_STYLE_AGENT ON)
set(QWINDOWKIT_INSTALL OFF)

add_subdirectory(3rdparty/qwindowkit)
#[1]

find_package(Qt6 6.5 REQUIRED COMPONENTS Quick)

qt_standard_project_setup(REQUIRES 6.5)

qt_add_executable(appframelessQML
    main.cpp
)

qt_add_qml_module(appframelessQML
    URI framelessQML
    VERSION 1.0
    QML_FILES
        Main.qml
)

# Qt for iOS sets MACOSX_BUNDLE_GUI_IDENTIFIER automatically since Qt 6.1.
# If you are developing for iOS or macOS you should consider setting an
# explicit, fixed bundle identifier manually though.
set_target_properties(appframelessQML PROPERTIES
#    MACOSX_BUNDLE_GUI_IDENTIFIER com.example.appframelessQML
    MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION}
    MACOSX_BUNDLE_SHORT_VERSION_STRING ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}
    MACOSX_BUNDLE TRUE
    WIN32_EXECUTABLE TRUE
)

target_link_libraries(appframelessQML
    PRIVATE Qt6::Quick
)

#[2]
target_link_libraries(appframelessQML
    PUBLIC QWindowKit::Quick
)
include_directories(3rdparty/qwindowkit/include)
#[2]

include(GNUInstallDirs)
install(TARGETS appframelessQML
    BUNDLE DESTINATION .
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
```

## 实现无边框窗口

参考官网的Example，修改main.cpp和Main.qml文件；

```cpp
//main.cpp
#include <QtQuick/QQuickWindow>
#include <QQmlApplicationEngine>
//[1]
#include <QWKQuick/qwkquickglobal.h>
//![1]

int main(int argc, char *argv[])
{
//[2]
    qputenv("QT_WIN_DEBUG_CONSOLE", "attach");
    qputenv("QSG_INFO", "1");
#if QT_VERSION >= QT_VERSION_CHECK(6, 0, 0)
    qputenv("QT_QUICK_CONTROLS_STYLE", "Basic");
#else
    qputenv("QT_QUICK_CONTROLS_STYLE", "Default");
#endif
    qputenv("QSG_RHI_BACKEND", "opengl");
    //qputenv("QSG_RHI_HDR", "scrgb");
    //qputenv("QT_QPA_DISABLE_REDIRECTION_SURFACE", "1");
    QGuiApplication::setHighDpiScaleFactorRoundingPolicy(
        Qt::HighDpiScaleFactorRoundingPolicy::PassThrough);
//![2]
    QGuiApplication app(argc, argv);
    QQuickWindow::setDefaultAlphaBuffer(true);
    QQmlApplicationEngine engine;
//[3]
    QWK::registerTypes(&engine);
//![3]
    QObject::connect(
        &engine,
        &QQmlApplicationEngine::objectCreationFailed,
        &app,
        []() { QCoreApplication::exit(-1); },
        Qt::QueuedConnection);
    engine.loadFromModule("framelessQML", "Main");

    return app.exec();
}
```

```qml
#Main.qml
import QtQuick
import QWindowKit 1.0

Window {
    id: window
    width: 800
    height: 600
    color: darkStyle.windowBackgroundColor
    title: qsTr("Hello, world!")
    Component.onCompleted: {
        windowAgent.setup(window)
        window.visible = true
    }

    QtObject {
        id: lightStyle
    }

    QtObject {
        id: darkStyle
        readonly property color windowBackgroundColor: "#1E1E1E"
    }

    WindowAgent {
        id: windowAgent
    }

    Rectangle {
        id: titleBar
        anchors {
            top: parent.top
            left: parent.left
            right: parent.right
        }
        height: 32
        color: window.active ? "#3C3C3C" : "#505050"
        Component.onCompleted: windowAgent.setTitleBar(titleBar)

        Text {
            anchors {
                verticalCenter: parent.verticalCenter
                left: iconButton.right
                leftMargin: 10
            }
            horizontalAlignment: Text.AlignHCenter
            verticalAlignment: Text.AlignVCenter
            text: window.title
            font.pixelSize: 14
            color: "#ECECEC"
        }
    }
}
```

## 构建/部署/运行

参考使用以下CMake参数进行CMake配置：

```shell
-DCMAKE_PREFIX_PATH=C:\Qt\6.7.2\mingw_64
-DCMAKE_GENERATOR:STRING=Ninja
-DCMAKE_CXX_COMPILER:FILEPATH=C:/Qt/Tools/mingw1120_64/bin/g++.exe
-DCMAKE_CXX_FLAGS_INIT:STRING=-DQT_QML_DEBUG
```

使用以下参数进行CMake构建：

```shell
--parallel --target all
```

构建完成后，除Windows以外的平台可以直接运行可执行文件。Windows平台下需要使用windeploy进行部署。

```powershell
C:\Qt\6.7.2\mingw_64\bin\windeployqt.exe --qmldir C:\Qt\6.7.2\mingw_64\qml .\appframelessQML.exe 
```

## 效果

<img src="/media/QQ20240811-150611.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">