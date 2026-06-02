



# AliceVision 编译指南（WSL Ubuntu 22.04）

## 环境信息

| 项目 | 说明 |
|------|------|
| 宿主机 | Windows 11 + Clash Verge VPN（代理端口 7897） |
| WSL | Ubuntu 22.04（镜像网络模式） |
| CMake | 3.31.5（免安装版，安装于 ~/.local/） |
| 编译器 | GCC 11+（C++17） |

---

## 一、系统准备

### 1.1 安装 CMake 3.31.5

```bash
cd ~
mkdir -p .local/bin

# 下载 CMake 官方编译好的免安装压缩包
wget https://github.com/Kitware/CMake/releases/download/v3.31.5/cmake-3.31.5-linux-x86_64.tar.gz

# 解压
tar -zxvf cmake-3.31.5-linux-x86_64.tar.gz

# 复制到局部环境变量路径
cp -r cmake-3.31.5-linux-x86_64/* ~/.local/

# 添加到 PATH
export PATH=$HOME/.local/bin:$PATH
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc

# 验证
cmake --version
```

### 1.2 安装系统依赖

```bash
sudo apt update sudo apt install -y \  libboost-all-dev libflann-dev bison
# 清空缓存文件
rm -rf CMakeCache.txt CMakeFiles/
# 1. 重新配置
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBoost_USE_STATIC_LIBS=OFF \  # 强制关闭boost静态库 显式指定使用动态 Boost 库
  -DLIB_SUFFIX=/x86_64-linux-gnu \
  -DALICEVISION_USE_OPENCV=ON \
  -DALICEVISION_USE_CCTAG=ON \
  -DALICEVISION_BUILD_EXAMPLES=OFF \
  -DALIZEVISION_BUILD_TESTS=OFF

# 2. 定向编译特征提取模块
make aliceVision_featureExtraction -j$(nproc)

```


### 1.3 Troubleshotting

```question
1. 一直提示找不到Boost
CMake Error at src/CMakeLists.txt:329 (message): Failed to find Boost.
解决：查看boost 版本：
root@lyinc:~/git_src/AliceVision/build# dpkg -s libboost-dev | grep Version 
Version: 1.74.0.3ubuntu7
查看AliceVision版本：git describe --tags

在CMakeList.txt中查看源码：
root@lyinc:~/git_src/AliceVision/build# sed -n '280,340p' ~/git_src/AliceVision/src/CMakeLists.txt
找到突破点：
find_package(Boost 1.76.0 QUIET COMPONENTS ${ALICEVISION_BOOST_COMPONENTS} ${ALICEVISION_BOOST_COMPONENT_UNITTEST})

要求Boost 1.76.0，而系统是Boost 1.74.0
两种办法：该CMakeList.txt ，但是有潜在问题：1.75引入了Boost.JSON，1.74中没有，后续编译依然会失败。


解决方案：升级Boost到1.76
（1）root@lyinc:~/git_src/AliceVision# cd third3p/
root@lyinc:~/git_src/AliceVision/third3p# wget https://archives.boost.io/release/1.86.0/source/boost_1_86_0.tar.gz
（2）tar xf boost_1_86_0.tar.gz
cd boost_1_86_0
./bootstrap.sh
./b2 install

```



```question
2. FLANN找不到的问题 CMake Error at src/CMakeLists.txt:505 (message): FLANN can not be found
安装 sudo apt update
sudo apt install libflann-dev
dpkg -l | grep flann # 检查是否已安装

root@lyinc:~/git_src/AliceVision/build# ls /usr/include/flann/flann.hpp # 检查头文件
/usr/include/flann/flann.hpp 

root@lyinc:~/git_src/AliceVision/build# ls /usr/lib/x86_64-linux-gnu/libflann* # 检查库文件
/usr/lib/x86_64-linux-gnu/libflann.so        /usr/lib/x86_64-linux-gnu/libflann_cpp.so.1.9
/usr/lib/x86_64-linux-gnu/libflann.so.1.9    /usr/lib/x86_64-linux-gnu/libflann_cpp.so.1.9.1
/usr/lib/x86_64-linux-gnu/libflann.so.1.9.1  /usr/lib/x86_64-linux-gnu/libflann_cpp_s.a
/usr/lib/x86_64-linux-gnu/libflann_cpp.so    /usr/lib/x86_64-linux-gnu/libflann_s.a


root@lyinc:~/git_src/AliceVision/build# sed -n '470,520p' ~/git_src/AliceVision/src/CMakeLists.txt
        if (ALICEVISION_REQUIRE_CERES_WITH_SUITESPARSE)
            # Ceres export include dirs but doesn't export suitesparse lib dependencies in CeresConfig.cmake
            # So here is a workaround:
            find_package(SuiteSparse)
            message(STATUS "SUITESPARSE_LIBRARIES: ${SUITESPARSE_LIBRARIES}")
            if (SUITESPARSE_LIBRARIES)
                list(APPEND CERES_LIBRARIES ${SUITESPARSE_LIBRARIES})
            endif()
        endif()
        message(STATUS "CERES_LIBRARIES: ${CERES_LIBRARIES}")
        message(STATUS "CERES_INCLUDE_DIRS: ${CERES_INCLUDE_DIRS}")
        if (WIN32)
            # avoid 'ERROR' macro clashing on Windows
            add_definitions(-DGLOG_NO_ABBREVIATED_SEVERITIES)
        endif()
        include_directories(${CERES_INCLUDE_DIRS})
    else()
        message(FATAL_ERROR "External CERES not found. Not found in Ceres_DIR: ${Ceres_DIR}")
    endif()
endif()

# ==============================================================================
# Flann
# ==============================================================================
if (ALICEVISION_BUILD_SFM)
    find_package(lz4 REQUIRED)
    find_package(flann REQUIRED)

    if (TARGET lz4::lz4)
        set(FLANN_LIBRARIES flann::flann_cpp lz4::lz4)
    elseif (TARGET LZ4::lz4_static)
        set(FLANN_LIBRARIES flann::flann_cpp LZ4::lz4_static)
    elseif (TARGET LZ4::lz4_shared)
        set(FLANN_LIBRARIES flann::flann_cpp LZ4::lz4_shared)
    else()
        message(FATAL_ERROR "FLANN can not be found")
    endif()
endif()

找到问题：真正的是LZ4 target 没有找到，而不是flann

root@lyinc:~/git_src/AliceVision/build# dpkg -l | grep lz4
ii  liblz4-1:amd64                         1.9.3-2build2                           amd64        Fast LZ compression algorithm library - runtime
ii  liblz4-dev:amd64                       1.9.3-2build2                           amd64        Fast LZ compression algorithm library - development files
ii  lz4                                    1.9.3-2build2                           amd64        Fast LZ

root@lyinc:~/git_src/AliceVision/build# find /usr -name "*lz4*cmake" 2>/dev/null
/usr/local/lib/cmake/lz4/lz4Config.cmake

Ubunutu本身有lz4，彻底删除：
rm -rf /usr/local/lib/cmake/lz4*
执行：cmake .. 


原因：有lz4.so,但是不支持 lz4Config.cmake

git clone https://github.com/lz4/lz4.git
cd build/cmake
编译安装：
cmake . \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local

make -j$(nproc)
sudo make install

===============================

# 重新编 AliceVision
rm -rf ~/git_src/AliceVision/build
mkdir ~/git_src/AliceVision/build
cd ~/git_src/AliceVision/build
cmake .. \
  -DBoost_ROOT=/usr/local \
  -DCMAKE_PREFIX_PATH=/usr/local
  
```



