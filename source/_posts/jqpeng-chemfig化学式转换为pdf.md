---
title: chemfig化学式转换为pdf
tags: ["chemfig","jqpeng"]
categories: ["博客","jqpeng"]
date: 2021-04-25 19:37
---
文章作者:jqpeng
原文链接: [chemfig化学式转换为pdf](https://www.cnblogs.com/xiaoqi/p/chemfig.html)

## SMILES 与 chemfig

针对化学分子结构，可以用`SMILES` （用ASCII字符串明确描述分子结构的规范）来定义。


> SMILES（Simplified molecular input line entry specification），简化分子线性输入规范，是一种用ASCII字符串明确描述分子结构的规范。SMILES由Arthur Weininger和David Weininger于20世纪80年代晚期开发，并由其他人，尤其是日光化学信息系统有限公司（Daylight Chemical Information Systems Inc.），修改和扩展


但是`SMILES`需要一定的化学基础，而`chemfig`则是从图形层面定义了一套规范，方便定义和显示化学式。当然，`SMILES`可以方便的转换到`chemfig`.

比如:


    CN1C=NC2=C1C(=O)N(C(=O)N2C)C


可通过mol2chemfig进行转换：


    mol2chemfig -wz -i direct 'CN1C=NC2=C1C(=O)N(C(=O)N2C)C' > caffeine.tex


转换后：


    \chemfig{-[:138]N-[:84]=^[:156]N-[:228]=[:300](-[:240](-[:180]N(-[:240]%
    )-[:120](-[:60]N(-[:120])-)=[:180]O)=[:300]O)-[:12]\phantom{N}}


## chemfig 转换为pdf

我们可以通过pdflatex（textlive的一个工具）来转换tex为pdf：

拉取txtlive镜像：


    docker pull listx/texlive:2020
    docker run -it --rm -v `pwd`:/app listx/texlive:2020 bash



然后用pdflatex转换。首先，我们生成一个tex文件`test.tex`，一个空tex文件，使用`mol2chemfig`(可从mol2chemfig下载),中间放上`\chemfig{H_3C-[:30]N**6(-(=O)-(**5(-N(-CH_3)--N-))--N(-CH_3)-(=O)-)}`，


    \documentclass{minimal}
    \usepackage{xcolor, mol2chemfig}
    \usepackage[margin=(margin)spt,papersize={%(width)spt, %(height)spt}]{geometry}
    
    \usepackage[helvet]{sfmath}
    \setcrambond{2.5pt}{0.4pt}{1.0pt}
    \setbondoffset{1pt}
    \setdoublesep{2pt}
    \setatomsep{%(atomsep)spt}
    \renewcommand{\printatom}[1]{\fontsize{8pt}{10pt}\selectfont{\ensuremath{\mathsf{#1}}}}
    
    \setlength{\parindent}{0pt}
    \setlength{\fboxsep}{0pt}
    \begin{document}
    \vspace*{\fill}
    \vspace{-8pt}
    \begin{center}
    
    \chemfig{H_3C-[:30]N**6(-(=O)-(**5(-N(-CH_3)--N-))--N(-CH_3)-(=O)-)}
    
    \end{center}
    \vspace*{\fill}
    \end{document}
    


然后执行转换：


    pdflatex -interaction=nonstopmode  test.tex


等待1~2s，可以看到生成的pdf，打开：

![PDF](https://gitee.com/jadepeng/pic/raw/master/pic/2021/4/25/1619350402339.png)

如何返回给前端呢，可以读取文件，然后转换为base64，python代码：


    pdfstring = open('test.pdf').read()
    encoded = base64.encodestring(pdfstring)
    pdflink = "data:application/pdf;base64,{}".format(encoded)
    


* * *

感谢您的认真阅读。

如果你觉得有帮助，欢迎点赞支持！

不定期分享软件开发经验，欢迎关注作者, 一起交流软件开发：

