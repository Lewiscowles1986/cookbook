# 使用pre-signed URLs通过浏览器上传 [![Slack](https://slack.min.io/slack?type=svg)](https://slack.min.io)

使用presigned URLs,你可以让浏览器直接上传一个文件到S3服务，而不需要暴露S3服务的认证信息给这个用户。下面就是使用[minio-js](https://github.com/minio/minio-js)实现的一个示例程序。

### 服务端代码

```js
const MinIO = require('minio')

var client = new MinIO.Client({
    endPoint: 'play.min.io',
    port: 9000,
    useSSL: true,
    accessKey: 'Q3AM3UQ867SPQQA43P2F',
    secretKey: 'zuf+tfteSlswRu7BJ86wekitnifILbZam1KYY3TG'
})
```

初始化MinIO client对象，用于生成presigned upload URL。

```js
// express是一个小巧的Http server封装，不过这对任何HTTP server都管用。
const server = require('express')()

server.get('/presignedUrl', (req, res) => {
    client.presignedPutObject('uploads', req.query.name, (err, url) => {
        if (err) throw err
        res.end(url)
    })
})

server.get('/', (req, res) => {
    res.sendFile(__dirname + '/index.html');
})

server.listen(8080)
```

这里是[`presignedPutObject`](https://docs.min.io/docs/javascript-client-api-reference#presignedPutObject)的文档。

### 客户端代码

程序使用了[jQuery](http://jquery.com/).

用户通过浏览器选择了一个文件进行上传，然后在方法内部从Node.js服务端获得了一个URL。然后通过`XMLHttpRequest()`往这个URL发请求，直接把文件上传到`play.min.io:9000`。

```html
<input type="file" id="selector" multiple>
<button onclick="upload()">Upload</button>

<div id="status">No uploads</div>

<script src="//code.jquery.com/jquery-3.1.0.min.js"></script>
<script type="text/javascript">
  // `upload` iterates through all files selected and invokes a helper function called `retrieveNewURL`.
  function upload() {
    // Reset status text on every upload.
    $('#status').text(`No uploads`)
    // Get selected files from the input element.
    var files = $("#selector")[0].files
    for (var i = 0; i < files.length; i++) {
      var file = files[i]
      // 从服务器获取一个URL
      retrieveNewURL(file, (file, url) => {
        // 上传文件到服务器
        uploadFile(file, url)
      })
    }
  }

  // 发请求到Node.js server获取上传URL。
  //`retrieveNewURL` accepts the name of the current file and invokes the `/presignedUrl` endpoint to
  // generate a pre-signed URL for use in uploading that file: 
  function retrieveNewURL(file, cb) {
    $.get(`/presignedUrl?name=${file.name}`, (url) => {
      cb(file, url)
    })
  }

  // 使用XMLHttpRequest来上传文件到S3。
  // ``uploadFile` accepts the current filename and the pre-signed URL. It then invokes `XMLHttpRequest()`
  // to upload this file to S3 at `play.min.io:9000` using the URL:
  function uploadFile(file, url) {
    var xhr = new XMLHttpRequest()
    xhr.open('PUT', url, true)
    xhr.send(file)
    xhr.onload = () => {
      if (xhr.status == 200) {
        if ($('#status').text() === 'No uploads') {
          // For the first file being uploaded, change upload status text directly.
          $('#status').text(`Uploaded ${file.name}.`)
        } else {
          // If multiple files are uploaded, append upload status on the next line.
          $('#status').append(`<br>Uploaded ${file.name}.`)
        }
      }
    }
  }
</script>
```

现在你就可以让别人访问网页，并直接上传文件到S3服务，而不需要暴露S3服务的认证信息。