```问题3
3. CoinUtils not found

CMake Error at src/CMakeLists.txt:513 (find_package):
  By not providing "FindCoinUtils.cmake" in CMAKE_MODULE_PATH this project
  has asked CMake to find a package configuration file provided by
  "CoinUtils", but CMake did not find one.

  Could not find a package configuration file provided by "CoinUtils" with
  any of the following names:

    CoinUtilsConfig.cmake
    coinutils-config.cmake

  Add the installation prefix of "CoinUtils" to CMAKE_PREFIX_PATH or set
  "CoinUtils_DIR" to a directory containing one of the above files.  If
  "CoinUtils" provides a separate development package or SDK, be sure it has
  been installed.
  
  (1) sudo apt install -y \
    coinor-libcoinutils-dev \
    coinor-libosi-dev \
    coinor-libclp-dev
    
    (2)安装带config的CoinUtils.git 使用AliceVision自带的 CoinUtils.git
    cd third3p 
    
    git clone https://github.com/alicevision/CoinUtils.git  # 带有CMakeList.txt
    cd CoinUtils

    mkdir build
    cd build
    cmake .. \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local

    make -j$(nproc)
    sudo make install
    
  =======
  # 重新编 AliceVision
rm -rf ~/git_src/AliceVision/build
mkdir ~/git_src/AliceVision/build
cd ~/git_src/AliceVision/build
cmake .. \
  -DBoost_ROOT=/usr/local \
  -DCMAKE_PREFIX_PATH=/usr/local
   
```

```question
4. 找不到CLP
CMake Error at src/CMakeLists.txt:514 (find_package):
  By not providing "FindClp.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "Clp", but
  CMake did not find one.

  Could not find a package configuration file provided by "Clp" with any of
  the following names:

    ClpConfig.cmake
    clp-config.cmake

  Add the installation prefix of "Clp" to CMAKE_PREFIX_PATH or set "Clp_DIR"
  to a directory containing one of the above files.  If "Clp" provides a
  separate development package or SDK, be sure it has been installed.
  
  同样使用AliceVision自带的fork https://github.com/alicevision/Clp
  
  编译安装时出现FindOsi找不到
  CMake Error at CMakeLists.txt:36 (find_package):
  By not providing "FindOsi.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "Osi", but
  CMake did not find one.

  Could not find a package configuration file provided by "Osi" with any of
  the following names:

    OsiConfig.cmake
    osi-config.cmake

  Add the installation prefix of "Osi" to CMAKE_PREFIX_PATH or set "Osi_DIR"
  to a directory containing one of the above files.  If "Osi" provides a
  separate development package or SDK, be sure it has been installed.
  
  安装Osi 同样使用AliceVision自带的fork：
  git clone  https://github.com/alicevision/Osi.git
  mkdir build
  cd build
  cmake .. \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local
  cmake .. \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON \
  -DCMAKE_BUILD_TYPE=Release
  make -j$(nproc)
    sudo make install
    
   
  
  
  
  继续安装CLP
  cmake . \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local
 
 cmake .. \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON \
  -DCMAKE_BUILD_TYPE=Release

    make -j$(nproc)
    sudo make install
    
   CLP安装完成
   
   ====================================
   重新编 AliceVision
   rm -rf ~/git_src/AliceVision/build
mkdir ~/git_src/AliceVision/build
cd ~/git_src/AliceVision/build
cmake .. \
  -DBoost_ROOT=/usr/local \
  -DCMAKE_PREFIX_PATH=/usr/local
  
```

```question
5.找不到 LEMONConfig
CMake Error at src/CMakeLists.txt:522 (find_package):
  By not providing "FindLEMON.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "LEMON", but
  CMake did not find one.

  Could not find a package configuration file provided by "LEMON" with any of
  the following names:

    LEMONConfig.cmake
    lemon-config.cmake

  Add the installation prefix of "LEMON" to CMAKE_PREFIX_PATH or set
  "LEMON_DIR" to a directory containing one of the above files.  If "LEMON"
  provides a separate development package or SDK, be sure it has been
  installed.
  
  安装图论包 lemon
  sudo apt install -y liblemon-dev
  将 AliceVision/src/CMakeLists.txt中
  if (ALICEVISION_BUILD_SFM)
    find_package(LEMON REQUIRED)
endif()
改成
if (ALICEVISION_BUILD_SFM)
    set(LEMON_FOUND TRUE)
    set(LEMON_INCLUDE_DIRS "/usr/include")
    set(LEMON_LIBRARIES "/usr/lib/x86_64-linux-gnu/liblemon.so")
    message(STATUS "Manually set LEMON for SfM")
endif()
  
  ==== 
重新编 AliceVision
   rm -rf ~/git_src/AliceVision/build
mkdir ~/git_src/AliceVision/build
cd ~/git_src/AliceVision/build
cmake .. \
  -DBoost_ROOT=/usr/local \
  -DCMAKE_PREFIX_PATH=/usr/local
```



```question
6. 出现新问题：缺少三维模型导入导出的核心库 assimp
-- Manually set LEMON for SfM
CMake Warning at src/CMakeLists.txt:533 (find_package):
  By not providing "FindE57Format.cmake" in CMAKE_MODULE_PATH this project
  has asked CMake to find a package configuration file provided by
  "E57Format", but CMake did not find one.

  Could not find a package configuration file provided by "E57Format" with
  any of the following names:

    E57FormatConfig.cmake
    e57format-config.cmake

  Add the installation prefix of "E57Format" to CMAKE_PREFIX_PATH or set
  "E57Format_DIR" to a directory containing one of the above files.  If
  "E57Format" provides a separate development package or SDK, be sure it has
  been installed.


CMake Error at src/CMakeLists.txt:547 (find_package):
  By not providing "Findassimp.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "assimp", but
  CMake did not find one.

  Could not find a package configuration file provided by "assimp" with any
  of the following names:

    assimpConfig.cmake
    assimp-config.cmake

  Add the installation prefix of "assimp" to CMAKE_PREFIX_PATH or set
  "assimp_DIR" to a directory containing one of the above files.  If "assimp"
  provides a separate development package or SDK, be sure it has been
  installed.
  
  直接安装，在重新编译
  sudo apt install -y libassimp-dev
  
  重新编 AliceVision
   rm -rf ~/git_src/AliceVision/build
mkdir ~/git_src/AliceVision/build
cd ~/git_src/AliceVision/build
  cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=ON \
  -DALICEVISION_BUNDLE_DEPENDENCIES=ON \
  -DALICEVISION_BUILD_SFM=ON \
  -DALICEVISION_USE_OPENCV=ON \
  -DALICEVISION_USE_CCTAG=ON \
  -DALICEVISION_BUILD_EXAMPLES=OFF \
  -DALIZEVISION_BUILD_TESTS=OFF
  
  
```



