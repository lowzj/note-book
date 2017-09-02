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

<script type="text/javascript" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
效果跟方法一差不多, 不过是文字不是图片, 同时右击攻击还有选项卡: \\(R\_i=\frac{P\_{i\_{0}}-P\_{i\_{m}}}{\sum\_{i=0}^{n}(P\_{i\_{0}}-P\_{i\_{m}})}\\)
