'use strict'

var fs = require('fs'),
    http = require('http'),
    kew = require('kew'),
    npmconf = require('npmconf'),
    mkdirp = require('mkdirp'),
    path = require('path'),
    rimraf = require('rimraf').sync,
    url = require('url'),
    util = require('util'),
    helper = {path: "lib"},
    libPath = path.join(__dirname, 'lib'),
    downloadUrl = 'http://selenium-release.storage.googleapis.com/2.43/selenium-server-standalone-2.43.1.jar',
    fileName = "selenium-server-standalone-2.43.1.jar";

npmconf.load(function(err, conf) {
  if (err) {
    console.log('Error loading npm config')
    console.error(err)
    process.exit(1)
    return
  }

  var tmpPath = findSuitableTempDirectory(conf),
      downloadedFile = path.join(tmpPath, fileName),
      promise = kew.resolve(true);

  // Start the install.
  promise = promise.then(function () {
    console.log('Downloading', downloadUrl);
    console.log('Saving to', downloadedFile);
    return requestBinary(getRequestOptions(conf.get('proxy')), downloadedFile);
  })

  promise.then(function () {
    return copyIntoPlace(tmpPath, libPath);
  })
  .then(function () {
    return fixFilePermissions();
  })
  .then(function () {
    console.log('Done. selenium standalone jar available at', helper.path);
  })
  .fail(function (err) {
    console.error('selenium standalone jar installation failed', err.stack);
    process.exit(1);
  })
})


function findSuitableTempDirectory(npmConf) {
  var now = Date.now(),
    candidateTmpDirs = [
    process.env.TMPDIR || '/tmp',
    npmConf.get('tmp'),
    path.join(process.cwd(), 'tmp')
  ];

  for (var i = 0; i < candidateTmpDirs.length; i++) {
    var candidatePath = path.join(candidateTmpDirs[i], 'Selenium');

    try {
      mkdirp.sync(candidatePath, '0777');
      var testFile = path.join(candidatePath, now + '.tmp');
      fs.writeFileSync(testFile, 'test');
      fs.unlinkSync(testFile);
      return candidatePath;
    } catch (e) {
      console.log(candidatePath, 'is not writable:', e.message);
    }
  }

  console.error('Can not find a writable tmp directory, please report issue on http://www.seleniumhq.org/support/ with as much information as possible.');
  process.exit(1);
}


function getRequestOptions(proxyUrl) {
  if (proxyUrl) {
    var options = url.parse(proxyUrl);
    options.path = downloadUrl;
    options.headers = { Host: url.parse(downloadUrl).host };
    // Turn basic authorization into proxy-authorization.
    if (options.auth) {
      options.headers['Proxy-Authorization'] = 'Basic ' + new Buffer(options.auth).toString('base64');
      delete options.auth;
    }
    return options;
  } else {
    return url.parse(downloadUrl);
  }
}

function requestBinary(requestOptions, filePath) {
  var deferred = kew.defer(),
    count = 0,
    notifiedCount = 0,
    outFile = fs.openSync(filePath, 'w');

  var client = http.get(requestOptions, function (response) {
    var status = response.statusCode;
    console.log('Receiving...');

    if (status === 200) {
      response.addListener('data',   function (data) {
        fs.writeSync(outFile, data, 0, data.length, null);
        count += data.length;
        if ((count - notifiedCount) > 800000) {
          console.log('Received ' + Math.floor(count / 1024) + 'K...');
          notifiedCount = count;
        }
      });

      response.addListener('end',   function () {
        console.log('Received ' + Math.floor(count / 1024) + 'K total.');
        fs.closeSync(outFile);
        deferred.resolve(true);
      });

    } else {
      client.abort();
      deferred.reject('Error with http request: ' + util.inspect(response.headers));
    }
  });
  return deferred.promise;
}

function copyIntoPlace(tmpPath, targetPath) {
  rimraf(targetPath);
  console.log("Copying to target path", targetPath);
  fs.mkdirSync(targetPath);

  // Look for the extracted directory, so we can rename it.
  var files = fs.readdirSync(tmpPath);
  var promises = files.map(function (name) {
    var deferred = kew.defer();

    var file = path.join(tmpPath, name);
    var reader = fs.createReadStream(file);

    var targetFile = path.join(targetPath, name);
    var writer = fs.createWriteStream(targetFile);
    writer.on("close", function() {
      deferred.resolve(true);
    });

    reader.pipe(writer);
    return deferred.promise;
  });

  return kew.all(promises);
}

function fixFilePermissions() {
    console.log('Fixing file permissions');
    fs.chmodSync(helper.path, '755');
}