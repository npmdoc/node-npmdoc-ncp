# api documentation for  [ncp (v2.0.0)](https://github.com/AvianFlu/ncp)  [![npm package](https://img.shields.io/npm/v/npmdoc-ncp.svg?style=flat-square)](https://www.npmjs.org/package/npmdoc-ncp) [![travis-ci.org build-status](https://api.travis-ci.org/npmdoc/node-npmdoc-ncp.svg)](https://travis-ci.org/npmdoc/node-npmdoc-ncp)
#### Asynchronous recursive file copy utility.

[![NPM](https://nodei.co/npm/ncp.png?downloads=true&downloadRank=true&stars=true)](https://www.npmjs.com/package/ncp)

[![apidoc](https://npmdoc.github.io/node-npmdoc-ncp/build/screenCapture.buildCi.browser.apidoc.html.png)](https://npmdoc.github.io/node-npmdoc-ncp/build/apidoc.html)

![npmPackageListing](https://npmdoc.github.io/node-npmdoc-ncp/build/screenCapture.npmPackageListing.svg)

![npmPackageDependencyTree](https://npmdoc.github.io/node-npmdoc-ncp/build/screenCapture.npmPackageDependencyTree.svg)



# package.json

```json

{
    "author": {
        "name": "AvianFlu"
    },
    "bin": {
        "ncp": "./bin/ncp"
    },
    "bugs": {
        "url": "https://github.com/AvianFlu/ncp/issues"
    },
    "dependencies": {},
    "description": "Asynchronous recursive file copy utility.",
    "devDependencies": {
        "mocha": "1.15.x",
        "read-dir-files": "0.0.x",
        "rimraf": "1.0.x"
    },
    "directories": {},
    "dist": {
        "shasum": "195a21d6c46e361d2fb1281ba38b91e9df7bdbb3",
        "tarball": "https://registry.npmjs.org/ncp/-/ncp-2.0.0.tgz"
    },
    "engine": {
        "node": ">=0.10"
    },
    "gitHead": "93c7c6c719e2a4944dc09a16178b09aef428cdf0",
    "homepage": "https://github.com/AvianFlu/ncp",
    "keywords": [
        "cli",
        "copy"
    ],
    "license": "MIT",
    "main": "./lib/ncp.js",
    "maintainers": [
        {
            "name": "avianflu"
        },
        {
            "name": "mmalecki"
        }
    ],
    "name": "ncp",
    "optionalDependencies": {},
    "repository": {
        "type": "git",
        "url": "git+https://github.com/AvianFlu/ncp.git"
    },
    "scripts": {
        "test": "mocha -R spec"
    },
    "version": "2.0.0"
}
```



# <a name="apidoc.tableOfContents"></a>[table of contents](#apidoc.tableOfContents)

#### [module ncp](#apidoc.module.ncp)
1.  [function <span class="apidocSignatureSpan"></span>ncp (source, dest, options, callback)](#apidoc.element.ncp.ncp)
1.  [function <span class="apidocSignatureSpan">ncp.</span>toString ()](#apidoc.element.ncp.toString)

#### [module ncp.ncp](#apidoc.module.ncp.ncp)
1.  [function <span class="apidocSignatureSpan">ncp.</span>ncp (source, dest, options, callback)](#apidoc.element.ncp.ncp.ncp)



# <a name="apidoc.module.ncp"></a>[module ncp](#apidoc.module.ncp)

#### <a name="apidoc.element.ncp.ncp"></a>[function <span class="apidocSignatureSpan"></span>ncp (source, dest, options, callback)](#apidoc.element.ncp.ncp)
- description and source-code
```javascript
function ncp(source, dest, options, callback) {
  var cback = callback;

  if (!callback) {
    cback = options;
    options = {};
  }

  var basePath = process.cwd(),
      currentPath = path.resolve(basePath, source),
      targetPath = path.resolve(basePath, dest),
      filter = options.filter,
      rename = options.rename,
      transform = options.transform,
      clobber = options.clobber !== false,
      modified = options.modified,
      dereference = options.dereference,
      errs = null,
      started = 0,
      finished = 0,
      running = 0,
      limit = options.limit || ncp.limit || 16;

  limit = (limit < 1) ? 1 : (limit > 512) ? 512 : limit;

  startCopy(currentPath);

  function startCopy(source) {
    started++;
    if (filter) {
      if (filter instanceof RegExp) {
        if (!filter.test(source)) {
          return cb(true);
        }
      }
      else if (typeof filter === 'function') {
        if (!filter(source)) {
          return cb(true);
        }
      }
    }
    return getStats(source);
  }

  function getStats(source) {
    var stat = dereference ? fs.stat : fs.lstat;
    if (running >= limit) {
      return setImmediate(function () {
        getStats(source);
      });
    }
    running++;
    stat(source, function (err, stats) {
      var item = {};
      if (err) {
        return onError(err);
      }

      // We need to get the mode from the stats object and preserve it.
      item.name = source;
      item.mode = stats.mode;
      item.mtime = stats.mtime; //modified time
      item.atime = stats.atime; //access time

      if (stats.isDirectory()) {
        return onDir(item);
      }
      else if (stats.isFile()) {
        return onFile(item);
      }
      else if (stats.isSymbolicLink()) {
        // Symlinks don't really need to know about the mode.
        return onLink(source);
      }
    });
  }

  function onFile(file) {
    var target = file.name.replace(currentPath, targetPath);
    if(rename) {
      target =  rename(target);
    }
    isWritable(target, function (writable) {
      if (writable) {
        return copyFile(file, target);
      }
      if(clobber) {
        rmFile(target, function () {
          copyFile(file, target);
        });
      }
      if (modified) {
        var stat = dereference ? fs.stat : fs.lstat;
        stat(target, function(err, stats) {
            //if souce modified time greater to target modified time copy file
            if (file.mtime.getTime()>stats.mtime.getTime())
                copyFile(file, target);
            else return cb();
        });
      }
      else {
        return cb();
      }
    });
  }

  function copyFile(file, target) {
    var readStream = fs.createReadStream(file.name),
        writeStream = fs.createWriteStream(target, { mode: file.mode });

    readStream.on('error', onError);
    writeStream.on('error', onError);

    if(transform) {
      transform(readStream, writeStream, file);
    } else {
      writeStream.on('open', function() {
        readStream.pipe(writeStream);
      });
    }
    writeStream.once('finish', function() {
        if (modified) {
            //target file modified date sync.
            fs.utimesSync(target, file.atime, file.mtime);
            cb();
        }
        else cb();
    });
  }

  function rmFile(file, done) {
    fs.unlink(file, function (err) {
      if (err) {
        return onError(err);
      }
      return done();
    });
  }

  function onDir(dir) {
    var target = dir.name.replace(currentPath, targetPath);
    isWritable(target, function (writable) {
      if (writable) {
        return mkDir(dir, target);
      }
      copyDir(dir.name);
    });
  }

  function mkDir(dir, target) {
    fs.mkdir(target, dir.mode, function (err) {
      if (err) {
        return onError(err);
      }
      copyDir(dir.name);
    });
  }

  function copyDir(dir) {
    fs.readdir(dir, function (err, items) {
      if (err) {
        return onError(err);
      }
      items.forEach(function (item) {
        startCopy(path.join(dir, item));
      });
      retur ...
```
- example usage
```shell
n/a
```

#### <a name="apidoc.element.ncp.toString"></a>[function <span class="apidocSignatureSpan">ncp.</span>toString ()](#apidoc.element.ncp.toString)
- description and source-code
```javascript
toString = function () {
    return toString;
}
```
- example usage
```shell
n/a
```



# <a name="apidoc.module.ncp.ncp"></a>[module ncp.ncp](#apidoc.module.ncp.ncp)

#### <a name="apidoc.element.ncp.ncp.ncp"></a>[function <span class="apidocSignatureSpan">ncp.</span>ncp (source, dest, options, callback)](#apidoc.element.ncp.ncp.ncp)
- description and source-code
```javascript
function ncp(source, dest, options, callback) {
  var cback = callback;

  if (!callback) {
    cback = options;
    options = {};
  }

  var basePath = process.cwd(),
      currentPath = path.resolve(basePath, source),
      targetPath = path.resolve(basePath, dest),
      filter = options.filter,
      rename = options.rename,
      transform = options.transform,
      clobber = options.clobber !== false,
      modified = options.modified,
      dereference = options.dereference,
      errs = null,
      started = 0,
      finished = 0,
      running = 0,
      limit = options.limit || ncp.limit || 16;

  limit = (limit < 1) ? 1 : (limit > 512) ? 512 : limit;

  startCopy(currentPath);

  function startCopy(source) {
    started++;
    if (filter) {
      if (filter instanceof RegExp) {
        if (!filter.test(source)) {
          return cb(true);
        }
      }
      else if (typeof filter === 'function') {
        if (!filter(source)) {
          return cb(true);
        }
      }
    }
    return getStats(source);
  }

  function getStats(source) {
    var stat = dereference ? fs.stat : fs.lstat;
    if (running >= limit) {
      return setImmediate(function () {
        getStats(source);
      });
    }
    running++;
    stat(source, function (err, stats) {
      var item = {};
      if (err) {
        return onError(err);
      }

      // We need to get the mode from the stats object and preserve it.
      item.name = source;
      item.mode = stats.mode;
      item.mtime = stats.mtime; //modified time
      item.atime = stats.atime; //access time

      if (stats.isDirectory()) {
        return onDir(item);
      }
      else if (stats.isFile()) {
        return onFile(item);
      }
      else if (stats.isSymbolicLink()) {
        // Symlinks don't really need to know about the mode.
        return onLink(source);
      }
    });
  }

  function onFile(file) {
    var target = file.name.replace(currentPath, targetPath);
    if(rename) {
      target =  rename(target);
    }
    isWritable(target, function (writable) {
      if (writable) {
        return copyFile(file, target);
      }
      if(clobber) {
        rmFile(target, function () {
          copyFile(file, target);
        });
      }
      if (modified) {
        var stat = dereference ? fs.stat : fs.lstat;
        stat(target, function(err, stats) {
            //if souce modified time greater to target modified time copy file
            if (file.mtime.getTime()>stats.mtime.getTime())
                copyFile(file, target);
            else return cb();
        });
      }
      else {
        return cb();
      }
    });
  }

  function copyFile(file, target) {
    var readStream = fs.createReadStream(file.name),
        writeStream = fs.createWriteStream(target, { mode: file.mode });

    readStream.on('error', onError);
    writeStream.on('error', onError);

    if(transform) {
      transform(readStream, writeStream, file);
    } else {
      writeStream.on('open', function() {
        readStream.pipe(writeStream);
      });
    }
    writeStream.once('finish', function() {
        if (modified) {
            //target file modified date sync.
            fs.utimesSync(target, file.atime, file.mtime);
            cb();
        }
        else cb();
    });
  }

  function rmFile(file, done) {
    fs.unlink(file, function (err) {
      if (err) {
        return onError(err);
      }
      return done();
    });
  }

  function onDir(dir) {
    var target = dir.name.replace(currentPath, targetPath);
    isWritable(target, function (writable) {
      if (writable) {
        return mkDir(dir, target);
      }
      copyDir(dir.name);
    });
  }

  function mkDir(dir, target) {
    fs.mkdir(target, dir.mode, function (err) {
      if (err) {
        return onError(err);
      }
      copyDir(dir.name);
    });
  }

  function copyDir(dir) {
    fs.readdir(dir, function (err, items) {
      if (err) {
        return onError(err);
      }
      items.forEach(function (item) {
        startCopy(path.join(dir, item));
      });
      retur ...
```
- example usage
```shell
n/a
```



# misc
- this document was created with [utility2](https://github.com/kaizhu256/node-utility2)
