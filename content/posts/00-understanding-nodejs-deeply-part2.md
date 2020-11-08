---
title: "《深入浅出 Node.js》学习笔记 Part2"
date: 2019-01-09T16:41:00+08:00
description: "Buffer, 网络编程"
draft: false
tags: [Node]
---

## 第 6 章 理解 Buffer

在 Node 中，应用需要处理网络协议、操作数据库、处理图片、接受文件上传等，在网络流和文件的操作中，还要处理大量二进制数据，JavaScript 自有的字符串远远不能满足这些需求，于是有了 Buffer. 

### Buffer 结构

Buffer 类似于 Array, 但主要用于操作字节。Buffer 是一个由 JavaScript 和 C++ 结合的模块，C++ 实现性能相关的部分。Buffer 所占用的内存不通过 V8 分配，属于堆外内存，有更高效和专用的内存分配策略。Buffer 位于全局对象 global 上，无需导入。

Buffer 内存分配：在 C++ 层面申请内存，在 JavaScript 上分配内存。当进行小而频繁的 Buffer 操作时，采用 slab 的机制进行预先申请和时候分配，使得 JavaScript 到操作系统之间不必有过多的内存申请方面的系统调用。大块的 Buffer 直接调用 C++ 层面提供的内存。

### Buffer 的转换

**字符串 -> Buffer 对象**：[Buffer.from(string[, encoding])](http://nodejs.cn/api/buffer.html#buffer_static_method_buffer_from_string_encoding) encoding 默认为 'utf8'. 一个 Buffer 对象可以存储不同编码类型的字符串转码的值，调用 `write()` 方法实现。  

**Buffer 对象 -> 字符串**：[buf.toString([encoding[, start[, end]]])](http://nodejs.cn/api/buffer.html#buffer_buf_tostring_encoding_start_end)

Buffer 支持的编码类型：[Buffer 与字符编码](http://nodejs.cn/api/buffer.html#buffer_buffers_and_character_encodings)  
- 字符编码：utf8 utf16le latin1
- 二进制转文本的编码：base64 hex
- 传统的字符串编码：ascii binary ucs2

### Buffer 的拼接

从常见的输入流从读取内容的示例代码：  
```js
// 从输入流读取内容
import fs from "fs";
import { dirname } from "path";
import { fileURLToPath } from "url";

// 使用 ECMAScript modules 之后，需要如下两行代码来使用 __dirname 或 __filename
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
console.log(__filename);
console.log(__dirname);

var rs = fs.createReadStream(__dirname + "/README.md");
var data = "";
rs.on("data", function (chunk) {
  // 这里隐藏了 data = data.toString() + chunk.toString() chunk 为 Buffer 时，会以 utf-8 的转换成字符串
  data += chunk;
});
rs.on("end", function () {
  console.log(data);
});
```

以上代码在遇到宽字节的中文时可能会出现乱码。将文件可读流的每次读取的 Buffer 的长度限制为 11 模拟乱码的情况：  
```js
var rs = fs.createReadStream(__dirname + "/README.md", { highWaterMark: 11 });
```
此时会输出：  
```js
代码来��于 《深入浅出 Node.js》 ���书
```
出现乱码的原因在于文件可读流在读取时会逐个读取 Buffer。代码中限定了每个 Buffer 的长度为 11 个字节。第一个 Buffer 按照每个中文字 3 个字节的 utf-8 格式解码时，只能解码出  9 个字节，剩余两个字节无法解码，出现乱码。下一个 Buffer 头部也会出现一个字节无法被解码，产生一个乱码。
```js
console.log(Buffer.from("代码来自于 《深入浅出 Node.js》 一书"));
// 使用前 11 个字节，输出就会乱码
console.log(Buffer.from("e4bba3e7a081e69da5e887", "hex").toString());
// <Buffer e4 bb a3 e7 a0 81 e6 9d a5 e8 87 aa e4 ba 8e 20 e3 80 8a e6 b7 b1 e5 85 a5 e6 b5 85 e5 87 ba 20 4e 6f 64 65 2e 6a 73 e3 80 8b 20 e4 b8 80 e4 b9 a6>
// 代码来�
```
解决：  
1. 添加 `setEncoding('utf-8')`, 设置编码后让 data 事件中传递的不再是一个 Buffer 对象，而是编码后的字符串。调用该方法时，可读流对象在内部设置了一个 decoder 对象，每次 data 事件都通过该 decoder 对象进行 Buffer 到字符串的解码，然后传递给调用者。decoder 对象来自于 string_decoder 模块 [StringDecoder](https://nodejs.org/dist/latest-v14.x/docs/api/string_decoder.html#string_decoder_string_decoder) 的实例。
    ```js
    var rs = fs.createReadStream(__dirname + "/README.md", { highWaterMark: 11 });
    rs.setEncoding("utf-8"); 
    ```
    当把 Buffer 对象写入到 StringDecoder 实例时，decoder 内部会使用一个 buffer 确保编码的文本不包含任何不完整的多字节字符，这些不完整的字符被保留在内部 buffer 里，直到下一次调用 `stringDecoder.write()` 或调用 `stringDecoder.end()` 方法时，将不完整的字符与这一次写入的 Buffer 对象合并。

#### 正确拼接 Buffer

使用 [Buffer.concat(list[, totalLength])](https://nodejs.org/dist/latest-v14.x/docs/api/buffer.html#buffer_static_method_buffer_concat_list_totallength) 拼接多个 buffer 对象：  
```js
// 正确拼接 Buffer
import fs from "fs";
import { dirname } from "path";
import { fileURLToPath } from "url";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
console.log(__filename);
console.log(__dirname);

var res = fs.createReadStream(__dirname + "/README.md");
var chunks = [];
var size = 0;
res.on("data", function (chunk) {
  chunks.push(chunk);
  size += chunk.length;
});
res.on("end", function () {
  var buf = Buffer.concat(chunks, size);
  console.log(buf.toString("utf-8"));
});
```

### Buffer 与性能

在 web 应用中，提高字符串转换到 Buffer 的效率，可以达成的地提高网络吞吐率。  
使用 [apache ab](https://www.tutorialspoint.com/apache_bench/apache_bench_environment_setup.htm) 工具可以测试 web 服务器的性能，比较转换到 buffer 和不转换的 QPS(每秒查询次数)。  
在 Node 构建的 web 应用中，可以选择将页面中的动态内容和精通内容分离，静态内容部分可以通过预先转换为 Buffer 的方式提高性能。文件读取尽量只读取为 buffer 然后直接传输，不做额外的转换。

## 第 7 章 网络编程

直接看原书。

## 地 8 章 构建 web 应用

直接看原书。