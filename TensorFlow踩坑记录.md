<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# TensorFlow踩坑记录

## 前言

本文从编译、安装和使用等方面叙述`TensorFlow`可能出现的问题及解决方法。TensorFlow版本为`1.4.1`，操作系统为`Ubuntu 16.04`，不保证文中出现的问题在其它版本或系统也会出现，更不保证文中解决方法对于其它版本或系统也有效，全文内容仅供参考。另，文中的问题针对C++，以后若有需要可能会加入Python和Java等语言。

建议将本文与<a href="Ubuntu_16.04安装C++版TensorFlow_1.4.1.md">《Ubuntu 16.04 安装 C++ 版 TensorFlow 1.4.1》</a>搭配阅读。

## 编译之坑

### bazel编译器

1. **版本必须选对**，过高或高低都不行。对应TensorFlow `1.4.1`，对应的bazel版本是`0.5.4`；`1.6.0`则对应`0.9.0`；等等。

2. **语法规则**很怪异，增加学习成本，且目前还很不稳定，换台机器分分钟就编译出错，关键是你还不知道怎么改。有些必要的命令，例如`install`、`uninstall`等，貌似没有提供，编译好的库和头文件，需要手动拷贝。

3. **编译行为**很奇葩，缓存很多乱七八糟的东西在用户家目录内，要找点什么就得掘地三尺。

### 纯CPU（即cpu-only）也依赖CUDA库

所以，还是老老实实先把CUDA库装上吧，不然会编译报错。这个坑倒是情有可原，要生成不同种类（指纯CPU还是有GPU参与）的库，可通过设置宏开关来实现，但代码数量一多，就难保某个角落没正确加上宏开关，编译报错就so easy啦。这跟写跨平台代码有点类似，体验过的人都知道有多操蛋。

### protobuf的链接报错

这个问题比较蛋疼，一种原因是已安装的`protobuf`版本并非`TensorFlow`所需，加装或重装即可解决。而另一种原因就是，如果你在编译`TensorFlow`之前先安装了`Caffe`（另一个深度学习框架）并且这货生成的一个`libproto.a`版本（实际上也是protobuf库）与`TensorFlow`所需的`protobuf`版本不一样时，也会报错。这种情况的解法办法也很简单，将`libproto.a`移到另一个地方再继续编译`TensorFlow`就行了。至于后面还要不要移回来，就随你喜欢了，貌似这个`libproto.a`没有也不会造成什么影响，应该是需要的东西都编译进`Caffe`库了。对了，以上两种报错的现象都是差不多的，会打印`undefined symbol **google**protobuf***`，如果用工具查看相应的库，要么找不到函数符号，要么找到的符号没有地址，例如：

```
$ nm libproto.a | grep _ZNK6google8protobuf7Message11GetTypeNameB5cxx11Ev
                 U _ZNK6google8protobuf7Message11GetTypeNameB5cxx11Ev
```

而正常的库应该类似这样的：

```
$ nm libprotobuf.so | grep _ZNK6google8protobuf7Message11GetTypeNameB5cxx11Ev
000000000016c020 T _ZNK6google8protobuf7Message11GetTypeNameB5cxx11Ev
```

值得一提的是，有些经过编译器优化的库用`nm`命令是查看不了的，这时可尝试使用`readelf`命令并加上`-a`选项。

最后，对于各种库的冲突，只能说林子大了，什么鸟都有，免不了要掐架，习惯就好……

## 安装之坑

就是编译好的库文件和生成的头文件，要手动拷贝到系统目录。库文件倒也罢，头文件可不好拷，数量多，附带的垃圾也多，让人没有精力一一清除。这事得怪`bazel`编译器，不带`install`命令的编译器，跟咸鱼有啥区别？

## 使用之坑

重点来了！如果说前面的编译和安装相当于餐厅收费贵、规矩多，但饭菜味道好（即使用方便，功能性能满足要求等），还是可以忍受的。然而现实却是，餐厅给你上了一道N多骨刺的鱼，稍不留心就会刺到喉咙。没办法，餐厅老板`Google`作为大佬，就是这么霸气侧漏、任性不羁。

### 日志格式

TensorFlow不使用独立的`glog`，而是内嵌在其中，并且还改了部分代码，其中就包括了日志格式，最神奇的是去掉了`线程ID`的打印，这就让人呵呵了……如果你的项目同时使用`TensorFlow`和`glog`，日志格式改变是小事，运行崩溃也不是不可能。但如果发现得早并采取措施，程序不但能毫发无损，日志格式也不用变。关子就不多卖了，下面奉上解决方案：

