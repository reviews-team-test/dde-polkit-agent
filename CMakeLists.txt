cmake_minimum_required(VERSION 3.13)

set(BIN_NAME dde-polkit-agent)

project(${BIN_NAME} CXX)

#set(CMAKE_VERBOSE_MAKEFILE ON)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_CXX_FLAGS "-g -Wall")

# 增加安全编译参数
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fstack-protector-all")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fstack-protector-all")
set(CMAKE_EXE_LINKER_FLAGS  "-z relro -z now -z noexecstack -pie")

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -fsanitize=address -O2")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fsanitize=address -O2")
endif()

if (DEFINED ENABLE_MIEEE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mieee")
endif()

# Install settings
if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX /usr)
endif ()

if (NOT (${CMAKE_BUILD_TYPE} MATCHES "Debug"))
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Ofast")

    # generate qm
    execute_process(COMMAND bash "translate_generation.sh"
                    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
endif ()

# Find the library
find_package(PkgConfig REQUIRED)
find_package(DtkWidget REQUIRED)
find_package(Qt5 COMPONENTS Core Widgets DBus Concurrent REQUIRED)

pkg_check_modules(Polkit-qt5 REQUIRED polkit-qt5-1)
pkg_check_modules(DFrameworkDBus REQUIRED dframeworkdbus)


function(generation_dbus_adaptor xml class_name class_file option)
    execute_process(COMMAND qdbusxml2cpp ${option} -a ${class_file} -c ${class_name} ${xml}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
endfunction(generation_dbus_adaptor)

generation_dbus_adaptor(
    ${CMAKE_SOURCE_DIR}/com.deepin.Polkit1AuthAgent.xml
    Polkit1AuthAgentAdaptor
    ${CMAKE_CURRENT_BINARY_DIR}/polkit1authagent_adaptor
    -N
)

# test deepin-polkit
try_compile(DEEPIN_POLKIT_TEST
              ${CMAKE_CURRENT_BINARY_DIR}/try_compile
            SOURCES
              ${CMAKE_CURRENT_SOURCE_DIR}/tests/polkit.cc
            LINK_LIBRARIES
              ${Polkit-qt5_LIBRARIES}
              Qt5::Core
)

if (DEEPIN_POLKIT_TEST)
    add_definitions(-DUSE_DEEPIN_POLKIT)
endif()

set(AuthAgent_DBUS_SCRS
    ${CMAKE_CURRENT_BINARY_DIR}/polkit1authagent_adaptor.h
    ${CMAKE_CURRENT_BINARY_DIR}/polkit1authagent_adaptor.cpp
)

add_executable(${BIN_NAME}
    policykitlistener.cpp
    AuthDialog.cpp
    main.cpp
    usersmanager.cpp
    pluginmanager.cpp
    images.qrc
    ${AuthAgent_DBUS_SCRS}
)
target_include_directories(${BIN_NAME} PUBLIC
    ${DtkWidget_INCLUDE_DIRS}
    ${Polkit-qt5_INCLUDE_DIRS}
    ${DFrameworkDBus_INCLUDE_DIRS}
    Qt5::Core
    Qt5::Widgets
    Qt5::DBus
    Qt5::Concurrent
)

target_link_libraries(${BIN_NAME} PRIVATE
    ${DtkWidget_LIBRARIES}
    ${Polkit-qt5_LIBRARIES}
    ${DFrameworkDBus_LIBRARIES}
    Qt5::Core
    Qt5::Widgets
    Qt5::DBus
    Qt5::Concurrent
)

## qm files
file(GLOB QM_FILES "translations/*.qm")
install(FILES ${QM_FILES} DESTINATION share/${BIN_NAME}/translations)

## header files
file(GLOB HEADER_FILES "agent-extension-proxy.h" "agent-extension.h")
install(FILES ${HEADER_FILES} DESTINATION include/dpa)

# bin
install(TARGETS ${BIN_NAME} DESTINATION lib/polkit-1-dde)