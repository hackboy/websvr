WebSvr
==============
A simple web server, implement with filter and handler.


Features
--------------
- Filter (Middleware):  A request will try to match all the filters first.
- Handler: When a request matched a handler, it will returned, only one handler will be executed.
- Session: By config sessionDir, you can store session in files, with JSON format
- File:  Support uploading files
- Cache: Cahce template in release mode

Install
--------------

    npm install websvr


Start
--------------
It's simple to start the websvr.

    //import WebSvr module
    var WebSvr = require("websvr");

    //Start the WebSvr, runnting at parent folder, default port is 8054, directory browser enabled;
    //Trying at: http://localhost:8054
    var webSvr = WebSvr({
        home: "./web"
      , listDir:  true
      , debug:    true
      , sessionTimeout: 60 * 1000
    });


Filter (HttpModule)
--------------
Session based authentication, basically useage:

    /*
    Session support; 
    */
    webSvr.filter(function(req, res) {
      //Link to next filter
      req.filter.next();
    }, {session:true});


    /*
    * filter equal to use
    */
    webSvr.use('/home', function(req, res) {
      //do sth.
      req.filter.next();
    });


Filter all the requests that begin with "test/", check the user permission in session, except the "index.htm" and "login.do".

    /*
    Session Filter: protect web/* folder => (validation by session);
    */
    webSvr.filter(function(req, res) {
      //It's not index.htm/login.do, do the session validation
      if (req.url.indexOf("login.htm") < 0 && req.url.indexOf("login.do") < 0 && req.url !== '/') {
        var val = req.session.get("username");

        console.log("session username:", val);
        !val && res.end("You must login, first!");

        //Link to next filter
        req.filter.next();
      } else {
        req.filter.next();
      }
    }, { session: true });

Handler (HttpHandler, Servlet)
--------------
Handle Login and put the username in session

    /*
    Handler: login.do => (validate the username & password)
      username: admin
      password: 12345678
    webSvr.url equal to webSvr.get/webSvr.post/webSvr.handle
    */
    webSvr.url("login.do", function(req, res) {
      var qs = req.body;
      console.log(qs);
      if (qs.username == "admin" && qs.password == "12345678") {
        //Add username in session
        var session = req.session.set("username", qs.username);
        console.log(session);
        res.redirect("setting.htm");
      } else {
        res.writeHead(401);
        res.end("Wrong username/password");
      }
    }, 'qs');

Note:
--------------
Filter and Handler doesn't have the same match rules when you sending a request

Filter  : Match any section in the request url, for example

    websvr.filter(".svr", cb);

The result is

    request: "domain.com/admin/root/login.svr"   match: true

Handler : Match from the begining (ignore the first '/'), etc: 

    websvr.handle("root/login", cb)   //equal to
    websvr.handle("/root/login", cb)

etc:

    request: "domain.com/root/login.svr"         match: true
    request: "domain.com/admin/root/login.svr"   match: false

You can use regular expression to match any part of url in Handler.

Cookies
--------------

    //get cookie value by key
    req.cookies[key];


Template
--------------
Render template with params, using doT template engine

    res.render([view, model]);

View is optional, it will get the location of template from req.url

    res.render({json: true});

View is a relative path, relative to web home

    //means related to Setting.home
    res.render("list.tmpl", {json: true});

View is a absolute path, relative to web root

    //means related to Setting.root
    res.render("/list.tmpl", {json: true});

Render raw HTML views

    res.renderRaw(viewContent, model);

You can change template engine, 

    webSvr.engine(engineFunc);

for example:

    webSvr.engine(require("doT").compile);
    webSvr.engine(require("jade").compile);

You can define some default properties in model, for example header/footer, this parameters will be overridden if they have the same key in your custom model.

    webSvr.model({
        title   : "WebSvr Page"
      , username: "WebSvr"
    });

You can use template and render it by using websvr.render(tmplPath, model, callback), tmplPath relative to webSvr.home;

    //pre-defined model
    var model = {};
    webSvr.model(model);

    //render a template using model, callbak argument is result html
    webSvr.render("header.tmpl", {categoryList: category.categoryList}, function(html) {
      //store rendered html to header
      model.header = html;
      console.log(model);
    });

Include file, you can using "#include" to include a file during rendering a template, in order to make the process easier, the file will fetched from the cache pool so the first refresh will not work when you first start the server;

###Be ware: include file: relative to web home, not the template file itself.###

    <body>
    <!--#include="header.part"-->
    <div id="articles" class="container home">

Cache templates, by default, server will cache the templates(include the "include file" in the templates), turn it off via:

    var webSvr = WebSvr({
      templateCache: false
    });

Clear the cached templates

    webSvr.clear()



Enable template engine and '<!--#include=""-->', using: res.render()/res.render(model)/res.render(tmplPath, model), etc

webSvr.url(['login.htm', 'setting.htm'], function(req, res) {
  res.render();
});


It also support include file in include files, but you need to refresh more times after the first running.


Settings
--------------
Return configuration of current WebSvr instance

    webSvr.settings