```
#include "TensorFlow的头文件"

#undef LOG

#include "glog的头文件"
```

重点就两个：（1）glog头文件要在TensorFlow之后；（2）需要用`#undef`将TensorFlow日志接口屏蔽掉。

### protobuf的问题

如果你只认为protobuf的问题只出现的TensorFlow的编译阶段，那就too young too simple了！如果你的系统存在多个版本的protobuf（包括TensorFlow使用的这个）并且它们的头文件和库文件路径都暴露出来，那么编译冲突和链接冲突分分钟会发生。怎么办呢？与TensorFlow用的protobuf版本保持一致吧，将其它依赖protobuf的库和程序重新编译一遍。原因在前面已经说过了，TensorFlow巨复杂，你想改也改不了，只能改别的容易改的程序。

### jpeg库（坑中之坑）

TensorFlow将jpeg库内嵌于其中，如果你的项目同时用了另一个版本的jpeg库以及TensorFlow库，那么程序在运行的时候可能会调用TensorFlow库里的jpeg函数，而部分jpeg函数会检查版本号，发现版本号不一致就直接罢工。这个问题不是那么容易发现，得用`GDB`去调试，层层跟进，而且生产环境的库和程序多采用`-O3`进行优化，把源码和符号给优化掉，得下载一份目标库对应版本的源码，加上`-O0 -g`编译出调试版本来进行调试，期间免不了要鼓捣`GNU configure`脚本、`CMakeLists.txt`、`Makefile`之类的东东，有更多的意外发生也不奇怪，时间上的耗费是跑不掉的。下面结合实际项目（关键信息已隐去），说说大致的的GDB调试和问题解决步骤：

1. **重新编译jpeg库及依赖它的库**（本项目用到了OpenCV和Caffe，Caffe不用重编)，并安装到单独的目录内（本文选择安装到用户家目录）：

```
# 分别到libjpeg和OpenCV的GitHub下载对应版本的源码包，放在$HOME/src目录下，
# 即：libjpeg-turbo-1.5.1.tar.gz和opencv-2.4.9.zip
# GitHub网址分别为：
# https://github.com/libjpeg-turbo/libjpeg-turbo.git
# https://github.com/opencv/opencv.git

# --------------------------------------------------
# 安装libjpeg
# --------------------------------------------------

cd $HOME/src
export CFLAGS="-DGCC_HASCLASSVISIBILITY -g -ggdb -O0 -Wall -W"
tar -zxvf libjpeg-turbo-1.5.1.tar.gz
cd libjpeg-turbo-1.5.1/
autoreconf -fiv
./configure --prefix=$HOME
make
make install

# --------------------------------------------------
# 安装OpenCV
# --------------------------------------------------

cd $HOME/src
unzip opencv-2.4.9.zip
cd opencv-2.4.9/
mkdir .build/
cd .build/
cmake -DCMAKE_BUILD_TYPE=Debug -DWITH_CUDA=OFF -DCMAKE_INSTALL_PREFIX=$HOME -DCMAKE_INCLUDE_PATH=$HOME/include -DCMAKE_LIBRARY_PATH=$HOME/lib ..
make
make install
```

事实上，在用GDB调试找出问题之前，不会知道jpeg库用的是哪个版本，只能根据系统已安装的jpeg共享库（例如/usr/lib/x86_64-linux-gnu/libjpeg.so.8.0.2）的版本后缀猜一个版本（例如libjpeg-turbo-jpeg-8、libjpeg-turbo-jpeg-8a、libjpeg-turbo-jpeg-8b等）。通过后续步骤找出准确版本号后，才能下载正确版本的压缩包重新编译安装，为了减少撰写的麻烦，以上直接只列出最终版`1.5.1`的编译和安装。

2. 通过**GDB调试**找出问题所在：