```
7. 缺少几何库：Geogram 
安装geogram
安装缺失的 Xinerama 及其相关的 X11 开发包
sudo apt install -y libxinerama-dev libxcursor-dev libxi-dev libxrandr-dev
git clone https://github.com/BrunoLevy/geogram
git submodule update --init --recursive
./configure.sh # 官方自带了配置脚本
cd build/Linux64-gcc-release
cmake .
make -j$(nproc)
sudo make install
sudo ldconfig

重新编 AliceVision
   rm -rf ~/git_src/AliceVision/build
mkdir ~/git_src/AliceVision/build
cd ~/git_src/AliceVision/build
  cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=ON \
  -DALICEVISION_BUNDLE_DEPENDENCIES=ON \
  -DALICEVISION_BUILD_SFM=ON \
  -DALICEVISION_USE_OPENCV=ON \
  -DALICEVISION_USE_CCTAG=ON \
  -DALICEVISION_BUILD_EXAMPLES=OFF \
  -DALIZEVISION_BUILD_TESTS=OFF

```



```question
8. OpenMeshConfig
CMake Error at src/CMakeLists.txt:691 (find_package):
  By not providing "FindOpenMesh.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "OpenMesh",
  but CMake did not find one.

  Could not find a package configuration file provided by "OpenMesh" with any
  of the following names:

    OpenMeshConfig.cmake
    openmesh-config.cmake

  Add the installation prefix of "OpenMesh" to CMAKE_PREFIX_PATH or set
  "OpenMesh_DIR" to a directory containing one of the above files.  If
  "OpenMesh" provides a separate development package or SDK, be sure it has
  been installed.
  
  sudo apt install -y libopenmesh-dev -- 未找到该包
  wget https://www.graphics.rwth-aachen.de/media/openmesh_static/Releases/11.0/OpenMesh-11.0.0.tar.gz
  tar -zxvf OpenMesh-11.0.0.tar.gz
  cd OpenMesh-11.0.0
  mkdir build && cd build
# 2. 配置 CMake（指定发布模式，并关闭其自带的可选应用以节省编译时间）
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DOPENMESH_BUILD_APPS=OFF
# 3. 全核并行编译
make -j$(nproc)
# 4. 安装到系统根路径（默认会写入 /usr/local/include 和 /usr/local/lib）
sudo make install
# 5. 刷新系统的动态库链接缓存
sudo ldconfig

重新编 AliceVision
   rm -rf ~/git_src/AliceVision/build
mkdir ~/git_src/AliceVision/build
cd ~/git_src/AliceVision/build
  cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=ON \
  -DALICEVISION_BUNDLE_DEPENDENCIES=ON \
  -DALICEVISION_BUILD_SFM=ON \
  -DALICEVISION_USE_OPENCV=ON \
  -DALICEVISION_USE_CCTAG=ON \
  -DALICEVISION_BUILD_EXAMPLES=OFF \
  -DALIZEVISION_BUILD_TESTS=OFF

```

```question
9. CMake Error at src/CMakeLists.txt:716 (message):
  Failed to find a CUDA compiler.
  跳过CUDA 先不开
  ALICEVISION_USE_CUDA= OFF
  

```

```question
10. 暂时不开CCTag
Could NOT find CCTag (missing: CCTag_DIR)
CMake Error at src/CMakeLists.txt:855 (message):
  Failed to find CCTAG.
  
  ALICEVISION_USE_CUDA=OFF
  
 重新编译 
 重新编 AliceVision
   rm -rf ~/git_src/AliceVision/build
mkdir ~/git_src/AliceVision/build
cd ~/git_src/AliceVision/build
  cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=ON \
  -DALICEVISION_BUNDLE_DEPENDENCIES=ON \
  -DALICEVISION_BUILD_SFM=ON \
  -DALICEVISION_USE_OPENCV=ON \
  -DALICEVISION_USE_CCTAG=OFF \
  -DALICEVISION_BUILD_EXAMPLES=OFF \
  -DALIZEVISION_BUILD_TESTS=OFF \
  -DALICEVISION_USE_CUDA=OFF
```



