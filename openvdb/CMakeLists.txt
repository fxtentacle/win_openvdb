# yue.nicholas@gmail.com
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

PROJECT ( OpenVDB )

FIND_PACKAGE ( GLFW3 REQUIRED )
IF (WIN32)
  FIND_PACKAGE ( GLEW REQUIRED )
ENDIF()
FIND_PACKAGE ( Boost REQUIRED COMPONENTS iostreams system thread )
FIND_PACKAGE ( ILMBase REQUIRED )
FIND_PACKAGE ( OpenEXR REQUIRED )
FIND_PACKAGE ( TBB REQUIRED )
FIND_PACKAGE ( ZLIB REQUIRED )
FIND_PACKAGE ( OpenGL REQUIRED )
FIND_PACKAGE ( Blosc REQUIRED )
FIND_PACKAGE ( Doxygen REQUIRED )
IF (APPLE)
  FIND_LIBRARY ( COCOA_LIBRARY Cocoa )
  FIND_LIBRARY ( IOKIT_LIBRARY IOKit )
  FIND_LIBRARY ( COREVIDEO_LIBRARY CoreVideo )
ENDIF ()

IF ( UNIX AND NOT APPLE )
  MESSAGE ( "UNIX" )
  SET ( XI_LIBRARY Xi )
  SET ( XRANDR_LIBRARY Xrandr )
ENDIF ()

IF ( USE_GLFW3 )
  SET ( OPENVDB_USE_GLFW_FLAG "-DOPENVDB_USE_GLFW_3" )
ENDIF ()

# MESSAGE ( "ZLIB_INCLUDE_DIR = ${ZLIB_INCLUDE_DIR}")

INCLUDE_DIRECTORIES ( ${Boost_INCLUDE_DIR} )
INCLUDE_DIRECTORIES ( ${ILMBASE_INCLUDE_DIRS} )
INCLUDE_DIRECTORIES ( ${TBB_INCLUDE_DIR} )
INCLUDE_DIRECTORIES ( ${OPENGL_INCLUDE_DIR} )
INCLUDE_DIRECTORIES ( ${ZLIB_INCLUDE_DIR} )
INCLUDE_DIRECTORIES ( ${GLFW3_INCLUDE_DIR} )
IF (WIN32)
  INCLUDE_DIRECTORIES ( ${GLEW_INCLUDE_DIR} )
ENDIF ()
INCLUDE_DIRECTORIES ( .. ) # So they can simulate referencing installed headers
INCLUDE_DIRECTORIES ( . ) # So they can simulate referencing installed headers

IF (WIN32)
  # Because of implicit linking!
  LINK_DIRECTORIES ( ${Boost_LIBRARY_DIR} )
ENDIF ()

INCLUDE_DIRECTORIES ( ${BLOSC_INCLUDE_DIR} )

IF (WIN32)
  ADD_DEFINITIONS ( -D_WIN32 -DNOMINMAX -DHALF_EXPORTS -DOPENVDB_DLL )
ELSE ()
  ADD_DEFINITIONS ( -pthread -fPIC )
ENDIF ()

