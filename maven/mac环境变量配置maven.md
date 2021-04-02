homebrew 下载maven3.5

~~~ shell
admindeMacBook-Pro-2:libexec admin$ brew search maven
==> Formulae
maven                     maven-completion          maven-shell               maven@3.2                 maven@3.3                 maven@3.5 ✔

==> Casks
homebrew/cask-fonts/font-maven-pro                                            homebrew/cask-fonts/font-maven-pro-vf-beta
admindeMacBook-Pro-2:libexec admin$ brew install maven@3.5
~~~

修改用户环境变量

![image-20201221175203027](https://tva1.sinaimg.cn/large/0081Kckwly1glvmjq039fj31do0mc44p.jpg)

~~~shell
admindeMacBook-Pro-2:libexec admin$ echo 'export PATH="/usr/local/opt/maven@3.5/bin:$PATH"' >> ~/.bash_profile
admindeMacBook-Pro-2:libexec admin$ source ~/.bash_profile
admindeMacBook-Pro-2:libexec admin$ mvn -v
Apache Maven 3.5.4 (1edded0938998edf8bf061f1ceb3cfdeccf443fe; 2018-06-18T02:33:14+08:00)
Maven home: /usr/local/Cellar/maven@3.5/3.5.4/libexec
Java version: 1.8.0_211, vendor: Oracle Corporation, runtime: /Library/Java/JavaVirtualMachines/jdk1.8.0_211.jdk/Contents/Home/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "mac os x", version: "10.15.2", arch: "x86_64", family: "mac"
~~~