```
11. configuration done , but build failed

-- Configuring done (40.8s)
CMake Error at src/cmake/Helpers.cmake:72 (target_link_libraries):
  Target "aliceVision_matching" links to:

    flann::flann_cpp
  but the target was not found.  Possible reasons include:

    * There is a typo in the target name.
    * A find_package call is missing for an IMPORTED target.
    * An ALIAS target is missing.
Call Stack (most recent call first):
  src/aliceVision/matching/CMakeLists.txt:33 (alicevision_add_library)
CMake Error at src/cmake/Helpers.cmake:72 (target_link_libraries):
  Target "aliceVision_matchingImageCollection" links to:

    flann::flann_cpp
  but the target was not found.  Possible reasons include:

    * There is a typo in the target name.
    * A find_package call is missing for an IMPORTED target.
    * An ALIAS target is missing.
Call Stack (most recent call first):
  src/aliceVision/matchingImageCollection/CMakeLists.txt:33 (alicevision_add_library)
-- Generating done (0.5s)


编译flann 
git clone https://github.com/alicevision/flann
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  -DBUILD_MATLAB_BINDINGS=OFF \
  -DBUILD_PYTHON_BINDINGS=OFF \
  -DBUILD_EXAMPLES=OFF \
  -DBUILD_TESTS=OFF
 make -j$(nproc)
 sudo make install
 sudo ldconfig
 
  重新编译 
 重新编 AliceVision
   rm -rf ~/git_src/AliceVision/build
mkdir ~/git_src/AliceVision/build
cd ~/git_src/AliceVision/build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=ON \
  -DALICEVISION_BUNDLE_DEPENDENCIES=OFF \
  -DALICEVISION_BUILD_DEPENDENCIES=OFF \
  -DALICEVISION_BUILD_SFM=ON \
  -DALICEVISION_USE_OPENCV=ON \
  -DALICEVISION_USE_CCTAG=OFF \
  -DALICEVISION_BUILD_EXAMPLES=OFF \
  -DALIZEVISION_BUILD_TESTS=OFF \
  -DALICEVISION_USE_CUDA=OFF \
  -DCMAKE_CXX_STANDARD=17 \
  -DCMAKE_PREFIX_PATH="/usr;/usr/local" \
  -DLEMON_FOUND=TRUE \
  -DLEMON_INCLUDE_DIRS="/usr/local/include" \
  -DLEMON_LIBRARIES="/usr/local/lib/libemon.a" \
  -DOpenEXR_ROOT=/usr
  -DALICEVISION_BUILD_LIDAR=OFF # 关闭激光点云
  
  cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=ON \
  -DALICEVISION_BUILD_DEPENDENCIES=ON \
  -DALICEVISION_BUILD_SFM=ON \
  -DALICEVISION_USE_OPENCV=ON \
  -DALICEVISION_USE_CCTAG=OFF \
  -DALICEVISION_USE_CUDA=OFF \
  -DALICEVISION_BUILD_LIDAR=OFF \
  -DALICEVISION_BUILD_EXAMPLES=OFF \
  -DALIZEVISION_BUILD_TESTS=OFF \
  -DCMAKE_CXX_STANDARD=17 \
  -DLEMON_FOUND=TRUE \
  -DLEMON_INCLUDE_DIRS="/usr/local/include" \
  -DLEMON_LIBRARIES="/usr/local/lib/libemon.a" \
  -DALICEVISION_BUILD_LIDAR=OFF 
  
  cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=ON \
  -DALICEVISION_BUILD_DEPENDENCIES=ON \
  -DUSE_EXTERNAL_LEMON=ON \
  -DLEMON_FOUND=TRUE \
  -DLEMON_INCLUDE_DIR="/usr/local/include" \
  -DLEMON_LIBRARY="/usr/local/lib/libemon.a" \
  -DALICEVISION_BUILD_SFM=ON \
  -DALICEVISION_USE_OPENCV=ON \
  -DALICEVISION_USE_CCTAG=OFF \
  -DALICEVISION_USE_CUDA=OFF \
  -DCMAKE_CXX_STANDARD=17 \
  -DALICEVISION_BUILD_LIDAR=OFF
  
  
  
  
  
-DAV_USE_AVX=OFF -DCMAKE_POSITION_INDEPENDENT_CODE=ON

```

```
12.安装 git clone https://github.com/ceres-solver/ceres-solver
wget http://ceres-solver.org/ceres-solver-2.2.0.tar.gz
tar zxf ceres-solver-2.2.0.tar.gz
mkdir ceres-bin
cd ceres-bin
cmake ../ceres-solver-2.2.0
make -j3
make test
# Optionally install Ceres, it can also be exported using CMake which
# allows Ceres to be used without requiring installation, see the documentation
# for the EXPORT_BUILD_DIR option for more information.
make install


安装 cmake ..   -DBoost_ROOT=/usr/local   -DCMAKE_PREFIX_PATH=/usr/local
需要安装 abslConfig
CMake Error at CMakeLists.txt:173 (find_package):
  By not providing "Findabsl.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "absl", but
  CMake did not find one.

  Could not find a package configuration file provided by "absl" with any of
  the following names:

    abslConfig.cmake
    absl-config.cmake

  Add the installation prefix of "absl" to CMAKE_PREFIX_PATH or set
  "absl_DIR" to a directory containing one of the above files.  If "absl"
  provides a separate development package or SDK, be sure it has been
  installed.
  
  ========
  git clone https://github.com/abseil/abseil-cpp.git
  cd abseil-cpp
  mkdir build & cd build
  cmake .. \
    -DCMAKE_CXX_STANDARD=17 \
    -DCMAKE_INSTALL_PREFIX=/usr/local
  make -j$(nproc)
sudo make install
```





![image-20260601175812873](C:\Users\31408\AppData\Roaming\Typora\typora-user-images\image-20260601175812873.png)



<h3>
    1.2 至此 cmake 已经成功，接下来make -j$(nproc)
</h3>