# Working around hardcoding of path at DWA
FILE ( GLOB VIEWER_SOURCE_FILES RELATIVE ${CMAKE_SOURCE_DIR} viewer/* )
FILE ( MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/dwa/openvdb_viewer )
FILE ( COPY ${VIEWER_SOURCE_FILES} DESTINATION ${CMAKE_BINARY_DIR}/dwa/openvdb_viewer )
INCLUDE_DIRECTORIES ( ${CMAKE_BINARY_DIR}/dwa )

SET ( OPENVDB_LIBRARY_SOURCE_FILES
  Grid.cc
  io/Archive.cc
  io/Compression.cc
  io/File.cc
  io/GridDescriptor.cc
  io/Queue.cc
  io/Stream.cc
  io/TempFile.cc
  math/Hermite.cc
  math/Maps.cc
  math/Proximity.cc
  math/QuantizedUnitVec.cc
  math/Transform.cc
  metadata/Metadata.cc
  metadata/MetaMap.cc
  openvdb.cc
  Platform.cc
  util/Formats.cc
  util/Util.cc
  )

SET_SOURCE_FILES_PROPERTIES ( ${OPENVDB_LIBRARY_SOURCE_FILES}
  PROPERTIES
  COMPILE_FLAGS "-DOPENVDB_PRIVATE -DOPENVDB_USE_BLOSC ${OPENVDB_USE_GLFW_FLAG}"
)

ADD_LIBRARY ( openvdb STATIC
  ${OPENVDB_LIBRARY_SOURCE_FILES}
  )

TARGET_LINK_LIBRARIES ( openvdb
  ${Boost_LIBRARIES}
  ${Tbb_TBB_LIBRARY}
  ${Ilmbase_HALF_LIBRARY}
  ${ZLIB_LIBRARY}
  )

SET ( VDB_PRINT_SOURCE_FILES  cmd/openvdb_print/main.cc )
SET_SOURCE_FILES_PROPERTIES ( ${VDB_PRINT_SOURCE_FILES}
  PROPERTIES
  COMPILE_FLAGS "-DOPENVDB_USE_BLOSC ${OPENVDB_USE_GLFW_FLAG}"
)

ADD_EXECUTABLE ( vdb_print
  ${VDB_PRINT_SOURCE_FILES}
  )

TARGET_LINK_LIBRARIES ( vdb_print
  openvdb
  ${BLOSC_blosc_LIBRARY} 
  )

SET ( VDB_RENDER_SOURCE_FILES cmd/openvdb_render/main.cc )
SET_SOURCE_FILES_PROPERTIES ( ${VDB_RENDER_SOURCE_FILES}
  PROPERTIES
  COMPILE_FLAGS "-DOPENVDB_USE_BLOSC ${OPENVDB_USE_GLFW_FLAG}"
)
ADD_EXECUTABLE ( vdb_render
  ${VDB_RENDER_SOURCE_FILES}
  )

TARGET_LINK_LIBRARIES ( vdb_render
  openvdb
  ${BLOSC_blosc_LIBRARY} 
  ${Ilmbase_ILMTHREAD_LIBRARY}
  ${Ilmbase_IEX_LIBRARY}
  ${Openexr_ILMIMF_LIBRARY}
  )

SET ( VDB_VIEW_SOURCE_FILES
  cmd/openvdb_view/main.cc
  viewer/Camera.cc
  viewer/ClipBox.cc
  viewer/Font.cc
  viewer/RenderModules.cc
  viewer/Viewer.cc
 )
SET_SOURCE_FILES_PROPERTIES ( ${VDB_VIEW_SOURCE_FILES}
  PROPERTIES
  COMPILE_FLAGS "-DOPENVDB_USE_BLOSC ${OPENVDB_USE_GLFW_FLAG} -DGL_GLEXT_PROTOTYPES=1"
)
ADD_EXECUTABLE ( vdb_view
  ${VDB_VIEW_SOURCE_FILES}
  )

TARGET_LINK_LIBRARIES ( vdb_view
  openvdb
  ${BLOSC_blosc_LIBRARY} 
  ${OPENGL_gl_LIBRARY}
  ${OPENGL_glu_LIBRARY}
  ${COCOA_LIBRARY}
  ${IOKIT_LIBRARY}
  ${COREVIDEO_LIBRARY}
  ${GLFW3_glfw_LIBRARY}
  ${XI_LIBRARY}
  ${XRANDR_LIBRARY}
  ${GLEW_GLEW_LIBRARY}
  )

# Doxygen docmentation
FILE ( READ ${CMAKE_SOURCE_DIR}/doxygen-config DOXYGEN_CONFIG_CONTENT )
FILE ( WRITE ${CMAKE_CURRENT_BINARY_DIR}/cmake-doxygen-config ${DOXYGEN_CONFIG_CONTENT} )
FILE ( APPEND ${CMAKE_CURRENT_BINARY_DIR}/cmake-doxygen-config "OUTPUT_DIRECTORY=${CMAKE_BINARY_DIR}/doc\n" )

ADD_CUSTOM_TARGET ( doc ALL
  ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/cmake-doxygen-config
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  COMMENT "Generating API documentation with Doxygen" VERBATIM
)

INSTALL ( DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc/html/ DESTINATION doc)

# Installation
INSTALL ( TARGETS
  vdb_view
  vdb_print
  vdb_render
  DESTINATION
  bin
  )

INSTALL ( TARGETS
  openvdb
  DESTINATION
  lib
  )

INSTALL ( DIRECTORY ${CMAKE_SOURCE_DIR} DESTINATION include
          FILES_MATCHING PATTERN "*.h")