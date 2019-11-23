---
title: js实现文件上传的几种方式
date: 2017-10-12 13:17:21
tags: 技术
---

##文件上传
###准备：

[四种常见的 POST 提交数据方式](https://imququ.com/post/four-ways-to-post-data-in-http.html)

form可以用以下四种方式发送：<br/>

POST方法，并设置enctype 属性为 application/x-www-form-urlencoded <br/>
post方法设置 enctype 属性为 text/plain<br/>
post方法，并设置 enctype 属性为 multipart/form-data<br/>
使用get方法<br/><br/><br/>


### 文件、图片上传方式方法与相关。

###一、传统形式
图片、文件上传的传统形式，是使用表单元素file。
![](./filebiaodan.png)

#### 二、iframe上传

思路:
	1: 点击"提交"时的瞬间,生成一个iframe对象,插入body中<br/>
	2: 修改form的target ,为iframe的name值<br/>
	3: 给iframe加1个事件 ,onload<br/>


```
<div id="progress"></div>
<form action="/upload" method="post" enctype="multipart/form-data" target="upfile">
    <input type="file" name="pic" />
    <br/>
    <input type="button" value="提交"/>
</form>
```


####注意：
<p>1、需要设置iframe的name值与form的target属性值一样，就是把form表单上传文件的刷新转嫁到iframe里去了；</p>
<p>2、form表单的enctype属性值必须设置成multipart/form-data，将文件转换成文件流供后端接收；</p>


#### 三、ajax上传
HTML5提出了XMLHttpRequest对象的第二版，从此ajax能够上传文件了。这是真正的"异步上传"。用iframe上传，可以用作老式浏览器的替代方案。该方案主要用的是FormData对象，它能够构建类似表单的键值对。用FormData的最大优点就是我们可以异步上传一个二进制文件。

###### 与普通的 Ajax 相比，使用 FormData的最大优点就是我们可以异步上传二进制文件。类似序列化对象

```
<form action="/upload" id="form" enctype="multipart/form-data" method="post">
    <input type="file" id="File" name="name1">
    <div><input type="submit"></div>
</form>
<script>
    var fd = new FormData(document.getElementById("File"));
    fd.append("CustomField", "This is some extra data");
    $.ajax({
        url: "/upload",
        type: "POST",
        data: fd,
    });
</script>
```


### -—- 图片上传时，可能会用到一些其他的api

###关于FileReader
FileReader是html5为我们提供的读取文件的api。它的作用就是把文本流按指定格式读取到缓存，以供js调用。

FileReader有四种读取文件的方式：<br/>
1.readAsBinaryString读取为二进制码<br/>
2.readAsDataURL读取为DataURL<br/>
3.readAsText读取为文本<br/>
4.readAsArrayBuffer<br/>

![](./doc.png)


#####举例，预览图片
![](./reader.png)

文件一旦开始读取，无论成功或失败，实例的 result 属性都会被填充。如果读取失败，则 result 的值为 null ，否则即是读取的结果


#####举例，其他类型文件，实现进度条

```
<form action="/upload" enctype="multipart/form-data" method="post">
        <fieldset>
            <legend>读取文件：</legend>
            <input type="file" id="File" name="name1">
            <input type="button" value="中断" id="Abort">
            <p>
                <lable>读取进度：</lable>
                <progress id="Progress" value="0" max="100"></progress>
            </p>
        </fieldset>
        <div><input type="submit"></div>
    </form>
    <script>

        var progress = document.getElementById('Progress');
        var events = {
            load: function () {
                console.log('loaded');
            },
            progress: function (percent) {
                console.log(percent);
                progress.value = percent;
            },
            success: function () {
                console.log('success');
            }
        };
        var loader;

        // 选择好要上传的文件后触发onchange事件
        document.getElementById('File').onchange = function (e) {
            var file = this.files[0];
            loader = new FileLoader(file, events);
        };

        document.getElementById('Abort').onclick = function () {
            loader.abort();
        }
    </script>
```



#### 相关： 
[FileReader.js](https://github.com/bgrins/filereader.js)


#####关于上传：
AjaxFileUpload实现ajax文件上传。
react-fileupload
jQuery插件AjaxFileUpload实现ajax文件上传
jQuery File Upload支持多文件上传、取消、删除，上传前缩略图预览、列表显示图片大小，支持上传进度条显示。


#### 问题1：文件下载的实现方式，每种方式的区别与优劣。
#### 问题2：HTTP协议下文件上传断点续传的实现。


[github](https://github.com/swingboy/upload)