Settings API:

    var Settings = {
      //home folder of web
      home: "../"

      //http start
      //default port of http
      , port: 8054

      //default port of https
      , httpsPort:  8443
      , httpsKey:   ""
      , httpsCert:  ""

      //list files in directory
      , listDir: false
      //enable client-side cache(304)
      , cache: true
      //enable debug information output
      , debug: true
      //enable cache of template/include file (when enabled templates will not be refreshed before restart)
      , templateCache: true

      //default pages, only one is supported
      , defaultPage: "index.html"
      //404 template/static file
      , 404:         "404.tmpl"
      //show errors to user(displayed in response)
      , showError: true

      /*
      Session timeout, in milliseconds.
      */
      , sessionTimeout: 1440000

      //session file stored here
      , sessionDir: os.tmpDir()

      //session domain, e.g. ".google.com"
      , sessionDomain: ""

      //tempary upload file stored here
      , uploadDir:  os.tmpDir()
    };

Response
--------------
Extension on reponse object

Ouput file, filepath relative to the root

    res.sendRootFile(filePath, [callback]);

Ouput file, filepath relative to the home (web dir)

    res.sendFile(filePath);
    res.sendHomeFile(filePath);

Reidrect request

    res.redirect(url);

Return request object

    res.req

Set Content-Type

    res.type('xml');

Set/Remove Cookie

    //Set Cookie
    res.cookie(name, value [, {domain: string, path: string, expires: date, secure, httponly }])
    //Remove Cookie
    res.cookie(name, null);


Change default charset

    res.charset = 'utf-8'


WebSvr APIs
--------------
Mapping url to file, webSvr.url equal to webSvr.handle

    webSvr.url("sitetest", ["svr/sitetest.js"]);

Mapping url to string

    webSvr.url("hello", "Hello WebSvr!")

Handle post

    webSvr.post("post.htm", function(req, res) {
        res.end('Received : ' + req.body);
    });

    //Equal to
    webSvr.handle("post.htm", function(req, res) {
        res.end('Received : ' + req.body);
    }, {post: true});

Post type

    post: true/"json"/"qs"

Handle session

    webSvr.session("session required url", function(req, res) {
        console.log(req.session);
        res.end();
    });


Handle upload file, it's a specfic filter

    webSvr.file("upload.do", function(req, res) {
      res.writeHead(200, {"Content-Type": "text/plain"});
      //Upload file is stored in req.files
      //form fields is stored in req.body
      res.write(JSON.stringify(req.body));
      res.end(JSON.stringify(req.files));
    });

Valid File beofre receing it

    /*
    * Valid request before receiving
    */
    webSvr.file("upload.do", function(req, res) {
      res.writeHead(200, {"Content-Type": "text/plain"});
      res.send(req.files);
    }).before(function(req, res) {
      if ((req.headers['content-length'] || 0) > 245760) {
        res.send('Posting is too large, should less than 240K')
      } else {
        return true
      }
    });

Multi-Mapping in Handler or Filter

    webSvr.handle(["about", "help", "welcome"], function(req, res) {
        res.writeFile(req.url + ".shtml");
    }, {post: true});

Pickup parameters from url expression

    webSvr.handle("/verify/:id", function(req, res) {
      var id = req.params.id;
    });

Parse parameters in url

    * expression = /home/:key/:pager
    *   /home/JavaScript => { id: 'JavaScript', pager: '' }
    *   /key/JavaScript  => false 

    var params = webSvr.parseUrl(expression, reqUrl);

Send API

  webSvr.send([type or statusCode, ] content);

Send JSON

    webSvr.send('json', { a: 1, b: 2 });

Send String

    webSvr.send(401, 'No permission');


Multi-instance support
--------------
Start a https server, make sure that the port will no conflict with others.

    var httpsSvr = new WebSvr({
        home: "./"

      //disable http server
      , port:      null

      //enable https server
      , httpsPort: 8443
      , httpsKey:  require("fs").readFileSync("svr/cert/privatekey.pem")
      , httpsCert: require("fs").readFileSync("svr/cert/certificate.pem")

    }).start();

Do you want to re-use the filters & handlers?

    httpsSvr.filters   = webSvr.filters;
    httpsSvr.handlers  = webSvr.handlers;


Store session in redis
--------------
Install: npm install websvr-redis

    var RedisStore = require('websvr-redis');

    RedisStore.start({ 
        port: 6379
      , host: 'ourjs.org'
      , auth: 'your-password-if-needed'
      , select: 0
    });

    httpsSvr.sessionStore = RedisStore;


Clear expired sessions, only 1 refresh timer is needed

    setInterval(RedisStore.clear, 1000000);




Lincenses
----
MIT, see our license file











Demo Sites
----
1. ourjs: url [ourjs.com](http://ourjs.com)
2. icalc: url [icalc.cn](http://icalc.cn),  source code [github](https://github.com/newghost/websvr-icalc/)


Websvr
====
基于NodeJS的一个极简Web服务器, 专为ARM设计。
假设嵌入式设备需要保持长时间稳定运行，当遇到问题时也可自动重启并恢复此前用户的Session会话。
