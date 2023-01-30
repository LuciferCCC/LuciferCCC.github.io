**两点注意**：

- 每次更新 `js` 文件之后记着重新打包文件，在文件夹 `/home/acs/acapp/scripts` 下执行：`./compress_game_js.sh ` 即可。现在由于配置到了 `nginx` 上，它需要执行 `python3 manage.py collectstatic`，下面第三步中修改了打包文件的命令，现在两个命令合二为一，只需要<font color=red>在目录 `/acapp/` 下</font>需执行 `./scripts/compress_game_js.sh` 即可。
- 每一次执行 `python3 manage.py collectstatic`，都需要清除浏览器缓存：
    - 硬刷新：`shift + command + r`，可以解决绝大多数问题。
    - 清除缓存并硬刷新：开发者模式下，使用右键点击刷新按钮，选择 `清除缓存并硬刷新`

## 一 、增加容器的映射端口：443(https)、80(http)

- 登录容器，关闭所有运行中的任务。
- 登录<font color=green>运行容器的服务器</font>，然后执行：

```dockerfile
docker commit CONTAINER_NAME django_lesson:1.1  # 将容器保存成镜像，将CONTAINER_NAME替换成容器名称

docker stop CONTAINER_NAME  # 关闭容器

docker rm CONTAINER_NAME # 删除容器

# 使用保存的镜像重新创建容器
docker run -p 20000:22 -p 8000:8000 -p 80:80 -p 443:443 --name CONTAINER_NAME -itd django_lesson:1.1
```

- 去云服务器控制台，在安全组配置中开放80和443端口。

## 二、在 Acwing 上创建 Acapp

<img src="https://cdn.acwing.com/media/article/image/2021/11/12/1_391b8d1443-QQ%E6%88%AA%E5%9B%BE20211112143851.png">

系统分配的域名、nginx配置文件及https证书在如下位置：

<img src="https://cdn.acwing.com/media/article/image/2021/11/12/1_7dff276143-QQ%E5%9B%BE%E7%89%8720211112144041.png">

在 `服务器IP` 一栏填入自己服务器的ip地址。

将 `nginx.conf` 中的内容写入服务器 /etc/nginx/`nginx.conf` 文件中。
将 `acapp.key` 中的内容写入服务器 /etc/nginx/cert/`acapp.key` 文件中。
将 `acapp.pem` 中的内容写入服务器 /etc/nginx/cert/`acapp.pem` 文件中。

然后在服务器中启动 `nginx` 服务：`sudo /etc/init.d/nginx start`

> 成功启动之后会提示 `OK`，如果存在错误则输入 `sudo nginx -s reload`，会提示错在什么地方，如果没问题，则提示 `OK`。
>
> 或者在文件 `/var/log/nginx/error.log` 也可以查看存在的错误。

## 三、修改 django 项目的配置

- 修改文件：`/acs/acapp/acapp/setting.py`，将第二步中系统分配的域名加入到 ``ALLOWED_HOSTS` 列表中，注意只需要添加 `https://` 后面的内容，添加之后是这样子的：`ALLOWED_HOSTS = ["123.60.154.120", "app4588.acapp.acwing.com.cn"]`。
- 修改文件：`/acs/acapp/acapp/setting.py`，将 `DEBUG = True` 修改为 `False`。

- 归档`static`文件：在文件夹 `/acapp` 下执行：`python3 manage.py collectstatic`

> 在每次修改 `js` 文件之后都需要先打包然后执行归档命令，这里将二者命令合二为一，在 `/acapp/scripts/compress_game_js.sh` 中尾部添加命令：
>
> `echo yes | python3 manage.py collectstatic`
>
> 后面仅<font color=red>在目录 `/acapp/` 下</font>需执行 `./scripts/compress_game_js.sh` 即可。

## 四、配置 uwsgi

> 上面的设置，使用户通过 `80 or 443` 端口访问网站的时候使先访问 `nginx` 服务器，而 `uwsgi` 文件就是连接 `nginx` 和 `django` 之间的桥梁。