```
make错误：
/usr/local/include/OpenImageIO/imagebufalgo.h:2758:13: note: declared here
 2758 | inline bool resize (ImageBuf &dst, const ImageBuf &src,
      |             ^~~~~~
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp: In instantiation of ‘void aliceVision::imageAlgo::resizeImage(OpenImageIO::v3_1::TypeDesc, int, int, int, int, int, const T*, T*, const string&, float) [with T = float; std::string = std::__cxx11::basic_string<char>]’:
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:273:16:   required from here
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:234:31: warning: ‘bool OpenImageIO::v3_1::ImageBufAlgo::resize(OpenImageIO::v3_1::ImageBuf&, const OpenImageIO::v3_1::ImageBuf&, OpenImageIO::v3_1::string_view, float, OpenImageIO::v3_1::ROI, int)’ is deprecated: prefer the kind that takes keyword args (3.0) [-Wdeprecated-declarations]
  234 |     oiio::ImageBufAlgo::resize(outBuf, inBuf, filter, filterSize, oiio::ROI::All());
      |     ~~~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
In file included from /usr/local/include/OpenImageIO/imagebufalgo_util.h:11,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.hpp:8,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:7:
/usr/local/include/OpenImageIO/imagebufalgo.h:2758:13: note: declared here
 2758 | inline bool resize (ImageBuf &dst, const ImageBuf &src,
      |             ^~~~~~
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp: In instantiation of ‘void aliceVision::imageAlgo::resizeImage(OpenImageIO::v3_1::TypeDesc, int, int, int, int, int, const T*, T*, const string&, float) [with T = aliceVision::image::Rgb<unsigned char>; std::string = std::__cxx11::basic_string<char>]’:
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:304:16:   required from here
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:234:31: warning: ‘bool OpenImageIO::v3_1::ImageBufAlgo::resize(OpenImageIO::v3_1::ImageBuf&, const OpenImageIO::v3_1::ImageBuf&, OpenImageIO::v3_1::string_view, float, OpenImageIO::v3_1::ROI, int)’ is deprecated: prefer the kind that takes keyword args (3.0) [-Wdeprecated-declarations]
  234 |     oiio::ImageBufAlgo::resize(outBuf, inBuf, filter, filterSize, oiio::ROI::All());
      |     ~~~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
In file included from /usr/local/include/OpenImageIO/imagebufalgo_util.h:11,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.hpp:8,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:7:
/usr/local/include/OpenImageIO/imagebufalgo.h:2758:13: note: declared here
 2758 | inline bool resize (ImageBuf &dst, const ImageBuf &src,
      |             ^~~~~~
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp: In instantiation of ‘void aliceVision::imageAlgo::resizeImage(OpenImageIO::v3_1::TypeDesc, int, int, int, int, int, const T*, T*, const string&, float) [with T = aliceVision::image::Rgb<float>; std::string = std::__cxx11::basic_string<char>]’:
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:335:16:   required from here
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:234:31: warning: ‘bool OpenImageIO::v3_1::ImageBufAlgo::resize(OpenImageIO::v3_1::ImageBuf&, const OpenImageIO::v3_1::ImageBuf&, OpenImageIO::v3_1::string_view, float, OpenImageIO::v3_1::ROI, int)’ is deprecated: prefer the kind that takes keyword args (3.0) [-Wdeprecated-declarations]
  234 |     oiio::ImageBufAlgo::resize(outBuf, inBuf, filter, filterSize, oiio::ROI::All());
      |     ~~~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
In file included from /usr/local/include/OpenImageIO/imagebufalgo_util.h:11,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.hpp:8,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:7:
/usr/local/include/OpenImageIO/imagebufalgo.h:2758:13: note: declared here
 2758 | inline bool resize (ImageBuf &dst, const ImageBuf &src,
      |             ^~~~~~
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp: In instantiation of ‘void aliceVision::imageAlgo::resizeImage(OpenImageIO::v3_1::TypeDesc, int, int, int, int, int, const T*, T*, const string&, float) [with T = aliceVision::image::Rgba<unsigned char>; std::string = std::__cxx11::basic_string<char>]’:
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:366:16:   required from here
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:234:31: warning: ‘bool OpenImageIO::v3_1::ImageBufAlgo::resize(OpenImageIO::v3_1::ImageBuf&, const OpenImageIO::v3_1::ImageBuf&, OpenImageIO::v3_1::string_view, float, OpenImageIO::v3_1::ROI, int)’ is deprecated: prefer the kind that takes keyword args (3.0) [-Wdeprecated-declarations]
  234 |     oiio::ImageBufAlgo::resize(outBuf, inBuf, filter, filterSize, oiio::ROI::All());
      |     ~~~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
In file included from /usr/local/include/OpenImageIO/imagebufalgo_util.h:11,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.hpp:8,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:7:
/usr/local/include/OpenImageIO/imagebufalgo.h:2758:13: note: declared here
 2758 | inline bool resize (ImageBuf &dst, const ImageBuf &src,
      |             ^~~~~~
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp: In instantiation of ‘void aliceVision::imageAlgo::resizeImage(OpenImageIO::v3_1::TypeDesc, int, int, int, int, int, const T*, T*, const string&, float) [with T = aliceVision::image::Rgba<float>; std::string = std::__cxx11::basic_string<char>]’:
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:397:16:   required from here
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:234:31: warning: ‘bool OpenImageIO::v3_1::ImageBufAlgo::resize(OpenImageIO::v3_1::ImageBuf&, const OpenImageIO::v3_1::ImageBuf&, OpenImageIO::v3_1::string_view, float, OpenImageIO::v3_1::ROI, int)’ is deprecated: prefer the kind that takes keyword args (3.0) [-Wdeprecated-declarations]
  234 |     oiio::ImageBufAlgo::resize(outBuf, inBuf, filter, filterSize, oiio::ROI::All());
      |     ~~~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
In file included from /usr/local/include/OpenImageIO/imagebufalgo_util.h:11,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.hpp:8,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:7:
/usr/local/include/OpenImageIO/imagebufalgo.h:2758:13: note: declared here
 2758 | inline bool resize (ImageBuf &dst, const ImageBuf &src,
      |             ^~~~~~
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp: In instantiation of ‘void aliceVision::imageAlgo::resizeImage(OpenImageIO::v3_1::TypeDesc, int, int, int, int, int, const T*, T*, const string&, float) [with T = unsigned int; std::string = std::__cxx11::basic_string<char>]’:
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:427:16:   required from here
/root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:234:31: warning: ‘bool OpenImageIO::v3_1::ImageBufAlgo::resize(OpenImageIO::v3_1::ImageBuf&, const OpenImageIO::v3_1::ImageBuf&, OpenImageIO::v3_1::string_view, float, OpenImageIO::v3_1::ROI, int)’ is deprecated: prefer the kind that takes keyword args (3.0) [-Wdeprecated-declarations]
  234 |     oiio::ImageBufAlgo::resize(outBuf, inBuf, filter, filterSize, oiio::ROI::All());
      |     ~~~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
In file included from /usr/local/include/OpenImageIO/imagebufalgo_util.h:11,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.hpp:8,
                 from /root/git_src/AliceVision/src/aliceVision/image/imageAlgo.cpp:7:
/usr/local/include/OpenImageIO/imagebufalgo.h:2758:13: note: declared here
 2758 | inline bool resize (ImageBuf &dst, const ImageBuf &src,
      |             ^~~~~~
[ 12%] Linking CXX shared library ../../../../Linux-x86_64/libaliceVision_lInftyComputerVision.so
/root/git_src/AliceVision/src/aliceVision/image/dcp.cpp: In function ‘int aliceVision::image::get4(FILE*, aliceVision::image::Endianness)’:
/root/git_src/AliceVision/src/aliceVision/image/dcp.cpp:356:10: warning: ignoring return value of ‘size_t fread(void*, size_t, size_t, FILE*)’ declared with attribute ‘warn_unused_result’ [-Wunused-result]
  356 |     fread(str, 1, 4, f);
      |     ~~~~~^~~~~~~~~~~~~~
/root/git_src/AliceVision/src/aliceVision/image/dcp.cpp: In member function ‘void aliceVision::image::DCPProfile::Load(const string&)’:
/root/git_src/AliceVision/src/aliceVision/image/dcp.cpp:919:10: warning: ignoring return value of ‘size_t fread(void*, size_t, size_t, FILE*)’ declared with attribute ‘warn_unused_result’ [-Wunused-result]
  919 |     fread(&bo, 1, 2, file);
      |     ~~~~~^~~~~~~~~~~~~~~~~
/root/git_src/AliceVision/src/aliceVision/image/dcp.cpp: In function ‘short unsigned int aliceVision::image::get2(FILE*, aliceVision::image::Endianness)’:
/root/git_src/AliceVision/src/aliceVision/image/dcp.cpp:350:10: warning: ignoring return value of ‘size_t fread(void*, size_t, size_t, FILE*)’ declared with attribute ‘warn_unused_result’ [-Wunused-result]
  350 |     fread(str, 1, 2, f);
      |     ~~~~~^~~~~~~~~~~~~~
/usr/bin/ld: /usr/local/lib/libOsi.a(OsiPresolve.cpp.o): warning: relocation against `_ZTI17make_fixed_action' in read-only section `.text'
/usr/bin/ld: /usr/local/lib/libClp.a(ClpModel.cpp.o): relocation R_X86_64_PC32 against symbol `_ZTI15ClpPackedMatrix' can not be used when making a shared object; recompile with -fPIC
/usr/bin/ld: final link failed: bad value
collect2: error: ld returned 1 exit status
make[2]: *** [src/aliceVision/linearProgramming/lInfinityCV/CMakeFiles/aliceVision_lInftyComputerVision.dir/build.make:115: Linux-x86_64/libaliceVision_lInftyComputerVision.so.3.4] Error 1
make[1]: *** [CMakeFiles/Makefile2:2657: src/aliceVision/linearProgramming/lInfinityCV/CMakeFiles/aliceVision_lInftyComputerVision.dir/all] Error 2
make[1]: *** Waiting for unfinished jobs....
[ 12%] Linking CXX executable ../../../Linux-x86_64/aliceVision_mergeMeshes
[ 12%] Built target aliceVision_mergeMeshes_exe
[ 13%] Linking CXX shared library ../../../Linux-x86_64/libaliceVision_image.so
[ 13%] Built target aliceVision_image
[ 13%] Linking CXX executable ../../../Linux-x86_64/MeshSDFilter
/usr/bin/c++  -Werror=return-type -Werror=switch -Werror=return-local-addr -faligned-new -Wall -Wextra -O3 CMakeFiles/MeshSDFilter.dir/MeshSDFilter.cpp.o -o ../../../Linux-x86_64/MeshSDFilter  -Wl,-rpath,/usr/local/lib /usr/local/lib/libOpenMeshCore.so.11.0 /usr/lib/gcc/x86_64-linux-gnu/11/libgomp.so /usr/lib/x86_64-linux-gnu/libpthread.a
[ 13%] Built target MeshSDFilter
[ 13%] Linking CXX executable ../../../Linux-x86_64/MeshDenoiser
/usr/bin/c++  -Werror=return-type -Werror=switch -Werror=return-local-addr -faligned-new -Wall -Wextra -O3 CMakeFiles/MeshDenoiser.dir/MeshDenoiser.cpp.o -o ../../../Linux-x86_64/MeshDenoiser  -Wl,-rpath,/usr/local/lib /usr/local/lib/libOpenMeshCore.so.11.0 /usr/lib/gcc/x86_64-linux-gnu/11/libgomp.so /usr/lib/x86_64-linux-gnu/libpthread.a
[ 13%] Built target MeshDenoiser
[ 13%] Linking CXX shared library ../../../Linux-x86_64/libaliceVision_geometry.so
[ 13%] Built target aliceVision_geometry
make: *** [Makefile:136: all] Error 2

解决方法：精准编译每个模块

```

