# Markdown里输入数学公式

### 一. 使用图片

使用[CodeCogs](http://latex.codecogs.com/eqneditor/editor.php?lang=zh-cn)线上生成公式图片, 再在markdown里引用图片.

```markdown
![](http://latex.codecogs.com/svg.latex?R_i=\frac{P_{i_{0}}-P_{i_{m}}}{\sum_{i=0}^{n}(P_{i_{0}}-P_{i_{m}})})
```

有的markdown编辑器可能会解析错误, 因为公式里有右括号`)`, 匹配了引用图片的左括号`(`. 这时候可以使用`img`标签:

```html
<img src='http://latex.codecogs.com/svg.latex?R_i=\frac{P_{i_{0}}-P_{i_{m}}}{\sum_{i=0}^{n}(P_{i_{0}}-P_{i_{m}})}'/>
```

显示效果如下:
<img src='http://latex.codecogs.com/svg.latex?R_i=\frac{P_{i_{0}}-P_{i_{m}}}{\sum_{i=0}^{n}(P_{i_{0}}-P_{i_{m}})}'/>

### 二. MathJax插件

[MathJax](https://www.mathjax.org)允许你在你的网页中包含公式，无论是使用LaTeX、MathML或者AsciiMath符号，这些公式都会被javascript处理为HTML、SVG或者MathML符号。[MathJax中文指南](https://mathjax-chinese-doc.readthedocs.io/en/latest/start.html)

但是github, gitlab都不支持引入js, 这个方法在github, gitlab上都无效. 我本地的gitbook是可以的, 或者说在自己的网站上都可以.

在markdown中添加引入MathJax的js文件:

```html
<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
```

然后就可以编写自己的数学公式了, 比如数学公式:

```latex
\frac{P_{i_{0}}-P_{i_{m}}}{\sum_{i=0}^{n}(P_{i_{0}}-P_{i_{m}})}
```

在markdown里最好把`_`给转译下, 变成`\_`; `\( \)`表示行内, 即公式显示在当前行中, 不会另起一行, 同样在markdown里需要转译下, 变成`\\( \\)`, 则上面的公式为:

```
\\(R\_i=\frac{P\_{i\_{0}}-P\_{i\_{m}}}{\sum\_{i=0}^{n}(P\_{i\_{0}}-P\_{i\_{m}})}\\)
```

效果跟方法一差不多, 不过是文字不是图片, 同时右击还有选项卡, 效果同方法三.

### 三. Install GitBook-KaTex Plugin

修改`book.json`, 在`plugins`字段添加`katex`插件:

```json
{
    "plugins": ["katex"]
}
```

然后安装一下:

```sh
gitbook install .
```

这时候会多出来一个`node_modules`目录, 在`.gitignore`里忽略掉. 使用`{% math %} {% endmath %}`表示行内, 或者是公式两边各加两个`$`分割, 还是上面的例子, 这种方法的公式为:

```
$$R_i=\frac{P_{i_{0}}-P_{i_{m}}}{\sum_{i=0}^{n}(P_{i_{0}}-P_{i_{m}})}$$
```

效果: $$R_i=\frac{P_{i_{0}}-P_{i_{m}}}{\sum_{i=0}^{n}(P_{i_{0}}-P_{i_{m}})}$$

也可以安装`mathjax`插件, 不过加载速度略慢.

**NOTE**:

* 在GitHub上看不出效果, 可以到[点这](https://lowzj.com/notes/math-formula.html#三-install-gitbook-katex-plugin).
* vim支持LaTex语法, 可以下载[vim-syntax-markdonw](https://github.com/drmingdrmer/vim-syntax-markdown/blob/master/syntax/markdown.vim)到你的`./vim/syntax/`目录下. [GitHub地址](https://github.com/drmingdrmer/vim-syntax-markdown)

