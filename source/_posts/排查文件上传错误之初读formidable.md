---
title: formidable模块-排查文件上传错误
date: 2020-03-24 23:59:51
tags: [node.js,formidable]
---
## 背景
对于文件上传之类功能的认识，一直以来都很有限，起初的认识是浏览器将选择的文件转为二进制数据后，通过HTTP发送到后端，后端再将二进制数据转为文件进行存储。后来在一次文件上传报错的debug过程中尝试了阅读一下formidable模块的源码，重新认识了文件上传的过程和原理。
<!--more-->
## 异常错误
某天，接到了一个有关文件上传错误的反馈，返回的错误大概如下：
```
{
  "errMsg":"parser error, 920 of 1780 bytes parsed"
}
```
首先排查了一下这个错误不是在业务层面定义的，根据服务日志的error stack可以看出错误的来源是formidable模块，根据错误信息定位到该错误是模块中的**incoming_form.js**文件抛出的。抛出的部分代码如下：
```javascript
if (bytesParsed !== buffer.length) {
  this._error(new Error('parser error, '+bytesParsed+' of '+buffer.length+' bytes parsed'));
}
```
从这里的代码可以知道是左边的bytes不等于右边的buffer长度导致的错误，那么接下来就可以顺着这两个表达式的来源或者生成方式进一步定位到错误产生的真正原因：
* bytesParsed 是由当前方法调用对象的 _parser 属性中的write方法生成，传入的参数为当前函数定义的入参
* buffer.length 则是直接用当前函数入参的length

## 入口
formidable模块的介绍 <u>1.2.1版本</u>
> A Node.js module for parsing form data, especially file uploads.

一个解析表单数据，尤其是上传文件的模块。这里的重点就是解析两个字，说明原始的请求数据不好直接处理，所以这个模块主要工作便是把请求数据处理转换成可直接进一步使用的对象。知道了主要功能，接下来就通过formidable的使用来进进一步步了解formidable模块的运行过程。
```javascript
app.use(async (ctx, next) => {
  const form = new formidable.IncomingForm();;
  form.parse(ctx.req, (err, fields, files) => {
    if (err) {
      reject(err);
      return;
    }
    prese_ret = { fields, files };
  });
});
```
首先，创建一个IncomingForm的实例，然后将http请求对象和接收解析结果函数作为参数传入实例的parse方法。那么接下来就深入模块内部，看看formidable.IncomingForm和parse方法的定义。
```javascript
function IncomingForm(opts) {
  if (!(this instanceof IncomingForm)) return new IncomingForm(opts);
  EventEmitter.call(this);

  opts=opts||{};

  this.error = null;
  this.ended = false;

  this.maxFields = opts.maxFields || 1000;
  this.maxFieldsSize = opts.maxFieldsSize || 20 * 1024 * 1024;
  this.maxFileSize = opts.maxFileSize || 200 * 1024 * 1024;
  this.keepExtensions = opts.keepExtensions || false;
  this.uploadDir = opts.uploadDir || (os.tmpdir && os.tmpdir()) || os.tmpDir();
  this.encoding = opts.encoding || 'utf-8';
  this.headers = null;
  this.type = null;
  this.hash = opts.hash || false;
  this.multiples = opts.multiples || false;

  this.bytesReceived = null;
  this.bytesExpected = null;

  this._parser = null;
  this._flushing = 0;
  this._fieldsSize = 0;
  this._fileSize = 0;
  this.openedFiles = [];

  return this;
}
```
上面是IncomingForm方法的定义，初始参数的赋值暂时不需要过多关注，需要注意的是 **EventEmitter.call(this)**, 这里会将EventEmitter构造函数声明的属性附加到IncomingForm的实例，相当于ES6中的super(),也就是继承。继承后的IncomingForm实例便可以添加监听器和发射事件，formidable模块的监听解析进度等功能就是借此特性实现的。接下来就到了重点的parse函数：
## 过程
```javascript
IncomingForm.prototype.parse = function(req, cb) {
  this.pause = function() {
    req.pause();
  };

  this.resume = function() {
    req.resume();
  };

  if (cb) {
    var fields = {}, files = {};
    this
      .on('field', function(name, value) {
        fields[name] = value;
      })
      .on('file', function(name, file) {
        files[name] = file;
      })
  }

  // Parse headers and setup the parser, ready to start listening for data.
  this.writeHeaders(req.headers);

  // Start listening for data.
  var self = this;
  req
    .on('error', function(err) {
      self._error(err);
    })
    .on('aborted', function() {
      self.emit('aborted');
      self._error(new Error('Request aborted'));
    })
    .on('data', function(buffer) {
      self.write(buffer);
    })
    .on('end', function() {
    });

  return this;
};
```
上面是缩减后定义，可以看出这里主要做了几件事
* 定义实例的暂停和恢复方法，实际上调用的是request的暂停和恢复
* 在实例的field、file事件监听里返回了回调函数的结果
* 将请求头传入writeHeaders函数
* 监听request的data事件，将数据块写入到实例的write函数

