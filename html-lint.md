### HTML代码检查工具对比

相比Javascript代码检查工具，HTML代码检查工具的位置略显尴尬：HTML比Javascript出现更早，在互联网发展的早期，使用量也大大超过后者。这导致HTML相关的工具自然分为两种：

1. 较早时期使用C、Perl等开发的标记语言Validator
2. 基于nodejs的lint、format工具

nodejs的出现引发了前端工具链的爆发，几乎所有主流的JS检查工具都出现在这之后，但HTML的特殊性使得我们在讨论其相关工具的时候无法忽视第1类的存在。

#### 对比角度

* 解决的问题

* 规则实现及自定义

* 使用及配置

* 对模板语言的支持

* format功能

* 性能

#### AriaLinter

https://github.com/globant-ui/arialinter

* 解决的问题

	语义层面（针对dom树）的检查，基于jsdom获取页面执行的结果document，并对其内容、结构进行检查。

* 规则实现及自定义

	偏语义的规则，如`tableMustHaveTh`、`inputImageHasAlt`、`htmlLang`等。

	不支持自定义规则。

* 使用及配置

	Grunt, Nodejs & cli

	Grunt, Nodejs方式调用可传入规则配置

	不支持配置文件、及行内注释配置

* 对模板语言的支持

	支持template参数，开启时部分规则不做检查

* format

	不支持

* 性能

	较差

#### htmllint/htmllint

https://github.com/htmllint/htmllint

* 解决的问题

	代码&语义层面的检查，对代码进行parse，parse完成后分别进行逐行lint及逐节点lint

* 规则实现及自定义

	https://github.com/htmllint/htmllint/wiki/Options

	偏代码格式的规则，如`tag-name-lowercase`，`attr-no-dup`，也有一些简单的如`doctype-html5`偏dom结构的规则

	支持自定义规则

* 使用及配置

	Grunt, Nodejs

	Grunt, Nodejs方式调用可传入规则配置

	不支持配置文件，支持行内注释配置

* 对模板语言的支持

	无

* format

	不支持

* 性能

	基于htmlparser2，性能较好，优于jsdom

#### tidy

* [tidy-html5](https://github.com/htacg/tidy-html5)

	http://www.html-tidy.org/

* [petdance/html-tidy](https://github.com/petdance/html-tidy)

* [vavere/htmltidy](https://github.com/vavere/htmltidy)

* 解决的问题

	HTML语法检查及修复，而不是代码风格，做的事是将HTML代码parse为语法树后再stringify一次，对一些简单的语法错误给出warning，并输出格式化后的内容。

* 规则实现及自定义

	规则较少，且不支持自定义。

* 使用及配置

	cli工具，C实现。安装相对繁琐。支持自定义配置文件。

* 对模板语言的支持

	无

* format

	支持

* 性能

	非常好

#### HTMLHint

https://github.com/yaniswang/HTMLHint

国人出品的HTML代码检查工具，介绍是“A Static Code Analysis Tool for HTML”。

* 解决的问题

	相对完善的HTML代码风格检查。

* 规则实现及自定义

	与AriaLinter相反，仅在对代码进行parse的过程中进行检查。因而相对偏重语法层面风格检查，如`tagname-lowercase`，但也实现了很多如`id-unique`这种偏语义的规则。

	支持自定义rule，`HTMLHint.addRule({...})`

* 使用及配置

	Browser, Nodejs & cli

	支持调用方法时传入配置及配置文件

* 对模板语言的支持

	无

* format

	不支持

* 性能

	较好，基于自己实现的HTML parser，仅有parse过程，无AST/Document的查找过程。

设计不合理之处：

1. 对于语义层面的rule实现不自然

2. rule仅支持配置开启/关闭

#### htmlcs

https://github.com/ecomfe/htmlcs

支持对script及style内容使用外部linter进行检查