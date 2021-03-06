# s3-sync #

A streaming upload tool for Amazon S3, taking input from a
[`readdirp`](http://npmjs.org/package/readdirp) stream, and outputting the
resulting files.

s3-sync is also optionally backed by a [level](http://github.com/level/level)
database to use as a local cache for file uploads. This way, you can minimize
the frequency you have to hit S3 and speed up the whole process considerably.

You can use this to sync complete directory trees with S3 when deploying static
websites. It's a work in progress, so expect occasional API changes and
additional features.

## Installation ##

``` bash
npm install s3-sync
```

## Usage ##

### `require('s3-sync').createStream([db, ]options)` ###

Creates an upload stream. Passes its options to [knox](http://ghub.io/knox),
so at a minimum you'll need:

* `key`: Your AWS access key.
* `secret`: Your AWS secret.
* `bucket`: The bucket to upload to.

The following are also specific to s3-sync:

* `concurrency`: The maximum amount of files to upload concurrently.
* `headers`: Additional headers to include on each file.

If you want more control over the files and their locations that you're
uploading, you can write file objects directly to the stream, e.g.:

``` javascript
var stream = s3sync({
    key: process.env.AWS_ACCESS_KEY
  , secret: process.env.AWS_SECRET_KEY
  , bucket: 'sync-testing'
})

stream.write({
    src: __filename
  , dest: '/uploader.js'
})

stream.end({
    src: __dirname + '/README.md'
  , dest: '/README.md'
})
```

Where `src` is the *absolute* local file path, and `dest` is the location to
upload the file to on the S3 bucket.

`db` is an optional argument - pass it a *level* database and it'll keep a
local cache of file hashes, keeping S3 requests to a minimum.

## Example ##

Here's an example using `level` and `readdirp` to upload a local directory to
an S3 bucket:

``` javascript
var level = require('level')
  , s3sync = require('s3-sync')
  , readdirp = require('readdirp')

// To cache the S3 HEAD results and speed up the
// upload process. Usage is optional.
var db = level(__dirname + '/cache')

var files = readdirp({
    root: __dirname
  , directoryFilter: ['!.git', '!cache']
})

// Takes the same options arguments as `knox`,
// plus some additional options listed above
var uploader = s3sync(db, {
    key: process.env.AWS_ACCESS_KEY
  , secret: process.env.AWS_SECRET_KEY
  , bucket: 'sync-testing'
  , concurrency: 16
}).on('data', function(file) {
  console.log(file.fullPath + ' -> ' + file.url)
})

files.pipe(uploader)
```