这里有个知识点记录一下，request对象其实实现了ReadableStream接口，这个信息流可以被监听，当接受到了一个 POST 或者 PUT 请求时，可以通过监听 'data' 和 'end' 事件把 数据给取出来，所以接下来要了解的就是writeHeaders函数和实例的write函数做了什么事情。按照代码顺序，首先看writeHeaders函数：
```javascript
IncomingForm.prototype.writeHeaders = function(headers) {
  this.headers = headers;
  this._parseContentLength();
  this._parseContentType();
};
IncomingForm.prototype._parseContentLength = function() {
  this.bytesReceived = 0;
  if (this.headers['content-length']) {
    this.bytesExpected = parseInt(this.headers['content-length'], 10);
  } else if (this.headers['transfer-encoding'] === undefined) {
    this.bytesExpected = 0;
  }

  if (this.bytesExpected !== null) {
    this.emit('progress', this.bytesReceived, this.bytesExpected);
  }
};
IncomingForm.prototype._parseContentType = function() {
  if (this.bytesExpected === 0) {
    this._parser = dummyParser(this);
    return;
  }

  if (this.headers['content-type'].match(/octet-stream/i)) {
    this._initOctetStream();
    return;
  }

  if (this.headers['content-type'].match(/urlencoded/i)) {
    this._initUrlencoded();
    return;
  }

  if (this.headers['content-type'].match(/multipart/i)) {
    var m = this.headers['content-type'].match(/boundary=(?:"([^"]+)"|([^;]+))/i);
    if (m) {
      this._initMultipart(m[1] || m[2]);
    } else {
      this._error(new Error('bad content-type header, no multipart boundary'));
    }
    return;
  }

  if (this.headers['content-type'].match(/json/i)) {
    this._initJSONencoded();
    return;
  }
};
```
writeHeaders只做了两件事件，首先根据请求头的content-length属性知道了本次请求的数据主体大小，可用于监听进度功能的实现，然后根据content-type区分不同的内容类型，调用相应的初始化方法，这里先看其中一种比较常见和复杂的类型 **multipart**：首先从multipart类型的content-type里包含了boundary，boundary用于封装消息多个部分的边界，一般是一组1~70的字符组，简单理解为分割请求体中多个部分的边界符，当遇到这个边界符时，则说明可以开始了一个新部分的解析；通过正则匹配将boundary解析出来并传入 _initMultipart 函数。
> POST /foo HTTP/1.1
> Content-Length: 68137
> Content-Type: multipart/form-data; boundary=---------------------------974767299852498929531610575
> 
> ---------------------------974767299852498929531610575
> Content-Disposition: form-data; name="description" 
> 
> some text
> ---------------------------974767299852498929531610575
> Content-Disposition: form-data; name="myFile"; filename="foo.txt" 
> Content-Type: text/plain 
> 
> (content of the uploaded file foo.txt)
> ---------------------------974767299852498929531610575
_initMultipart函数主要是添加一些函数对解析出来的字段、值做处理。利用boundary将请求中多个part做分割，每个part代表一个字段参数或者文件参数，都有自己的headerField和headerValue，参考上面例子；声明onPartBegin函数，当被调用时则初始化一些part的属性，例如headers、filename、mime；声明onHeaderField、onHeaderValue、onHeaderEnd函数，将各个part的headerField、headerValue存起来，然后是onHeadersEnd函数，当执行到这个函数则说明part的header部分解析完了，开始处理字段值或文件内容，处理文件内容时分为两种情况：binary/7bit/8bit、base64，这里base64比较特殊是因为base64中每6个比特为1个可打印字符，而一个字节有8个比特，即3个打印字符可由4个字节来表示，所以需要对base64类型的进行4的整数倍截取，然后把处理后的buffer发送给part的data事件，onHeadersEnd 函数声明的结尾调用了 IncomingForm.prototype.handlePart 函数，将part作为参数传进去，handlePart函数里对part的data事件进行监听，通过part的filename判断part是字段还是文件做不同处理，如果是字段则简单判断一下buffer大小是否超出大小，然后将buffer传给string_decoder模块转成字符串，最后将字符串作为IncomingForm实例的field事件进行发送，如果是文件类型，则将buffer写入_uploadPath指定的临时文件路径中part.filename文件，然后将part.name和file对象作为IncomingForm实例的file事件进行发送。到这里处理解析结果的步骤就结束了，前面讲到req.data事件里将buffer转发给了内部解析器的write函数，即MultipartParser实例的write函数，接下来就看看_parser.write解析buffer的过程。
```
{ 
  PARSER_UNINITIALIZED: s++,
  START: s++,
  START_BOUNDARY: s++,
  HEADER_FIELD_START: s++,
  HEADER_FIELD: s++,
  HEADER_VALUE_START: s++,
  HEADER_VALUE: s++,
  HEADER_VALUE_ALMOST_DONE: s++,
  HEADERS_ALMOST_DONE: s++,
  PART_DATA_START: s++,
  PART_DATA: s++,
  PART_END: s++,
  END: s++
}
```
MultipartParser定义了上面这些状态值，对应不同的进度，例如开始读取part的headerFiled时则用HEADER_FIELD_START代表；然后看看MultipartParser的重点：write函数，该函数接收一个buffer作为参数，然后循环buffer里的每个字节，每个字节都是0\~255的整数，代表不同字符，例如CR=13（回车）、LF=10（换行）、SPACE=32（空格），循环里定义了一个switch，对不同状态时接收到的字节做处理，例如当前状态为S.HEADER_FIELD时，如果循环到CR这个字节，则把状态改为S.HEADERS_ALMOST_DONE，下个字节循环时的处理则对应S.HEADERS_ALMOST_DONE里的逻辑，如果当前循环到的字节是 58（代表冒号），则调用前面的 onheaderField 回调，将当前buffer和field在buffer中的起始结束位置作为参数传入，然后修改状态为S.HEADER_VALUE_START进行下一次循环，如果之前条件没命中，则判断当前字节是否在A~Z字符集内，如果不是，则返回当前循环位置。到这里bug的原因就很清晰了，通过判断解析中断的位置是在哪个状态中产生的，并且根据这个字符与状态希望的字符进行对比就可以找到错误的根本原因了。这次的bug原因就是在S.HEADER_FIELD状态下接收到非期望字符产生的，原理搞清后，调试就非常简单了，这里不做记录。
## 总结
最后总结一下，formidable模块处理文件上传，其实就是通过对buffer流按一定的规则进行解析，其中的文件写入到一个临时目录，然后将文件信息和解析处来的字段返回到回调函数中。