```
编译aliceVision_imageMatching_exe 出错
编译make aliceVision_imageMatching_exe -j$(nproc)
出现错误： In file included from /root/git_src/AliceVision/src/aliceVision/dataio/E57Reader.cpp:7: /root/git_src/AliceVision/src/aliceVision/dataio/E57Reader.hpp:13:10: fatal error: E57SimpleData.h: No such file or directory 13 | #include <E57SimpleData.h> | ^~~~~~~~~~~~~~~~~ compilation terminated. make[3]: *** [src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/build.make:163: src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/E57Reader.cpp.o] Error 1 make[3]: *** Waiting for unfinished jobs.... In file included from /root/git_src/AliceVision/src/aliceVision/sfm/bundle/costfunctions/constraint2d.hpp:15, from /root/git_src/AliceVision/src/aliceVision/sfm/bundle/BundleAdjustmentCeres.cpp:12: /root/git_src/AliceVision/src/aliceVision/sfm/bundle/costfunctions/dynamic_cost_function_to_functor.h:41:10: fatal error: ceres/internal/export.h: No such file or directory 41 | #include <ceres/internal/export.h> | ^~~~~~~~~~~~~~~~~~~~~~~~~ compilation terminated.

（1）E57 是 E57 point cloud format，用于 激光点云 描仪数据 MVS 输入，可以关闭
cmake .. -DALICEVISION_USE_E57=OFF 或 cmake .. -DALICEVISION_USE_LAS=OFF

（2）Ceres 错误（关键问题）
```

```
安装AliceVision官方lemon库
git@github.com:alicevision/lemon.git
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local -DCMAKE_BUILD_TYPE=Release
make -j4
make install

```




### 1.4 各模块单独编译


```
1. 编译模块 featureExtraction
root@lyinc:~/git_src/AliceVision/build# make help | grep -i feature
make aliceVision_featureExtraction_exe -j$(nproc)
编译后的文件在：
/root/git_src/AliceVision/build/Linux-x86_64/aliceVision_featureExtraction

```

![image-20260601181036859](C:\Users\31408\AppData\Roaming\Typora\typora-user-images\image-20260601181036859.png)



```
2. 编译图片处理模块
make aliceVision_cameraInit_exe -j$(nproc)
[100%] Building CXX object src/software/pipeline/CMakeFiles/aliceVision_cameraInit_exe.dir/main_cameraInit.cpp.o
[100%] Linking CXX executable ../../../Linux-x86_64/aliceVision_cameraInit
[100%] Built target aliceVision_cameraInit_exe
```

