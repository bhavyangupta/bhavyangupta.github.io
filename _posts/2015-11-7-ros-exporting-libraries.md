---
layout: post
title: Making custom C++ libraries in catkin workspace 
---


In some cases you want to have a custom standalone library that provides a set 
of nodes/ functionality that any package in your workspace can use. Recently, I 
wanted the same functionality for the PANDUbot project. In this case, I wanted 
a lot of packages to be able to use some actionlib clients and boilerplate nodes
I wrote and so I looked into doing this.

## Creating the library

I first created a new package called pandubot_libraries and moved my source and
header files inside it. Here's how my CMakeLists.txt looks for that package, 
followed by the directory contents:

{% highlight bash %}
cmake_minimum_required(VERSION 2.8.3)
project(pandubot_libraries)

find_package(catkin REQUIRED)
find_package(catkin REQUIRED COMPONENTS roscpp
                                        actionlib
                                        pandubot_object_recognition
                                        pandubot_face_detection)

include_directories(${catkin_INCLUDE_DIRS}
                    include/${PROJECT_NAME}/)

catkin_package(INCLUDE_DIRS   include
               CATKIN_DEPENDS pandubot_object_recognition 
                              move_base
                              pandubot_face_detection
               LIBRARIES      pandubotActionClients)

add_library(pandubotActionClients src/object_action_client.cpp
                                  src/face_action_client.cpp
                                  src/navigation_action_client.cpp)

add_dependencies(pandubotActionClients pandubot_object_recognition_generate_messages_cpp
                                       pandubot_face_detection_generate_message_cpp 
                                       ${catkin_EXPORTED_TARGETS})

target_link_libraries(pandubotActionClients ${catkin_LIBRARIES})


## Install Targets
install(TARGETS pandubotActionClients
        ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
        LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
       )
       
install(DIRECTORY include/${PROJECT_NAME}/
        DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION})
{% endhighlight %}

{% highlight bash %}
├── CMakeLists.txt
├── include
│   └── pandubot_libraries
│       ├── face_action_client.hpp
│       ├── navigation_action_client.hpp
│       └── object_action_client.hpp
├── package.xml
└── src
    ├── face_action_client.cpp
    ├── navigation_action_client.cpp
    └── object_action_client.cpp
{% endhighlight %}

- First, you need to move the headers in a special directory : 
  **package_name/include/${project_name}**. This is the reccommended location for 
  header files in ROS. 
- Secondly, you need to *export* the header files and the library archive in the
  catkin_package directive. 

Other important directives in the file are as follows:
- **add_library** : this creates a new target library
- **add_dependencies** : just like building executables, this adds dependencies 
of the library. This was needed since I used custom messages in the library - so
this tells cmake to generate the messages for pandubot_object_recognition and 
pandubot_face_detection before using it in this library.
- **install** : This is used to install the library when built using catkin_make install

## Using the library

The library can be used like any other package- just add the name of the package
to your find_package directive and the corresponding catkin_package dependency 
on the same library package.

One change needs to be made to the *source* files though, the files must use 
`#include <library_package_name/header_name.h>` instead of `#include<header_name.h>`.
Note that this too, is just like headers exported by any other ROS package. 



## Common mistake:
- A mistake I made while doing this was that I wasn't placing the header file 
in the right directory i.e. my headers were in include/ instead of include/${PROJECT_NAME}


# References
1. https://cmake.org/cmake-tutorial/
2. http://answers.ros.org/question/65716/which-is-the-correct-way-to-install-header-files-in-catkin-packages/
3. http://docs.ros.org/api/catkin/html/howto/format2/building_libraries.html
4. http://answers.ros.org/question/195467/catkin-unable-to-include-custom-libraries/
5. http://wiki.ros.org/catkin/CMakeLists.txt
6. http://answers.ros.org/question/201977/include-header-file-from-another-package-indigo/