```
$ gdb ./predict_porn 
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./predict_porn...done.
(gdb) break 33
Breakpoint 1 at 0x423f22: file /home/foo/git/lvtu/03_biz/predict_server/src/main.cpp, line 33.
(gdb) run -running_mode=local
Starting program: /home/foo/git/lvtu/03_biz/predict_server/predict_porn -running_mode=local

[省略一系列日志内容……]

Thread 1 "predict_porn" hit Breakpoint 1, local_test_template (classifier=...) at /home/foo/git/lvtu/03_biz/predict_server/src/main.cpp:33
33				std::vector<lvtu::Classifier::prediction_t>& predictions = classifier.classify(line);
(gdb) s
lvtu::Classifier::classify (this=0x7fffffffd990, img="./input/cat.jpg", dimensions=5) at /home/foo/git/lvtu/03_biz/predict_server/src/classifier.cpp:34
34		predict(img, _predictions);
(gdb) s
lvtu::CaffeClassifier::predict (this=0x7fffffffd990, img="./input/cat.jpg", result=std::vector of length 0, capacity 0, result_count=5) at /home/foo/git/lvtu/03_biz/predict_server/src/caffe_classifier.cpp:102
102	{
(gdb) l
97	
98		return 0;
99	}
100	
101	int CaffeClassifier::predict(const std::string& img, std::vector<Classifier::prediction_t> &result, int result_count/* = DEFAULT_PREDICTION_COUNT*/) /*override*/
102	{
103		result.clear();
104	
105		cv::Mat img_mat(std::move(cv::imread(img, -1)));
106	
(gdb) n
103		result.clear();
(gdb) n
105		cv::Mat img_mat(std::move(cv::imread(img, -1)));
(gdb) s
cv::imread (filename=<error: Cannot access memory at address 0x9d>, flags=32767) at /home/foo/src/opencv-2.4.9/modules/highgui/src/loadsave.cpp:259
259	{
(gdb) l
254	    return hdrtype == LOAD_CVMAT ? (void*)matrix :
255	        hdrtype == LOAD_IMAGE ? (void*)image : (void*)mat;
256	}
257	
258	Mat imread( const string& filename, int flags )
259	{
260	    Mat img;
261	    imread_( filename, flags, LOAD_MAT, &img );
262	    return img;
263	}
(gdb) n
260	    Mat img;
(gdb) n
261	    imread_( filename, flags, LOAD_MAT, &img );
(gdb) s
cv::imread_ (filename=<error: Cannot access memory at address 0xc0012000033d3>, flags=0, hdrtype=0, mat=0x0) at /home/foo/src/opencv-2.4.9/modules/highgui/src/loadsave.cpp:197
197	{
(gdb) l
192	
193	enum { LOAD_CVMAT=0, LOAD_IMAGE=1, LOAD_MAT=2 };
194	
195	static void*
196	imread_( const string& filename, int flags, int hdrtype, Mat* mat=0 )
197	{
198	    IplImage* image = 0;
199	    CvMat *matrix = 0;
200	    Mat temp, *data = &temp;
201	
(gdb) n
198	    IplImage* image = 0;
(gdb) n
199	    CvMat *matrix = 0;
(gdb) n
200	    Mat temp, *data = &temp;
(gdb) n
202	    ImageDecoder decoder = findDecoder(filename);
(gdb) n
203	    if( decoder.empty() )
(gdb) n
205	    decoder->setSource(filename);
(gdb) n
206	    if( !decoder->readHeader() )
(gdb) s
cv::Ptr<cv::BaseImageDecoder>::operator-> (this=0x7fffffffd2c0) at /home/foo/src/opencv-2.4.9/modules/core/include/opencv2/core/operations.hpp:2637
2637	template<typename _Tp> inline _Tp* Ptr<_Tp>::operator -> () { return obj; }
(gdb) s
cv::JpegDecoder::readHeader (this=0x7ffff6e667fa <cv::BaseImageDecoder::setSource(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&)+54>) at /home/foo/src/opencv-2.4.9/modules/highgui/src/grfmt_jpeg.cpp:215
215	{
(gdb) n
216	    bool result = false;
(gdb) n
217	    close();
(gdb) n
219	    JpegState* state = new JpegState;
(gdb) n
220	    m_state = state;
(gdb) n
221	    state->cinfo.err = jpeg_std_error(&state->jerr.pub);
(gdb) n
222	    state->jerr.pub.error_exit = error_exit;
(gdb) n
224	    if( setjmp( state->jerr.setjmp_buffer ) == 0 )
(gdb) n
226	        jpeg_create_decompress( &state->cinfo );
(gdb) s
jpeg_CreateDecompress (cinfo=0x2b7ffbf0, version=80, structsize=656) at jdapimin.c:36
36	  cinfo->mem = NULL;		/* so jpeg_destroy knows mem mgr not called */
(gdb) l
31	jpeg_CreateDecompress (j_decompress_ptr cinfo, int version, size_t structsize)
32	{
33	  int i;
34	
35	  /* Guard against version mismatches between library and caller. */
36	  cinfo->mem = NULL;		/* so jpeg_destroy knows mem mgr not called */
37	  if (version != JPEG_LIB_VERSION)
38	    ERREXIT2(cinfo, JERR_BAD_LIB_VERSION, JPEG_LIB_VERSION, version);
39	  if (structsize != SIZEOF(struct jpeg_decompress_struct))
40	    ERREXIT2(cinfo, JERR_BAD_STRUCT_SIZE, 
(gdb) p version
$1 = 80
(gdb) p JPEG_LIB_VERSION
No symbol "JPEG_LIB_VERSION" in current context.
```

