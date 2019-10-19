###安装Flutter时执行pod setup速度慢的解决办法

当我们去执行pod setup的时候，会发现那是一个相当的慢。估计一天的时间都浪费再这上面。这是因为使用的国外的镜像，只要使用国内的镜像就很好的解决了。
只要使用 cd ~/.cocoapods/repos
然后 执行 pod repo remove master来删除master文件
再执行 git clone https://mirrors.tuna.tsinghua.edu.cn/git/CocoaPods/Specs.git master