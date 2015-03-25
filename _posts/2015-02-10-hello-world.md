---
title: 使用 ZLib 压缩/解压 ZIP 文件
---
From http://www.cppblog.com/Streamlet/archive/2010/09/22/127368.aspx

调用 rar.exe, unzip.exe 等
使用某现成库
完全手写
第一种虽然能完成任务，但是没法知晓结果。曾经有人对说，可以抓命令行输出结果来判断……这种依靠界面文字来进行精确判断的行为个人认为相当不靠谱。第三种，既然我是个“造轮主义”者，当然说好，但是现在我不了解 ZIP 格式，也不了解 ZIP 算法，所以这个日后再说。今天我们就来切切实实地用一次轮子。

ZIP 相关的库中比较有名的可能就是 ZLib 和 InfoZip（unzip60）了。InfoZip 我了解的不多，其外层接口似乎也不大好，一堆回调——回调是个很烦人的东西，专门用来打乱代码结构。另外，这个库也已经有好多年没更新了吧，太久的东西给人的感觉总是不太舒服。ZLib 最新版本是 1.2.5，今年 4 月 19 日出的。确切的说，ZLib 可能并不是一个针对 ZIP 文件的库，它只是一个针对 gzip 以及 deflate 算法的库。它提供了一个叫做 minizip (contrib\minizip) 例子来给出操作 ZIP 文件的方法。下文将从 ZLib 出发，归结出两个傻瓜接口：

BOOL ZipCompress(LPCTSTR lpszSourceFiles, LPCTSTR lpszDestFile);
BOOL ZipExtract(LPCTSTR lpszSourceFile, LPCTSTR lpszDestFolder);

要引入的源文件

ZLib 主目录下的代码，除 minigzip.c、example.c 外；
contrib\minizip 下的代码，除 minizip.c、miniunz.c 外。
相关 API

虽然 minizip 更像是个例子，但是除去其主程序 minizip.c 和 miniunz.c 后，剩下的部分我们可以看作是 ZLib 的一个上层库，它封装了与 ZIP 文件格式相关的操作。而 minizip.c 和 miniunz.c 就是我们要改写的——把它从命令行程序改为上述傻瓜接口。minizip.c 和 miniunz.c 中用到的 API 主要有：

压缩相关：

zipOpen64
zipClose
zipOpenNewFileInZip
zipCloseFileInZip
zipWriteInFileInZip
解压相关：

unzOpen64
unzClose
unzGetGlobalInfo64
unzGoToNextFile
unzGetCurrentFileInfo64
unzOpenCurrentFile
unzCloseCurrentFile
unzReadCurrentFile
想必看到这些名字都能猜到怎么用了吧。好的接口果然能带给人愉悦的。minizip 中的这些函数有的是带“64”的有的是不带的，有的还有“2”、“3”、“4”版本。这里一律用带 64 的，不带“2”、“3”、“4”的。

具体操作

下文涉及的所有操作，其相关代码都可以在 http://zlibwrap.codeplex.com/ 上找到（Change Set 2450）。这里就不贴长篇代码了。另外有个 DLL版本 和 Lib版本，供拿来主义者用。

首先是压缩操作。使用 zipOpen64 来打开/创建一个 ZIP 文件，然后开始遍历要被放到压缩包中去的文件。针对每个文件，先调用一次 zipOpenNewFileInZip，然后开始读原始文件数据，使用 zipWriteInFileInZip 来写入到 ZIP 文件中去。zipOpenNewFileInZip 的第三个参数是一个 zip_fileinfo 结构，该结构数据可全部置零，其中 dosDate 可用于填入一个时间（LastModificationTime）。它的第二个参数是 ZIP 中的文件名，若要保持目录结构，该参数中可以保留路径，如 foo/bar.txt。

解压操作稍微复杂一点点。打开一个 ZIP 文件后，需要先使用 unzGetGlobalInfo64 来取得该文件的一些信息，来了解这个压缩包里一共包含了多少个文件，等等。目前我们用得着的就是这个文件数目。然后开始遍历 ZIP 中的文件，初始时自动会定位在第一个文件，以后处理完一个后用 unzGoToNextFile 来跳到下一个文件。对于每个内部文件，可用 unzGetCurrentFileInfo64 来查内部文件名。这个文件名和刚才 zipOpenNewFileInZip 的第二个参数是一样的形式，所以有可能包含路径。也有可能会以路径分隔符（/）结尾，表明这是个目录项（其实压缩操作的时候也可以针对目录写入这样的内部文件，上面没有做）。所以接下来要根据情况创建（多级）目录。unzGetCurrentFileInfo64 的第三个参数是 unz_file_info64 结构，其中也有一项包含了 dosDate 信息，可以还原文件时间。对于非目录的内部文件，用 unzOpenCurrentFile，打开，然后 unzReadCurrentFile 读取文件内容，写入到真实文件中。unzReadCurrentFile 返回 0 表示文件读取结束。

局限性

只能压缩、解压采用 deflate 算法的 ZIP 文件。（不过此类 ZIP 应该占了绝大多数）
由于 minizip 中相关 API 的限制，以及 ZIP 文件格式的限制，被压缩/解压的相关文件名必须与系统的当前代码页相符合。（虽然 ZIP 格式最近一次更新加入了使用 UTF8 编码文件名的选项，但是不能保证所遇到的 ZIP 文件都是新格式的，minizip 中似乎也没有针对此选项做什么动作。）