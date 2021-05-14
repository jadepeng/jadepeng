---
title: 易企秀前端压缩源码分析与还原
tags: ["JavaScript","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-03-08 17:50
---
文章作者:jqpeng
原文链接: [易企秀前端压缩源码分析与还原](https://www.cnblogs.com/xiaoqi/p/8529899.html)

你是否想知道易企秀炫酷的H5是如何实现的，原理是什么，本文会为你揭秘并还原压缩过的源代码。

易企秀是一款h5页面制作工具，因方便易用成为业界标杆。后续一个项目会用到类似易企秀这样的自定义H5的功能，因此首先分析下易企秀的前端代码，看看他们是怎么实现的，再取其精华去其糟粕。  
 由于代码较多，且是压缩处理过的，阅读和还原起来较为困难，不过可以先大概分析下原理，然后有针对性的看主要代码，并借助VS Code等工具对变量、函数进行重命名，稍微耐心一点就能大概还原源代码。

## 分析数据模型

前端分析第一步，看看易企秀的数据模型：

![数据模型](https://raw.githubusercontent.com/jadepeng/blogpic/master/pic/2018/1520492536143.jpg "1520492536143")

dataList是页面配置信息，elemengts是页面元素的配置信息，obj是H5的配置信息，

## 加载流程分析

查看H5源代码，发现入口函数是：


     eqShow.bootstrap();


顺藤摸瓜，大概分析下，主要流程如下：

![加载主要流程](https://raw.githubusercontent.com/jadepeng/blogpic/master/pic/2018/1520493739334.jpg "1520493739334")

主要的功能函数在eqxiu和window对象下面，其中的重点是parsePage、renderPage和app,下面一一来分析。

## parsePage

先看主要代码（重命名后的），主要功能是为每一页生成一个section并appendTo(".nr")，另外如果页面有特效，加载相关js库并执行，最后再renderPage。


    function parsePage(dataList, response) {
    
            for (var pageIndex = 1; pageIndex <= dataList.length; pageIndex++) {
                // 分页容器
                $('<section class="main-page"><div class="m-img" id="page' + pageIndex + '"></div></section>').appendTo(".nr");
    
                if (10 == pageMode) {
                    $("#page" + pageIndex).parent(".main-page").wrap('<div class="flip-mask" id="flip' + pageIndex + '"></div>'),
                        $(".main-page").css({
                            width: $(".nr").width() + "px",
                            height: $(".nr").height() + "px"
                        });
                }
    
                if (dataList.length > 1 && 14 != pageMode && !response.obj.property.forbidHandFlip) {
                    if (0 == pageMode || 1 == pageMode || 2 == pageMode || 6 == pageMode || 7 == pageMode ||
                        8 == pageMode || 11 == pageMode || 12 == pageMode || 13 == pageMode || 14 == pageMode) {
                        $('<section class="u-arrow-bottom"><div class="pre-wrap"><div class="pre-box1"><div class="pre1"></div></div><div class="pre-box2"><div class="pre2"></div></div></div></section>')
                            .appendTo("#page" + pageIndex)
                    } else if (3 == pageMode || 4 == pageMode || 5 == pageMode || 9 == pageMode || 10 == pageMode) {
                        $('<section class="u-arrow-right"><div class="pre-wrap-right"><div class="pre-box3"><div class="pre3"></div></div><div class="pre-box4"><div class="pre4"></div></div></div></section>')
                            .appendTo("#page" + pageIndex);
                    }
                }
    
                  ....
      	     renderPage(eqShow, pageIndex, dataList);
    
                    // 最后一页
                    if (pageIndex == dataList.length) {
                        eqxiu.app($(".nr"), response.obj.pageMode, dataList, response);
                        addEnabledClassToPageCtrl(response);
                    }
                }
    
            }
    
            hasSymbols || addReportToLastPage(dataList, response);
        }


## 渲染页面和组件

parsePage搭建了页面框架，renderPage实现页面渲染。

rendepage里，核心代码是：


    eqShow.templateParser("jsonParser").parse({
        def: dataList[pageIndex - 1],
        appendTo: "#page" + pageIndex,
        mode: "view",
        disEvent: disEvent
    });


templateParser负责将页面上的elements还原为组件，因此这里核心是要了解下templateParser，大致还原的代码如下：


                var jsonTemplateParser = eqShow.templateParser("jsonParser", function () {
    
                    function createContainerFunction(container) {
                        return function (key, value) {
                            container[key] = value
                        }
                    }
    
                    function wrapComp(element, mode) {
                        try {
                            var comp = components[("" + element.type).charAt(0)](element, mode)
                        } catch (e) {
                            return
                        }
                        if (comp) {
                            var elementContainer = $('<li comp-drag comp-rotate class="comp-resize comp-rotate inside" id="inside_' + element.id + '" num="' +
                                    element.num + '" ctype="' + element.type + '"></li>'),
                                elementType = ("" + element.type).charAt(0);
    
                            if ("3" !== elementType && "1" !== elementType) {
                                elementContainer.attr("comp-resize", "")
                            }
    
                            // 组件类型
                            /**
                             *  2 文本
                             *  3 背景
                             *  9 音乐
                             *  v video
                             *  4 图片
                             *  h shape形状
                             *  p 图集
                             *  5 输入框
                             *  r radio
                             *  c checkbox
                             *  z 多选按钮
                             *  a 评分组件
                             *  b 留言板
                             *  6 提交按钮
                             */
                            switch (elementType) {
                                case "p":
                                    elementContainer.removeAttr("comp-rotate");
                                    break;
                                case "1":
                                    elementContainer.removeAttr("comp-drag");
                                    break;
                                case "2": // 文本
                                    elementContainer.addClass("wsite-text");
                                    break;
                                case "3":
                                    // 背景
                                    break;
                                case "x":
                                    elementContainer.addClass("show-text");
                                    break;
                                case "4":
                                    // image
                                    element.properties.imgStyle && $(comp).css(element.properties.imgStyle), elementContainer.addClass("wsite-image");
                                    break;
                                case "n":
                                    elementContainer.addClass("wsite-image");
                                    break;
                                case "h":
                                    elementContainer.addClass("wsite-shape")
                                    break;
                                case "5":
                                    elementContainer.removeAttr("comp-input");
                                    break;
                                case "6":
                                case "8":
                                    elementContainer.removeAttr("comp-button");
                                    break;
                                case "v":
                                    elementContainer.removeAttr("comp-video");
                                    elementContainer.addClass("wsite-video");
                                    if (element.properties && element.properties.lock) {
                                        elementContainer.addClass("alock")
                                    }
                                    break;
                                case "b":
                                    elementContainer.removeAttr("comp-boards");
                                    elementContainer.attr("min-h", 60),
                                        elementContainer.attr("min-w", 230);
                                    break;
                                default:
                                    break;
                            }
    
                            elementContainer.mouseenter(function () {
                                    $(this).addClass("inside-hover")
                                }),
                                elementContainer.mouseleave(function () {
                                    $(this).removeClass("inside-hover")
                                });
    
                            // edit或者非文本type，再套一层
                            if ("edit" === jsonTemplateParser.mode || "x" !== ("" + element.type).charAt(0)) {
                                var elementBoxContent = $('<div class="element-box-contents">'),
                                    elementBox = $('<div class="element-box">').append(elementBoxContent.append(comp));
                                elementContainer.append(elementBox),
                                    "5" !== ("" + element.type).charAt(0) && "6" !== ("" + element.type).charAt(0) && "r" !== element.type && "c" !== element.type && "a" !== element.type && "8" !== element.type && "l" !== element.type && "s" !== element.type && "i" !== element.type && "h" !== element.type && "z" !== element.type || "edit" !== mode || $(comp).before($('<div class="element" style="position: absolute; height: 100%; width: 100%;z-index: 1;">'))
                            }
    
                            // 文本类型，处理font
                            var k, eleFonts = element.fonts || element.css.fontFamily || element.fontFamily;
                            if ("2" === elementType || "x" === elementType) {
                                for (var content = element.content, font_pattern = /font-family:(.*?);/g, matchResults = [], fonts = []; null !== (matchResults = font_pattern.exec(content));)
                                    fonts.push(matchResults[1].trim());
                                if (1 !== fonts.length || "defaultFont" !== fonts[0] && "moren" !== fonts[0] || (eleFonts = null),
                                    eleFonts) {
                                    if ("view" === jsonTemplateParser.mode && element.css.fontFamily && window.scene && (window.scene.publishTime || !mobilecheck() && !tabletCheck() || (k = "@font-face{font-family:" + element.css.fontFamily + ';src: url("' + element.properties.localFontPath + '") format("truetype");}',
                                            b(k))),
                                        "object" == typeof eleFonts && eleFonts.constructor === Object) {
                                        if (!jQuery.isEmptyObject(eleFonts))
                                            for (var q in eleFonts)
                                                u[q] || ("edit" === jsonTemplateParser.mode ? k = "@font-face{font-family:" + q + ";src: url(" + PREFIX_FILE_HOST + eleFonts[q] + ") format(woff);}" : window.scene && window.scene.publishTime && (k = "@font-face{font-family:" + q + ';src: url("' + PREFIX_S2_URL + "fc/" + q + "_" + element.sceneId + "_" + scene.publishTime + '.woff") format("woff");}'),
                                                    b(k),
                                                    u[q] = !0)
                                    } else
                                        u[eleFonts] || ("edit" === jsonTemplateParser.mode ? k = "@font-face{font-family:" + eleFonts + ";src: url(" + PREFIX_FILE_HOST + element.preWoffPath + ") format(woff);}" : window.scene && window.scene.publishTime && (k = "@font-face{font-family:" + eleFonts + ';src: url("' + PREFIX_S2_URL + "fc/" + eleFonts + "_" + element.sceneId + "_" + scene.publishTime + '.woff") format("woff");}'),
                                            b(k),
                                            u[eleFonts] = !0);
                                    "edit" === jsonTemplateParser.mode && localStorage.setItem("shoppingFontFamily", JSON.stringify(u))
                                }
                            }
    
                            // 处理css
                            if (element.css) {
                                var elementWidth = 320 - parseInt(element.css.left, 10);
                                elementContainer.css({
                                    width: elementWidth
                                });
                                elementContainer.css({
                                    width: element.css.width,
                                    height: element.css.height,
                                    left: element.css.left,
                                    top: element.css.top,
                                    zIndex: element.css.zIndex,
                                    bottom: element.css.bottom,
                                    transform: element.css.transform
                                });
                                if (0 === element.css.boxShadowSize || "" + element.css.boxShadowSize == "0") {
                                    element.css.boxShadow = "0px 0px 0px rgba(0,0,0,0.5)";
                                    if ("edit" !== jsonTemplateParser.mode && "x" === ("" + element.type).charAt(0)) {
                                        return elementContainer.append(comp),
                                            elementContainer.find(".element-box").css({
                                                borderStyle: element.css.borderStyle,
                                                borderWidth: element.css.borderWidth,
                                                borderColor: element.css.borderColor,
                                                borderTopLeftRadius: element.css.borderTopLeftRadius,
                                                borderTopRightRadius: element.css.borderTopRightRadius,
                                                borderBottomRightRadius: element.css.borderBottomRightRadius,
                                                borderBottomLeftRadius: element.css.borderBottomLeftRadius,
                                                boxShadow: element.css.boxShadow,
                                                backgroundColor: element.css.backgroundColor,
                                                opacity: element.css.opacity,
                                                width: "100%",
                                                height: "100%",
                                                overflow: "hidden"
                                            }),
                                            elementContainer.find("img").css({
                                                width: "100%"
                                            }),
                                            elementContainer;
                                    }
                                }
    
                                // Android 微信，图片，设置borderColor
                                isAndroid() &&
                                    isWeixin() &&
                                    "" + element.type == "4" &&
                                    "0px" !== element.css.borderRadius &&
                                    0 === element.css.borderWidth &&
                                    element.properties.anim && (element.css.borderWidth = 1, element.css.borderColor = "rgba(0,0,0,0)");
    
                                var elementCss = $.extend(!0, {}, element.css);
                                delete elementCss.fontFamily,
                                    elementBox.css(elementCss).css({
                                        width: "100%",
                                        height: "100%",
                                        transform: "none"
                                    }),
                                    elementBox.children(".element-box-contents").css({
                                        width: "100%",
                                        height: "100%"
                                    }),
                                    // 设置宽高
                                    "4" !== ("" + element.type).charAt(0) &&
                                    "n" !== ("" + element.type).charAt(0) &&
                                    "p" !== ("" + element.type).charAt(0) &&
                                    "h" !== ("" + element.type).charAt(0) && "t" !== ("" + element.type).charAt(0) &&
                                    "z" !== ("" + element.type).charAt(0) &&
                                    $(comp).css({
                                        width: element.css.width,
                                        height: element.css.height
                                    }),
                                    // w01 w02 设置lineHeight
                                    ("w01" === element.type || "w02" === element.type) &&
                                    $(comp).css({
                                        lineHeight: element.css.height + "px"
                                    }),
                                    // shape 类型
                                    "h" === ("" + element.type).charAt(0) &&
                                    ($(comp).find("g").length ?
                                        $(comp).find("g").attr("fill", element.css.color) :
                                        $(comp).children().attr("fill", element.css.color),
                                        elementBox.children(".element-box-contents").css("position", "relative"))
                            }
                            return elementContainer
                        }
                    }
    
                    /**
                     * 将element按zindex排序
                     */
                    function sortElementsByZindex(elements) {
                        for (var i = 0; i < elements.length - 1; i++)
                            for (var j = i + 1; j < elements.length; j++)
                                if (parseInt(elements[i].css.zIndex, 10) > parseInt(elements[j].css.zIndex, 10)) {
                                    var element = elements[i];
                                    elements[i] = elements[j],
                                        elements[j] = element
                                }
                        for (var e = 0; e < elements.length; e++)
                            elements[e].css.zIndex = e + 1 + "";
                        return elements
                    }
    
                    function parseElements(pageDef, $edit_wrapper, mode) {
                        $edit_wrapper = $edit_wrapper.find(".edit_area");
                        var i, elements = pageDef.elements;
                        if (elements)
                            for (elements = sortElementsByZindex(elements),
                                i = 0; i < elements.length; i++)
                                if (elements[i].sceneId = pageDef.sceneId,
                                    "" + elements[i].type == "3") {
                                    // type == 3 
                                    var component = components[("" + elements[i].type).charAt(0)](elements[i], $edit_wrapper);
    
                                    // if is edit mode, dispatch edit event
                                    "edit" === mode
                                        &&
                                        editEvents[("" + elements[i].type).charAt(0)] &&
                                        editEvents[("" + elements[i].type).charAt(0)](component, elements[i])
                                } else {
                                    var comp = wrapComp(elements[i], mode);
                                    if (!comp)
                                        continue;
                                    $edit_wrapper.append(comp);
    
                                    // invoke interceptors
                                    for (var n = 0; n < interceptors.length; n++)
                                        interceptors[n](comp, elements[i], mode);
    
                                    afterRenderEvents[("" + elements[i].type).charAt(0)] &&
                                        (
                                            afterRenderEvents[("" + elements[i].type).charAt(0)](comp, elements[i]),
                                            "edit" !== mode && (
                                                parseElementTrigger(comp, elements[i]),
                                                r(comp, elements[i])
                                            )
                                        ),
    
                                        "edit" === mode &&
                                        editEvents[("" + elements[i].type).charAt(0)] &&
                                        editEvents[("" + elements[i].type).charAt(0)](comp, elements[i])
                                }
                    }
    
                    function getEventHandlers() {
                        return editEvents
                    }
    
                    function getComponents() {
                        return components
                    }
    
                    function addInterceptor(interceptor) {
                        interceptors.push(interceptor)
                    }
    
                    function getInterceptors() {
                        return interceptors
                    }
                    var components = {},
                        editEvents = {},
                        afterRenderEvents = {},
                        interceptors = [],
                        _width = containerWidth = 320,
                        _height = containerHeight = 486,
                        p = 1,
                        s = 1,
                        parser = {
                            getComponents: getComponents,
                            getEventHandlers: getEventHandlers,
                            addComponent: createContainerFunction(components),
                            bindEditEvent: createContainerFunction(editEvents),
                            bindAfterRenderEvent: createContainerFunction(afterRenderEvents),
                            addInterceptor: addInterceptor,
                            getInterceptors: getInterceptors,
                            wrapComp: wrapComp,
                            disEvent: !1,
                            mode: "view",
                            parse: function (parseInfo) {
                                var edit_wrapper = $('<div class="edit_wrapper" data-scene-id="' + parseInfo.def.sceneId + '"><ul eqx-edit-destroy id="edit_area' + parseInfo.def.id + '" paste-element class="edit_area weebly-content-area weebly-area-active"></div>'),
                                    mode = this.mode = parseInfo.mode;
                                // page 定义
                                this.def = parseInfo.def,
                                    parseInfo.disEvent && (this.disEvent = !0),
                                    "view" === mode && tplCount++;
                                // 页面容器
                                var pageContainer = $(parseInfo.appendTo);
                                return containerWidth = pageContainer.width(),
                                    containerHeight = pageContainer.height(),
                                    p = _width / containerWidth,
                                    s = _height / containerHeight,
                                    parseElements(parseInfo.def, edit_wrapper.appendTo($(parseInfo.appendTo)), mode)
                            }
                        };
                    return parser
                });


上面的重点是parseElements，先把elements按zindex排序，然后逐个渲染。  
 注意，渲染是根据elementType,从components找到对应的组件，然后创建一个实例，因此这里要单独说下组件是如何定义的。

先看下一个组件的配置信息大概是这样，有id，css，type和动画等配置信息：


    	{
        "id": 29,
        "css": {
            "top": 124.93546211843,
            "left": 62.967731059217,
            "color": "#676767",
            "width": 195,
            "height": 195,
            "zIndex": "1",
            "opacity": 1,
            "boxShadow": "0px 0px 0px rgba(0,0,0,0.5)",
            "transform": "rotateZ(45deg)",
            "lineHeight": 1,
            "paddingTop": 0,
            "borderColor": "rgba(255,255,255,1)",
            "borderStyle": "double",
            "borderWidth": 4,
            "borderRadius": "0px",
            "boxShadowSize": 0,
            "paddingBottom": 0,
            "backgroundColor": "rgba(252,230,238,0.16)",
            "borderRadiusPerc": 0,
            "boxShadowDirection": 0,
            "textAlign": "left",
            "borderBottomRightRadius": "0px",
            "borderBottomLeftRadius": "0px",
            "borderTopRightRadius": "0px",
            "borderTopLeftRadius": "0px"
        },
        "type": "2",
        "pageId": "24642",
        "content": "<div style=\"text-align: center;\"><br></div>",
        "sceneId": 8831293,
        "properties": {
            "anim": {
                "type": 4,
                "delay": 0.6,
                "countNum": 1,
                "duration": 1,
                "direction": 0
            },
            "width": 195,
            "height": 195
        }
    }


jsonParser里用一个components对象存储组件，通过addComponent添加组件，key就是组件的type：


    addComponent: createContainerFunction(components)
    function createContainerFunction(container) {
                    return function (key, value) {
                        container[key] = value
                    }
                }


添加组件时，type 作为key，value为创建组件的函数：


    // 添加组件1jsonTemplateParser.addComponent("1", function (element, mode) {	var comp = document.createElement("div");	if (comp.id = element.id,		comp.setAttribute("class", "element comp_title"),		// 设置组件content		element.content && (comp.textContent = element.content),		element.css) {		var item, elementCss = element.css;		for (item in elementCss)			comp.style[item] = elementCss[item]	}	if (element.properties.labels)		for (var labels = element.properties.labels, f = 0; f < labels.length; f++)			$('<a class = "label_content" style = "display: inline-block;">')			.appendTo($(comp))			.html(labels[f].title)			.css(labels[f].color)			.css("width", 100 / labels.length + "%");	return comp});


这样渲染组件时，根据element的类型就能找到createCompFunction，从而创建组件。

## 组件动画播放

H5之所以炫酷，很大一部分因为每个组件都有定制好的CSS3动画，我们这里来看看这些动画是如何执行的。

代码还是上一部分的代码，我们注意到组件渲染后，有一段代码;


      // invoke interceptorsfor (var n = 0; n < interceptors.length; n++)	interceptors[n](comp, elements[i], mode);


执行interceptors，这个interceptors可以通过addInterceptor注册拦截器，在组件渲染完成后会调用定义的拦截器，组件的动画就是这样来调用的。


            // 添加拦截器执行动画
            jsonTemplateParser.addInterceptor(function (comp, element, mode) {
                eqxCommon.animation(comp, element, mode, jsonTemplateParser.def.properties)
            });


我们发现，eqxiu通过addInterceptor注册了一个拦截器，该拦截器调用eqxCommon.animation执行组件动画，因此分析eqxCommon.animation就可以了解动画是如何实现的。

还是先看element里的定义：


    	 "properties": {		"anim": {			"type": 4,			"delay": 0.6,			"countNum": 1,			"duration": 1,			"direction": 0		},


我们看到，anim里定义了type，delay，duration等配置信息，可以设想播放动画无非就是解析这个配置，然后执行，其中type应该是对应的各种动画类型，分析代码吧，下面给出破解后的代码：


    	    // 动画播放序号
            var animIndex = 0;
    
            // 处理动画属性
            if (element.properties && element.properties.anim) {
                var anim = [];
                element.properties.anim.length ? anim = element.properties.anim : anim.push(element.properties.anim);
                var elementBox = $(".element-box", comp);
                elementBox.attr("element-anim", "");
    
                // 找出animations
                for (var animType, animTypes = [], anims = [], index = 0, animLength = anim.length; animLength > index; index++)
                    if (null != anim[index].type &&
                        -1 != anim[index].type) {
                        animType = eqxCommon.convertType(anim[index]),
                            animTypes.push(animType),
                            anims.push(anim[index]);
                    }
    
                if (properties && properties.scale)
                    return;
    
                // 动画播放类型
                element.properties.anim.trigger ?
                    comp.click(function () {
                        // 点击播放
                        playAnimation(elementBox, animType, element.properties.anim)
                    }) :
                    properties && properties.longPage ?
                    playAnimation(elementBox, animTypes, anims, !0, element.css) // longpage
                    :
                    playAnimation(elementBox, animTypes, anims)
            }


上面的逻辑是先从element里找到anim，放入数组，然后再playAnimation。这里使用了convertType函数将数字type转换为真实的动画类型：


    var convertType = function (a) {
            var animType, c, d = a.type;
            return "typer" === d && (animType = "typer"),
                0 === d && (animType = "fadeIn"),
                1 === d && (c = a.direction,
                    0 === c && (animType = "fadeInLeft"),
                    1 === c && (animType = "fadeInDown"),
                    2 === c && (animType = "fadeInRight"),
                    3 === c && (animType = "fadeInUp")),
                6 === d && (animType = "wobble"),
                5 === d && (animType = "rubberBand"),
                7 === d && (animType = "rotateIn"),
                8 === d && (animType = "flip"),
                9 === d && (animType = "swing"),
                2 === d && (c = a.direction,
                    0 === c && (animType = "bounceInLeft"),
                    1 === c && (animType = "bounceInDown"),
                    2 === c && (animType = "bounceInRight"),
                    3 === c && (animType = "bounceInUp")),
                3 === d && (animType = "bounceIn"),
                4 === d && (animType = "zoomIn"),
                10 === d && (animType = "fadeOut"),
                11 === d && (animType = "flipOutY"),
                12 === d && (animType = "rollIn"),
                13 === d && (animType = "lightSpeedIn"),
                14 === d && (animType = "bounceOut"),
                15 === d && (animType = "rollOut"),
                16 === d && (animType = "lightSpeedOut"),
                17 === d && (c = a.direction,
                    0 === c && (animType = "fadeOutRight"),
                    1 === c && (animType = "fadeOutDown"),
                    2 === c && (animType = "fadeOutLeft"),
                    3 === c && (animType = "fadeOutUp")),
                18 === d && (animType = "zoomOut"),
                19 === d && (c = a.direction,
                    0 === c && (animType = "bounceOutRight"),
                    1 === c && (animType = "bounceOutDown"),
                    2 === c && (animType = "bounceOutLeft"),
                    3 === c && (animType = "bounceOutUp")),
                20 === d && (animType = "flipInY"),
                21 === d && (animType = "tada"),
                22 === d && (animType = "jello"),
                23 === d && (animType = "flash"),
                26 === d && (animType = "twisterInDown"),
                27 === d && (animType = "puffIn"),
                28 === d && (animType = "puffOut"),
                29 === d && (animType = "slideDown"),
                30 === d && (animType = "slideUp"),
                24 === d && (animType = "flipInX"),
                25 === d && (animType = "flipOutX"),
                31 === d && (animType = "twisterInUp"),
                32 == d && (animType = "vanishOut"),
                33 == d && (animType = "vanishIn"),
                animType
        };


播放动画函数在playAnimation里：

- 大概是先判断类型是否是typer，typer的话是打字机特效，调用相关代码
- 设置element的的css animation属性，本质上是CSS3的animation动画,可以参见（[http://www.w3school.com.cn/css3/css3\_animation.asp）](http://www.w3school.com.cn/css3/css3_animation.asp%EF%BC%89)



     elementBox.css("animation", "");
                    elementBox.css("animation", animTypes[animIndex] + " " + anims[animIndex].duration + "s ease " + anims[animIndex].delay + "s " +
                        (anims[animIndex].countNum ? anims[animIndex].countNum : ""));				 anims[animIndex].count && animIndex == anims.length - 1 && elementBox.css("animation-iteration-count", "infinite");
                        elementBox.css("animation-fill-mode", "both");


最后，如果有多个动画，在播放完成后继续播放下一个：


    // 动画播放结束，播放下一个动画（一个组件可能有多个动画）
                    elementBox.one("webkitAnimationEnd mozAnimationEnd MSAnimationEnd oanimationend animationend", function () {
                        animIndex++;
                        playAnimation(elementBox, animTypes, anims);
                    })


## 页面切换

由于是多页应用，因此涉及到页面切换，并且页面切换时还需要有对应的切换动画，改工作是由一个eqxiu对象来管理和实现的。

老套路，先看这块的配置吧，页面的配置在obj下面,其中pageMode定义了翻页效果：


    "obj": {
        "id": 8831293,
        "name": "房产广告",
        "createUser": "1",
        "type": 103,
        "pageMode": 4,
        "image": {},
        "property": "{\"triggerLoop\":true,\"slideNumber\":true,\"autoFlipTime\":4,\"shareDes\":\"\",\"eqAdType\":1,\"hideEqAd\":false,\"autoFlip\":true,\"lastPageId\":604964}",
        "timeout": "",
        "timeout_url": "",
        "accessCode": null,
        "cover": "syspic/pageimg/yq0KA1UrbkOAV_yiAAFuhyGx9LE397.jpg",
        "bgAudio": "{\"url\":\"syspic/mp3/yq0KA1RHT3iAMXYOAAgPq1MjV9M930.mp3\",\"type\":\"3\"}",
        "isTpl": 0,
        "isPromotion": 0,
        "status": 1,
        "openLimit": 0,
        "startDate": null,
        "endDate": null,
        "updateTime": 1426045746000,
        "createTime": 1426572693000,
        "publishTime": 1426572693000,
        "applyTemplate": 0,
        "applyPromotion": 0,
        "sourceId": null,
        "code": "U903078B74Q5",
        "description": "房产广告",
        "sort": 0,
        "pageCount": 0,
        "dataCount": 0,
        "showCount": 44,
        "eqcode": "",
        "userLoginName": null,
        "userName": null
    },


pagemode是这样定义的：


    pagemodes = [{		id: 0,		name: "上下翻页"	}, {		id: 4,		name: "左右翻页"	}, {		id: 1,		name: "上下惯性翻页"	}, {		id: 3,		name: "左右惯性翻页"	}, {		id: 11,		name: "上下连续翻页"	}, {		id: 5,		name: "左右连续翻页"	}, {		id: 6,		name: "立体翻页"	}, {		id: 7,		name: "卡片翻页"	}, {		id: 8,		name: "放大翻页"	}, {		id: 9,		name: "交换翻页"	}, {		id: 10,		name: "翻书翻页"	}, {		id: 12,		name: "掉落翻页"	}, {		id: 13,		name: "淡入翻页"	}];


在renderpage结束后，调用eqxiu.app：


     				 // 最后一页
                    if (pageIndex == dataList.length) {
                        eqxiu.app($(".nr"), response.obj.pageMode, dataList, response);
                        addEnabledClassToPageCtrl(response);
                    }


来分析eqxiu.app代码，通过pagemode，我们可以看出翻页打开分为上下翻页、左右翻页两个大类：


    if ("8" == pageMode || "9" == pageMode) {
                transformTime = 0.7;
                timeoutDelay = 800;
            }
            // 上下翻页  上下惯性翻页 立体翻页 卡片翻页 放大翻页 上下连续翻页 上下连续翻页
            if (0 == pageMode || (1 == pageMode || (2 == pageMode || (6 == pageMode || (7 == pageMode || (8 == pageMode || (11 == pageMode || 12 == pageMode))))))) {
                /** @type {boolean} */
                upDownMode = true;
            } else {
                // 左右惯性翻页 左右翻页 左右连续翻页  翻书翻页
                if (3 == pageMode || (4 == pageMode || (5 == pageMode || 10 == pageMode))) {
                    /** @type {boolean} */
                    leftRightMode = true;
                }
            }


然后配置里有一个autoFlip，代表是否自动翻页,通过setInterval设置定时翻页任务：


    		// 自动翻页
            if (response.obj.property.autoFlip) {
                // 自动翻页时间
                autoFlipTimeMS = 1000 * response.obj.property.autoFlipTime;
                setAndStartAutoFlip(autoFlipTimeMS);
            }	
        /**
         * 设置翻页时间间隔并启动翻页
         * @param {number} textStatus
         * @return {undefined}
         */
        function setAndStartAutoFlip(autoFlipTime) {
            autoFlipTime = autoFlipTime;
            pauseAutoFlip();
            startAutoFlip();
        }		   /**
         * 启动自动翻页
         * @return {undefined}
         */
        function startAutoFlip() {
            // 通过setInterval
            autoFlipIntervalId = setInterval(function () {
                if (!(10 === self._scrollMode)) {
                    if (!isTouching) {
                        nextPage();
                    }
                }
            }, autoFlipTimeMS);
        }


默认情况下H5是支持touch滑动翻页的，这种滑动操作一般是监听相关事件，开始滑动、滑动中和滑动结束，为了同时支持移动端和PC端，还需要加上鼠标点击事件：


            var isTouch = false;
            self._$app.on("mousedown touchstart", function (e) {
                if (!self._isforbidHandFlip) {
                    onTouchStart(e);
                    isTouch = true;
                }
            }).on("mousemove touchmove", function (e) {
                if (!self._isforbidHandFlip) {
                    if (isTouch) {
                        onTouchMove(e);
                    }
                }
            }).on("mouseup touchend mouseleave", function (events) {
                if (!self._isforbidHandFlip) {
                    onTouchEnd(events);
                    /** @type {boolean} */
                    isTouch = false;
                }
            });


翻页的核心无非就是判断位移是否超过特定的值，比如左右翻页X位移是否大于Y位移并且X的偏移量大于20。因此onTouchStart开始时，记录初始位置，onTouchMove时计算offset变化，按照pageMode执行对应的动画，onTouchEnd时判断位移是否足够，足够就切换页面，否则复位。


    /**
             * 开始滑动
             * @param {Object} e
             * @return {undefined}
             */
            onTouchStart = function (e) {
                /** @type {boolean} */
                fa = false;
                if (isMobile) {
                    if (e) {
                        e = event;
                    }
                }
                if (!self._isDisableFlipPage) {
                    self.$currentPage = self._$pages.filter(".z-current").get(0);
                    if (!C) {
                        /** @type {null} */
                        self.$activePage = null;
                    }
                    if (self.$currentPage) {
                        if (completeEffect($(self.$currentPage))) {
                            isTouching = true;
                            isCursorAtEnd = false;
                            ignoreEvent = true;
                            offsetX = 0;
                            offsetY = 0;
                            if (e && "mousedown" == e.type) {
                                currentPageX = e.pageX;
                                currentPageY = e.pageY;
                            } else if (e && "touchstart" == e.type) {
                                currentPageX = e.touches ? e.touches[0].pageX : e.originalEvent.touches[0].pageX;
                                currentPageY = e.touches ? e.touches[0].pageY : e.originalEvent.touches[0].pageY;
                            }
                            self.$currentPage.classList.add("z-move");
                            setAttribute(self.$currentPage.style, "Transition", "none");
                            if ("12" == self._scrollMode) {
                                /** @type {number} */
                                self.$currentPage.style.zIndex = 3;
                            }
                        }
                    }
                }
            };
    
            /**
             * 滑动处理
             * @param {Object} e
             * @return {undefined}
             */
            onTouchMove = function (e) {
                if (isMobile) {
                    if (e) {
                        e = event;
                    }
                }
                if (isTouching) {
                    if (self._$pages.length > 1) {
                        if (e && "mousemove" == e.type) {
                            offsetX = e.pageX - currentPageX;
                            offsetY = e.pageY - currentPageY;
                        } else {
                            if (e) {
                                if ("touchmove" == e.type) {
                                    offsetX = (e.touches ? e.touches[0].pageX : e.originalEvent.touches[0].pageX) - currentPageX;
                                    offsetY = (e.touches ? e.touches[0].pageY : e.originalEvent.touches[0].pageY) - currentPageY;
                                }
                            }
                        }
                        if (!fa) {
                            if (Math.abs(offsetX) > 20 || Math.abs(offsetY) > 20) {
                                /** @type {boolean} */
                                fa = true;
                            }
                        }
    
                        switch (self._scrollMode + "") {
                            case "0":
                            case "1":
                            case "2":
                            case "15":
                                //上下翻页
                                upDownFlip();
                                break;
                            case "3":
                            case "4":
                                // 左右翻页
                                leftRightFlip();
                                break;
                            case "5":
                                // 左右连续翻页
                                leftRightLoopFlip();
                                break;
                            case "7":
                                cardFlip();
                                break;
                            case "8":
                                scaleUpFlip();
                                break;
                            case "9":
                                switchFlip();
                                break;
                            case "11":
                                //上下连续翻页
                                upDownContinuousFlip();
                                break;
                            case "12":
                                //掉落翻页
                                dropFlip();
                                break;
                            case "13":
                            case "14":
                                //淡入翻页
                                fadeFlip();
                                break;
                            default:
                                break;
                        }
                    }
                }
            };
    
    
            /**
             *  滑动结束
             * @param {?} e
             * @return {undefined}
             */
            onTouchEnd = function (e) {
                if (isTouching && completeEffect($(self.$currentPage))) {
                    isTouching = false;
                    if (self.$activePage) {
                        self._isDisableFlipPage = true;
                        var ease;
                        ease = "6" == self._scrollMode || "7" == self._scrollMode ? "cubic-bezier(0,0,0.99,1)" : "12" == self._scrollMode ? "cubic-bezier(.17,.67,.87,.13)" : "linear";
                        self.$currentPage.style.webkitTransition = "-webkit-transform " + transformTime + "s " + ease;
                        self.$activePage.style.webkitTransition = "-webkit-transform " + transformTime + "s " + ease;
                        self.$currentPage.style.mozTransition = "-moz-transform " + transformTime + "s " + ease;
                        self.$activePage.style.mozTransition = "-moz-transform " + transformTime + "s " + ease;
                        self.$currentPage.style.transition = "transform " + transformTime + "s " + ease;
                        self.$activePage.style.transition = "transform " + transformTime + "s " + ease;
    
                        // 完成翻页
                        if ("0" == self._scrollMode || ("2" == self._scrollMode || ("1" == self._scrollMode || "15" == self._scrollMode))) {
                            endUpDownFlip();
                        } else if ("4" == self._scrollMode || "3" == self._scrollMode) {
                            // 左右翻页
                            endLeftRightFlip();
                        } else if ("5" == self._scrollMode) {
                            //左右连续翻页
                            endLeftRightContinueFlip();
                        } else if ("6" == self._scrollMode) {
                            //立体翻页
                            endCubeFlip();
                        } else if ("7" == self._scrollMode) {
                            //卡片翻页
                            endCardFlip();
                        } else if ("8" == self._scrollMode) {
                            //放大翻页
                            endScaleUpFlip();
                        } else if ("9" == self._scrollMode) {
                            //交换翻页
                            endSwitchFlip();
                        } else if ("11" == self._scrollMode) {
                            //上下连续翻页
                            endUpDownContinueFlip();
                        } else if ("12" == self._scrollMode) {
                            //掉落翻页
                            endDropFlip();
                        } else if ("13" == self._scrollMode || "14" == self._scrollMode) {
                            //淡入翻页
                            endFadeFlip();
                        } 
    
                        /** @type {number} */
                        var pageIndex = $(self.$activePage).find(".m-img").attr("id").replace("page", "") - 1;
                        if (self._pageData[pageIndex].properties) {
                            if (self._pageData[pageIndex].properties.longPage) {
                                $(document).trigger("clearTouchPos");
                            }
                        }
                        $(self.$activePage).find("li.comp-resize").each(function (dataAndEvents) {
                            /** @type {number} */
                            var i = 0;
                            for (; i < self._pageData[pageIndex].elements.length; i++) {
                                if (self._pageData[pageIndex].elements[i].id == parseInt($(this).attr("id").substring(7), 10)) {
                                    eqxCommon.animation($(this), self._pageData[pageIndex].elements[i], "view", self._pageData[pageIndex].properties);
                                    var r20 = getComp(self._pageData[pageIndex].elements[i].id);
                                    eqxCommon.bindTrigger(r20, self._pageData[pageIndex].elements[i]);
                                }
                            }
                        });
                        /** @type {number} */
                        var i = 0;
                        for (; i < self._pageData.length; i++) {
                            if (self._pageData[i].effObj) {
                                /** @type {boolean} */
                                self._pageData[i].effObj.pause = true;
                            }
                        }
                        if (self._pageData[pageIndex].effObj) {
                            self._pageData[pageIndex].effObj.startPlay();
                        }
                        eqShow.setPageHis(self._pageData[pageIndex].id);
                    } else {
                        self.$currentPage.classList.remove("z-move");
                    }
                }
                C = false;
            };


然后再来看自动翻页nextPage


      /**
         * 启动自动翻页
         * @return {undefined}
         */
        function startAutoFlip() {
            // 通过setInterval
            autoFlipIntervalId = setInterval(function () {
                if (!(10 === self._scrollMode)) {
                    if (!isTouching) {
                        nextPage();
                    }
                }
            }, autoFlipTimeMS);
        }


自动翻页比较简单，模拟滑动操作，当位移足够时就可以自动翻页了：


    /**
         *  上一页
         * @param {number} direction
         * @return {undefined}
         */
        function prePage(direction) {
            if (!(leftRightMode && 2 == direction || upDownMode && 1 == direction)) {
                if ("10" != self._scrollMode) {
                    var offset = 0;
                    // 开启滑动
                    onTouchStart();
                    // 定时器，增加offset,模拟滑动
                    var poll = setInterval(function () {
                        offset += 2;
                        if ("0" == self._scrollMode || ("1" == self._scrollMode || ("2" == self._scrollMode || ("6" == self._scrollMode || ("7" == self._scrollMode || ("8" == self._scrollMode || ("11" == self._scrollMode || ("12" == self._scrollMode || ("13" == self._scrollMode || ("14" == self._scrollMode || "15" == self._scrollMode)))))))))) {
                            // 纵向翻页，增加y
                            offsetY = offset;
                        } else {
                            if ("3" == self._scrollMode || ("4" == self._scrollMode || ("5" == self._scrollMode || "9" == self._scrollMode))) {
                                // 横向翻页，增加x
                                offsetX = offset;
                            }
                        }
                        // 触发move操作，模拟滑动
                        onTouchMove();
                        if (offset >= 21) {
                            // 位移超过20，
                            clearInterval(poll);
                            // 停止滑动,完成翻页
                            onTouchEnd();
                        }
                    }, 1);
                } else {
                    // 翻书
                    $(document).trigger("bookFlipPre");
                }
            }
        }
    
        /**
         * 下一页，逻辑和prePage类似
         * @param {number} direction
         * @return {undefined}
         */
        function nextPage(direction) {
            if (!(leftRightMode && 2 == direction || upDownMode && 1 == direction)) {
                if ("10" != self._scrollMode) {
                    u = false;
                    var offset = 0;
                    if ("block" == $("body .boards-panel").css("display")) {
                        $("body .boards-panel").hide();
                        $("body .z-current").show();
                    }
                    onTouchStart();
                    var poll = setInterval(function () {
                        offset -= 2;
                        if ("0" == self._scrollMode || ("1" == self._scrollMode || ("2" == self._scrollMode || ("6" == self._scrollMode || ("7" == self._scrollMode || ("8" == self._scrollMode || ("11" == self._scrollMode || ("12" == self._scrollMode || ("13" == self._scrollMode || ("14" == self._scrollMode || "15" == self._scrollMode)))))))))) {
                            offsetY = offset;
                        } else {
                            if ("3" == self._scrollMode || ("4" == self._scrollMode || ("5" == self._scrollMode || "9" == self._scrollMode))) {
                                offsetX = offset;
                            }
                        }
                        onTouchMove();
                        if (-21 >= offset) {
                            clearInterval(poll);
                            onTouchEnd();
                            if (!triggerLoop) {
                                if (!self.$activePage) {
                                    clearInterval(autoFlipIntervalId);
                                }
                            }
                        }
                    }, 1);
                } else {
                    $(document).trigger("bookFlipNext");
                }
            }
        }


## 总结

上面是花了大概一天多的时间阅读代码的成果，总结经验就是阅读代码先分析大的流程，再层层递进分析一些细节，就能一步一步接近真相。

另外，阅读压缩过的代码，可以借助VS Code，善用F2重命名，修改的越多，越接近本来的代码：）

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


