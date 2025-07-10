cmake_minimum_required(VERSION 3.10)

# 设置OpenGL策略（优先使用GLVND）
cmake_policy(SET CMP0072 NEW)  # 新增这一行，优先使用GLVND

project(OpenGLExample)

# 设置C++标准（跨平台统一）
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 配置输出目录（统一路径，避免平台差异）
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_BINARY_DIR}/bin/Debug)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR}/bin/Release)

# 配置vcpkg工具链（跨平台自动识别）
if(DEFINED ENV{VCPKG_ROOT} AND NOT DEFINED CMAKE_TOOLCHAIN_FILE)
    set(CMAKE_TOOLCHAIN_FILE "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
        CACHE STRING "Vcpkg toolchain file")
endif()

# 查找依赖库（跨平台统一接口）
find_package(glfw3 REQUIRED)
find_package(OpenGL REQUIRED)
find_package(glad REQUIRED)

# 源文件（共用一套代码）
set(SOURCE_FILES src/main.cpp)
add_executable(${PROJECT_NAME} ${SOURCE_FILES})

# 编译选项（平台差异化警告配置）
target_compile_options(${PROJECT_NAME} PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wno-unused-parameter>  # GCC/Clang
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /wd4100>  # MSVC（屏蔽无关警告）
)

# 链接库（跨平台统一链接目标）
target_link_libraries(${PROJECT_NAME}
    glfw
    ${OPENGL_LIBRARIES}  # 自动适配Windows的opengl32.lib、Linux的libGL.so等
    glad::glad
)

# ================= 跨平台动态库处理 =================
if(WIN32)
    # Windows：复制DLL到执行目录
    set(VCPKG_DLL_DIR ${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/bin)
    set(VCPKG_DLL_DEBUG_DIR ${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/debug/bin)
    
    find_file(GLFW_DLL NAMES glfw3.dll PATHS ${VCPKG_DLL_DIR} ${VCPKG_DLL_DEBUG_DIR} NO_DEFAULT_PATH)
    find_file(GLAD_DLL NAMES glad.dll PATHS ${VCPKG_DLL_DIR} ${VCPKG_DLL_DEBUG_DIR} NO_DEFAULT_PATH)
    
    if(GLFW_DLL AND GLAD_DLL)
        add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                ${GLFW_DLL} ${GLAD_DLL}
                $<TARGET_FILE_DIR:${PROJECT_NAME}>
            COMMENT "复制Windows依赖DLL"
        )
    endif()
elseif(UNIX)
    # Linux/macOS：设置RPATH，避免手动复制.so/.dylib
    set_target_properties(${PROJECT_NAME} PROPERTIES
        BUILD_RPATH_USE_ORIGIN TRUE  # 运行时优先从可执行文件所在目录找库
        INSTALL_RPATH "$ORIGIN"     # 安装后仍可从同目录找库
    )
endif()

# ================= 着色器文件复制（跨平台统一） =================
set(SHADER_FILES vertShader.glsl fragShader.glsl)
foreach(SHADER ${SHADER_FILES})
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_SOURCE_DIR}/${SHADER}
            $<TARGET_FILE_DIR:${PROJECT_NAME}>
        COMMENT "复制着色器文件: ${SHADER}"
    )
endforeach()
