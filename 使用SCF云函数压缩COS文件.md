使用腾讯云 SCF 云函数压缩 COS 对象存储文件
===


在使用腾讯云 COS 对象存储的过程中，我们经常有想要把整个 Bucket 打包下载的需求，但是 COS 并没有提供整个 Bucket 打包下载的能力。这时，我们可以利用腾讯云的 SCF 无服务器云函数，完成 COS Bucket 的打包，并重新保存压缩后的文件到 COS 中，然后通过 COS 提供的文件访问链接下载文件。

但是在使用 SCF 云函数进行 COS Bucket 打包的过程中，偶尔会碰到这样的问题：我期望将某个 COS Bucket 内的文件全部下载下来然后打包压缩，把压缩文件再上传到 COS 中进行备份；但是在这个过程中，COS Bucket 内的文件可能数量多体积大，而 SCF 云函数的运行环境，实际只有 512MB 的 /tmp 目录是可以读写的。这样算上下载的文件，和生成的 ZIP 包，可能仅支持一定体积的文件处理，满足不了所需。怎么办？

在这种情况下，可能有的同学会想到使用内存，将内存转变为文件系统，即内存文件系统，或者直接读取文件并放置在内存中，或者在内存中生成文件。这种方法能解决一部分问题，但同时也带来了些其他问题：

1. SCF 云函数的内存配置也是有上限的，目前上限是 1.5GB。
2. SCF 云函数的收费方式是按配置内存*运行时间。如果使用配置大内存的方法，实际是在为可能偶尔碰到的极端情况支付不必要的费用，不符合我们使用 SCF 云函数就是要精简费用的目的。


我们在这里尝试了一种流式文件处理的方式，通过单个文件压缩后数据立即提交 COS 写的方法，一次处理一个文件，使得被压缩文件无需在 SCF 的缓存空间内堆积，压缩文件也无需放在缓存或内存中，而是直接写入 COS。在这里，我们实际利用了两种特性：ZIP 文件的数据结构特性和 COS 的分片上传特性。

## zip 文件的数据结构

在官方文档中给出的 zip 文件格式如下：
```
  Overall .ZIP file format:

    [local file header 1]
    [file data 1]
    [data descriptor 1]
    . 
    .
    .
    [local file header n]
    [file data n]
    [data descriptor n]
    [archive decryption header] (EFS)
    [archive extra data record] (EFS)
    [central directory]
    [zip64 end of central directory record]
    [zip64 end of central directory locator] 
    [end of central directory record]
```
可以看到，实际的 zip 文件格式基本是`[文件头+文件数据+数据描述符]{此处可重复n次}+核心目录+目录结束标识
`组成的，压缩文件的文件数据和压缩数据是在文件头部，相关的目录结构，zip文件信息存储在文件尾部。这样的结构，为我们后续 COS 分片上传写入带来了方便，可以先写入压缩数据内容，再写入最终文件信息。

## COS 分片上传

COS 分片上传按照如下操作即可进行：

1. 初始化分片上传：通过初始化动作，获取到此次上传的唯一标识ID。此ID需要保存在本地并在后续上传分片时使用。
2. 上传分片：通过初始化时获取到的ID，配合文件分片的顺序编号，依次上传文件分片，获取到每个分片的ETag；COS 会通过 ID 和分片顺序编号，拼接文件。
3. 结束上传：通过初始化时获取到的ID，结合分片的顺序编号和ETag，通知 COS 分片上传已经完成，可以进行拼接。

在上传过程中，还随时可以查询已上传分片，或结束取消分片上传。

## 文件压缩处理流程设计

利用 zip 文件数据结构中文件压缩数据在前目录和额外标识在后的特性，和 COS 支持分片上传的特性，我们可以利用流式文件处理方式来依次处理文件，并且做到处理完成一个文件压缩就上传处理后的压缩数据分片。这种处理流程可以简化为如下说明：
1. 初始化 zip 文件数据结构，并将数据结构保存在内存中。
2. 初始化 COS 分片上传文件，保存好分片上传 ID。
3. 下载要放入压缩包的文件至本地，使用 zip 算法，生成压缩文件的数据内容并保存在内存中，并根据目录格式，更新zip数据格式中的目录标识。
4. 将压缩后的文件数据使用 COS 上传分片，上传至 COS 中。
5. 清理删除下载至本地的需压缩文件。
6. 根据需要，重复 3~5 步骤，增加压缩包内的文件。
7. 在压缩文件处理完成后，使用分片上传，将内存中的 zip 文件数据结构最后的目录结构部分上传至 COS。
8. 通知 COS 结束上传，完成最终 zip 文件的自动拼接。

