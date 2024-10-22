
# 前言


之前开发的接口需要用到json加签，有一次对接JAVA时，签名怎么都过不了，仔细对比了字符串，发现是**PHP进行json\_encode时，会将浮点型所有无意义的0给去掉(echo和var\_dump也会)**，而JAVA那边没有。遂在文档中写下： “json中请把无意义的0去掉”。 \#doge


最近又遇到这个事情，**需求直接要求：显示字符型，且精度要保留两位小数**，于是不得不开始研究PHP的json中，浮点型的精度该如何保留的问题。


删除无意义0的原理在这里：[文内跳转：原理\-PHP中浮点型的显示](https://github.com)


需求的解决方法在这里：[文内跳转：字符串处理\-正则](https://github.com)


下面是整体的解决过程和相关原理。


# 解决方案


## json\_encode常量参数(无法解决)


### 相关知识


json\_encode的函数原型如下：
`json_encode(mixed $value, int $flags = 0, int $depth = 512): string|false`


众所周知，json\_encode的第一个进阶用法，就是它的第二个参数flags，也就是“可选的json编码方式”，各种奇妙的常量。比如我最长用到的，`JSON_UNESCAPED_UNICODE`，让json不自动进行unicode转换，直接输出中文。所以第一个想到的，就是查看有没有对应的常量参数。


查看源码，json的常量参数都放在 `php-src/ext/json/php_json.h` 中，如下：



```


|  | /* json_encode() options */ |
| --- | --- |
|  | #define PHP_JSON_HEX_TAG                    (1<<0) |
|  | #define PHP_JSON_HEX_AMP                    (1<<1) |
|  | #define PHP_JSON_HEX_APOS                   (1<<2) |
|  | #define PHP_JSON_HEX_QUOT                   (1<<3) |
|  | #define PHP_JSON_FORCE_OBJECT               (1<<4) |
|  | #define PHP_JSON_NUMERIC_CHECK              (1<<5) |
|  | #define PHP_JSON_UNESCAPED_SLASHES          (1<<6) |
|  | #define PHP_JSON_PRETTY_PRINT               (1<<7) |
|  | #define PHP_JSON_UNESCAPED_UNICODE          (1<<8) |
|  | #define PHP_JSON_PARTIAL_OUTPUT_ON_ERROR    (1<<9) |
|  | #define PHP_JSON_PRESERVE_ZERO_FRACTION     (1<<10) |
|  | #define PHP_JSON_UNESCAPED_LINE_TERMINATORS (1<<11) |


```

PHP\_JSON\_UNESCAPED\_UNICODE，恰好对应的就是256，二进制的设计是为了他们可以方便的复合使用。写法也很多变，比如`json_encode($data, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES)`,`json_encode($data, JSON_UNESCAPED_UNICODE + JSON_UNESCAPED_SLASHES)`,`json_encode($data, 256 + 64)`。都是一样的实现。


[PHP json\_encode中文文档](https://github.com)
[PHP json\_encode常量文档](https://github.com)


[文内跳转：常量版本适用性](https://github.com)


其中和数字有关的，就是`PHP_JSON_NUMERIC_CHECK`，以及`PHP_JSON_PRESERVE_ZERO_FRACTION`。



```


|  | // 将所有数字字符串编码成数字（numbers）。 |
| --- | --- |
|  | // Encodes numeric strings as numbers. |
|  | JSON_NUMERIC_CHECK (int) |
|  |  |
|  | // 确保 float 值始终编码为为 float 值。 |
|  | // Ensures that float values are always encoded as a float value. |
|  | JSON_PRESERVE_ZERO_FRACTION (int) |


```

### 做排列组合试验



```


|  | $str_arr = [ |
| --- | --- |
|  | 'str1' => '1', |
|  | 'str2' => '1.0', |
|  | 'str3' => '1.00', |
|  | 'str4' => '1.1', |
|  | 'str5' => '1.10', |
|  | 'str6' => '1.110' |
|  | ]; |
|  | $s_j1 = json_encode($str_arr, JSON_NUMERIC_CHECK); |
|  | $s_j2 = json_encode($str_arr, JSON_PRESERVE_ZERO_FRACTION); |
|  | $s_j3 = json_encode($str_arr, JSON_NUMERIC_CHECK | JSON_PRESERVE_ZERO_FRACTION); |
|  | echo $s_j1,PHP_EOL; |
|  | echo $s_j2,PHP_EOL; |
|  | echo $s_j3,PHP_EOL; |
|  | echo PHP_EOL; |
|  |  |
|  | $float_arr = [ |
|  | 'f1' => 1, |
|  | 'f2' => 1.0, |
|  | 'f3' => 1.00, |
|  | 'f4' => 1.1, |
|  | 'f5' => 1.10, |
|  | 'f6' => 1.110 |
|  | ]; |
|  | $f_j1 = json_encode($float_arr, JSON_NUMERIC_CHECK); |
|  | $f_j2 = json_encode($float_arr, JSON_PRESERVE_ZERO_FRACTION); |
|  | $f_j3 = json_encode($float_arr, JSON_NUMERIC_CHECK | JSON_PRESERVE_ZERO_FRACTION); |
|  | echo $f_j1,PHP_EOL; |
|  | echo $f_j2,PHP_EOL; |
|  | echo $f_j3,PHP_EOL; |


```

### 结果



```


|  | {"str1":1,"str2":1,"str3":1,"str4":1.1,"str5":1.1} |
| --- | --- |
|  | {"str1":"1","str2":"1.0","str3":"1.00","str4":"1.1","str5":"1.10"} |
|  | {"str1":1,"str2":1.0,"str3":1.0,"str4":1.1,"str5":1.1} |
|  |  |
|  | {"f1":1,"f2":1,"f3":1,"f4":1.1,"f5":1.1} |
|  | {"f1":1,"f2":1.0,"f3":1.0,"f4":1.1,"f5":1.1} |
|  | {"f1":1,"f2":1.0,"f3":1.0,"f4":1.1,"f5":1.1} |


```

### 结论


可以看到`JSON_NUMERIC_CHECK`正如文档描述中的那样，将所有数字字符串都编码成了数字，**无意义的0仍旧会被处理掉**。


而`JSON_PRESERVE_ZERO_FRACTION`的表现形式就有些奇怪，**只能在有第一位小数且为0时，只保留一位0**。


[文内跳转：JSON\_PRESERVE\_ZERO\_FRACTION的处理](https://github.com)


显然，flags是无法满足需求的。


## 配置项"serialize\_precision"("precision")(无法解决)


文档中有这么一句话


如果参数是 array 或 object，则会递归序列化。


编码受传入的 flags 参数影响，**此外浮点值的编码依赖于 serialize\_precision**。


[serialize\_precision文档位置](https://github.com)



```


|  | serialize_precision int |
| --- | --- |
|  | 序列化浮点数时存储的有效数字的位数。-1 表示将使用增强算法来四舍五入此类数字。 |


```

PHP中，`serialize_precision`配置项用于序列化时控制浮点数的精度，而`precision`用于平常显示时的控制。


我们取一个数字，`echo json_encode(17.2);`，将`serialize_precision`，从低到高设置。得到下面的结果：



```


|  | 0   2.0e+1 |
| --- | --- |
|  | 1   2.0e+1 |
|  | 2   17 |
|  | 3   17.2 |
|  | 4   17.2 |


```

可以比较清楚的看出这个配置的效果了，而且显然，无法达成需求。


### 题外话：


测试时发现，在PHP7\.1以上的版本中，如果将`serialize_precision`的数值设置为很大，比如`5.*`版本默认的17，得到的结果是: `17.199999999999999`。`precision`同理，作用于`echo`,`var_dump`,`print_r`等。


所以建议日常使用，设置为默认的\-1就好。


## :[樱花宇宙官网](https://yzygzn.com)字符串处理\-正则


如此来看，从编码配置层面似乎无法解决这个需求了，那么就使用最简单直接的办法： 用正则，直接对字符串下手。



```


|  | foreach ($data as &$item) { |
| --- | --- |
|  | if (is_numeric($item)) { |
|  | $item = sprintf("%.2f", $item); |
|  | } |
|  | } |
|  | $json = json_encode($data); |
|  | // 浮点型转换为数值型 |
|  | $pattern = '/"(\d+\.\d+)"/'; |
|  | $replacement = '$1'; |
|  | $new_json = preg_replace($pattern, $replacement, $json); |


```

这段函数，是把数值全部先转换为保留2位小数的字符串，进行json\_encode后，再把字符串中所有带"."，左右是数字的，外层的双引号去掉。


如果你的json更为复杂，需要对正则进行调整。


# 原理\-PHP中浮点型的显示


我们来看这么一段代码，猜测下他的输出结果会是什么：



```


|  | echo 1.0; |
| --- | --- |
|  | var_dump(1); |
|  | var_dump(1.0); |
|  | var_dump(1.0 === 1); |
|  | var_dump(1.00 === 1.0); |


```

结果:



```


|  | 1 |
| --- | --- |
|  | int(1) |
|  | float(1) |
|  | bool(false) |
|  | bool(true) |


```

那么，为什么会出现`float(1)`,`1.00 === 1.0`这样奇怪的输出呢？原因在于PHP内核中变量容器Zval（Zend value）的实现，以及显示处理。


PHP是一个弱类型语言，一个变量，可以是任何类型，这也得益于Zval的实现。Zval，也就是\_zval\_struct这个结构体，主要记录了三块东西：值，类型，引用计数。并没有“显示精度”这种属性和配置。(引用计数和垃圾回收有关)


所以在var\_dump时，显示的是变量的类型float，以及和存储的值，最近似的有意义的数值，也就是float(1\)。而使用\=\=\=对比时，存储的值相等，类型也相等，自然就会显示成true。


## 对应源码


### a) 对浮点型的输出函数 smart\_str\_append\_double



```


|  | // Zend\zend_smart_str.c |
| --- | --- |
|  | ZEND_API void ZEND_FASTCALL smart_str_append_double( |
|  | smart_str *str, double num, int precision, bool zero_fraction) { |
|  | char buf[ZEND_DOUBLE_MAX_LENGTH]; |
|  | /* Model snprintf precision behavior. */ |
|  | zend_gcvt(num, precision ? precision : 1, '.', 'E', buf); |
|  | smart_str_appends(str, buf); |
|  | if (zero_fraction && zend_finite(num) && !strchr(buf, '.')) { |
|  | smart_str_appendl(str, ".0", 2); |
|  | } |
|  | } |


```

JSON\_PRESERVE\_ZERO\_FRACTION 是在这里进行的影响，会在最终判断是否整形，并加".0"


### b) smart\_str\_append\_double 的引用部分



```


|  | // ext\standard\var.c |
| --- | --- |
|  | PHPAPI zend_result php_var_export_ex(zval *struc, int level, smart_str *buf) { |
|  | ... |
|  | case IS_DOUBLE: |
|  | smart_str_append_double( |
|  | buf, Z_DVAL_P(struc), (int) PG(serialize_precision), /* zero_fraction */ true); |
|  | break; |
|  | ... |
|  | } |


```


```


|  | // Zend\zend_ast.c |
| --- | --- |
|  | static ZEND_COLD void zend_ast_export_zval(smart_str *str, zval *zv, int priority, int indent) { |
|  | ... |
|  | case IS_DOUBLE: |
|  | smart_str_append_double( |
|  | str, Z_DVAL_P(zv), (int) EG(precision), /* zero_fraction */ false); |
|  | break; |
|  | ... |
|  | } |


```

可以很明显的看到，serialize\_precision和precision，就是从这里进行的引入。


### c) smart\_str\_append\_double 对浮点型字符串的处理函数： zend\_gcvt



```


|  | // Zend\zend_strtod.c |
| --- | --- |
|  | ZEND_API char *zend_gcvt(double value, int ndigit, char dec_point, char exponent, char *buf) { |
|  | ... |
|  | if ((decpt >= 0 && decpt > ndigit) || decpt < -3) { /* use E-style */ |
|  | /* exponential format (e.g. 1.2345e+13) */ |
|  | ... |
|  | } else if (decpt < 0) { |
|  | /* standard format 0. */ |
|  | *dst++ = '0';   /* zero before decimal point */ |
|  | *dst++ = dec_point; |
|  | do { |
|  | *dst++ = '0'; |
|  | } while (++decpt < 0); |
|  | src = digits; |
|  | while (*src != '\0') { |
|  | *dst++ = *src++; |
|  | } |
|  | *dst = '\0'; |
|  | } else { |
|  | /* standard format */ |
|  | for (i = 0, src = digits; i < decpt; i++) { |
|  | if (*src != '\0') { |
|  | *dst++ = *src++; |
|  | } else { |
|  | *dst++ = '0'; |
|  | } |
|  | } |
|  | if (*src != '\0') { |
|  | if (src == digits) { |
|  | *dst++ = '0';   /* zero before decimal point */ |
|  | } |
|  | *dst++ = dec_point; |
|  | for (i = decpt; digits[i] != '\0'; i++) { |
|  | *dst++ = digits[i]; |
|  | } |
|  | } |
|  | *dst = '\0'; |
|  | } |
|  | zend_freedtoa(digits); |
|  | return (buf); |
|  | } |


```

e的写法，清除无意义的0，在这里被实现。


## 如何显示精度


如果要显示确切的精度，只能转换为字符串类型，有两种方法：



```


|  | $number = 1; |
| --- | --- |
|  | echo sprintf("%.2f", $number); |
|  | echo number_format($number, 2, '.', ''); |


```

两种方法都在PHP4的版本实装，可以放心使用。


需要注意的是，如果本身的位数超过精度，这两种方法**都会四舍五入**。


另外，`number_format`的第三个参数为“小数点符号”，**第四个参数为“千位分隔符”**。默认分别是"."和","。尤其是需要进行数字计算和正常显示时，需要注意“千位分隔符”的设置。


## 关于"double"和"float"


PHP中的浮点型，是使用c中的double型实现的，全部都是遵循 IEEE754 标准，64位的双精度浮点数，不存在单精度。


在PHP中，double和float的命名使用的很混乱。在源码中，多见double，类型判断用的也是`IS_DOUBLE`。但**在7以后，显示定义的类型，必须使用float。比如 `function(float $num): float`**，这似乎是为了与其他语言的命名方式保持一致。


获取类型的相关函数，使用不同版本进行了简单测试，很奇怪，尽量别用8\.2：



```


|  | gettype(1.0); // double |
| --- | --- |
|  | var_dump(1.0); // 8.2版本显示为double，8.3及其他版本都是float，同时8.2版本也多出了文件位置的输出 |


```

其他函数:



```


|  | // 都只是别名，功能一致 |
| --- | --- |
|  | is_float(); |
|  | is_double(); |
|  |  |
|  | floatval(); |
|  | doubleval(); |
|  |  |
|  | ... |


```

## 浮点型的对比和精确计算


[PHP float型文档](https://github.com)


从float文档中可以看到，由于精度问题，官方是不支持把浮点型进行直接对比和计算的，`“永远不要相信浮点数结果精确到了最后一位，也永远不要比较两个浮点数是否相等”`。(例如，0\.1 \+ 0\.2 在计算机中并不等于 0\.3，而是等于 0\.30000000000000004‌)


一般正常的四则运算其实影响不大，但如果对精度有很高的要求，推荐使用**BC系列函数**，或者**GMP函数**。


对比前，先使用`round()`函数，将浮点型进行四舍五入处理。(和官方给的处理方式类似，但更好理解)



```


|  | $x = 8 - 6.4;  // which is equal to 1.6 |
| --- | --- |
|  | $y = 1.6; |
|  | var_dump($x == $y); // is not true |
|  |  |
|  | PHP thinks that 1.6 (coming from a difference) is not equal to 1.6. To make it work, use round() |
|  |  |
|  | var_dump(round($x, 2) == round($y, 2)); // this is true |
|  |  |
|  | This happens probably because $x is not really 1.6, but 1.599999.. and var_dump shows it to you as being 1.6. |


```

## float型的下划线


7\.4以后，支持对浮点型添加下划线，只是增加可读性，和千分符类似：



```


|  | 1_000.0 == 1000.0; // true |
| --- | --- |


```

# 其他


## json\_encode常量参数版本适用性


* PHP\_JSON\_HEX\_TAG、PHP\_JSON\_HEX\_AMP、PHP\_JSON\_HEX\_APOS、PHP\_JSON\_HEX\_QUOT、PHP\_JSON\_FORCE\_OBJECT、PHP\_JSON\_NUMERIC\_CHECK、PHP\_JSON\_UNESCAPED\_SLASHES、PHP\_JSON\_PRETTY\_PRINT、PHP\_JSON\_UNESCAPED\_UNICODE：在 PHP 5\.3\.0 及以上版本可用。
* PHP\_JSON\_PARTIAL\_OUTPUT\_ON\_ERROR：在 PHP 5\.5\.0 及以上版本可用。
* PHP\_JSON\_PRESERVE\_ZERO\_FRACTION：在 PHP 5\.6\.6 及以上版本可用。
* PHP\_JSON\_UNESCAPED\_LINE\_TERMINATORS：在 PHP 7\.3\.0 及以上版本可用。


# 对象的序列化处理\-JsonSerializable（json\_encode的其他进阶用法）


阅读json\_encode文档时，还可以发现，


[JsonSerializable 文档位置](https://github.com)


**实现 JsonSerializable 的类可以 在 json\_encode() 时定制他们的 JSON 表示法(序列化)。**


go的json序列化比较常见，可以结合理解。


JAVA也有同名JsonSerializable方法，是将类信息也带入json中，可以实现反序列化，不常用。



```


|  | class IDou implements JsonSerializable |
| --- | --- |
|  | { |
|  | public function __construct(protected $name, protected $year) |
|  | {} |
|  |  |
|  | public function jsonSerialize() |
|  | { |
|  | return ['name' => $this->name, 'year' => $this->year]; |
|  | } |
|  | } |
|  |  |
|  | echo json_encode(new IDou('cxk', 2.5)); |


```

结果：



```


|  | {"name":"cxk","year":2.5} |
| --- | --- |


```

