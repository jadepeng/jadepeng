---
title: Latex 公式在线可视化编辑器
tags: ["latex","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-04-21 09:00
---
文章作者:jqpeng
原文链接: [Latex 公式在线可视化编辑器](https://www.cnblogs.com/xiaoqi/p/latex-editor.html)

## 寻觅

最近的一个demo需要用到Latex公式在线编辑器，从搜索引擎一般会得到类似[http://latex.codecogs.com/eqneditor/editor.php](http://latex.codecogs.com/eqneditor/editor.php)的结果，这个编辑器的问题在于使用成本高，并且界面不美观。  
![codecogs](https://ooo.0o0.ooo/2017/04/20/58f847b5b0998.jpg "codecogs")

继续探寻，发现了[wiris Editor](http://www.wiris.com/editor/demo/en/examples)：  
![wiris Editor](https://ooo.0o0.ooo/2017/04/20/58f8474b37a60.jpg "wiris")

支持mathml和latex：  
![wiris Editor](https://ooo.0o0.ooo/2017/04/20/58f847990789b.jpg "mathml和latex")

那么就它了！

## 选型

首先，我们不会直接使用这个编辑器，只是在编辑公式的时候才使用，所以要选择合适的版本。  
![wiris Editor](https://ooo.0o0.ooo/2017/04/20/58f849cd99f29.jpg "1492666829731")  
 以前用过CKEditor，所以就这它了！选用[java版本](http://www.wiris.com/en/downloads/files/1376/011ckeditor/java-demo_ckeditor_wiris4-4.2.0.1365.zip)  
 我们的数据已经是latex的，在wiris 编辑器显示需要注意latex需要用两个$$包括起来  
 例如:


    The history of $$\sqrt(2)$$.


但是CK版本的wiris对latex的支持是非可视化支持，在编辑器里输入latex还是显示为latex：  
![enter description here](https://ooo.0o0.ooo/2017/04/20/58f8490c90a9e.jpg "1492666637204")

将焦点移动到$$内部，再点击按钮出现wiris的公式编辑器：

![enter description here](https://ooo.0o0.ooo/2017/04/20/58f84a34a07e2.jpg "1492666933326")  
 这种设计适合对latex熟悉的人员，可以裸写latex，同时对不熟悉的人来说，可以使用公式编辑器。但是，这样不直观啊！你让不会latex的看到的就一堆符号！

## 适配

简单试用可以发现，如果直接使用公式编辑器插入公式，是直观显示的：  
![enter description here](https://ooo.0o0.ooo/2017/04/20/58f84c7433958.jpg "1492667508804")

可以看到保存的时候，mathml是：


    <math class="wrs_chemistry" xmlns="http://www.w3.org/1998/Math/MathML"><msqrt>	<mn>2</mn></msqrt>
    </math>


那么在latex输入情况下呢：


    <math xmlns="http://www.w3.org/1998/Math/MathML"><semantics>	<mrow>		<msqrt><mo>(</mo></msqrt><mn>2</mn><mo>)</mo>	</mrow>	<annotation encoding="LaTeX">\sqrt(2)</annotation></semantics>
    </math>


原来问题在这里，正是mathML的区别导致处理的区别。也就是说一开始就生成不带LaTeX的mathML，然后再放入编辑器。简单查看代码，可以知道先调用wrs\_endParse，再wrs\_initParse就可以了。


    CKEDITOR.on("instanceReady", function(event){	CKEDITOR.instances.example.focus();	var mathxml = wrs_endParse("已知向量$$\\vec{a}=(\\sqrt{3},2)$$,$$\\vec{b}=(0,-2)$$,向量$$\\vec{c}=(k,\\sqrt{2})$$.$$\\vec{a}-1\\vec{b}$$与$$\\vec{d}$$共线,$$k=$$__.");	CKEDITOR.instances.example.setData(wrs_initParse(mathxml));	// 等待完成	window.setTimeout(updateFunction,0);});


![Latex](https://ooo.0o0.ooo/2017/04/21/58f954fc81b80.jpg "Latex可视化")

直观显示没问题了，但是mathml如何再转换成Latex呢？core.js里的wrs\_parseMathmlToLatex函数是直接从mathml里将<annotation encoding="LaTeX">。。。</annotation>里的内容提取出来：


    function wrs_parseMathmlToLatex(content, characters){
        ....
        var openTarget = characters.tagOpener + 'annotation encoding=' + characters.doubleQuote + 'LaTeX' + characters.doubleQuote + characters.tagCloser;
     
            mathml = content.substring(start, end);
    
            startAnnotation = mathml.indexOf(openTarget);	// 包含 encoding=latex，保留latex
            if (startAnnotation != -1){
                startAnnotation += openTarget.length;
                closeAnnotation = mathml.indexOf(closeTarget);
                var latex = mathml.substring(startAnnotation, closeAnnotation);
                if (characters == _wrs_safeXmlCharacters) {
                    latex = wrs_mathmlDecode(latex);
                }
                output += '$$' + latex + '$$';
                // Populate latex into cache.
                wrs_populateLatexCache(latex, mathml);
            }else{
                output += mathml;
            }
       ......
    }


但是现在的mathml不包含这个信息，如何处理？查看官方文档，发现有一个mathml2latex的服务，查看官方给的java demo里servlet并不包含这个服务，但是jar包里存在代码，于是自己封装一个servlet即可：


    public class ServiceServlet extends com.wiris.plugin.dispatchers.MainServlet {
    
        @Override
        public void doGet(javax.servlet.http.HttpServletRequest request, javax.servlet.http.HttpServletResponse response)
                throws ServletException, IOException {
            PluginBuilder pb = newPluginBuilder(request);
            String origin = request.getHeader("origin");
            HttpResponse res = new HttpResponse(response);
            pb.addCorsHeaders(res, origin);
            String pathInfo = request.getServletPath();
            if (pathInfo.equals("/mathml2latex")) {
                response.setContentType("text/plain; charset=utf-8");
                ParamsProvider provider = pb.getCustomParamsProvider();
                String mml = provider.getParameter("mml", (String)null);
                String r = pb.newTextService().mathml2latex(mml);
                PrintWriter out = response.getWriter();
                out.print(r);
                out.close();
            }


js里，调用这个服务：


    var _wrs_mathmlCache = {};
    function wrs_getLatexFromMathML(mml) {
        if (_wrs_mathmlCache.hasOwnProperty(mml)) {
            return _wrs_mathmlCache[mml];
        }
        var data = {
            'service': 'mathml2latex',
            'mml': mml
        };
    
        var latex = wrs_getContent(_wrs_conf_servicePath, data);
        // Populate LatexCache.
        if (!_wrs_mathmlCache.hasOwnProperty(mml)) {
            _wrs_mathmlCache[mml] = latex;
        }
        return latex.split("\r").join('').split("\n").join(' ');
    }


wrs\_getLatexFromMathML只能将一个mathml转换为latex，对于编辑器里的内容来说，需要将mathML抽取出来逐一转换：


    function wrs_parseRawMathmlToLatex(content, characters){
        var output = '';
        var mathTagBegin = characters.tagOpener + 'math';
        var mathTagEnd = characters.tagOpener + '/math' + characters.tagCloser;
        var start = content.indexOf(mathTagBegin);
        var end = 0;
        var mathml, startAnnotation, closeAnnotation;
    
        while (start != -1) {
            output += content.substring(end, start);
            end = content.indexOf(mathTagEnd, start);
    
            if (end == -1) {
                end = content.length - 1;
            }
            else {
                end += mathTagEnd.length;
            }
    
            mathml = content.substring(start, end);
    
            output += wrs_getLatexFromMathML(mathml);
    
            start = content.indexOf(mathTagBegin, end);
        }
    
        output += content.substring(end, content.length);
        return output;
    }
    function wrs_getLatex(code) {
        return wrs_parseRawMathmlToLatex(code, _wrs_xmlCharacters);
    }


末了，为了方便获取，可以将latex放到\_current\_latex变量里：


    	// 获取数据editor.on('getData', function (e) {	e.data.dataValue = wrs_endParse(e.data.dataValue || "");	_current_latex = wrs_getLatex(e.data.dataValue || "");});
    


再简单修改下网页，显示latex：  
![enter description here](https://ooo.0o0.ooo/2017/04/21/58f9588ca5b8f.jpg "获取Latex")

收官!

