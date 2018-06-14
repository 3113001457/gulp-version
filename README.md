### 问题：

    1、当我们修改js和css文件时往往需要清除浏览器的缓存，否则有些效果就看不到更新过后的。
    2、通过对js,css文件内容进行hash运算，生成一个文件的唯一hash字符串(如果文件修改则hash号会发生变化)
    3、替换html中的js,css文件名，生成一个带版本号的文件名,这样加版本号的静态文件就不会存在缓存的问题了。

### 解决：

    1、gulp有自动化添加版本hash的插件gulp-rev，它的效果是更改文件名，如下：
    
        <link rel="stylesheet" href="../css/css-8710db1db1.css">
        <script src="../js/app-5340dc1df9.js"></script>
        
    2、gulp直接更改了文件名，而我们想要的效果则是下面这种添加版本号，如下：
        
        <link rel="stylesheet" href="../css/css.css?v=8710db1db1">
        <script src="../js/app.js?v=5340dc1df9"></script>
    
    3、所以我们需要更改一下gulp的文件，更改gulp-rev和gulp-rev-collector，如下：
        
        *a).打开node_modules\gulp-rev\index.js
        
            > 将 let cleanReplacement =  path.basename(json[key]).replace(new RegExp( opts.revSuffix ), '' );
            > 修改为：let cleanReplacement =  path.basename(json[key]).split('?')[0];
        
        *b).打开node_modules\gulp-rev-collector\index.js
        
            > 将 manifest[originalFile] = revisionedFile;
            > 修改为 manifest[originalFile] = originalFile + '?v=' + file.revHash;
        
        *c).打开nodemodules\rev-path\index.js
        
            > 将 return modifyFilename(pth, (filename, ext) => `${filename}-${hash}${ext}`)
            > 修改为 return filename + ext;
            
            > 将 return modifyFilename(pth, (filename, ext) => filename.replace(new RegExp(`-${hash}$`), '') + ext
            > 修改为 return filename + ext;
            
    4、再执行gulp命令，得到的结果如下:
    
        <link rel="stylesheet" href="../css/css.css?v=8710db1db1?v=8710db1db1">
        <script src="../js/app.js?v=5340dc1df9?v=5340dc1df9"></script>
    
    5、继续更改gulp-rev-collector
    
        打开node_modules\gulp-rev-collector\index.js
        
            > 将 regexp: new RegExp( '([\/\\\\\'"])' + pattern, 'g')
            > 修改为 regexp: new RegExp( '([\/\\\\\'"])' + pattern+'(\\?v=\\w{10})?', 'g')
    
    6、再执行gulp命令，得到的结果如下:
    
        <link rel="stylesheet" href="../css/css.css?v=8710db1db1">
        <script src="../js/app.js?v=5340dc1df9"></script>
    
### 具体实现步骤

###### 1、创建Node.js配置文件package.json

    > 1、npm init

配置目录：
```
{
  "name": "myapp",
  "version": "1.0.0",
  "description": "",
  "main": "gulpfile.js",
  "dependencies": {},
  "devDependencies": {
   "gulp": "^3.9.1",
    "gulp-clean": "^0.3.2",
    "gulp-clean-css": "^3.9.0",
    "gulp-jshint": "^2.0.4",
    "gulp-rename": "^1.2.2",
    "gulp-rev": "^7.1.2",
    "gulp-rev-collector": "^1.2.2",
    "gulp-uglify": "^3.0.0",
    "gulp-watch": "^4.3.11",
    "jshint": "^2.9.5",
    "pump": "^1.0.2",
    "run-sequence": "^2.2.0"
  },
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```
###### 2、创建项目目录

```
gulp-version
└─webapp
    |---gulpfile.js
    └─src
        ├─css
        │  └─css.css
        ├─js
        │  └─js.js
        └─html
            └─index.html
```

gulpfile.js 配置详情

```
//引入gulp和gulp插件
var gulp = require('gulp'),
runSequence = require('run-sequence'),
rev = require('gulp-rev'),
revCollector = require('gulp-rev-collector'),
rename = require('gulp-rename'),
uglify = require('gulp-uglify'),
clean = require('gulp-clean'),
pump = require('pump'),
watch = require('gulp-watch'),
jshint = require('gulp-jshint')
cleanCSS = require('gulp-clean-css');;

//定义css、js源文件路径
var cssSrc = 'src/**/*.css',
jsSrc = ['src/**/*.js','!gulpfile*.js'];

//监控文件变化
gulp.task('watch', function () {
    gulp.watch([jsSrc,cssSrc], ['default']);
});

//检查js语法
gulp.task('jslint', function() {
  return gulp.src(jsSrc)
  .pipe(jshint())
  .pipe(jshint.reporter('default'));
});

//清空目标文件
gulp.task('cleanDst', function () {
    return gulp.src(['dist','rev'], {read: false})
    .pipe(clean());
});

//CSS生成文件hash编码并生成 rev-manifest.json文件名对照映射
gulp.task('revCss', function(){
    return gulp.src(cssSrc)
    .pipe(rev())
    // 压缩css
    .pipe(cleanCSS({compatibility: 'ie8'}))
    .pipe(gulp.dest('dist'))
    .pipe(rev.manifest())
    .pipe(gulp.dest('rev/css'));
});

//js生成文件hash编码并生成 rev-manifest.json文件名对照映射
gulp.task('revJs', function(){

    return gulp.src(jsSrc)
    .pipe(rev())
    //压缩
    .pipe(uglify())
    .pipe(gulp.dest('dist'))
     //生成rev-manifest.json
     .pipe(rev.manifest())
     .pipe(gulp.dest('rev/js'));
 });

//Html替换css、js文件版本
gulp.task('revHtml', function () {
    return gulp.src(['rev/**/*.json', 'src/**/*.html'])
    .pipe(revCollector({
        replaceReved: true
    }))
    .pipe(gulp.dest('dist'));
});

// 将非js、非css移动到目标目录
gulp.task('mvNotDealAsset', function () {
    return gulp.src(['src/**/*','!src/**/*.css', '!src/**/*.js','!src/**/*.html'])
    .pipe(gulp.dest('dist'));
});

//开发构建
gulp.task('dev', function (done) {
    condition = false;
    runSequence(
        ['jslint'],
        ['cleanDst'],
        ['revCss'],
        ['revJs'],
        ['revHtml'],
        ['mvNotDealAsset'],
        ['watch'],
        done);
});

gulp.task('default', ['dev']);
```