在文件夹 `/acapp/scripts` 下面添加文件 `uwsgi.ini` 并添加如下内容：

```ini
[uwsgi]
socket          = 127.0.0.1:8000
chdir           = /home/acs/acapp
wsgi-file       = acapp/wsgi.py
master          = true
processes       = 2
threads         = 5
vacuum          = true
```

在文件夹 `/acapp` 下执行：`uwsgi --ini scripts/uwsgi.ini` ，启动 `uwsgi` 服务。

## 五、填写剩余信息

- 标题：应用的名称
- 关键字：应用的标签（选填）
- css地址：css文件的地址，例如：https://app76.acapp.acwing.com.cn/static/css/game.css
- js地址：js文件的地址，例如：https://app76.acapp.acwing.com.cn/static/js/dist/game.js
- 主类名：应用的main class，例如AcGame。
- 图标：4:3的图片
- 应用介绍：介绍应用，支持markdown + latex语法。

## 六、调整代码

打开菜单页面：

之前为了方便调试，打开网页直接进入游戏界面，注释掉了菜单页面。现在去掉注释： `/home/acs/acapp/game/static/js/src/zbase.js`  

```javascript
export class AcGame {
    constructor(id) {   
        this.id = id;
        this.$ac_game = $('#' + id);
        this.menu = new AcGameMenu(this); // 去掉注释
        this.playground = new AcGamePlayground(this);

        this.start();
    }

    start() {

    }
}
```

同时修改文件 `/home/acs/acapp/game/static/js/src/playground/zbase.js` 

```javascript
class AcGamePlayground {
    constructor(root) {
        this.root = root;
        this.$playground = $(`<div class="ac-game-playground"></div>`);

        this.hide(); // 注释

        this.start();
    }

    get_random_color() {
        let colors = ["blue", "red", "pink", "Fuchsia", "green", "yellow", "Aqua"]
        return colors[Math.floor(Math.random() * 7)]
    }

    start() {

    }

    show() { // 打开playground界面
        this.$playground.show();

        /* 加载界面之后才开始初始化！不然窗口缩放之后幕布不会改变大小 */

        this.root.$ac_game.append(this.$playground);

        // 存储界面的宽高，因为后面常用
        this.width = this.$playground.width();
        this.height = this.$playground.height();

        this.game_map = new GameMap(this); // 添加地图
        
        this.players = []; // 添加玩家
        this.players.push(new Player(this, this.width / 2, this.height / 2, this.height * 0.05, "white", this.height * 0.4, true));

        for (let i = 0; i < 5; i ++ ) { // 添加敌人
            this.players.push(new Player(this, this.width / 2, this.height / 2, this.height * 0.05, this.get_random_color(), this.height * 0.4, false)); // 注意敌人是 false
        }
    }

    hide() { // 关闭playground界面
        this.$playground.hide();
    }
}
```

还发现不管窗口页面如何移动，菜单页面永远固定，而且字体不美观，这里进入文件 `/home/acs/acapp/game/static/css/game.css` 文件夹修改：

```css
.ac-game-menu-field {
    width: 20vw;
    position: relative;
    top: 50%; /* 此处修改 */
    left: 50%; /* 此处修改 */
}

.ac-game-menu-field-item {
    height: 6vh; /* 此处修改 */
    width: 18vw;
    color: white;
    font-size: 4vh; /* 此处修改 */
    font-style: italic;
    /* 内边距删掉：padding: 2vh;*/
    cursor: pointer;
    text-align: center;
    background-color: rgba(39, 21, 28, 0.6);
    border-radius: 10px;
    letter-spacing: 0.5vw;
}
```

提交代码：

```shell
git add .
git commit -m "create a acapp"
git push	
```

> 到这里，部署到了 `nginx` 上，所以可以直接通过分配的网址访问。如果要像之前那样通过 `公网IP:端口` 访问需要执行 `python3 manage.py runserver 0.0.0.0:8000`
