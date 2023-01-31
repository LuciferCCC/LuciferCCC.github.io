上节课的一个 bug，在文件 `/home/acs/acapp/game/views/settings/getinfo.py` 中

```python linenums="1"
...
def getinfo_web(request):
    user = request.user
    if not user.is_authenticated: 
        return JsonResponse({
            'result': "未登录"
        })
    else:
        # player = Player.objects.all()[0] 这里每次都是查询第一个用户：管理员的信息
        player = Player.objects.get(user=user) # 取当前登录用户的信息
        return JsonResponse({
            'result': "success",
            'username': player.user.username,
            'photo': player.photo,
        })
...
```



## 一、Django 中集成 Redis 与基本操作

### 1. 安装并启动 redis

Django 自带数据库是 SqLite，通过几行命令可以迁移到 MySQL。

Redis 也是数据库，但是它是内存数据库，读取速度高于上面两个数据库。

Redis 数据库特点：

- Redis 中存储的全部是 `key:value` ，其余数据库存储的是表，表中存储一个一个条目；
- Redis 是单线程，不会出现读写冲突。

在 Django 中集成 Redis：

1. 安装 `django_redis`，在项目根目录 `/acapp/` 下执行指令：`pip isntall django_redis==5.0.0`

    > __注意__：这里一定要指定版本，跟课程保持一致，如果直接安装，即执行指令 `pip isntall django_redis` 错误：`ModuleNotFoundError: No module named 'django_redis'`，查询了原因是 Redis 和 Django 版本不兼容。

2. 在文件 `/home/acs/acapp/acapp/settings.py` 中配置 Django 的缓存机制，将下面代码添加进去即可

    ```python linenums="1"
    CACHES = {
        'default': {
            'BACKEND': 'django_redis.cache.RedisCache',
            'LOCATION': 'redis://127.0.0.1:6379/1',
            "OPTIONS": {
                "CLIENT_CLASS": "django_redis.client.DefaultClient",
            },
        },
    }
    USER_AGENTS_CACHE = 'default'
    ```

3. 启动 `redis-server`，在项目根目录 `/acapp/` 下执行指令：`sudo redis-server /etc/redis/redis.conf`

    启动完成之后，使用命令 `top` 可以查看到进程 `redis-server`

    如果需要关闭 `redis-server`，可以在 `top` 中找到 `redis-server` 的 `PID` 后执行 `kill -9 PID` 即可。

### 2. redis 在 Django 中的基本操作

在项目根目录 `/acapp/` 下执行指令 `python3 manage.py shell ` 进入 Django 后台系统，在这个系统里面如果使用 `Tab` 键会有命令提示。

```python linenums="1"
from django.core.cache import cache # 引入 cache

cache.keys('关键字') # 查找某个关键字，支持正则表达式

cache.set('lumoumou', 1, 5) # 设置：关键字, 值, 过期时间(单位是秒)
cache.set('lumoumou', 1, None) # 设置永不过期的关键字

cache.has_key('关键字') # 查询关键字是否存在

cache.get('关键字') # 查询关键字的值

cache.delete('关键字') # 删去某关键字

# 删除所有关键字
for key in cache.keys("*"):
    cache.delete(key)
```



## 二、WEB端AcWing一键登录

<img src="https://cdn.acwing.com/media/article/image/2021/11/25/1_1ddf070e4d-weboauth2.png">



首先由于每个用户都需要存储 openid（现阶段数据库只存储了用户的用户名和头像信息），所以需要添加一栏，在文件 `/home/acs/acapp/game/models/player/player.py` 中修改：

```python linenums="1"
from django.db import models
from django.contrib.auth.models import User

class Player(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    photo = models.URLField(max_length=256, blank=True) 
    
    # 此处添加
    openid = models.CharField(default="", max_length=50, blank=True, null=True)    

    def __str__(self):
        return str(self.user)
```

更新数据库：

