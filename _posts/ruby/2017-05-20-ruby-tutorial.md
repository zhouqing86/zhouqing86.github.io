---
layout: post
comments: false
categories: "Ruby"
date: 2017-5-19 18:30:54
title: Ruby基础导航
---

<div id="toc"></div>

由于新项目使用了Ruby On Rails，本着基于项目走深的原则，写个文档先记录总结复习后的基础帮助文档。

### Ruby是一门面向对象语言

```
song = Song.new('Ruby Programming')
```

每个对象有一个唯一的对象标识符。其次，可以定义一些实例变量，这些变量对于每个实例都是独一无二的。为了获取类的状态，需要定义实例方法，类的内部或外部调用实例方法，实例方法改变或者获取类中的变量或者状态。

Ruby首先分配一些内存来保存未初始化的对象，然后调用对象的initialize方法。

在Ruby中，类用于都不是封闭的：你永远可以向一个已有的的类中添加方法。

类的继承定义 class KaraokeSong < Song。

Ruby默认以Object类作为父类。

Ruby不支持多继承，但是其支持任何数量的mixin中引入(include)功能。

Ruby类中定义了三个访问方法来得到3个实例变量的值： attr_reader :name, :artist, :duration, attr_writer,

Ruby中特殊的对象：

- 数值是对象，如1.odd?

- nil是对象，如nil.nil?

### 数组
Array.new可以创建新数组，数组定义了一些列操作方法，如push, shift, pop方法等。

['ant', 'bee', 'cat']与%w{ant bee cat}是一样的。

数组上可以使用find方法:

```
[1, 3, 5, 7, 9].find {|v| v*v > 30 }
```

数组上使用each方法：

```
[1, 3, 5, 7, 9].each {|v| puts v }
```

数组上使用collect方法：

```
[1, 3, 5, 7, 9].collect {|v| v+1}
```

数组上使用inject方法：

```
[1, 3, 5, 7, 9].inject(0) {|sum, element| sum+element }
```

### 序列与区间

#### 创建序列

- 3.times

- 3.upto(6)

- 'a'..'e'

- 9.downto(1)

- 50.step(80, 5)

#### 区间作为条件

- 当区间的第一部分的条件为true时，它们就打开，当区间的第二部分为条件时关闭

```
while line = gets
    puts line if line =~ /start/ .. line =~ /end/
end

```

- 区间作为间隔，`(1..10)  === 5`将返回true

#### 可变长的参数列表

```
def varargs(arg1, *rest)
    "Got #{arg1} and #{rest.join(', ')}"
end
varargs('one', 'two')
varargs('one', 'two', 'three')
```

### 字符串处理

#### 正则匹配

- line =~ |Perl|Python|

- line.sub(/Perl/, 'Ruby')

- line.gsub(/Python/, 'Ruby')

- 使用match或匹配操作符=~(肯定匹配)和!~（否定匹配）对字符串进行匹配

#### 字符串

- 双引号字符串支持许多转义序列，如`\n`, 又如`#{ exper }`序列把任何Ruby代码的值放进字符串中

```
"Seconds/Day: #{24*60*60}"
```

- %q和%Q分别开始界定单引号和双引号的字符串

- 对于多行字符串

```
string = <<END_OF_STRING
    The body of the string
    is the input lines up to
END_OF_STRING
```

- 字符串缩进

```
pring <<-STRING1, <<-STRING2
    Concat
    STRING1
        enate
        STRING2
```

### 代码块

Block在代码中只和方法调用一起出现， block和方法调用的最后一个参数在同一行，并紧跟在其后（或者参数列表的右括号的后面)。

在方法内部，block可以像方法一样被yield语句调用。每执行一次yield，就会调用block中的代码。

```
def three_times
    yeild
    yeild
    yeild
end
three_times { puts 'hello' }
```

block也可以带参数，在函数中使用`yield [参数]`。也可以在函数中`if yield([参数])`。

如[1, 3, 5, 7, 9].find {|v| v*v > 30 } 返回结果为7。


在Ruby库中大量使用了block来实现迭代器，如%w{ant bee cat}.each{|animal| puts annimal}

怎么让方法既支持有block的情况也支持没有block的情况呢，使用`if block_given?`。

Block可以作为闭包：

```
songlist = SongList.new
class JukeboxButton < Button
    def initialize(label, &action)
        super(label)
        @action = action
    end

    def button_pressed
        @action.call(self)
    end
end
start_button = JukeboxButton.new('Start') { songlist.start }
pause_button = JukeboxButton.new('Pause') { songlist.pause }
```

### 链式写法

[3, 1, 7, 0].sort.reverse

链式调用的调试技巧

- 使用tap方法，tap 是 Object 的 instance_method，传递 self 给一个 block，最后返回 self

    ```
    (1..10)                .tap {|x| puts "original: #{x.inspect}"}
      .to_a                .tap {|x| puts "array: #{x.inspect}"}
      .select {|x| x%2==0} .tap {|x| puts "evens: #{x.inspect}"}
      .map { |x| x*x }     .tap {|x| puts "squares: #{x.inspect}"}
    ```
- 使用try方法，try 可以让你调用对象方法时不用担心对象是nil并抛出异常

    `@person && @persion.name` 可以改写成 `@person.try(:name)`

    try还可以这么使用:

    ```
    Person.try(:find, 1)
    @people.try(:collect) {|p| p.name}
    ```

### 参考资料

- Programming Ruby

- [Rails 技巧之 tap & try](https://ruby-china.org/topics/5348)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
