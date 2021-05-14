---
title: 易企秀H5 json配置文件解密分析
tags: ["易企秀","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-11-27 19:36
---
文章作者:jqpeng
原文链接: [易企秀H5 json配置文件解密分析](https://www.cnblogs.com/xiaoqi/p/10028472.html)

最近需要参考下易企秀H5的json配置文件，发现已经做了加密，其实前端的加密分析起来只是麻烦点。

## 抓包分析

先看一个H5： [https://h5.eqxiu.com/s/XvEn30op](https://h5.eqxiu.com/s/XvEn30op)

F12可以看到，配置json地址是：[https://s1-cdn.eqxiu.com/eqs/page/142626394?code=XvEn30op&time=1542972816000](https://s1-cdn.eqxiu.com/eqs/page/142626394?code=XvEn30op&amp;time=1542972816000)

对应的json：

{"success":true,"code":200,"msg":"操作成功","obj":"加密后的字符串"}

obj对应的是正式的配置信息

## 解码分析

需要对加密信息进行解密，首先可以定位到解密代码


    function _0x230bc7(_0x2fb175) {
        return _0x3c31('0xee') == typeof _0x2fb175[_0x3c31('0x25')] && _0x2fb175['\x6f\x62\x6a'][_0x3c31('0xe')] > 0x64 ? _0x54c90c[_0x3c31('0x31f')]()['\x74\x68\x65\x6e'](function() {
            _0x249a60();
            var _0x5ab652 = null
              , _0x2cf0a4 = null
              , _0x4d1175 = null;
            try {
                var _0x3dbfaa = _0x2fb175[_0x3c31('0x25')]['\x73\x75\x62\x73\x74\x72\x69\x6e\x67'](0x0, 0x13)
                  , _0x360e25 = _0x2fb175[_0x3c31('0x25')][_0x3c31('0xeb')](0x13 + 0x10);
                _0x2cf0a4 = _0x2fb175[_0x3c31('0x25')][_0x3c31('0xeb')](0x13, 0x13 + 0x10),
                _0x4d1175 = _0x2cf0a4,
                _0x5ab652 = _0x3dbfaa + _0x360e25,
                _0x2cf0a4 = CryptoJS['\x65\x6e\x63'][_0x3c31('0x320')][_0x3c31('0x6b')](_0x2cf0a4),
                _0x4d1175 = CryptoJS[_0x3c31('0x321')][_0x3c31('0x320')]['\x70\x61\x72\x73\x65'](_0x4d1175);
                var _0x57e61a = CryptoJS[_0x3c31('0x322')][_0x3c31('0x323')](_0x5ab652, _0x2cf0a4, {
                    '\x69\x76': _0x4d1175,
                    '\x6d\x6f\x64\x65': CryptoJS[_0x3c31('0x324')][_0x3c31('0x325')],
                    '\x70\x61\x64\x64\x69\x6e\x67': CryptoJS[_0x3c31('0x326')][_0x3c31('0x327')]
                });
                return _0x2fb175[_0x3c31('0x36')] = JSON[_0x3c31('0x6b')](CryptoJS[_0x3c31('0x321')][_0x3c31('0x320')][_0x3c31('0x267')](_0x57e61a)),
                _0x2fb175;
            } catch (_0x36fffe) {
                _0x5ab652 = _0x2fb175[_0x3c31('0x25')]['\x73\x75\x62\x73\x74\x72\x69\x6e\x67'](0x0, _0x2fb175[_0x3c31('0x25')][_0x3c31('0xe')] - 0x10),
                _0x2cf0a4 = _0x2fb175[_0x3c31('0x25')][_0x3c31('0xeb')](_0x2fb175[_0x3c31('0x25')][_0x3c31('0xe')] - 0x10),
                _0x4d1175 = _0x2cf0a4,
                _0x2cf0a4 = CryptoJS[_0x3c31('0x321')][_0x3c31('0x320')][_0x3c31('0x6b')](_0x2cf0a4),
                _0x4d1175 = CryptoJS[_0x3c31('0x321')][_0x3c31('0x320')][_0x3c31('0x6b')](_0x4d1175);
                var _0x4ff7de = CryptoJS[_0x3c31('0x322')][_0x3c31('0x323')](_0x5ab652, _0x2cf0a4, {
                    '\x69\x76': _0x4d1175,
                    '\x6d\x6f\x64\x65': CryptoJS['\x6d\x6f\x64\x65'][_0x3c31('0x325')],
                    '\x70\x61\x64\x64\x69\x6e\x67': CryptoJS[_0x3c31('0x326')][_0x3c31('0x327')]
                });
                return _0x2fb175[_0x3c31('0x36')] = JSON[_0x3c31('0x6b')](CryptoJS[_0x3c31('0x321')]['\x55\x74\x66\x38'][_0x3c31('0x267')](_0x4ff7de)),
                _0x2fb175;
            }
        }) : Promise[_0x3c31('0x1e')](_0x2fb175);


这个代码基本不可读，简单分析下可以发现，\_0x3c31('0x321')对应一个字符串，'\x6f\x62\x6a'等也可以转义：

先转义：


    function decode(xData) {
        return xData.replace(/\\x(\w{2})/g, function (_, $1) { return String.fromCharCode(parseInt($1, 16)) });
    }


然后替换下：


    Function.prototype.getMultiLine = function () {
        var lines = new String(this);
        lines = lines.substring(lines.indexOf("/*") + 3, lines.lastIndexOf("*/"));
        return lines;
    }
    
    function decode(xData) {
        return xData.replace(/\\x(\w{2})/g, function (_, $1) { return String.fromCharCode(parseInt($1, 16)) });
    }
    
    var str1 = function () {
        /* 
       var _0x5ab652 = _0x50019d(_0x3c31('0x30c'))
    , _0x2cf0a4 = _0x50019d('\x63\x6f\x6d\x70\x4b\x65\x79')
    , _0x5cba8a = {
      '\x74\x79\x70\x65': _0x3c31('0x16c'),
      '\x75\x72\x6c': _0x8aa6f1()
    }
    , _0xfca4af = {
      '\x74\x79\x70\x65': _0x3c31('0x16c'),
      '\x75\x72\x6c': _0xefaeb()
    };
    _0x31f1b8 && (_0x5cba8a[_0x3c31('0x2cb')] = {
      '\x70\x61\x73\x73\x77\x6f\x72\x64': _0x31f1b8
    });
    var _0x11871b = null
    , _0x170c7e = Promise[_0x3c31('0x1e')](null);
     
    var _0x256cd0 = _0x2cf0a4(0x16)
    , _0x36150e = _0x2cf0a4(0x15)
    , _0x42a8d9 = _0x36150e['\x61\x6a\x61\x78']
    , _0xdc46dc = _0x36150e[_0x3c31('0x1ef')]
    , _0xcca797 = _0x2cf0a4(0x18)
    , _0x50019d = _0xcca797[_0x3c31('0x5f')]
    , _0x5c77e3 = _0xcca797['\x70\x61\x72\x73\x65\x55\x72\x6c']
    , _0x36d54a = _0x2cf0a4(0x3a)['\x70\x65\x72\x66\x65\x63\x74\x4d\x65\x74\x61']
    , _0x4c6a9e = _0x2cf0a4(0x2c)[_0x3c31('0x328')]
    , _0x296a9b = _0x2cf0a4(0x2c)[_0x3c31('0x329')]
    , _0x54c90c = _0x2cf0a4(0x17)
    , _0x50f238 = _0x2cf0a4(0x3d)[_0x3c31('0x32a')]
    , 0x13 = 0x13
    , 0x0 = 0x0
    , 0x10 = 0x10
    , CryptoJS = null;
        function _0x230bc7(_0x2fb175) {
        return _0x3c31('0xee') == typeof _0x2fb175[_0x3c31('0x25')] && _0x2fb175['\x6f\x62\x6a'][_0x3c31('0xe')] > 0x64 ? _0x54c90c[_0x3c31('0x31f')]()['\x74\x68\x65\x6e'](function() {
            _0x249a60();
            var _0x5ab652 = null
              , _0x2cf0a4 = null
              , _0x4d1175 = null;
            try {
                var _0x3dbfaa = _0x2fb175[_0x3c31('0x25')]['\x73\x75\x62\x73\x74\x72\x69\x6e\x67'](0x0, 0x13)
                  , _0x360e25 = _0x2fb175[_0x3c31('0x25')][_0x3c31('0xeb')](0x13 + 0x10);
                _0x2cf0a4 = _0x2fb175[_0x3c31('0x25')][_0x3c31('0xeb')](0x13, 0x13 + 0x10),
                _0x4d1175 = _0x2cf0a4,
                _0x5ab652 = _0x3dbfaa + _0x360e25,
                _0x2cf0a4 = CryptoJS['\x65\x6e\x63'][_0x3c31('0x320')][_0x3c31('0x6b')](_0x2cf0a4),
                _0x4d1175 = CryptoJS[_0x3c31('0x321')][_0x3c31('0x320')]['\x70\x61\x72\x73\x65'](_0x4d1175);
                var _0x57e61a = CryptoJS[_0x3c31('0x322')][_0x3c31('0x323')](_0x5ab652, _0x2cf0a4, {
                    '\x69\x76': _0x4d1175,
                    '\x6d\x6f\x64\x65': CryptoJS[_0x3c31('0x324')][_0x3c31('0x325')],
                    '\x70\x61\x64\x64\x69\x6e\x67': CryptoJS[_0x3c31('0x326')][_0x3c31('0x327')]
                });
                return _0x2fb175[_0x3c31('0x36')] = JSON[_0x3c31('0x6b')](CryptoJS[_0x3c31('0x321')][_0x3c31('0x320')][_0x3c31('0x267')](_0x57e61a)),
                _0x2fb175;
            } catch (_0x36fffe) {
                _0x5ab652 = _0x2fb175[_0x3c31('0x25')]['\x73\x75\x62\x73\x74\x72\x69\x6e\x67'](0x0, _0x2fb175[_0x3c31('0x25')][_0x3c31('0xe')] - 0x10),
                _0x2cf0a4 = _0x2fb175[_0x3c31('0x25')][_0x3c31('0xeb')](_0x2fb175[_0x3c31('0x25')][_0x3c31('0xe')] - 0x10),
                _0x4d1175 = _0x2cf0a4,
                _0x2cf0a4 = CryptoJS[_0x3c31('0x321')][_0x3c31('0x320')][_0x3c31('0x6b')](_0x2cf0a4),
                _0x4d1175 = CryptoJS[_0x3c31('0x321')][_0x3c31('0x320')][_0x3c31('0x6b')](_0x4d1175);
                var _0x4ff7de = CryptoJS[_0x3c31('0x322')][_0x3c31('0x323')](_0x5ab652, _0x2cf0a4, {
                    '\x69\x76': _0x4d1175,
                    '\x6d\x6f\x64\x65': CryptoJS['\x6d\x6f\x64\x65'][_0x3c31('0x325')],
                    '\x70\x61\x64\x64\x69\x6e\x67': CryptoJS[_0x3c31('0x326')][_0x3c31('0x327')]
                });
                return _0x2fb175[_0x3c31('0x36')] = JSON[_0x3c31('0x6b')](CryptoJS[_0x3c31('0x321')]['\x55\x74\x66\x38'][_0x3c31('0x267')](_0x4ff7de)),
                _0x2fb175;
            }
        }) : Promise[_0x3c31('0x1e')](_0x2fb175);
    }"
      */
    }
    var js1 = decode(str1.getMultiLine());
    js1 = js1.replace(/_0x3c31\('([^\']+)'\)/g, function ($v, $g) { return _0x3c31($g); })
    


得到


    var _0x5ab652 = _0x50019d(userKey)
    , _0x2cf0a4 = _0x50019d('compKey')
    , _0x5cba8a = {
      'type': GET,
      'url': _0x8aa6f1()
    }
    , _0xfca4af = {
      'type': GET,
      'url': _0xefaeb()
    };
    _0x31f1b8 && (_0x5cba8a[data] = {
      'password': _0x31f1b8
    });
    var _0x11871b = null
    , _0x170c7e = Promise[resolve](null);
     
    var _0x256cd0 = _0x2cf0a4(0x16)
    , _0x36150e = _0x2cf0a4(0x15)
    , _0x42a8d9 = _0x36150e['ajax']
    , _0xdc46dc = _0x36150e[$ajax]
    , _0xcca797 = _0x2cf0a4(0x18)
    , _0x50019d = _0xcca797[getUrlParam]
    , _0x5c77e3 = _0xcca797['parseUrl']
    , _0x36d54a = _0x2cf0a4(0x3a)['perfectMeta']
    , _0x4c6a9e = _0x2cf0a4(0x2c)[isVipScene]
    , _0x296a9b = _0x2cf0a4(0x2c)[isTgScene]
    , _0x54c90c = _0x2cf0a4(0x17)
    , _0x50f238 = _0x2cf0a4(0x3d)[setJsCrypto]
    , 0x13 = 0x13
    , 0x0 = 0x0
    , 0x10 = 0x10
    , CryptoJS = null;
        function _0x230bc7(_0x2fb175) {
        return string == typeof _0x2fb175[obj] && _0x2fb175['obj'][length] > 0x64 ? _0x54c90c[$loadCryptoJS]()['then'](function() {
            _0x249a60();
            var _0x5ab652 = null
              , _0x2cf0a4 = null
              , _0x4d1175 = null;
            try {
                var _0x3dbfaa = _0x2fb175[obj]['substring'](0x0, 0x13)
                  , _0x360e25 = _0x2fb175[obj][substring](0x13 + 0x10);
                _0x2cf0a4 = _0x2fb175[obj][substring](0x13, 0x13 + 0x10),
                _0x4d1175 = _0x2cf0a4,
                _0x5ab652 = _0x3dbfaa + _0x360e25,
                _0x2cf0a4 = CryptoJS['enc'][Utf8][parse](_0x2cf0a4),
                _0x4d1175 = CryptoJS[enc][Utf8]['parse'](_0x4d1175);
                var _0x57e61a = CryptoJS[AES][decrypt](_0x5ab652, _0x2cf0a4, {
                    'iv': _0x4d1175,
                    'mode': CryptoJS[mode][CFB],
                    'padding': CryptoJS[pad][NoPadding]
                });
                return _0x2fb175[list] = JSON[parse](CryptoJS[enc][Utf8][stringify](_0x57e61a)),
                _0x2fb175;
            } catch (_0x36fffe) {
                _0x5ab652 = _0x2fb175[obj]['substring'](0x0, _0x2fb175[obj][length] - 0x10),
                _0x2cf0a4 = _0x2fb175[obj][substring](_0x2fb175[obj][length] - 0x10),
                _0x4d1175 = _0x2cf0a4,
                _0x2cf0a4 = CryptoJS[enc][Utf8][parse](_0x2cf0a4),
                _0x4d1175 = CryptoJS[enc][Utf8][parse](_0x4d1175);
                var _0x4ff7de = CryptoJS[AES][decrypt](_0x5ab652, _0x2cf0a4, {
                    'iv': _0x4d1175,
                    'mode': CryptoJS['mode'][CFB],
                    'padding': CryptoJS[pad][NoPadding]
                });
                return _0x2fb175[list] = JSON[parse](CryptoJS[enc]['Utf8'][stringify](_0x4ff7de)),
                _0x2fb175;
            }
        }) : Promise[resolve](_0x2fb175);
    }"


基板上可以读了，使用CryptoJS做的前端解密，然后直接给最后的代码：


    
    // 依赖： https://lib.eqh5.com/CryptoJS/1.0.1/cryptoJs.js
    function decrypt(result) {
        var ciphertext = null
            , key = null
            , iv = null;
        try {
            var part0 = result.obj.substring(0x0, 0x13)
                , part1 = result.obj.substring(0x13 + 0x10);
            key = result.obj.substring(0x13, 0x13 + 0x10),
                iv = key,
                ciphertext = part0 + part1,
                key = CryptoJS.enc.Utf8.parse(key),
                iv = CryptoJS.enc.Utf8.parse(iv);
            var decryptData = CryptoJS.AES.decrypt(ciphertext, key, {
                'iv': iv,
                'mode': CryptoJS.mode.CFB,
                'padding': CryptoJS.pad.NoPadding
            });
            return CryptoJS.enc.Utf8.stringify(decryptData);
        } catch (_0x36fffe) {
            ciphertext = result[obj]['substring'](0x0, result.obj.length - 0x10),
                key = result[obj][substring](result.obj.length - 0x10),
                iv = key,
                key = CryptoJS.enc.Utf8.parse(key),
                iv = CryptoJS.enc.Utf8.parse(iv);
            var decryptData = CryptoJS.AES.decrypt(ciphertext, key, {
                'iv': iv,
                'mode': CryptoJS.mode.CFB,
                'padding': CryptoJS.pad.NoPadding
            });
            return CryptoJS.enc.Utf8.stringify(decryptData)
        }
    }     
    


