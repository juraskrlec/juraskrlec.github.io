---
layout: post
title:  "CMake + Swift"
date:   2023-02-20
categories: ios swift cmake
---
It's been a long time since I've posted something here. I thought that I could do a post every two weeks, but I needed a little break. And a little break turned into a bigger break. Anyway, I'll do a post every month. 

I had an interesting problem last year that I wanted to tackle; how to add Swift files to a framework written in Objective-C through CMake and make it work. 

The basic concept is: 
	
	1. Add a Swift file to your sources. This is important, if you don't Swift Compiler will be turned off. You can create an empty Swift file (dummy).
	2. We need to turn on and force Swift Compiler (swiftc) for CMake to recognize its 	builds settings.
	3. Add CMake SET for swiftc build settings
	4. Remove other Swift flags
	5. Use Swift :)

Let's tackle one by one. 

* Add a Swift file to your sources; add this where you list your source files for cmake:

```
list( APPEND <your_source_list>
...
<path>/Empty.swift
...
)
```

* Force load `swiftc` in your `<project>.cmake`:

```
execute_process(
    COMMAND xcrun --find swiftc
    OUTPUT_VARIABLE SWIFTC_EXECUTABLE
    OUTPUT_STRIP_TRAILING_WHITESPACE)
set( SWIFTC_FOUND TRUE )
message( STATUS "Found swiftc: ${SWIFTC_EXECUTABLE}" )

if ( SWIFTC_FOUND )
    message(STATUS "Setting swiftc: ${SWIFTC_EXECUTABLE}")
    # We need to tell cmake where swiftc is
    set( CMAKE_Swift_COMPILER ${SWIFTC_EXECUTABLE} )
    # This line is important, we need to force load it
    set( CMAKE_Swift_COMPILER_FORCED ON ) 
endif()

if( CMAKE_Swift_COMPILER )
  # Enable Swift, tell cmake there's now new swiftc build settings, without this, cmake can't properly configure attributes
  enable_language(Swift)
else()
  message(FATAL_ERROR "FATAL ERROR! Did not find swiftc?!")
endif()
```

* Add `swiftc` build settings to your `<project>.cmake`:

```
# Swift Features
message( STATUS "Setting Swift Settings:" )
# Swift requires modules
set( CMAKE_XCODE_ATTRIBUTE_DEFINES_MODULE "YES" )
# Enable modules for ObjectiveC - Swift
set( CMAKE_XCODE_ATTRIBUTE_CLANG_ENABLE_MODULES "YES" )
# Set Swift to ObjC Interface name
set( CMAKE_XCODE_ATTRIBUTE_SWIFT_INSTALL_OBJC_HEADER "YES")
set( CMAKE_XCODE_ATTRIBUTE_SWIFT_OBJC_INTERFACE_HEADER_NAME "${FRAMEWORK_OUTPUT_NAME}-Swift.h" )
# Set name for the source code module constructed for this target, and which will be used to import the module in implementation source files
set( CMAKE_XCODE_ATTRIBUTE_PRODUCT_MODULE_NAME "${FRAMEWORK_OUTPUT_NAME}" )
# Set paths to be searched by the Swift compiler for additional Swift modules. This is used to use ObjectiveC in Swift. Set all ObjC private headers to use for internal use in module.modulemap
set( CMAKE_XCODE_ATTRIBUTE_SWIFT_INCLUDE_PATHS "<path>/ProjectModule" )
# Set Swift Version
set( CMAKE_XCODE_ATTRIBUTE_SWIFT_VERSION "5.0" )
set( CMAKE_XCODE_ATTRIBUTE_SWIFT_OPTIMIZATION_LEVEL "$<$<CONFIG:Release>:-O>$<$<CONFIG:Debug>:-Onone>" )
# Set for every target
set ( CMAKE_XCODE_ATTRIBUTE_OTHER_SWIFT_FLAGS "")
```

* Remove other Swift flags:

```
set_target_properties( ${PROJECT_NAME} PROPERTIES
FRAMEWORK TRUE
...
XCODE_ATTRIBUTE_OTHER_SWIFT_FLAGS ""
XCODE_ATTRIBUTE_BUILD_LIBRARY_FOR_DISTRIBUTION "YES"
...
)
```

* Use Swift. But be aware, you'll need to wrap some things when using Swift in the Objective-C file if you want them private :) 

Feel free to message me on Twitter if you want to talk about this, or anything tech related :)  