在这个处理流程中，一次只处理一个文件，对本地缓存和内存使用都只这一个文件的占用，相比下载全部文件再处理，大大减小了本地缓存占用和内存占用，这种情况下，使用少量缓存和内存就可以完成 COS 中大量文件的压缩打包处理。

## 使用SCF进行 COS 文件压缩处理实现

### 流式压缩文件库 archiver

我们这里使用 node.js 开发语言来实现 COS 文件压缩处理。我们这里使用了 cos-nodejs-sdk-v5 sdk 和 archiver 模块。其中 archiver 模块是实现zip和tar包压缩的流式处理库，能够通过 append 输入欲压缩文件，通过 stream 输出压缩后的文件流。archiver的简单用法如下：
```
// require modules
var fs = require('fs');
var archiver = require('archiver');

// create a file to stream archive data to.
var output = fs.createWriteStream(__dirname + '/example.zip');
var archive = archiver('zip', {
    zlib: { level: 9 } // Sets the compression level.
});

// pipe archive data to the file
archive.pipe(output);

// append a file from stream
var file1 = __dirname + '/file1.txt';
archive.append(fs.createReadStream(file1), { name: 'file1.txt' });

// append a file
archive.file('file1.txt', { name: 'file4.txt' });

// finalize the archive (ie we are done appending files but streams have to finish yet)
// 'close', 'end' or 'finish' may be fired right after calling this method so register to them beforehand
archive.finalize();
```
archiver 将会在每次 append 文件的时候，将文件的压缩数据输出到 pipe 指定的输出流上。因此，我们在这里可以通过实现我们自身的 WriteStream，获取到 archiver 的写请求，并把写入内容转移到 COS 模块的分片上传能力上。在这里，我们实现的 WriteStream 为：
```
var Writable = require('stream').Writable;
var util = require('util');

module.exports = TempWriteStream;

let handlerBuffer;

function TempWriteStream(options) {
  if (!(this instanceof TempWriteStream))
    return new TempWriteStream(options);
  if (!options) options = {};
  options.objectMode = true;
  handlerBuffer = options.handlerBuffer;
  Writable.call(this, options);
}

util.inherits(TempWriteStream, Writable);

TempWriteStream.prototype._write = function write(doc, encoding, next) {
  handlerBuffer(doc);
  process.nextTick(next)
};
```
通过集成 nodejs 中的 Writable stream，我们可以将写操作转移到我们所需的 handle 上去，handle 可以对接 COS 的分片上传功能。 
### COS 分片上传

COS 分片上传功能的实现如下，我们将其封装为 Upload 模块：
```
const cos = require('./cos')

let Duplex = require('stream').Duplex;
function bufferToStream(buffer) {
    let stream = new Duplex();
    stream.push(buffer);
    stream.push(null);
    return stream;
}

// 大于4M上传
const sliceSize = 4 * 1024 * 1024

function Upload(cosParams) {
    this.cosParams = cosParams;
    this.partNumber = 1;
    this.uploadedSize = 0;
    this.Parts = []
    this.tempSize = 0;
    this.tempBuffer = new Buffer('')
}

Upload.prototype.init = function (next) {
    const _this = this;
    cos.multipartInit(this.cosParams, function (err, data) {
        _this.UploadId = data.UploadId
        next()
    });
}
Upload.prototype.upload = function(partNumber, buffer) {
    const _this = this;
    const params = Object.assign({
        Body: bufferToStream(buffer),
        PartNumber: partNumber,
        UploadId: this.UploadId,
        ContentLength: buffer.length
    }, this.cosParams);
    cos.multipartUpload(params, function (err, data) {
        if (err) {
            console.log(err)
        } else {
            _this.afterUpload(data.ETag, buffer, partNumber)
        }
    });
}


Upload.prototype.sendData = function (buffer) {
    this.tempSize += buffer.length;
    if (this.tempSize >= sliceSize) {
        this.upload(this.partNumber, Buffer.concat([this.tempBuffer, buffer]))
        this.partNumber++;
        this.tempSize = 0;
        this.tempBuffer = new Buffer('')
    } else {
        this.tempBuffer = Buffer.concat([this.tempBuffer, buffer]);
    }
}

Upload.prototype.afterUpload = function (etag, buffer, partNumber) {
    this.uploadedSize += buffer.length
    this.Parts.push({ ETag: etag, PartNumber: partNumber })
    if (this.uploadedSize == this.total) {
        this.complete();
    }
}

Upload.prototype.complete = function () {
    this.Parts.sort((part1, part2) => {
        return part1.PartNumber - part2.PartNumber
    });
    const params = Object.assign({
        UploadId: this.UploadId,
        Parts: this.Parts,
    }, this.cosParams);
    cos.multipartComplete(params, function (err, data) {
        if (err) {
            console.log(err)
        } else {
            console.log('Success!')
        }
    });
}

Upload.prototype.sendDataFinish = function (total) {
    this.total = total;
    this.upload(this.partNumber, this.tempBuffer);
}

module.exports = Upload;

```

