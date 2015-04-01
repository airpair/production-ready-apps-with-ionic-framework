In this post i will explain how to prepare your project for production. We are going to:
- (**cordova hook**) [Lint your javascript](#lint-your-javascript "Lint your javascript") (this step is needed because before minifying and obfuscating we need to ensure that there are no javascript errors)
- (**gulp task**) [Load html templates as javascript angular templates](#html-templates-transformation "Load html templates as javascript angular templates") (this adds more obfuscation as your html won’t be visible for others)
- (**gulp task**) [Enable angular strict dependency injection](#enable-ng-strict-di "Enable angular strict dependency injection") (this step is needed because before obfuscating we need to ensure angular dependency injection won’t brake)
- (**gulp task**) [Concat js and css files](#concatenate-js-and-css-files "Concat js and css files") (this hides more details about your code)
- (**cordova hook**) And finally [uglify, minify and obfuscate your code](#uglify-minify-and-obfuscate "Uglify, minify and obfuscate your code").

For all the previous task we are going to use a mixture between ** *gulp tasks* ** and ** *cordova hooks* **. Gulp tasks will be “running” all the time after you run `ionic serve`. Cordova hooks will run each time you build or run your project with `ionic build ios [android]` or `ionic run android [ios]`

## <a name="lint-your-javascript">Lint your javascript</a>

1. For this procedure we are going to use these npm packages. Run these commands to install them.
    - `npm install jshint --save-dev`
    - `npm install async --save-dev`
2. Copy the following cordova hooks
    - In ** *before_prepare* ** folder copy [these](https://gist.github.com/agustinhaller/5e489e5419e43b11d7b7 "before_prepare cordova hooks") files
    - Give execution permissions to all of them, run
`chmod +x file_name`
3. We are ready to test javascript linting, run this command and you will see that jshint is working
    - `ionic build ios [android]`


## <a name="html-templates-transformation">HTML templates transformation</a>

Part of the obfuscation is transforming all the html templates in angular js templates (compressed inside a javascript file)

1. For this we are going to use ** *gulp-angular-templatecache* **. Run this command to install the npm package
    - `npm install gulp-angular-templatecache --save-dev`
2. Add the following lines to ** *gulpfile.js* **
    - ```javascript
        var templateCache = require('gulp-angular-templatecache');
    ```
    - ```javascript
        var paths = {
          templatecache: ['./www/templates/**/*.html']
        };
    ```
    - ```javascript
        gulp.task('templatecache', function (done) {
          gulp.src('./www/templates/**/*.html')
            .pipe(templateCache({standalone:true}))
            .pipe(gulp.dest('./www/js'))
            .on('end', done);
        });
    ```
    - ```javascript
        gulp.task('default', ['sass', 'templatecache']);
    ```
    - ```javascript
        gulp.task('watch', function() {
          gulp.watch(paths.sass, ['sass']);
          gulp.watch(paths.templatecache, ['templatecache']);
        });
    ```
3. Also we need to add this to ** *ionic.project* **
    - ```json
        "gulpStartupTasks": [
          "sass",
          "templatecache",
          "watch"
        ]
    ```
4. Add ** *templates* ** module in your ** *app.js* **
    - ```javascript
        angular.module('starter', ['ionic', 'starter.controllers', 'templates'])
    ```
5. Add reference to ** *templates.js* ** file in your ** *index.html* **
    - ```html
        <script src="js/templates.js"></script>    
    ```
6. Run
    - `ionic serve`
    - This will add a ** *templates.js* ** file inside ** *www/js* ** with all the html templates as angular js templates
    - **Note:** remember to update the angular ** *templateUrl* ** (for example in ** *app.js* ** and your ** *directives* **). By these I mean matching each ** *templateUrl* ** to the name in ** *templates.js* ** file
        - Before in ** *app.js* ** it was
            - ```javascript
                .state('intro', {
                  url: "/",
                  templateUrl: "templates/intro.html",
                  controller: 'IntroCtrl'
                });
            ```
        - After html templates transformation we should have this in templates.js
            - ```javascript
                $templateCache.put("intro.html", ...
            ```
        - So now in ** *app.js* ** we need to change it to
            - ```javascript
                .state('intro', {
                  url: "/",
                  templateUrl: "intro.html",
                  controller: 'IntroCtrl'
                });
            ```

## <a name="enable-ng-strict-di">Enable `ng-strict-di`</a>
Before minifying we need to enable angular strict dependency injection (for more information about why you need this, read [here](https://github.com/olov/ng-annotate#highly-recommended-enable-ng-strict-di-in-your-minified-builds "Why you need ng-strict-di"). This will save us from breaking angular dependency injection when minifying.

1. For that we are going to use ** *gulp-ng-annotate* **. Run this command to install the npm package
    - `npm install gulp-ng-annotate --save-dev`
2. Add the following lines to ** *gulpfile.js* **
    - ```javascript
        var ngAnnotate = require('gulp-ng-annotate');
    ```
    - ```javascript
        var paths = {
          ng_annotate: ['./www/js/*.js']
        };
    ```
    - ```javascript
        gulp.task('ng_annotate', function (done) {
          gulp.src('./www/js/*.js')
            .pipe(ngAnnotate({single_quotes: true}))
            .pipe(gulp.dest('./www/dist/dist_js/app'))
            .on('end', done);
        });
    ```
    - ```javascript
        gulp.task('default', ['sass', 'templatecache', 'ng_annotate']);
    ```
    - ```javascript
        gulp.task('watch', function() {
          gulp.watch(paths.sass, ['sass']);
          gulp.watch(paths.templatecache, ['templatecache']);
          gulp.watch(paths.ng_annotate, ['ng_annotate']);
        });
    ```
3. Also we need to add this to ** *ionic.project* **
    - ```json
        "gulpStartupTasks": [
          "sass",
          "templatecache",
          "ng_annotate",
          "watch"
        ]
    ```
4. Change the path of our angular js files in the ** *index.html* ** as follows
    - ```html
        <script src="dist/dist_js/app/app.js"></script>
    ```
5. Add `ng-strict-di` directive in `ng-app` tag (inside ** *index.html* **)
    - ```html
        <body ng-app="your-app" ng-strict-di>
    ```
6. Run
    - `ionic serve`
    - This will create a ** *dist* ** folder inside ** *www* ** folder with all our js files with strict dependency injection fixed

## <a name="concatenate-js-and-css-files">Concatenate js and css files</a>

1. For the concatenation of files we are going to use ** *gulp-useref* **. Run this command to install the npm package
    - `npm install gulp-useref --save-dev`
2. Add the following lines to ** *gulpfile.js* **
    - ```javascript
        var useref = require('gulp-useref');
    ```
    - ```javascript
        var paths = {
          useref: ['./www/*.html']
        };
    ```
    - ```javascript
        gulp.task('useref', function (done) {
          var assets = useref.assets();
          gulp.src('./www/*.html')
            .pipe(assets)
            .pipe(assets.restore())
            .pipe(useref())
            .pipe(gulp.dest('./www/dist'))
            .on('end', done);
        });
    ```
    - ```javascript
        gulp.task('default', ['sass', 'templatecache', 'ng_annotate', 'useref']);
    ```
    - ```javascript
        gulp.task('watch', function() {
          gulp.watch(paths.sass, ['sass']);
          gulp.watch(paths.templatecache, ['templatecache']);
          gulp.watch(paths.ng_annotate, ['ng_annotate']);
          gulp.watch(paths.useref, ['useref']);
        });
    ```
3. Also we need to add this to ** *ionic.project* **
    - ```json
        "gulpStartupTasks": [
          "sass",
          "templatecache",
          "ng_annotate",
          "useref",
          "watch"
        ]
    ```
4. Add the following to ** *index.html* ** to bundle css and js as you want
    - ```html
        <!-- build:css dist_css/styles.css -->
        <link href="css/ionic.app.css" rel="stylesheet">
        <!-- endbuild -->
    ```
    - ```html
        <!-- build:js dist_js/app.js -->
        <script src="dist/dist_js/app/app.js"></script>
        <script src="dist/dist_js/app/controllers.js"></script>
        <!-- endbuild -->
    ```
    - **Note:** if you require an external script/file don’t include it inside a bundle. For example:
        - ```javascript
            <script src="http://maps.google.com/maps/api/js"></script>
        ```
5. Run
    - `ionic serve`
    - This will create the bundled files inside you ** *www/dist* ** folder also a new ** *index.html* ** with the new path to the bundled files.

## <a name="uglify-minify-and-obfuscate">Uglify, minify and obfuscate</a>

1. For this procedure we are going to use these npm packages. Run these commands to install them.
    - `npm install cordova-uglify --save-dev`
    - `npm instal mv --save-dev`
2. Copy the following cordova hooks
    - In ** *after_prepare* ** folder copy [these](https://gist.github.com/agustinhaller/426351993c70a0329ad0 "after_prepare cordova hooks") files
    - Give execution permissions to all of them, run
        - `chmod +x file_name`
3. We are ready to get the obfuscated/minified/compressed apk, run this command and you will see your production ready app
    - `ionic build ios [android]`