下面划重点了：cv::imread() -> cv::imread_() -> ImageDecoder::readHeader() -> jpeg_create_decompress() -> jpeg_CreateDecompress() -> JPEG_LIB_VERSION，根据这样的调用关系追根溯源，顺藤摸瓜找出了元凶 **JPEG_LIB_VERSION** 及其所在头文件`jpeglib.h`（可利用IDE的代码跳转功能，这里不展开）。需要说明的是，如果使用自己加上`-O0 -g`重新编译的jpeg库，并且屏蔽了TensorFlow的使用，业务程序运行正确，并且GDB调试的时候是能进入jpeg_create_decompress()内部并能打印`version`参数值的，内容如上所示。如果加入了TensorFlow库，则业务程序运行出错，且GDB调试时进不去jpeg_create_decompress()，因为这时的jpeg_create_decompress()是TensorFlow库内部的。调试示例如下：

```
(gdb) n
221	    state->cinfo.err = jpeg_std_error(&state->jerr.pub);
(gdb) n
222	    state->jerr.pub.error_exit = error_exit;
(gdb) n
224	    if( setjmp( state->jerr.setjmp_buffer ) == 0 )
(gdb) n
226	        jpeg_create_decompress( &state->cinfo );
(gdb) s
0x00007fffedfadef2	224	    if( setjmp( state->jerr.setjmp_buffer ) == 0 )
```

3. **找出`TensorFlow`用的jpeg库版本**：

```
# 搜TensorFlow的头文件。TensorFlow的安装细节见文章开头的另一篇文章

$ find ~/include/tensorflow/ -name "*.h" | xargs grep "JPEG_LIB_VERSION" -n
/home/foo/include/tensorflow/bazel-genfiles/external/jpeg/jconfig.h: * Might be useful for tests like "#if JPEG_LIB_VERSION >= 60".
/home/foo/include/tensorflow/bazel-genfiles/external/jpeg/jconfig.h:#define JPEG_LIB_VERSION  62	/* Version 6b */
/home/foo/include/tensorflow/bazel-genfiles/external/jpeg/jconfig_nowin_nosimd.h: * Might be useful for tests like "#if JPEG_LIB_VERSION >= 60".
/home/foo/include/tensorflow/bazel-genfiles/external/jpeg/jconfig_nowin_nosimd.h:#define JPEG_LIB_VERSION  62	/* Version 6b */
/home/foo/include/tensorflow/bazel-genfiles/external/jpeg/jconfig_nowin_simd.h: * Might be useful for tests like "#if JPEG_LIB_VERSION >= 60".
/home/foo/include/tensorflow/bazel-genfiles/external/jpeg/jconfig_nowin_simd.h:#define JPEG_LIB_VERSION  62	/* Version 6b */
/home/foo/include/tensorflow/bazel-genfiles/external/jpeg/jconfig_win.h:#define JPEG_LIB_VERSION 62

# 找出版本号的定义在jconfig.h文件，再打开该文件看更多的详情，发现以下内容：

  1 /* Version ID for the JPEG library.                                                                                                                                                                                                                             
  2  * Might be useful for tests like "#if JPEG_LIB_VERSION >= 60".
  3  */
  4 #define JPEG_LIB_VERSION  62    /* Version 6b */
  5  
  6 /* libjpeg-turbo version */
  7 #define LIBJPEG_TURBO_VERSION 1.5.1

# 竟然还有一个TURBO的版本号。事实证明，就是靠这个版本号到GitHub上下载到正确的源码包的。
```

4. 下载正确版本的jpeg源码包，重新编译安装，并且OpenCV和业务程序最好也重新编译（见步骤1），重新运行，这回能正常运行了。

### png库

其实跟上面的jpeg坑是一样的，只不过这个库厚道一点，直接打印出警告，省却了自己用GDB去调试的麻烦，警告类似如下：

```
libpng warning: Application was compiled with png.h from libpng-1.6.34
libpng warning: Application  is  running with png.c from libpng-1.2.53
```

没啥好说的，重装这个库吧。并且，最好也重新编译依赖于这个库的组件，例如前文提到的`Caffe`(待定 )，
真不是一般的酸爽！