对于 COS 本身已经提供的 SDK，我们在其基础上封装了相关查询，分片上传初始化，分片上传等功能如下：

```
const COS = require('cos-nodejs-sdk-v5');

const cos = new COS({
    AppId: '125xxxx227',
    SecretId: 'AKIDutrojxxxxxxx5898Lmciu',
    SecretKey: '96VJ5tnlxxxxxxxl5To6Md2',
});
const getObject = (event, callback) => {
    const Bucket = event.Bucket;
    const Key = event.Key;
    const Region = event.Region
    const params = {
        Region,
        Bucket,
        Key
    };
    cos.getObject(params, function (err, data) {
        if (err) {
            const message = `Error getting object ${Key} from bucket ${Bucket}.`;
            callback(message);
        } else {
            callback(null, data);
        }
    });
};

const multipartUpload = (config, callback) => {
    cos.multipartUpload(config, function (err, data) {
        if (err) {
            console.log(err);
        }
        callback && callback(err, data);
    });
};

const multipartInit = (config, callback) => {
    cos.multipartInit(config, function (err, data) {
        if (err) {
            console.log(err);
        }
        callback && callback(err, data);
    });
};

const multipartComplete = (config, callback) => {
    cos.multipartComplete(config, function (err, data) {
        if (err) {
            console.log(err);
        }
        callback && callback(err, data);
    });
};

const getBucket = (config, callback) => {
    cos.getBucket(config, function (err, data) {
        if (err) {
            console.log(err);
        }
        callback && callback(err, data);
    });
};


module.exports = {
    getObject,
    multipartUpload,
    multipartInit,
    multipartComplete,
    getBucket
};
```

在具体使用时，需要将文件中 COS 相关登录信息的APPId，SecretId，SecretKey等替换为自身可用的真实内容。

### 功能入口实现函数

我们在最终入口函数 index.js 中使用各个组件来完成最终的目录检索，文件压缩打包上传。在这里，我们利用函数入参来确定要访问的 bucket 名称和所属地域，期望压缩的文件夹和最终压缩后文件名。云函数入口函数仍然为 main_handler。

```
// require modules
const fs = require('fs');
const archiver = require('archiver');

const cos = require('./cos');

const Upload = require('./Upload')

const TempWriteStream = require('./TempWriteStream')

const start = new Date();

const getDirFileList = (region, bucket, dir, next) => {
    const cosParams = {
        Bucket: bucket,
        Region: region,
    }
    const params = Object.assign({ Prefix: dir }, cosParams);

    cos.getBucket(params, function (err, data) {
        if (err) {
            console.log(err)
        } else {
            let fileList = [];
            data.Contents.forEach(function (item) {
                if (!item.Key.endsWith('/')) {
                    fileList.push(item.Key)
                }
            });
            next && next(fileList)
        }
    })
}

const handler = (region, bucket, source, target) => {

    const cosParams = {
        Bucket: bucket,
        Region: region,
    }
    const multipartUpload = new Upload(Object.assign({ Key: target}, cosParams));

    const output = TempWriteStream({ handlerBuffer: multipartUpload.sendData.bind(multipartUpload) })

    var archive = archiver('zip', {
        zlib: { level: 9 } // Sets the compression level.
    });
    output.on('finish', function () {
        multipartUpload.sendDataFinish(archive.pointer());
    });

    output.on('error', function (error) {
        console.log(error);
    });

    archive.on('error', function (err) {
        console.log(err)
    });

    archive.pipe(output);

    multipartUpload.init(function () {
        getDirFileList(region, bucket, source, function(fileList) {
            let count = 0;
            const total = fileList.length;
            for (let fileName of fileList) {
                ((fileName) => {
                    let getParams = Object.assign({ Key: fileName }, cosParams)
                    cos.getObject(getParams, (err, data) => {
                        if (err) {
                            console.log(err)
                            return
                        }
                        var buffer = data.Body;
                        console.log("download file "+fileName);
                        archive.append(buffer, { name: fileName.split('/').pop() });
                        console.log("zip file "+fileName);
                        count++;
                        if (count == total) {
                            console.log("finish zip "+count+" files")
                            archive.finalize();
                        }
                    })
                })(fileName)
            }
        })
    })
}

exports.main_handler = (event, context, callback) => {
    var region = event["region"];
    var bucket = event["bucket"];
    var source = event["source"];
    var zipfile = event["zipfile"];
    //handler('ap-guangzhou', 'testzip', 'pic/', 'pic.zip');
    handler(region, bucket, source, zipfile)
}

```