```
3. 编译image matching 模块
make aliceVision_imageMatching_exe -j$(nproc)
遇到问题：
[ 70%] Building CXX object src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/E57Reader.cpp.o In file included from /root/git_src/AliceVision/src/aliceVision/dataio/E57Reader.cpp:7: /root/git_src/AliceVision/src/aliceVision/dataio/E57Reader.hpp:13:10: fatal error: E57SimpleData.h: No such file or directory 13 | #include <E57SimpleData.h> | ^~~~~~~~~~~~~~~~~ compilation terminated. make[3]: *** [src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/build.make:163: src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/E57Reader.cpp.o] Error 1 make[3]: *** Waiting for unfinished jobs.... [ 70%] Linking CXX shared library ../../../../Linux-x86_64/libaliceVision_lInftyComputerVision.so /usr/bin/ld: /usr/local/lib/libCoinUtils.a(CoinStructuredModel.cpp.o): warning: relocation against _ZTI13CoinBaseModel' in read-only section .text' /usr/bin/ld: /usr/local/lib/libCoinUtils.a(CoinConflictGraph.cpp.o): relocation R_X86_64_PC32 against symbol _ZTV17CoinConflictGraph' can not be used when making a shared object; recompile with -fPIC /usr/bin/ld: final link failed: bad value collect2: error: ld returned 1 exit status make[3]: *** [src/aliceVision/linearProgramming/lInfinityCV/CMakeFiles/aliceVision_lInftyComputerVision.dir/build.make:115: Linux-x86_64/libaliceVision_lInftyComputerVision.so.3.4] Error 1 make[2]: *** [CMakeFiles/Makefile2:2657: src/aliceVision/linearProgramming/lInfinityCV/CMakeFiles/aliceVision_lInftyComputerVision.dir/all] Error 2 make[2]: *** Waiting for unfinished jobs.... [ 70%] Linking CXX shared library ../../../Linux-x86_64/libaliceVision_voctree.so [ 70%] Built target aliceVision_voctree make[2]: *** [CMakeFiles/Makefile2:2366: src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/all] Error 2 [ 70%] Linking CXX shared library ../../../Linux-x86_64/libaliceVision_sfm_bundle.so [ 70%] Built target aliceVision_sfm_bundle [ 70%] Linking CXX shared library ../../../Linux-x86_64/libaliceVision_matching.so [ 70%] Built target aliceVision_matching make[1]: *** [CMakeFiles/Makefile2:3878: src/software/pipeline/CMakeFiles/aliceVision_imageMatching_exe.dir/rule] Error 2 make: *** [Makefile:871: aliceVision_imageMatching_exe] Error 2

分析：
开始以为是 E57，将E57关闭后依然出现，细看才发现原来是 libCoinUtils， 才知道：重新编译了Osi 和 Clp，没有重新编译CoinUtils，重新编译CoinUtil后已经到达81%
[ 81%] Building CXX object src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/VideoFeed.cpp.o In file included from /root/git_src/AliceVision/src/aliceVision/dataio/E57Reader.cpp:7: /root/git_src/AliceVision/src/aliceVision/dataio/E57Reader.hpp:13:10: fatal error: E57SimpleData.h: No such file or directory 13 | #include <E57SimpleData.h> | ^~~~~~~~~~~~~~~~~ compilation terminated. make[3]: *** [src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/build.make:163: src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/E57Reader.cpp.o] Error 1 make[3]: *** Waiting for unfinished jobs.... make[2]: *** [CMakeFiles/Makefile2:2366: src/aliceVision/dataio/CMakeFiles/aliceVision_dataio.dir/all] Error 2 make[2]: *** Waiting for unfinished jobs.... [ 81%] Linking CXX shared library ../../../Linux-x86_64/libaliceVision_matchingImageCollection.so [ 81%] Built target aliceVision_matchingImageCollection make[1]: *** [CMakeFiles/Makefile2:3878: src/software/pipeline/CMakeFiles/aliceVision_imageMatching_exe.dir/rule] Error 2 make: *** [Makefile:871: aliceVision_imageMatching_exe] Error 2

逃不过去，下载E57进行编译
git clone https://github.com/asmaloney/libE57Format.git
cmake -B E57-build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=E57-install libE57Format
cmake --build E57-build --parallel
cmake --install E57-build

(base) root@lyinc:~/git_src/AliceVision/build# sed -n '1,50p' ~/git_src/AliceVision/src/aliceVision/dataio/CMakeLists.txt
# Headers
set(dataio_files_headers
    FeedProvider.hpp
    IFeed.hpp
    ImageFeed.hpp
    SfMDataFeed.hpp
    json.hpp
)

# Sources
set(dataio_files_sources
    FeedProvider.cpp
    IFeed.cpp
    ImageFeed.cpp
    SfMDataFeed.cpp
    json.cpp
)

if (ALICEVISION_HAVE_OPENCV)
    list(APPEND dataio_files_headers VideoFeed.hpp)
    list(APPEND dataio_files_sources VideoFeed.cpp)
endif()

if (NOT ALICEVISION_BUILD_LIDAR STREQUAL "OFF")
    list(APPEND dataio_files_headers E57Reader.hpp)
    list(APPEND dataio_files_sources E57Reader.cpp)
endif()

alicevision_add_library(aliceVision_dataio
    SOURCES ${dataio_files_headers} ${dataio_files_sources}
    PUBLIC_LINKS
        aliceVision_camera
        aliceVision_image
    PRIVATE_LINKS
        aliceVision_sfmData
        aliceVision_sfmDataIO
        aliceVision_system
        Boost::boost
        Boost::json
)

if (ALICEVISION_HAVE_OPENCV)
    target_link_libraries(aliceVision_dataio PRIVATE ${OpenCV_LIBS})
endif()

if (NOT ALICEVISION_BUILD_LIDAR STREQUAL "OFF")
    target_link_libraries(aliceVision_dataio PRIVATE E57Format)
endif()
(base) root@lyinc:~/git_src/AliceVision/build#
```



<h3>1.2 主要流程
</h3>

 Boost  -> FLANN -> COIN-OR -> LZ4 -> Alembic



### 1.3 安装系统依赖

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    git \
    libpng-dev \
    libjpeg-dev \
    libtiff-dev \
    libxxf86vm1 \
    libxxf86vm-dev \
    libxi-dev \
    libxrandr-dev \
    libgraphviz-dev \
    graphviz \
    autoconf \
    automake \
    libtool \
    nasm \
    pkg-config
```

> **说明**：
> - `autoreconf` / `libtool` / `nasm`：编译 libturbo-jpeg 所需
> - `automake`：编译 libpng 所需
> - `graphviz`：查看图结构 SVG 日志（可选）

## 二、推荐方案：嵌入式依赖编译

> AliceVision 官方推荐 Linux 使用嵌入式依赖模式（`ALICEVISION_BUILD_DEPENDENCIES=ON`），CMake 会自动下载并编译所有依赖，无需手动逐个安装。

> 

### 2.2 执行 CMake 配置

```bash
mkdir build && cd build

cmake -DALICEVISION_BUILD_DEPENDENCIES=ON \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=$PWD/../install \
      -DALICEVISION_USE_OPENMP=ON \
      ..
