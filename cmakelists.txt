cmake_minimum_required(VERSION 3.12)
cmake_policy(SET CMP0074 NEW)

project(onscripter-jh VERSION 1.0)

# ==========================================
# 强制开启 C++17 标准，解决渲染器的编译报错
# ==========================================
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ==========================================
# 1. 查找依赖包
# ==========================================
find_package(SDL2 CONFIG REQUIRED)
find_package(SDL2_mixer CONFIG REQUIRED) 
find_package(SDL2_image CONFIG REQUIRED) 
find_package(SDL2_ttf CONFIG REQUIRED)   
find_package(Ogg CONFIG REQUIRED)
find_package(Vorbis CONFIG REQUIRED)
find_package(Opus CONFIG REQUIRED)
find_package(BZip2 REQUIRED)
find_package(libjpeg-turbo CONFIG REQUIRED)
find_package(Lua REQUIRED)

include_directories(
    src src/renderer src/simd
    ${LUA_INCLUDE_DIR}
)

# =========================================================
# 产物重定向
# =========================================================
string(REPLACE "/out/build/" "/out/dist/" MY_DIST_DIR "${CMAKE_BINARY_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${MY_DIST_DIR}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${MY_DIST_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${MY_DIST_DIR}")

# =========================================================
# [终极精准修复] 生成变量存根
# =========================================================
set(MISSING_GLOBALS_STUB "${CMAKE_BINARY_DIR}/missing_globals.cpp")
file(WRITE ${MISSING_GLOBALS_STUB} "
class Coding2UTF16; class SJIS2UTF16; class GBK2UTF16;
Coding2UTF16* coding2utf16 = nullptr;
SJIS2UTF16* sjis2utf16 = nullptr;
GBK2UTF16* gbk2utf16 = nullptr;
")

# ==========================================
# 2. 定义源文件集合
# ==========================================
set(CHARSET_SUPPORT_OBJS 
    src/coding2utf16.cpp
    src/sjis2utf16.cpp
    src/gbk2utf16.cpp
)

set(DECODER_OBJS 
    src/DirectReader.cpp
    src/SarReader.cpp
    src/NsaReader.cpp
    ${CHARSET_SUPPORT_OBJS}
)

set(GUI_OBJS  
    src/ONScripter.cpp 
    src/ONScripter_animation.cpp 
    src/ONScripter_command.cpp 
    src/ONScripter_effect.cpp 
    src/ONScripter_effect_breakup.cpp 
    src/ONScripter_event.cpp 
    src/ONScripter_file.cpp 
    src/ONScripter_file2.cpp 
    src/ONScripter_image.cpp 
    src/ONScripter_lut.cpp 
    src/ONScripter_rmenu.cpp 
    src/ONScripter_sound.cpp 
    src/ONScripter_text.cpp 
    src/AnimationInfo.cpp 
    src/FontInfo.cpp 
    src/DirtyRect.cpp 
    src/resize_image.cpp 
    src/renderer/gles_renderer.cpp
)

set(BUILTIN_DLL_OBJS
    src/builtin_dll/layer_snow.cpp 
    src/builtin_dll/ONScripter_effect_cascade.cpp 
    src/builtin_dll/ONScripter_effect_trig.cpp 
    src/builtin_dll/layer_oldmovie.cpp
)

set(EXT_OBJS
    src/LUAHandler.cpp
    src/Parallel.cpp
)

set(ONSCRIPTER_OBJS 
    src/onscripter_main.cpp
    src/ScriptHandler.cpp 
    src/ScriptParser.cpp 
    src/ScriptParser_command.cpp
    ${GUI_OBJS}
    ${BUILTIN_DLL_OBJS}
    ${DECODER_OBJS}
    ${EXT_OBJS}
)

# 精准分发：NSA 系列自带变量不需要存根；SAR 系列需要存根
set(SARDEC_OBJS src/sardec.cpp src/DirectReader.cpp src/SarReader.cpp ${CHARSET_SUPPORT_OBJS} ${MISSING_GLOBALS_STUB})
set(SARCONV_OBJS src/sarconv.cpp src/conv_shared.cpp src/resize_image.cpp src/DirectReader.cpp src/SarReader.cpp ${CHARSET_SUPPORT_OBJS} ${MISSING_GLOBALS_STUB})

set(NSADEC_OBJS src/nsadec.cpp ${DECODER_OBJS})
set(NSACONV_OBJS src/nsaconv.cpp src/conv_shared.cpp src/resize_image.cpp ${DECODER_OBJS})

# ==========================================
# 3. 依赖库与编译定义
# ==========================================
set(COMMON_LIBS 
    SDL2::SDL2main 
    $<IF:$<TARGET_EXISTS:SDL2::SDL2>,SDL2::SDL2,SDL2::SDL2-static>
    $<IF:$<TARGET_EXISTS:SDL2_mixer::SDL2_mixer>,SDL2_mixer::SDL2_mixer,SDL2_mixer::SDL2_mixer-static>
    $<IF:$<TARGET_EXISTS:SDL2_image::SDL2_image>,SDL2_image::SDL2_image,SDL2_image::SDL2_image-static>
    $<IF:$<TARGET_EXISTS:SDL2_ttf::SDL2_ttf>,SDL2_ttf::SDL2_ttf,SDL2_ttf::SDL2_ttf-static>
    Ogg::ogg Vorbis::vorbis Vorbis::vorbisfile Opus::opus BZip2::BZip2
    $<IF:$<TARGET_EXISTS:libjpeg-turbo::jpeg>,libjpeg-turbo::jpeg,libjpeg-turbo::jpeg-static>
    ${LUA_LIBRARIES}
)

set(CONVERTER_LIBS 
    $<IF:$<TARGET_EXISTS:SDL2::SDL2>,SDL2::SDL2,SDL2::SDL2-static>
    $<IF:$<TARGET_EXISTS:SDL2_mixer::SDL2_mixer>,SDL2_mixer::SDL2_mixer,SDL2_mixer::SDL2_mixer-static>
    $<IF:$<TARGET_EXISTS:SDL2_image::SDL2_image>,SDL2_image::SDL2_image,SDL2_image::SDL2_image-static>
    $<IF:$<TARGET_EXISTS:SDL2_ttf::SDL2_ttf>,SDL2_ttf::SDL2_ttf,SDL2_ttf::SDL2_ttf-static>
    Ogg::ogg Vorbis::vorbis Vorbis::vorbisfile Opus::opus BZip2::BZip2
    $<IF:$<TARGET_EXISTS:libjpeg-turbo::jpeg>,libjpeg-turbo::jpeg,libjpeg-turbo::jpeg-static>
)

set(COMMON_DEFINITIONS "_CRT_SECURE_NO_WARNINGS;WIN32;USE_OGG_VORBIS;UTF8_CAPTION;USE_SDL_RENDERER;_CONSOLE;USE_LUA")

# ==========================================
# 4. 注册可执行文件目标
# ==========================================
add_executable(onscripter ${ONSCRIPTER_OBJS}) 
set_target_properties(onscripter PROPERTIES COMPILE_DEFINITIONS "${COMMON_DEFINITIONS}"OUTPUT_NAME "ONScripter-Jh")
target_link_libraries(onscripter PRIVATE ${COMMON_LIBS})

add_executable(sarconv ${SARCONV_OBJS}) 
set_target_properties(sarconv PROPERTIES COMPILE_DEFINITIONS "${COMMON_DEFINITIONS}")
target_link_libraries(sarconv PRIVATE ${CONVERTER_LIBS})

add_executable(nsaconv ${NSACONV_OBJS}) 
set_target_properties(nsaconv PROPERTIES COMPILE_DEFINITIONS "${COMMON_DEFINITIONS}")
target_link_libraries(nsaconv PRIVATE ${CONVERTER_LIBS})

add_executable(sardec ${SARDEC_OBJS}) 
set_target_properties(sardec PROPERTIES COMPILE_DEFINITIONS "${COMMON_DEFINITIONS}")
target_link_libraries(sardec PRIVATE ${CONVERTER_LIBS})

add_executable(nsadec ${NSADEC_OBJS}) 
set_target_properties(nsadec PROPERTIES COMPILE_DEFINITIONS "${COMMON_DEFINITIONS}")
target_link_libraries(nsadec PRIVATE ${CONVERTER_LIBS})

# nscriptdecode 也缺少变量，精确分发存根
add_executable(nscriptdecode src/nscriptdecode.cpp ${DECODER_OBJS} ${MISSING_GLOBALS_STUB})
set_target_properties(nscriptdecode PROPERTIES COMPILE_DEFINITIONS "${COMMON_DEFINITIONS}")
target_link_libraries(nscriptdecode PRIVATE ${CONVERTER_LIBS})

# ==========================================
# 5. 自动收集并拷贝 LICENSE 文件 (Post-Build)
# ==========================================
# 定义你的依赖库的 vcpkg 包名列表 (包含底层依赖)
set(VCPKG_LIBS 
    sdl2 sdl2-mixer sdl2-image sdl2-ttf 
    libogg libvorbis opus bzip2 libjpeg-turbo lua 
    flac freetype libpng tiff zlib libwebp
)

# 动态定位 vcpkg 的 share 目录路径
if(DEFINED VCPKG_INSTALLED_DIR AND DEFINED VCPKG_TARGET_TRIPLET)
    set(VCPKG_SHARE_DIR "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/share")
else()
    # 兼容 Manifest 模式的常规备用路径
    set(VCPKG_SHARE_DIR "${CMAKE_BINARY_DIR}/vcpkg_installed/x64-windows/share") 
endif()

# 1. 拷贝主项目的开源协议 
# (如果在项目根目录检测到了 COPYING 文件，就会自动拷贝过去)
if(EXISTS "${CMAKE_SOURCE_DIR}/COPYING")
    add_custom_command(TARGET onscripter POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different 
            "${CMAKE_SOURCE_DIR}/COPYING" "${MY_DIST_DIR}/COPYING"
        COMMENT "Copying ONScripter-Jh COPYING file..."
    )
endif()

# 2. 自动遍历并拷贝所有第三方依赖库的 copyright 文件
foreach(LIB_NAME IN LISTS VCPKG_LIBS)
    set(SRC_LICENSE "${VCPKG_SHARE_DIR}/${LIB_NAME}/copyright")
    set(DST_LICENSE "${MY_DIST_DIR}/LICENSE.${LIB_NAME}.txt")
    
    # 只有在配置阶段确认该协议文件存在，才会向构建系统添加拷贝指令
    if(EXISTS "${SRC_LICENSE}")
        add_custom_command(TARGET onscripter POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different "${SRC_LICENSE}" "${DST_LICENSE}"
            COMMENT "Copying license for ${LIB_NAME}..."
        )
    endif()
endforeach()