### 测试及输出

最终我们将如上的代码文件及相关依赖库打包为zip代码包，创建函数并上传代码包。同时我们准备好一个 COS Bucket命名为 testzip， 在其中创建 pic 文件夹，并在文件夹中传入若干文件。通过函数页面的测试功能，我们使用如下模版测试函数：
```
{
"region":"ap-guangzhou",
"bucket":"testzip",
"source":"pic/",
"zipfile":"pic.zip"
}
```
函数输出日志为：
```
...
2017-10-13T12:18:18.579Z 9643c683-b010-11e7-a4ea-5254001df6c6 download file pic/DSC_3739.JPG
2017-10-13T12:18:18.579Z 9643c683-b010-11e7-a4ea-5254001df6c6 zip file pic/DSC_3739.JPG
2017-10-13T12:18:18.689Z 9643c683-b010-11e7-a4ea-5254001df6c6 download file pic/DSC_3775.JPG
2017-10-13T12:18:18.690Z 9643c683-b010-11e7-a4ea-5254001df6c6 zip file pic/DSC_3775.JPG
2017-10-13T12:18:18.739Z 9643c683-b010-11e7-a4ea-5254001df6c6 download file pic/DSC_3813.JPG
2017-10-13T12:18:18.739Z 9643c683-b010-11e7-a4ea-5254001df6c6 finish zip 93 files
2017-10-13T12:18:56.887Z 9643c683-b010-11e7-a4ea-5254001df6c6 Success!
```
可以看到函数执行成功，并从 COS Bucket 根目录看到新增加的 pic.zip 文件。

## 项目源代码及改进方向

目前项目所有源代码已经放置在 Github 上，路径为 [https://github.com/qcloud-scf/demo-scf-compress-cos](https://github.com/qcloud-scf/demo-scf-compress-cos)。可以通过下载或 git clone 项目，获取到项目源代码，根据自身帐号信息，修改 cos 文件内的帐号 APPId、SecretId、SecretKey这些认证信息，然后将根目录下所有文件打包至 zip 压缩包后，通过 SCF 创建函数并通过 zip 文件上传代码来完成函数创建，根据上面所属的“测试及输出”步骤来测试函数的可用性。

函数在此提供的仍然只是个demo代码，更多的是为大家带来一种新的思路及使用腾讯云 SCF 无服务器云函数和 COS 对象存储。基于此思路，Demo本身后续还有很多可以改进的方法，或根据业务进行变化的思路：

1. 文件的处理目前还是下载一个处理一个，其实我们可以使用多线程和队列来加速处理过程，使用若干线程持续下载文件，使用队列对已经下载完成待处理的文件进行排队，然后使用一个压缩线程从队列中读取已下载的文件后进行压缩上传处理。这种方式可以进一步加快大量文件的处理速度，只是需要小心处理好缓存空间被使用占满后的等待和文件处理完成后的删除释放空间。
2. 目前 Demo 从入参接受的是单个地域、Bucket、目录和输出文件，我们完全可以改造为从多个地域或Bucket拉取文件，也可以传递指定的文件列表而不是仅一个目录，同时函数执行触发可以使用 COS 触发或 CMQ 消息队列触发，能够形成更加通用的压缩处理函数。

后续对于此 Demo 如果有更多疑问，想法，或改进需求，欢迎大家提交 git pr 或 issue。项目地址：[https://github.com/qcloud-scf/demo-scf-compress-cos](https://github.com/qcloud-scf/demo-scf-compress-cos)