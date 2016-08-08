---
title: DeviceIoControl 的一个用处
---

## 背景
最近一个月一直在做快速扫描磁盘文件的东西，主要是通过扫描NTFS格式的磁盘中的索引树（B-树），可以得到整个磁盘文件的索引，然后通过遍历其索引树，即可得到目录下有哪些文件。这个需要对NTFS格式的磁盘的内部数据结构有一定的了解，这个可以参考我的另外一篇文章（这里还没有完成，待我完成时发布出来，当然网上也有很多资料）。

预期是希望能够比windows的API（FindFirstFile和FindNextFile）效率要高，但是实际结果却不好，原因是磁盘IO太多了，因为首先找到根目录，然后找到根目录下的文件以及目录，然后再通过目录寻找其目录下的文件以及文件夹，所以需要不断的磁盘IO。

这里我使用的文件映射（CreateFileMapping好像不可以映射磁盘）、异步IO以及windows的完成端口（IOCP），最终的效果都不好。

然后经过老大的提醒，DeviceIOControl这个API也可以进行磁盘操作。查阅MSDN，这个API功能比较强大，功能比较多，发现其可以通过控制码直接和驱动交互，读取磁盘的File Entry。

因为它的效率比较高，最终采用的读取磁盘所有File Entry，而后自建索引树。

## API详细介绍
### API参数
``` bash
BOOL WINAPI DeviceIoControl(
  _In_        HANDLE       hDevice,
  _In_        DWORD        dwIoControlCode,
  _In_opt_    LPVOID       lpInBuffer,
  _In_        DWORD        nInBufferSize,
  _Out_opt_   LPVOID       lpOutBuffer,
  _In_        DWORD        nOutBufferSize,
  _Out_opt_   LPDWORD      lpBytesReturned,
  _Inout_opt_ LPOVERLAPPED lpOverlapped
);
```
参数一：设备句柄
参数二：控制码
参数三：输入缓冲区
参数四：输入缓冲区长度
参数五：输出缓冲区
参数六：输出缓冲区长度
参数七：返回的字节长度
参数八：一个OverLapped结构的指针

### 控制码
关于控制码主要有以下几个主题：
``` bash
Communications Control Codes
Device Management Control Codes
Directory Management Control Codes
Disk Management Control Codes
File Management Control Codes
Power Management Control Codes
Volume Management Control Codes
```
控制码比较多，这里只介绍读取MFT的控制码，其他有兴趣可以参考MSDN。这个控制是`File Management Control Codes`中的 `FSCTL_GET_NTFS_FILE_RECORD`。

这个控制在MSDN中的解释是`Retrieves the first file record that is in use and is of a lesser than or equal ordinal value to the requested file reference number.`主要意思是通过文件参考号（file reference number）获得File Entry。

### 控制码对应的API参数
再看该控制码对应的API参数结构：
``` bash
BOOL DeviceIoControl( (HANDLE) hDevice,              // handle to device
                      FSCTL_GET_NTFS_FILE_RECORD,    // dwIoControlCode
                      (LPVOID) lpInBuffer,           // input buffer
                      (DWORD) nInBufferSize,         // size of input buffer
                      (LPVOID) lpOutBuffer,          // output buffer
                      (DWORD) nOutBufferSize,        // size of output buffer
                      (LPDWORD) lpBytesReturned,     // number of bytes returned
                      (LPOVERLAPPED) lpOverlapped ); // OVERLAPPED structure

```
这里主要注意两个参数：
参数三：NTFS_FILE_RECORD_INPUT_BUFFER 结构指明你要读取的File Entry的编号，例如$MFT的编号为0，那么你就给该参数传入0，需要注意的是LARGER_INTEGER有64位，分为高32位和低32位。
``` bash
typedef struct {
  LARGE_INTEGER FileReferenceNumber;
} NTFS_FILE_RECORD_INPUT_BUFFER, *PNTFS_FILE_RECORD_INPUT_BUFFER;
```

参数五：NTFS_FILE_RECORD_OUTPUT_BUFFER 结构，用来保存读取的数据缓冲区。注意，`FileRecordBuffer`只占一个字节，如果数据没有读完，该API返回相应的提示。一般一个File Entry占2个扇区，一个扇区一般512字节，所以这里我一般将`FileRecordBuffer`改为占用1024字节的数组。
``` bash
typedef struct {
  LARGE_INTEGER FileReferenceNumber;
  DWORD         FileRecordLength;
  BYTE          FileRecordBuffer[1];     //可以改成FileRecordBuffer[1024]
} NTFS_FILE_RECORD_OUTPUT_BUFFER, *PNTFS_FILE_RECORD_OUTPUT_BUFFER;
```
### 相关代码
以上就是利用`DeviceIoControl1`读取`File Entry`时需要注意的地方，下面看下完整代码：
首先打开磁盘，注意CreateFile的各个参数的含义：
``` bash
CString wStrVolum = L"\\\\.\\C:";        //以C盘为例
hVolume = CreateFile(
	wStrVolum,
	GENERIC_READ,
	FILE_SHARE_READ | FILE_SHARE_WRITE,
	0,
	OPEN_EXISTING,
	0,
	0);
```
然后读取File Entry：
``` bash
typedef struct {
  LARGE_INTEGER FileReferenceNumber;
  DWORD         FileRecordLength;
  BYTE          FileRecordBuffer[1024];
} NTFS_FILE_RECORD_OUTPUT_BUFFER, *PNTFS_FILE_RECORD_OUTPUT_BUFFER;

int ReadMFT(FILE_RECORD_HEADER* mftRecord, ULONGLONG mftID)
{
	if (mftRecord == NULL)
		return -1;

	NTFS_FILE_RECORD_INPUT_BUFFER nfrib;
	NTFS_FILE_RECORD_OUTPUT_BUFFER2 nfrob;

	if (hVolume != INVALID_HANDLE_VALUE)
	{
		nfrib.FileReferenceNumber.QuadPart = mftID;
		DWORD bytesReturned = 0;
		BOOL ret = DeviceIoControl(
			hVolume,
			FSCTL_GET_NTFS_FILE_RECORD,
			&nfrib,
			sizeof(NTFS_FILE_RECORD_INPUT_BUFFER),
			&nfrob,
			sizeof(NTFS_FILE_RECORD_OUTPUT_BUFFER2),
			&bytesReturned,
			NULL);

		if (bytesReturned>0)
		{
			memcpy(mftRecord, nfrob.FileRecordBuffer, 1024);
			return -1;
		}
	}
	return 0;
}
```