```python linenums="1"
python3 manage.py makemigrations
python3 manage.py migrate
```

重启服务即可在后台看见每个用户下面多了一栏：`uwsgi --ini scripts/uwsgi.ini `

![](https://cdn.jsdelivr.net/gh/LuciferCCC/blogs_images@main/%E6%88%AA%E5%B1%8F2023-01-31%2010.11.34.png)

### 1. 申请授权码 code

__请求地址__

`https://www.acwing.com/third_party/api/oauth2/web/authorize/`

参考示例：`?` 开始， `&`隔开每个参数

```
请求方法：GET
https://www.acwing.com/third_party/api/oauth2/web/authorize/？appid=APPID&redirect_uri=REDIRECT_URI&scope=SCOPE&state=STATE
```

__参数说明__

| 参数         | 是否必须 |                             说明                             |
| :----------- | -------- | :----------------------------------------------------------: |
| appid        | 是       |       应用的唯一id，可以在AcWing编辑AcApp的界面里看到        |
| redirect_uri | 是       | 接收授权码的地址。需要用对链接进行编码：Python3中使用urllib.parse.quote；Java中使用URLEncoder.encode |
| scope        | 是       |              申请授权的范围。目前只需填userinfo              |
| state        | 否       | 用于判断请求和回调的一致性，授权成功后后原样返回。该参数可用于防止csrf攻击（跨站请求伪造攻击），建议第三方带上该参数，可设置为简单的随机数（如果是将第三方授权登录绑定到现有账号上，那么推荐用随机数 + user_id作为state的值，可以有效防止CSRF攻击） |

> Tips：state是存放在 redis 中的

__返回说明__

用户同意授权后会重定向到 `redirect_url`，返回参数为 `code` 和 `state`。链接格式如下：

```
redirect_uri?code=CODE&state=STATE
```

如果用户拒绝授权，则不会发生重定向。

__后端实现过程__

在文件夹 `/home/acs/acapp/game/views/settings` 下面新建文件夹 `acwing` 并在里面创建文件 `__init__.py`，再在该文件夹下面创建文件夹 `web` 和 `acapp` ，并分别在两个文件夹下创建文件 `__init__.py`。

在文件夹 `web` 下面创建文件 `apply_code.py` 用于实现申请 `code`。

```python linenums="1"
from django.http import JsonResponse

def apply_code(request):
    appid = "4588" # 在 Acwing 网站 acapp 应用编辑界面找
    
    return JsonResponse({
        'result': "success",
    })
```

在文件夹 `web` 下面创建文件 `receive_code.py` 用于实现接收 `code`。

```python linenums="1"
from django.shortcuts import redirect

def receive_code(request):
    return redirect("index")
```

在文件夹 `/home/acs/acapp/game/urls/settings` 下面创建一个文件夹 `acwing` 并创建文件夹 `__init__.py` 和 `index.py`，在文件 `index.py` 里修改用于实现 `acwing` 的路由：

```python linenums="1"
from django.urls import path
from game.views.settings.acwing.web.apply_code import apply_code
from game.views.settings.acwing.web.receive_code import receive_code

urlpatterns = [
    path("web/apply_code/", apply_code, name="settings_acwing_web_apply_code"),
    path("web/receive_code/", receive_code, name="settings_acwing_web_receive_code"),
]
```

在文件 `/home/acs/acapp/game/urls/settings/index.py` 中将 `acwing` 中的目录导入进来。

```python linenums="1"
from django.urls import path, include # 添加 include
from game.views.settings.getinfo import getinfo
from game.views.settings.login import signin
from game.views.settings.logout import signout
from game.views.settings.register import register

urlpatterns = [
    path("getinfo/", getinfo, name="settings_getinfo"),
    path("login/", signin, name="settings_login"),
    path("logout/", signout, name="settings_logout"),
    path("register/", register, name="settings_register"),
    path("acwing/", include("game.urls.settings.acwing.index")), # 此处添加
]
```

重启服务后可以在浏览器中输入 `游戏网址/settings/acwing/web/apply_code` 后看见 `{"result": "success"}`，在浏览器中输入 `游戏网址/settings/acwing/web/recei_code` 后看见游戏登录界面。 

继续修改文件 `/home/acs/acapp/game/views/settings/acwing/web/apply_code.py`：

```python linenums="1"
from django.http import JsonResponse
from urllib.parse import quote
from random import randint
from django.core.cache import cache

def get_state():
    """
    获取一个8位的每个数值随机在0~8之间的数
    """
    res = ""
    for i in range(8):
        res += str(randint(0, 9))
    return res


def apply_code(request):
    appid = "4588" # 在 Acwing 网站 acapp 应用编辑界面找
    redirect_uri = quote("https://app4588.acapp.acwing.com.cn/settings/acwing/web/receive_code/")
    scope = "userinfo"
    state = get_state()

    cache.set(state, True, 7200)  # 将 state 存储 redis，值为 True，有效时间 2 小时(7200秒)

    apply_code_url = "https://www.acwing.com/third_party/api/oauth2/web/authorize/"
    return JsonResponse({
        'result': "success",
        'apply_code_url': apply_code_url + "?appid=%s&redirect_uri=%s&scope=%s&state=%s" % (appid, redirect_uri, scope, state)
    })
```

重启服务后在浏览器中输入  `游戏网址/settings/acwing/web/apply_code`  后可以看见返回信息 `{"result": "success", "apply_code_url": "https://www.acwing.com/third_party/api/oauth2/web/authorize/?appid=4588&redirect_uri=https%3A//app4588.acapp.acwing.com.cn/settings/acwing/web/receive_code/&scope=userinfo&state=39902428"}`，在浏览器中输入链接

```
https://www.acwing.com/third_party/api/oauth2/web/authorize/?appid=4588&redirect_uri=https%3A//app4588.acapp.acwing.com.cn/settings/acwing/web/receive_code/&scope=userinfo&state=39902428
```

会显示出请求授权界面：

![](https://cdn.jsdelivr.net/gh/LuciferCCC/blogs_images@main/20230131113403.png)

__前端实现过程__

在文件 `/home/acs/acapp/game/static/js/src/settings/zbase.js` 中修改：

```javascript linenums="1"
	...
		this.$register_login = this.$register.find(".ac-game-settings-option");

        this.$register.hide();

        this.$acwing_login = this.$settings.find('.ac-game-settings-acwing img'); // 此处添加

        this.root.$ac_game.append(this.$settings);

        this.start();
    }
    
    add_listening_events() {
        let outer = this; // 此处添加
        this.add_listening_events_login();
        this.add_listening_events_register();

        this.$acwing_login.click(function() { // 此处添加
            outer.acwing_login();
        })
    }
	...
    acwing_login() { // 此处添加
        console.log("click acwing login");
    }
	...
```

打包之后重启服务器，进入游戏登录界面，点击一键登录会在控制台看到输出 `click acwing login`。

```shell linenums="1"
# 项目根目录 /acapp/下
./scripts/compress_game_js.sh 
uwsgi --ini scripts/uwsgi.ini 
```

> Tips：如果没有反应就清理一下缓存。

上述步骤执行完毕之后，说明程序无误。

下面继续修改文件 `/home/acs/acapp/game/static/js/src/settings/zbase.js` ：

```javascript linenums="1"
	...
		});
        this.$register_submit.click(function() {
            outer.register_on_remote();
        });
    }

    // 修改这个函数
    acwing_login() {
        $.ajax({
            url: "https://app4588.acapp.acwing.com.cn/settings/acwing/web/apply_code",
            type: "GET",
            success: function(resp) {
                console.log(resp);
                if (resp.result === "success") {
                    window.location.replace(resp.apply_code_url); // 重定向
                }
            }
        });
    }

    login_on_remote() { // 在远程服务器上登录
    ...
```

打包之后重启服务器，进入游戏登录界面，点击一键登录会跳转到![](https://cdn.jsdelivr.net/gh/LuciferCCC/blogs_images@main/20230131113403.png)

```shell linenums="1"
# 项目根目录 /acapp/下
./scripts/compress_game_js.sh 
uwsgi --ini scripts/uwsgi.ini 
```

> Tips：如果没有反应就清理一下缓存。

修改文件 `/home/acs/acapp/game/views/settings/acwing/web/receive_code.py` 用于接收 `code`：

```python linenums="1"
from django.shortcuts import redirect

def receive_code(request):
    # 此处添加
    data = request.GET
    code = data.get('code')
    state = data.get('state')
    print("code:%s  state:%s" %(code, state))
    
    return redirect("index")
```

这时候重启服务，刷新页面，进入一键登录并点击授权，会在重启服务的命令行窗口看见 `acwing` 返回的 `code and state`：

![](https://cdn.jsdelivr.net/gh/LuciferCCC/blogs_images@main/%E6%88%AA%E5%B1%8F2023-01-31%2012.31.00.png)

此时回到项目根目录 `/acapp/` 下输入：

```python linenums="1"
python3 manage.py shell
from django.core.cache import cache
cache.keys('*')
```

可以看见 `redis` 中存储的所有有效的 `state`，如果有多个是因为申请了多次。

由于点击了多次生成了多个 `state`，为了不出问题，使用下面代码将 redis 中的数据删除：

```python linenums="1"
for key in cache.keys("*"): 
	cache.delete(key)
```

下面对接收到的  `code and state` 进行处理，继续修改文件 `/home/acs/acapp/game/views/settings/acwing/web/receive_code.py` ：

```python linenums="1"
from django.shortcuts import redirect
from django.core.cache import cache # 此处添加

def receive_code(request):
    data = request.GET
    code = data.get('code')
    state = data.get('state')
    print("code:%s  state:%s" %(code, state))

    # 此处添加
    if not cache.has_key(state):
        return redirect("index")
    cache.delete(state) 

    return redirect("index")
```

### 2. 申请授权令牌 access_token 和用户的 openid

__请求地址__

```
https://www.acwing.com/third_party/api/oauth2/access_token/
```

__参考示例__

```shell linenums="1"
# 请求方法：GET
https://www.acwing.com/third_party/api/oauth2/access_token/?appid=APPID&secret=APPSECRET&code=CODE
```

**参数说明**

| 参数   | 是否必须 | 说明                                            |
| ------ | -------- | ----------------------------------------------- |
| appid  | 是       | 应用的唯一id，可以在AcWing编辑AcApp的界面里看到 |
| secret | 是       | 应用的秘钥，可以在AcWing编辑AcApp的界面里看到   |
| code   | 是       | 第一步中获取的授权码                            |

**返回说明**

```shell linenums="1"
# 申请成功示例
{ 
    "access_token": "ACCESS_TOKEN", 
    "expires_in": 7200, 
    "refresh_token": "REFRESH_TOKEN",
    "openid": "OPENID", 
    "scope": "SCOPE",
}

# 申请失败示例
{
    "errcode": 40001,
    "errmsg": "code expired",  # 授权码过期
}
```

**返回参数说明**

| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| access_token  | 授权令牌，有效期2小时                                        |
| expires_in    | 授权令牌还有多久过期，单位（秒）                             |
| refresh_token | 用于刷新access_token的令牌，有效期30天                       |
| openid        | 用户的id。每个AcWing用户在每个acapp中授权的openid是唯一的,可用于识别用户。 |
| scope         | 用户授权的范围。目前范围为userinfo，包括用户名、头像         |

**刷新access_token的有效期**

- `access_token` 的有效期为2小时，时间较短
- `refresh_token` 的有效期为30天，可用于刷新 `access_token`
- 刷新结果有两种：
    - 如果 `access_token` 已过期，则生成一个新的 `access_token`
    - 如果 `access_token` 未过期，则将当前的 `access_token` 的有效期延长为 `2` 小时



**参考示例**

```shell linenums="1"
# 请求方法：GET
https://www.acwing.com/third_party/api/oauth2/refresh_token/?appid=APPID&refresh_token=REFRESH_TOKEN
```

返回结果的格式与申请access_token相同。

修改文件 `/home/acs/acapp/game/views/settings/acwing/web/receive_code.py` ：

```python linenums="1"
from django.shortcuts import redirect
from django.core.cache import cache
import requests # 此处添加

def receive_code(request):
    data = request.GET
    code = data.get('code')
    state = data.get('state')
    print("code:%s  state:%s" %(code, state))

    if not cache.has_key(state): 
        return redirect("index")
    cache.delete(state) 

    # 添加下面到文件中
  
    # 申请授权令牌和用户openid
    apply_access_token_url = "https://www.acwing.com/third_party/api/oauth2/access_token/"
    
    # appid在acapp设置界面中的AppID
    # secret在acapp设置界面中的AppSecret
    params = {
        'appid': "4588",
        'secret': "ea08584c059441c297b952e06395ea08",
        'code': code,
    }

    # 通过链接访问数据
    access_token_res = requests.get(apply_access_token_url, params=params).json()

    return redirect("index")
```



### 3. 申请用户信息

**请求地址**

```
https://www.acwing.com/third_party/api/meta/identity/getinfo/
```

**参考示例**

```shell linenums="1"
# 请求方法：GET
https://www.acwing.com/third_party/api/meta/identity/getinfo/?a
```

**参数说明**

| 参数         | 是否必须 | 说明                     |
| ------------ | -------- | ------------------------ |
| access_token | 是       | 第二步中获取的授权令牌   |
| openid       | 是       | 第二步中获取的用户openid |

**返回说明**

```shell linenums="1"
# 申请成功示例：
{
    'username': "USERNAME",
    'photo': "https:cdn.acwing.com/xxxxx"
}

# 申请失败示例：
{
    'errcode': "40004",
    'errmsg': "access_token expired"  # 授权令牌过期
}
```

修改文件 `/home/acs/acapp/game/views/settings/acwing/web/receive_code.py` ：

```python linenums="1"
from django.shortcuts import redirect
from django.core.cache import cache
import requests
from django.contrib.auth.models import User # 此处添加
from game.models.player.player import Player # 此处添加
from django.contrib.auth import login # 此处添加
from random import randint # 此处添加

def receive_code(request):
    data = request.GET
    code = data.get('code')
    state = data.get('state')
    print("code:%s  state:%s" %(code, state))

    if not cache.has_key(state): 
        return redirect("index")
    cache.delete(state) 

    apply_access_token_url = "https://www.acwing.com/third_party/api/oauth2/access_token/"

    params = {
        'appid': "4588",
        'secret': "ea08584c059441c297b952e06395ea08",
        'code': code,
    }

    access_token_res = requests.get(apply_access_token_url, params=params).json()

    # 将下面添加到文件中
    access_token = access_token_res['access_token']
    openid = access_token_res['openid']

    players = Player.objects.filter(openid=openid)
    if players.exists(): # 如果用户已经注册，直接登录即可
        login(request, players[0].user)
        return redirect("index")

    get_userinfo_url = "https://www.acwing.com/third_party/api/meta/identity/getinfo/"
    params = {
        "access_token": access_token,
        "openid": openid
    }

    # 获取用户信息
    userinfo_res = requests.get(get_userinfo_url, params=params).json()
    username = userinfo_res['username']
    photo = userinfo_res['photo']

    # 防止用户重名，每次在username后面随机加一位数即可
    while User.objects.filter(username=username).exists():
        username += str(randint(0, 9))

    # 创建用户并且登录
    user = User.objects.create(username=username)
    player = Player.objects.create(user=user, photo=photo, openid=openid)
    login(request, user)


    return redirect("index")
```

这个时候重新登录服务器即可实现一键登录。

保存代码：

```shell linenums="1"
git add .
git commit -m "implement web acwing oauth2 login"
git push
```



## 三、AcApp端AcWing一键登录

<img src="https://cdn.acwing.com/media/article/image/2021/11/25/1_308359794d-acapp.png">

### 1. 申请授权码 code

**请求授权码的API**

```shell linenums="1"
AcWingOS.api.oauth2.authorize(appid, redirect_uri, scope, state, callback);
```

**参数说明**

| 参数         | 是否必须 | 说明                                                         |
| ------------ | -------- | ------------------------------------------------------------ |
| appid        | 是       | 应用的唯一id，可以在AcWing编辑AcApp的界面里看到              |
| redirect_uri | 是       | 接收授权码的地址。需要用对链接进行编码：Python3中使用urllib.parse.quote；Java中使用URLEncoder.encode |
| scope        | 是       | 申请授权的范围。目前只需填userinfo                           |
| state        | 是       | 用于判断请求和回调的一致性，授权成功后后原样返回。该参数可用于防止csrf攻击（跨站请求伪造攻击），建议第三方带上该参数，可设置为简单的随机数（如果是将第三方授权登录绑定到现有账号上，那么推荐用随机数 + user_id作为state的值，可以有效防止CSRF攻击） |
| callback     | 是       | redirect_uri返回后的回调函数                                 |

**返回说明**

用户同意授权后，会将 `code` 和 `state` 传递给 `redirect_uri`。

如果用户拒绝授权，则将会收到如下错误码：

```shell linenums="1"
{
    errcode: "40010"
    errmsg: "user reject"
}
```

### 2. 申请授权令牌 `access_token` 和用户的 `openid`

**请求地址**

```
https://www.acwing.com/third_party/api/oauth2/access_token/
```

**参考示例**

```shell linenums="1"
# 请求方法：GET
https://www.acwing.com/third_party/api/oauth2/access_token/?appid=APPID&secret=APPSECRET&code=CODE
```

**参数说明**

| 参数   | 是否必须 | 说明                                            |
| ------ | -------- | ----------------------------------------------- |
| appid  | 是       | 应用的唯一id，可以在AcWing编辑AcApp的界面里看到 |
| secret | 是       | 应用的秘钥，可以在AcWing编辑AcApp的界面里看到   |
| code   | 是       | 第一步中获取的授权码                            |

**返回说明**

```shell linenums="1"
# 申请成功示例：
{ 
    "access_token": "ACCESS_TOKEN", 
    "expires_in": 7200, 
    "refresh_token": "REFRESH_TOKEN",
    "openid": "OPENID", 
    "scope": "SCOPE",
}

# 申请失败示例：
{
    "errcode": 40001,
    "errmsg": "code expired",  # 授权码过期
}
```

**返回参数说明**

| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| access_token  | 授权令牌，有效期2小时                                        |
| expires_in    | 授权令牌还有多久过期，单位（秒）                             |
| refresh_token | 用于刷新access_token的令牌，有效期30天                       |
| openid        | 用户的id。每个AcWing用户在每个acapp中授权的openid是唯一的,可用于识别用户。 |
| scope         | 用户授权的范围。目前范围为userinfo，包括用户名、头像         |

**刷新access_token的有效期**

- `access_token` 的有效期为2小时，时间较短
- `refresh_token` 的有效期为30天，可用于刷新 `access_token`
- 刷新结果有两种：
    - 如果 `access_token` 已过期，则生成一个新的 `access_token`
    - 如果 `access_token` 未过期，则将当前的 `access_token` 的有效期延长为 `2` 小时



**参考示例**

```shell linenums="1"
# 请求方法：GET
https://www.acwing.com/third_party/api/oauth2/refresh_token/?appid=APPID&refresh_token=REFRESH_TOKEN
```

返回结果的格式与申请access_token相同。

### 3. 申请用户信息

**请求地址**

```
https://www.acwing.com/third_party/api/meta/identity/getinfo/
```

**参考示例**

```shell linenums="1"
# 请求方法：GET
https://www.acwing.com/third_party/api/meta/identity/getinfo/?a
```

**参数说明**

| 参数         | 是否必须 | 说明                     |
| ------------ | -------- | ------------------------ |
| access_token | 是       | 第二步中获取的授权令牌   |
| openid       | 是       | 第二步中获取的用户openid |

**返回说明**

```shell linenums="1"
# 申请成功示例：
{
    'username': "USERNAME",
    'photo': "https:cdn.acwing.com/xxxxx"
}

# 申请失败示例：
{
    'errcode': "40004",
    'errmsg': "access_token expired"  # 授权令牌过期
}
```

### 4. 具体实现

实现过程同上面 二、WEB端AcWing一键登录 十分相似。

**后端实现**

首先将文件夹 `/home/acs/acapp/game/views/settings/acwing/web` 下所有文件复制到文件夹 `/home/acs/acapp/game/views/settings/acwing/acapp` 下面。

然后在文件 `/home/acs/acapp/game/views/settings/acwing/acapp/apply_code.py` 中修改：

```python linenums="1"
from django.http import JsonResponse
from urllib.parse import quote
from random import randint
from django.core.cache import cache

def get_state():
    """
    获取一个8位的每个数值随机在0~8之间的数
    """
    res = ""
    for i in range(8):
        res += str(randint(0, 9))
    return res


def apply_code(request):
    appid = "4588"
    redirect_uri = quote("https://app4588.acapp.acwing.com.cn/settings/acwing/acapp/receive_code/")
    scope = "userinfo"
    state = get_state()

    cache.set(state, True, 7200)

    # 删掉 apply_code_url = "https://www.acwing.com/third_party/api/oauth2/web/authorize/"
    return JsonResponse({
        'result': "success",
        # 删掉掉'apply_code_url': apply_code_url + "?appid=%s&redirect_uri=%s&scope=%s&state=%s" % (appid, redirect_uri, scope, state)
        # 添加下面的参数
        'appid': appid,
        'redirect_uri': redirect_uri,
        'scope': scope,
        'state': state,
    })
```

在文件 `/home/acs/acapp/game/urls/settings/acwing/index.py` 里面修改路由：

```python linenums="1"
from django.urls import path
from game.views.settings.acwing.web.apply_code import apply_code as web_apply_code # 此处修改
from game.views.settings.acwing.web.receive_code import receive_code as web_receive_code # 此处修改
from game.views.settings.acwing.acapp.apply_code import apply_code as acapp_apply_code # 此处添加
from game.views.settings.acwing.acapp.receive_code import receive_code as acapp_receive_code  # 此处添加

urlpatterns = [
    path("web/apply_code/", web_apply_code, name="settings_acwing_web_apply_code"), # 此处修改
    path("web/receive_code/", web_receive_code, name="settings_acwing_web_receive_code"), # 此处修改
    path("acapp/apply_code/", acapp_apply_code, name="settings_acwing_acapp_apply_code"),  # 此处添加
    path("acapp/receive_code/", acapp_receive_code, name="settings_acwing_acapp_receive_code"),  # 此处添加
]
```

这个时候在重启服务后在浏览器中输入 `https://app4588.acapp.acwing.com.cn/settings/acwing/acapp/apply_code/`可以得到结果 `{"result": "success", "appid": "4588", "redirect_uri": "https%3A//app4588.acapp.acwing.com.cn/settings/acwing/acapp/receive_code/", "scope": "userinfo", "state": "88516570"}`。

**前端实现**

在文件 `/home/acs/acapp/game/static/js/src/settings/zbase.js` 中修改：

```javascript linenums="1"
	...
		this.start();
    }

    start() { // 修改这个函数
        if (this.platform === "ACAPP") {
            this.getinfo_acapp();
        } else {
            this.getinfo_web(); 
            this.add_listening_events();
        }
    }

    add_listening_events() {
        let outer = this;
	...
    login() { 
        this.$register.hide();
        this.$login.show();
    }

    // 添加
    acapp_login(appid, redirect_uri, scope, start) {
        let outer = this;
        this.root.AcWingOS.api.oauth2.authorize(appid, redirect_uri, scope, state, function(resp) {
            console.log("called from acapp_login function");
            console.log(resp);
            if (resp.result === "success") {
                outer.username = resp.username;
                outer.photo = resp.photo;
                outer.hide();
                outer.root.menu.show();
            }
        });
    }

    // 添加
    getinfo_acapp() {
        let outer = this;
        $.ajax ({
            url: "https://app4588.acapp.acwing.com.cn/settings/acwing/acapp/apply_code/",
            tpye: "GET",
            success: function(resp) {
                if (resp.result === "success") {
                    outer.acapp_login(resp.appid, resp.redirect_uri, resp.scope, resp.state);
                }
            }
        });
    }

    getinfo_web() { // 修改函数名称
        let outer = this;
        $.ajax({
...
```

修改文件 `/home/acs/acapp/game/views/settings/acwing/acapp/receive_code.py`：

```python linenums="1"
from django.http import JsonResponse # 此处修改
from django.core.cache import cache
import requests
from django.contrib.auth.models import User
from game.models.player.player import Player
# 删除 from django.contrib.auth import login
from random import randint

def receive_code(request):
    data = request.GET

    if "errcode" in data: # 此处修改
        return JsonResponse({
            'result': "apply failed",
            'errcode': data['errcode'],
            'errmsg': data['errmsg'],
        })

    code = data.get('code')
    state = data.get('state')
    print("code:%s  state:%s" %(code, state))

    if not cache.has_key(state): 
        return JsonResponse({ # 此处修改
            'result': "state not exist"
        })
    cache.delete(state) 

    apply_access_token_url = "https://www.acwing.com/third_party/api/oauth2/access_token/"
    
    params = {
        'appid': "4588",
        'secret': "ea08584c059441c297b952e06395ea08",
        'code': code,
    }

    access_token_res = requests.get(apply_access_token_url, params=params).json()

    access_token = access_token_res['access_token']
    openid = access_token_res['openid']

    players = Player.objects.filter(openid=openid)
    if players.exists():
        # 删除login(request, players[0].user) 
        # 删除return redirect("index")
        # 添加下面代码
        player = players[0]
        return JsonResponse({
            'result': "success",
            'username': player.user.username,
            'photo': player.photo,
        })

    get_userinfo_url = "https://www.acwing.com/third_party/api/meta/identity/getinfo/"
    params = {
        "access_token": access_token,
        "openid": openid
    }

    userinfo_res = requests.get(get_userinfo_url, params=params).json()
    username = userinfo_res['username']
    photo = userinfo_res['photo']

    while User.objects.filter(username=username).exists():
        username += str(randint(0, 9))

    user = User.objects.create(username=username)
    player = Player.objects.create(user=user, photo=photo, openid=openid)
    # 删除 login(request, user)


    # 删除return redirect("index")
    # 添加下面代码
    return JsonResponse({
            'result': "success",
            'username': player.user.username,
            'photo': player.photo,
        })
```

重新打包后重启服务，就可以一键登录了。

提交代码：

```shell linenums="1"
git add .
git commit -m "implement acapp acwing auth2 login system"
```