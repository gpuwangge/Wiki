# Download Flask
```
pip install Flask
```
Verify Flask installed:
```
python -m flask --version
```

# Hello Flask Example
app.py
```
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, Flask!"
```
代码解释：
```
app = Flask(__name__)
```
作用是：创建一个 Flask Web 应用对象，并把当前 Python 模块的信息交给 Flask。之后你写的 @app.route("/")、app.config 等，都是在配置这个 app 应用对象。  
```__name__ ```
是 Python 自动提供的特殊变量，表示当前这个 Python 文件/模块的名字。  
假设文件名是：app.py  
如果它被 Flask 用模块方式加载，例如：
```
python -m flask --app app run
```
此时 
```__name__```
通常为："app"  
为什么要把它传给 Flask：Flask 需要知道“你的应用代码位于哪里”。
```__name__```
给它提供了当前模块或包的身份信息，Flask 据此定位应用的根目录，并进一步寻找资源  
```
@app.route("/")
```
作用是：告诉 Flask：当浏览器请求网站根路径 / 时，调用它下面紧挨着定义的 Python 函数，并把函数返回值作为 HTTP 响应发回去。这叫做“路由（routing）”。  

# Run Flask
VSCode terminal:
```
python -m flask --app app run
```
可能的运行结果：
```
* Serving Flask app 'app'
* Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
* Running on http://127.0.0.1:5000
Press CTRL+C to quit
127.0.0.1 - - [14/Sep/2026 21:36:32] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [14/Sep/2026 21:36:32] "GET /favicon.ico HTTP/1.1" 404 -
```
分析结果：
```
* Serving Flask app 'app'
```
Flask 成功找到了你的 app.py 模块，并加载了其中的 Flask 应用对象。
```
 * Debug mode: off
```
当前没有开启调试模式。这不是错误；开发时可以按需加上 --debug。
```
WARNING: This is a development server...
```
这是 Flask 每次使用内置开发服务器时都会显示的正常警告。它只是提醒你：该服务器适合本机开发和测试，不应该直接用于公开生产部署。开发阶段可以忽略它。
```
Running on http://127.0.0.1:5000
```
你的 Web 服务正在本机的 5000 端口运行：
- 127.0.0.1 指本机，也就是只有你这台电脑可以访问。
- 5000 是 Flask 开发服务器默认常用端口。
- 在浏览器打开 http://127.0.0.1:5000 就会请求你的 / 路由。
```
127.0.0.1 - - [14/Sep/2026 21:36:32] "GET / HTTP/1.1" 200 -
```
这是成功日志：
- GET /：浏览器请求根路径 /
- 200：服务器成功返回了页面内容
- 127.0.0.1：请求来自你的本机浏览器

这时候程序会一直运行，如果按CTRL+C to quit推出之后，http://127.0.0.1:5000就立即没有信息了。  

# 端口设置
```
http://127.0.0.1:5000/
       │         │
       │         └─ 端口（port）
       └─ 主机/监听地址（host）
```
- 127.0.0.1：代表当前这台电脑，也常可写为 localhost
- 5000：Flask 开发服务器的默认端口
- /：你的首页路由，例如 @app.get("/")

如果想改为 8000 端口：
```
python -m flask --app app run --port 8000
```

# favicon.ico
favicon.ico 是网站的小图标文件，相当于网站在浏览器里的“头像”。你在浏览器标签页标题左边、书签/收藏夹、历史记录以及有时的地址栏左侧看到的小 Logo，通常就是它。  
它有什么用
- 帮助识别网站：同时打开很多标签时，可以靠图标快速区分 GitHub、Google、自己的网站等。
- 品牌展示：通常会放公司或产品的简化 Logo。
- 书签和快捷方式：用户把网页加入收藏夹、固定到浏览器或桌面时，也可能使用这个图标。
- 默认兼容方案：许多浏览器会自动尝试请求网站根目录下的 /favicon.ico；即使 HTML 没明确声明图标，也可能被加载。

.ico 是 Windows 常见的图标容器格式；一个文件可以包含多种尺寸和色深，例如：
- 16×16：传统浏览器标签页
- 32×32：高 DPI 或某些界面
- 48×48：系统/快捷方式场景

现代网站也常使用 PNG、SVG，并通过 HTML 显式声明。浏览器对 PNG 等格式已有广泛支持。  

如果没有设定favicon.ico就会出现如下这一句
```
127.0.0.1 - - [14/Sep/2026 21:36:32] "GET /favicon.ico HTTP/1.1" 404 -
```
浏览器在打开网页时，会自动尝试请求网站标签页图标，而项目中暂时没有这个文件，所以 Flask 返回：
```
404 Not Found
```
如果希望浏览器标签有图标，并让这条 404 消失，按 Flask 的静态文件约定创建：
```
my_flask_project/
├─ app.py
└─ static/
   └─ favicon.ico
```
然后在你的 HTML <head> 中引用
```
<link rel="icon" href="{{ url_for('static', filename='favicon.ico') }}">
```
不过如果返回的是纯文本，没有 HTML 页面，暂时不需要处理。



