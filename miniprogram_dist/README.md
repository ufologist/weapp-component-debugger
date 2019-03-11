## TODO 构建出兼容微信小程序原生支持的 npm 自定义组件

```javascript
// weapp-after-build.js
var fs = require('fs');
var path = require('path');

var chokidar = require('chokidar');
var glob = require('glob');
var copy = require('copy');
var log = require('fancy-log');

// TODO
// 监听 src 文件的变动, 再扫描全部的 json 配置, 再改写自定义组件的引用路径

var watcher = chokidar.watch('./src/**/*.{wxa,wxp,wxc}').on('ready', () => {
    console.log('ready');
    glob('./dist/**/*.json', function(error, files) {
        files.forEach(function(file) {
            new WeappConfig(file).rewriteNpmComponentPath();
        });
    });

    watcher.on('all', (event, path) => {
        console.log('all', event, path);
        setTimeout(function() {
            glob('./dist/**/*.json', function(error, files) {
                files.forEach(function(file) {
                    new WeappConfig(file).rewriteNpmComponentPath();
                });
            });
        }, 1000)
    });
});

function WeappConfig(configFilePath) {
    /**
     * 微信小程序配置文件的路径
     */
    this.configFilePath = configFilePath;
}
/**
 * 读取小程序的配置文件
 */
WeappConfig.prototype._readConfig = function(successCb) {
    fs.readFile(this.configFilePath, function(error, data) {
        if (error) {
            log.error(error, data);
        } else {
            var config = {};
            try {
                config = JSON.parse(data);
            } catch (error) {
                log.error(error);
            }
            successCb(config);
        }
    });
};
/**
 * 重写 npm 自定义组件的路径
 * 
 * - packageName1     -> {root}/npm_weapp_components/packageName1/index
 * - packageName1/a   -> {root}/npm_weapp_components/packageName1/a
 * - packageName1/a/b -> {root}/npm_weapp_components/packageName1/a/b
 */
WeappConfig.prototype.rewriteNpmComponentPath = function() {
    var configFilePath = this.configFilePath;

    this._readConfig(function(config) {
        var usingComponents = config.usingComponents;
        if (usingComponents) {
            log('[rewriteNpmComponentPath]', configFilePath);
            log(JSON.stringify(config, null, 4));

            for (var tagName in usingComponents) {
                var componentPath = usingComponents[tagName];
                if (isNpmWeappComponent(componentPath)) {
                    var npmPackageName = getNpmPackageName(componentPath);
                    var relativePath = getNpmWeappComponentRelativePath(configFilePath, componentPath);

                    copyNpmWeappComponent(npmPackageName, componentPath, function(files, source, dist) {
                        log('[copyNpmWeappComponent]', '[' + npmPackageName + ']', source, '->', dist);
                        log('------------------------------------');
                        log(files.map(function(file) {
                            return file.path;
                        }));
                        log('------------------------------------');
                    });

                    log('------------------------------------');
                    log('relativePath', componentPath, '->', relativePath);
                    log('------------------------------------');
                    usingComponents[tagName] = relativePath;
                }
            }

            fs.writeFile(configFilePath, JSON.stringify(config, null, 4), function() {

            });
        }
    });
};

/**
 * 组件路径是否指向了 npm 自定义组件(路径直接是字母开头的)
 * 
 * - "myPackage": "packageName"
 * - "package-other": "packageName/other"
 * 
 * @param {string} componentPath 自定义组件的引用路径
 * @return {boolean}
 * @see [使用 npm 包中的自定义组件](https://developers.weixin.qq.com/miniprogram/dev/devtools/npm.html)
 */
function isNpmWeappComponent(componentPath) {
    var isNpmPkg = false;

    var firstChar = componentPath.substring(0, 1);
    if (/\w/.test(firstChar)) {
        isNpmPkg = true;
    }

    return isNpmPkg;
}

/**
 * 获取相对路径
 * 
 * @param {string} usingComponentsConfigFilePath 
 * @param {string} componentPath 
 */
function getNpmWeappComponentRelativePath(usingComponentsConfigFilePath, componentPath) {
    var npmPackageName = getNpmPackageName(componentPath);
    var componentName = getComponentName(componentPath);

    var configDir = path.dirname(usingComponentsConfigFilePath);
    var npmWeappComponentDir = './dist/npm_weapp_components/' + npmPackageName + '/' + componentName;

    var relativePath = path.relative(configDir, npmWeappComponentDir);
    relativePath = relativePath.replace(/\\/g, '/');

    return relativePath;
}

/**
 * 从自定义组件路径中提取出 npm 包名
 * 
 * "packageName1"       -> packageName1
 * "packageName2/other" -> packageName2
 * 
 * @param {string} componentPath 
 * @return {string} npm 包名
 */
function getNpmPackageName(componentPath) {
    var pathSepIndex = componentPath.indexOf('/');

    var npmPackageName = '';
    if (pathSepIndex !== -1) {
        npmPackageName = componentPath.substring(0, pathSepIndex);
    } else {
        npmPackageName = componentPath;
    }

    return npmPackageName;
}

/**
 * 从自定义组件路径中提取出组件的名称
 * 
 * - "packageName1"         -> index
 * - "packageName2/other"   -> other
 * - "packageName2/other/b" -> other/b
 * 
 * @param {string} componentPath 
 * @return {string} 组件文件名
 */
function getComponentName(componentPath) {
    var componentName = 'index';

    var pathSepIndex = componentPath.indexOf('/');
    if (pathSepIndex !== -1) {
        componentName = componentPath.substring(pathSepIndex + 1);
    }

    return componentName;
}

/**
 * 从自定义组件路径中提取出组件的文件夹名称
 * 
 * - packageName1     -> '.'            -> ''
 * - packageName1/a   -> '.'            -> ''
 * - packageName1/a/b -> packageName1/a -> a
 * 
 * @param {string} componentPath 
 */
function getNpmComponentDir(componentPath) {
    var packageName = getNpmPackageName(componentPath);
    var dir = path.dirname(componentPath);

    if (dir === '.') {
        dir = '';
    } else {
        dir = dir.replace(packageName + '/', '');
    }

    return dir;
}

/**
 * 复制 npm 包中的组件文件到 dist 目录
 * 
 * - packageName1     -> ./node_modules/packageName1/miniprogram_dist/**
 * - packageName1/a   -> ./node_modules/packageName1/miniprogram_dist/**
 * - packageName1/a/b -> ./node_modules/packageName1/miniprogram_dist/a/**
 * 
 * @param {string} npmPackageName 
 * @param {string} componentPath 
 * @param {Function} successCb 
 */
function copyNpmWeappComponent(npmPackageName, componentPath, successCb) {
    var npmComponentDir = getNpmComponentDir(componentPath);
    var sourceFiles = npmComponentDir ? '/' + npmComponentDir + '/**' : '/**';
    var source = './node_modules/' + npmPackageName + '/miniprogram_dist' + sourceFiles;
    var dist = './dist/npm_weapp_components/' + npmPackageName + '/' + npmComponentDir;

    copy(source, dist, function(error, files) {
        if (error) {
            console.error(error);
        } else {
            successCb(files, source, dist);
        }
    });
}
```