```

**关键参数说明**：

| 参数 | 值 | 说明 |
|------|-----|------|
| `ALICEVISION_BUILD_DEPENDENCIES` | `ON` | 自动下载编译所有依赖 |
| `CMAKE_BUILD_TYPE` | `Release` | 发布版本，含优化 |
| `CMAKE_INSTALL_PREFIX` | 指定路径 | 安装目录（建议用绝对路径） |
| `ALICEVISION_USE_OPENMP` | `ON`（默认） | 多线程加速，性能影响巨大 |

### 2.3 编译

```bash
make -j$(nproc)
```

> `-j$(nproc)` 使用全部 CPU 核心并行编译。
> 如果内存不足或遇到编译器崩溃，可减少并行数：`make -j4`。

### 2.4 安装

```bash
make install
```

安装完成后，二进制文件位于 `~/git_src/AliceVision/install/bin/`。

### 2.5 设置环境变量

```bash
export ALICEVISION_ROOT=$HOME/git_src/AliceVision/install
echo 'export ALICEVISION_ROOT=$HOME/git_src/AliceVision/install' >> ~/.bashrc
export PATH=$ALICEVISION_ROOT/bin:$PATH
echo 'export PATH=$ALICEVISION_ROOT/bin:$PATH' >> ~/.bashrc
```

---

## 三、方案二：外部依赖编译（手动控制依赖版本）

如果需要使用系统已安装的特定版本依赖，或需要精细控制每个依赖的版本，可使用外部依赖模式。

### 3.1 完整依赖列表

| 依赖 | 最低版本 | 来源 |
|------|---------|------|
| Assimp | 5.0.0 | [GitHub](https://github.com/assimp/assimp) |
| Boost | 1.74.0 | [boost.org](https://www.boost.org) |
| Ceres Solver | 1.10.0 | [GitHub](https://github.com/ceres-solver/ceres-solver) |
| CoinUtils | 2.9.3 | [AliceVision fork](https://github.com/alicevision/CoinUtils) |
| Clp | — | [AliceVision fork](https://github.com/alicevision/Clp) |
| Eigen | 3.3.4 | [GitLab](https://gitlab.com/libeigen/eigen) |
| Expat | 2.4.8 | [libexpat](https://libexpat.github.io/) |
| Flann | 1.8.4 | [AliceVision fork](https://github.com/alicevision/flann) |
| Geogram | 1.7.5（推荐 1.8.8+） | [GitHub](https://github.com/BrunoLevy/geogram) |
| nanoflann | 1.5.4 | [GitHub](https://github.com/jlblancoc/nanoflann) |
| OpenEXR | 2.5 | [GitHub](https://github.com/AcademySoftwareFoundation/openexr) |
| OpenImageIO | 3.0.0 | [GitHub](https://github.com/OpenImageIO/oiio) |
| OpenMesh | 9.0 | [官网](https://www.graphics.rwth-aachen.de/software/openmesh/) |
| Osi | 0.106.10 | [AliceVision fork](https://github.com/alicevision/Osi) |
| zlib | — | 系统自带 |

### 3.2 可选依赖

| 依赖 | 用途 | CMake 开关 |
|------|------|------------|
| Alembic | 数据 I/O | `ALICEVISION_USE_ALEMBIC` |
| CCTag | 特征提取/匹配/定位 | `ALICEVISION_USE_CCTAG` |
| CUDA 11.0+ | GPU 加速（特征提取 + 深度图） | `ALICEVISION_USE_CUDA` |
| OpenCV 3.4.11+ | 特征提取、标定、视频 I/O | `ALICEVISION_USE_OPENCV` |
| PopSift | GPU SIFT | `ALICEVISION_USE_POPSIFT` |
| PCL 1.12.1+ | 点云配准 | — |
| Mosek 6+ | 线性规划 | — |
| Python3 + NumPy | Python 绑定 | `ALICEVISION_BUILD_SWIG_BINDING` |

### 3.3 外部依赖编译示例

```bash
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=$PWD/../install \
      -DALICEVISION_BUILD_DEPENDENCIES=OFF \
      -DCeres_DIR=/path/to/ceres/install/share/Ceres/ \
      -DFLANN_INCLUDE_DIR_HINTS=/path/to/flann/include/ \
      -DOPENIMAGEIO_LIBRARY_DIR_HINTS=/path/to/oiio/lib/ \
      -DOPENIMAGEIO_INCLUDE_DIR=/path/to/oiio/include/ \
      ..
```

---

## 四、CMake 编译选项速查

### 模块开关

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `ALICEVISION_BUILD_SFM` | ON | Structure from Motion |
| `ALICEVISION_BUILD_MVS` | ON | Multi-View Stereo |
| `ALICEVISION_BUILD_HDR` | ON | HDR 处理 |
| `ALICEVISION_BUILD_SEGMENTATION` | ON | ONNX 分割 |
| `ALICEVISION_BUILD_PANORAMA` | ON | 全景拼接 |
| `ALICEVISION_BUILD_PHOTOMETRICSTEREO` | ON | 光度立体 |
| `ALICEVISION_BUILD_LIDAR` | AUTO | LiDAR 支持 |
| `ALICEVISION_BUILD_TESTS` | OFF | 单元测试 |
| `ALICEVISION_BUILD_DOC` | AUTO | 文档生成 |
| `ALICEVISION_BUILD_SWIG_BINDING` | OFF | Python 绑定 |

### 功能开关

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `ALICEVISION_USE_OPENMP` | ON | 多线程加速 |
| `ALICEVISION_USE_CUDA` | ON | GPU 加速 |
| `ALICEVISION_USE_OPENCV` | OFF | OpenCV 集成 |
| `ALICEVISION_USE_CCTAG` | AUTO | CCTag 标记 |
| `ALICEVISION_USE_ALEMBIC` | AUTO | Alembic 文件格式 |
| `ALICEVISION_USE_POPSIFT` | AUTO | GPU SIFT |
| `BUILD_SHARED_LIBS` | ON | 构建动态库 |
| `ALICEVISION_REQUIRE_CERES_WITH_SUITESPARSE` | ON | Ceres + SuiteSparse |

---

## 五、验证安装

```bash
# 查看安装的二进制工具
ls ~/git_src/AliceVision/install/bin/

# 查看版本（示例）
~/git_src/AliceVision/install/bin/aliceVision_cameraInit --help
```

---

## 六、配合 Meshroom 使用（可选）

AliceVision 通常与 [Meshroom](https://github.com/alicevision/Meshroom) 配合使用进行三维重建。编译时建议开启以下选项：

```bash
cmake ... \
    -DALICEVISION_USE_OPENCV=ON \
    -DALICEVISION_BUILD_SWIG_BINDING=ON \
    -DALICEVISION_USE_POPSIFT=ON \
    -DALICEVISION_USE_CCTAG=ON \
    -DALICEVISION_INSTALL_MESHROOM_PLUGIN=ON
```

---

## 七、常见问题

### Q1: 内存不足导致编译失败

减少并行编译数：
```bash
make -j2
```

或增加 swap：
```bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Q2: CMake 找不到依赖

检查 CMake 输出末端的依赖报告，确认每个库的版本和来源（internal/external）。

### Q3: 网络问题导致依赖下载失败

确保 WSL 代理已正确配置：
```bash
# 确认镜像网络模式生效
ip addr show | grep 192.168

# 确认代理可达
curl -x http://127.0.0.1:7897 https://github.com -o /dev/null -s -w "%{http_code}"
```

### Q4: 编译时间

嵌入式依赖模式首次编译约需 **1–3 小时**（取决于 CPU 核心数和网络速度），建议使用 `make -j$(nproc)` 充分利用多核。

---

> 参考：[AliceVision INSTALL.md](https://github.com/alicevision/AliceVision/blob/develop/INSTALL.md)
