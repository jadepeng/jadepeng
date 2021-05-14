---
title: webpack升级到4.x 不完全指南
tags: ["webpack","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-06-08 12:40
---
文章作者:jqpeng
原文链接: [webpack升级到4.x 不完全指南](https://www.cnblogs.com/xiaoqi/p/webpack-upgrade-to-4.html)

最近在团队推行ts，顺便将webpack做了升级，升级到最新的4.X版本，下面记录一些迁移指南。

## VueLoader

### VueLoaderPlugin，显示的引用：


    const VueLoaderPlugin = require('vue-loader/lib/plugin')
    
    module.exports = {
      module: {
        rules: [
          // ... other rules
          {
            test: /\.vue$/,
            loader: 'vue-loader'
          }
        ]
      },
      plugins: [
        // make sure to include the plugin!
        new VueLoaderPlugin()
      ]
    }


### less问题

less需要单独的配置规则,注意一定要加上vue-style-loader，否则样式不生效


    {
      module: {
        rules: [
          // ... other rules
          {
            test: /\.less$/,
            use: [
              'vue-style-loader',
              'css-loader',
              'less-loader'
            ]
          }
        ]
      }
    }


## ts 支持

加上ts-loader


     {
                test: /\.tsx?$/,
                loader: 'ts-loader',
                exclude: /node_modules/,
                options: {
                    appendTsSuffixTo: [/\.vue$/],
                }
            },


## UglifyJsPlugin

UglifyJsPlugin 配置位置发生了变化，放到`optimization`里，如果不想开启`minimize`，可以配置`minimize：false`,开启优化的话，可以配置minimizer：


    // webpack.optimize.UglifyJsPlugin
        optimization: {
            // minimize: false,
            minimizer: [
                new UglifyJsPlugin({
                    cache: true,
                    parallel: true,
                    uglifyOptions: {
                        compress: false
                    },
                    sourceMap: false
                })
            ]
        }


## CopyWebpackPlugin

配置发生了变化：


            // copy custom static assets
            new CopyWebpackPlugin({
                patterns: [{
                    from: path.resolve(__dirname, '../static'),
                    to: config.dev.assetsSubDirectory
                }]
            })


## ExtractTextPlugin

之前：


     new ExtractTextPlugin({
          filename: utils.assetsPath('css/[name].[contenthash].css'),
          allChunks: true,
        }),


contenthash已不能使用，换成hash即可


        new ExtractTextPlugin({
          filename: utils.assetsPath('css/[name].[hash].css'),
          allChunks: true,
        }),


