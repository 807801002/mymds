目录

[前言](https://calibre-internal.invalid/OEBPS/text00002.html)

[致谢](https://calibre-internal.invalid/OEBPS/text00003.html)

[第1章　Maven简介](https://calibre-internal.invalid/OEBPS/text00004.html)

[1.1　何为Maven](https://calibre-internal.invalid/OEBPS/text00005.html)

[1.1.1　何为构建](https://calibre-internal.invalid/OEBPS/text00006.html)

[1.1.2　Maven是优秀的构建工具](https://calibre-internal.invalid/OEBPS/text00007.html)

[1.1.3　Maven不仅仅是构建工具](https://calibre-internal.invalid/OEBPS/text00008.html)

[1.2　为什么需要Maven](https://calibre-internal.invalid/OEBPS/text00010.html)

[1.2.1　组装PC和品牌PC](https://calibre-internal.invalid/OEBPS/text00010.html)

[1.2.2　IDE不是万能的](https://calibre-internal.invalid/OEBPS/text00011.html)

[1.2.3　Make](https://calibre-internal.invalid/OEBPS/text00012.html)

[1.2.4　Ant](https://calibre-internal.invalid/OEBPS/text00013.html)

[1.2.5　不重复发明轮子](https://calibre-internal.invalid/OEBPS/text00014.html#ch1)

[1.3　Maven与极限编程](https://calibre-internal.invalid/OEBPS/text00015.html)

[1.4　被误解的Maven](https://calibre-internal.invalid/OEBPS/text00016.html)

[1.5　小结](https://calibre-internal.invalid/OEBPS/text00017.html)

[第2章　Maven的安装和配置](https://calibre-internal.invalid/OEBPS/text00018.html)

[2.1　在Windows上安装Maven](https://calibre-internal.invalid/OEBPS/text00019.html)

[2.1.1　检查JDK安装](https://calibre-internal.invalid/OEBPS/text00019.html)

[2.1.2　下载Maven](https://calibre-internal.invalid/OEBPS/text00020.html)

[2.1.3　本地安装](https://calibre-internal.invalid/OEBPS/text00021.html)

[2.1.4　升级Maven](https://calibre-internal.invalid/OEBPS/text00022.html)

[2.2　在基于UNIX的系统上安装Maven](https://calibre-internal.invalid/OEBPS/text00024.html)

[2.2.1　下载和安装](https://calibre-internal.invalid/OEBPS/text00024.html)

[2.2.2　升级Maven](https://calibre-internal.invalid/OEBPS/text00025.html)

[2.3　安装目录分析](https://calibre-internal.invalid/OEBPS/text00027.html)

[2.3.1　M2_HOME](https://calibre-internal.invalid/OEBPS/text00027.html)

[2.3.2　~/.m2](https://calibre-internal.invalid/OEBPS/text00028.html)

[2.4　设置HTTP代理](https://calibre-internal.invalid/OEBPS/text00029.html)

[2.5　安装m2eclipse](https://calibre-internal.invalid/OEBPS/text00030.html)

[2.6　安装NetBeans　Maven插件](https://calibre-internal.invalid/OEBPS/text00031.html)

[2.7　Maven安装最佳实践](https://calibre-internal.invalid/OEBPS/text00033.html)

[2.7.1　设置MAVEN_OPTS环境变量](https://calibre-internal.invalid/OEBPS/text00033.html)

[2.7.2　配置用户范围settings.xml](https://calibre-internal.invalid/OEBPS/text00034.html)

[2.7.3　不要使用IDE内嵌的Maven](https://calibre-internal.invalid/OEBPS/text00035.html)

[2.8　小结](https://calibre-internal.invalid/OEBPS/text00036.html)

[第3章　Maven使用入门](https://calibre-internal.invalid/OEBPS/text00037.html)

[3.1　编写POM](https://calibre-internal.invalid/OEBPS/text00038.html)

[3.2　编写主代码](https://calibre-internal.invalid/OEBPS/text00039.html)

[3.3　编写测试代码](https://calibre-internal.invalid/OEBPS/text00040.html)

[3.4　打包和运行](https://calibre-internal.invalid/OEBPS/text00041.html)

[3.5　使用Archetype生成项目骨架](https://calibre-internal.invalid/OEBPS/text00042.html)

[3.6　m2eclipse简单使用](https://calibre-internal.invalid/OEBPS/text00043.html)

[3.6.1　导入Maven项目](https://calibre-internal.invalid/OEBPS/text00044.html)

[3.6.2　创建Maven项目](https://calibre-internal.invalid/OEBPS/text00045.html)

[3.6.3　运行mvn命令](https://calibre-internal.invalid/OEBPS/text00046.html)

[3.7　NetBeans　Maven插件简单使用](https://calibre-internal.invalid/OEBPS/text00048.html)

[3.7.1　打开Maven项目](https://calibre-internal.invalid/OEBPS/text00048.html)

[3.7.2　创建Maven项目](https://calibre-internal.invalid/OEBPS/text00049.html)

[3.7.3　运行mvn命令](https://calibre-internal.invalid/OEBPS/text00050.html)

[3.8　小结](https://calibre-internal.invalid/OEBPS/text00051.html)

[第4章　背景案例](https://calibre-internal.invalid/OEBPS/text00052.html)

[4.1　简单的账户注册服务](https://calibre-internal.invalid/OEBPS/text00053.html)

[4.2　需求阐述](https://calibre-internal.invalid/OEBPS/text00055.html)

[4.2.1　需求用例](https://calibre-internal.invalid/OEBPS/text00055.html)

[4.2.2　界面原型](https://calibre-internal.invalid/OEBPS/text00056.html)

[4.3　简要设计](https://calibre-internal.invalid/OEBPS/text00057.html)

[4.3.1　接口](https://calibre-internal.invalid/OEBPS/text00057.html)

[4.3.2　模块结构](https://calibre-internal.invalid/OEBPS/text00058.html)

[4.4　小结](https://calibre-internal.invalid/OEBPS/text00059.html)

[第5章　坐标和依赖](https://calibre-internal.invalid/OEBPS/text00060.html)

[5.1　何为Maven坐标](https://calibre-internal.invalid/OEBPS/text00061.html)

[5.2　坐标详解](https://calibre-internal.invalid/OEBPS/text00062.html)

[5.3　account-email](https://calibre-internal.invalid/OEBPS/text00063.html)

[5.3.1　account-email的POM](https://calibre-internal.invalid/OEBPS/text00064.html)

[5.3.2　account-email的主代码](https://calibre-internal.invalid/OEBPS/text00065.html)

[5.3.3　account-email的测试代码](https://calibre-internal.invalid/OEBPS/text00066.html)

[5.3.4　构建account-email](https://calibre-internal.invalid/OEBPS/text00067.html)

[5.4　依赖的配置](https://calibre-internal.invalid/OEBPS/text00068.html)

[5.5　依赖范围](https://calibre-internal.invalid/OEBPS/text00069.html)

[5.6　传递性依赖](https://calibre-internal.invalid/OEBPS/text00070.html)

[5.6.1　何为传递性依赖](https://calibre-internal.invalid/OEBPS/text00070.html)

[5.6.2　传递性依赖和依赖范围](https://calibre-internal.invalid/OEBPS/text00071.html)

[5.7　依赖调解](https://calibre-internal.invalid/OEBPS/text00072.html)

[5.8　可选依赖](https://calibre-internal.invalid/OEBPS/text00073.html)

[5.9　最佳实践](https://calibre-internal.invalid/OEBPS/text00074.html)

[5.9.1　排除依赖](https://calibre-internal.invalid/OEBPS/text00075.html)

[5.9.2　归类依赖](https://calibre-internal.invalid/OEBPS/text00076.html)

[5.9.3　优化依赖](https://calibre-internal.invalid/OEBPS/text00077.html)

[5.10　小结](https://calibre-internal.invalid/OEBPS/text00078.html)

[第6章　仓库](https://calibre-internal.invalid/OEBPS/text00079.html)

[6.1　何为Maven仓库](https://calibre-internal.invalid/OEBPS/text00080.html)

[6.2　仓库的布局](https://calibre-internal.invalid/OEBPS/text00081.html)

[6.3　仓库的分类](https://calibre-internal.invalid/OEBPS/text00082.html)

[6.3.1　本地仓库](https://calibre-internal.invalid/OEBPS/text00083.html)

[6.3.2　远程仓库](https://calibre-internal.invalid/OEBPS/text00084.html)

[6.3.3　中央仓库](https://calibre-internal.invalid/OEBPS/text00085.html)

[6.3.4　私服](https://calibre-internal.invalid/OEBPS/text00086.html)

[6.4　远程仓库的配置](https://calibre-internal.invalid/OEBPS/text00087.html)

[6.4.1　远程仓库的认证](https://calibre-internal.invalid/OEBPS/text00088.html)

[6.4.2　部署至远程仓库](https://calibre-internal.invalid/OEBPS/text00089.html)

[6.5　快照版本](https://calibre-internal.invalid/OEBPS/text00090.html)

[6.6　从仓库解析依赖的机制](https://calibre-internal.invalid/OEBPS/text00091.html)

[6.7　镜像](https://calibre-internal.invalid/OEBPS/text00092.html)

[6.8　仓库搜索服务](https://calibre-internal.invalid/OEBPS/text00093.html)

[6.8.1　Sonatype　Nexus](https://calibre-internal.invalid/OEBPS/text00094.html)

[6.8.2　Jarvana](https://calibre-internal.invalid/OEBPS/text00095.html)

[6.8.3　MVNbrowser](https://calibre-internal.invalid/OEBPS/text00096.html)

[6.8.4　MVNrepository](https://calibre-internal.invalid/OEBPS/text00097.html)

[6.8.5　选择合适的仓库搜索服务](https://calibre-internal.invalid/OEBPS/text00098.html)

[6.9　小结](https://calibre-internal.invalid/OEBPS/text00099.html)

[第7章　生命周期和插件](https://calibre-internal.invalid/OEBPS/text00100.html)

[7.1　何为生命周期](https://calibre-internal.invalid/OEBPS/text00101.html)

[7.2　生命周期详解](https://calibre-internal.invalid/OEBPS/text00103.html)

[7.2.1　三套生命周期](https://calibre-internal.invalid/OEBPS/text00103.html)

[7.2.2　clean生命周期](https://calibre-internal.invalid/OEBPS/text00104.html)

[7.2.3　default生命周期](https://calibre-internal.invalid/OEBPS/text00105.html)

[7.2.4　site生命周期](https://calibre-internal.invalid/OEBPS/text00106.html)

[7.2.5　命令行与生命周期](https://calibre-internal.invalid/OEBPS/text00107.html)

[7.3　插件目标](https://calibre-internal.invalid/OEBPS/text00108.html)

[7.4　插件绑定](https://calibre-internal.invalid/OEBPS/text00109.html)

[7.4.1　内置绑定](https://calibre-internal.invalid/OEBPS/text00110.html)

[7.4.2　自定义绑定](https://calibre-internal.invalid/OEBPS/text00111.html)

[7.5　插件配置](https://calibre-internal.invalid/OEBPS/text00112.html)

[7.5.1　命令行插件配置](https://calibre-internal.invalid/OEBPS/text00113.html)

[7.5.2　POM中插件全局配置](https://calibre-internal.invalid/OEBPS/text00114.html)

[7.5.3　POM中插件任务配置](https://calibre-internal.invalid/OEBPS/text00115.html)

[7.6　获取插件信息](https://calibre-internal.invalid/OEBPS/text00116.html)

[7.6.1　在线插件信息](https://calibre-internal.invalid/OEBPS/text00117.html)

[7.6.2　使用maven-help-plugin描述插件](https://calibre-internal.invalid/OEBPS/text00118.html)

[7.7　从命令行调用插件](https://calibre-internal.invalid/OEBPS/text00119.html)

[7.8　插件解析机制](https://calibre-internal.invalid/OEBPS/text00120.html)

[7.8.1　插件仓库](https://calibre-internal.invalid/OEBPS/text00121.html)

[7.8.2　插件的默认groupId](https://calibre-internal.invalid/OEBPS/text00122.html)

[7.8.3　解析插件版本](https://calibre-internal.invalid/OEBPS/text00123.html)

[7.8.4　解析插件前缀](https://calibre-internal.invalid/OEBPS/text00124.html)

[7.9　小结](https://calibre-internal.invalid/OEBPS/text00125.html)

[第8章　聚合与继承](https://calibre-internal.invalid/OEBPS/text00126.html)

[8.1　account-persist](https://calibre-internal.invalid/OEBPS/text00127.html)

[8.1.1　account-persist的POM](https://calibre-internal.invalid/OEBPS/text00128.html)

[8.1.2　account-persist的主代码](https://calibre-internal.invalid/OEBPS/text00129.html)

[8.1.3　account-persist的测试代码](https://calibre-internal.invalid/OEBPS/text00130.html)

[8.2　聚合](https://calibre-internal.invalid/OEBPS/text00131.html)

[8.3　继承](https://calibre-internal.invalid/OEBPS/text00132.html)

[8.3.1　account-parent](https://calibre-internal.invalid/OEBPS/text00133.html)

[8.3.2　可继承的POM元素](https://calibre-internal.invalid/OEBPS/text00134.html)

[8.3.3　依赖管理](https://calibre-internal.invalid/OEBPS/text00135.html)

[8.3.4　插件管理](https://calibre-internal.invalid/OEBPS/text00136.html)

[8.4　聚合与继承的关系](https://calibre-internal.invalid/OEBPS/text00137.html)

[8.5　约定优于配置](https://calibre-internal.invalid/OEBPS/text00138.html)

[8.6　反应堆](https://calibre-internal.invalid/OEBPS/text00139.html)

[8.6.1　反应堆的构建顺序](https://calibre-internal.invalid/OEBPS/text00140.html)

[8.6.2　裁剪反应堆](https://calibre-internal.invalid/OEBPS/text00141.html)

[8.7　小结](https://calibre-internal.invalid/OEBPS/text00142.html)

[第9章　使用Nexus创建私服](https://calibre-internal.invalid/OEBPS/text00143.html)

[9.1　Nexus简介](https://calibre-internal.invalid/OEBPS/text00144.html)

[9.2　安装Nexus](https://calibre-internal.invalid/OEBPS/text00146.html)

[9.2.1　下载Nexus](https://calibre-internal.invalid/OEBPS/text00146.html)

[9.2.2　Bundle方式安装Nexus](https://calibre-internal.invalid/OEBPS/text00147.html)

[9.2.3　WAR方式安装Nexus](https://calibre-internal.invalid/OEBPS/text00148.html)

[9.2.4　登录Nexus](https://calibre-internal.invalid/OEBPS/text00149.html)

[9.3　Nexus的仓库与仓库组](https://calibre-internal.invalid/OEBPS/text00150.html)

[9.3.1　Nexus内置的仓库](https://calibre-internal.invalid/OEBPS/text00151.html)

[9.3.2　Nexus仓库分类的概念](https://calibre-internal.invalid/OEBPS/text00152.html)

[9.3.3　创建Nexus宿主仓库](https://calibre-internal.invalid/OEBPS/text00153.html)

[9.3.4　创建Nexus代理仓库](https://calibre-internal.invalid/OEBPS/text00154.html)

[9.3.5　创建Nexus仓库组](https://calibre-internal.invalid/OEBPS/text00155.html)

[9.4　Nexus的索引与构件搜索](https://calibre-internal.invalid/OEBPS/text00156.html)

[9.5　配置Maven从Nexus下载构件](https://calibre-internal.invalid/OEBPS/text00157.html)

[9.6　部署构件至Nexus](https://calibre-internal.invalid/OEBPS/text00158.html)

[9.6.1　使用Maven部署构件至Nexus](https://calibre-internal.invalid/OEBPS/text00159.html)

[9.6.2　手动部署第三方构件至Nexus](https://calibre-internal.invalid/OEBPS/text00160.html)

[9.7　Nexus的权限管理](https://calibre-internal.invalid/OEBPS/text00161.html)

[9.7.1　Nexus的访问控制模型](https://calibre-internal.invalid/OEBPS/text00162.html)

[9.7.2　为项目分配独立的仓库](https://calibre-internal.invalid/OEBPS/text00163.html)

[9.8　Nexus的调度任务](https://calibre-internal.invalid/OEBPS/text00164.html)

[9.9　其他私服软件](https://calibre-internal.invalid/OEBPS/text00165.html)

[9.10　小结](https://calibre-internal.invalid/OEBPS/text00166.html)

[第10章　使用Maven进行测试](https://calibre-internal.invalid/OEBPS/text00167.html)

[10.1　account-captcha](https://calibre-internal.invalid/OEBPS/text00168.html)

[10.1.1　account-captcha的POM](https://calibre-internal.invalid/OEBPS/text00169.html)

[10.1.2　account-captcha的主代码](https://calibre-internal.invalid/OEBPS/text00170.html)

[10.1.3　account-captcha的测试代码](https://calibre-internal.invalid/OEBPS/text00171.html)

[10.2　maven-surefire-plugin简介](https://calibre-internal.invalid/OEBPS/text00172.html)

[10.3　跳过测试](https://calibre-internal.invalid/OEBPS/text00173.html)

[10.4　动态指定要运行的测试用例](https://calibre-internal.invalid/OEBPS/text00174.html)

[10.5　包含与排除测试用例](https://calibre-internal.invalid/OEBPS/text00175.html)

[10.6　测试报告](https://calibre-internal.invalid/OEBPS/text00177.html)

[10.6.1　基本的测试报告](https://calibre-internal.invalid/OEBPS/text00177.html)

[10.6.2　测试覆盖率报告](https://calibre-internal.invalid/OEBPS/text00178.html)

[10.7　运行TestNG测试](https://calibre-internal.invalid/OEBPS/text00179.html)

[10.8　重用测试代码](https://calibre-internal.invalid/OEBPS/text00180.html)

[10.9　小结](https://calibre-internal.invalid/OEBPS/text00181.html)

[第11章　使用Hudson进行持续集成](https://calibre-internal.invalid/OEBPS/text00182.html)

[11.1　持续集成的作用、过程和优势](https://calibre-internal.invalid/OEBPS/text00183.html)

[11.2　Hudson简介](https://calibre-internal.invalid/OEBPS/text00184.html)

[11.3　安装Hudson](https://calibre-internal.invalid/OEBPS/text00185.html)

[11.4　准备Subversion仓库](https://calibre-internal.invalid/OEBPS/text00186.html)

[11.5　Hudson的基本系统设置](https://calibre-internal.invalid/OEBPS/text00187.html)

[11.6　创建Hudson任务](https://calibre-internal.invalid/OEBPS/text00188.html)

[11.6.1　Hudson任务的基本配置](https://calibre-internal.invalid/OEBPS/text00189.html)

[11.6.2　Hudson任务的源码仓库配置](https://calibre-internal.invalid/OEBPS/text00190.html)

[11.6.3　Hudson任务的构建触发配置](https://calibre-internal.invalid/OEBPS/text00191.html)

[11.6.4　Hudson任务的构建配置](https://calibre-internal.invalid/OEBPS/text00192.html)

[11.7　监视Hudson任务状态](https://calibre-internal.invalid/OEBPS/text00193.html)

[11.7.1　全局任务状态](https://calibre-internal.invalid/OEBPS/text00194.html)

[11.7.2　自定义任务视图](https://calibre-internal.invalid/OEBPS/text00195.html)

[11.7.3　单个任务状态](https://calibre-internal.invalid/OEBPS/text00196.html)

[11.7.4　Maven项目测试报告](https://calibre-internal.invalid/OEBPS/text00197.html)

[11.8　Hudson用户管理](https://calibre-internal.invalid/OEBPS/text00198.html)

[11.9　邮件反馈](https://calibre-internal.invalid/OEBPS/text00199.html)

[11.10　Hudson工作目录](https://calibre-internal.invalid/OEBPS/text00200.html)

[11.11　小结](https://calibre-internal.invalid/OEBPS/text00201.html)

[第12章　使用Maven构建Web应用](https://calibre-internal.invalid/OEBPS/text00202.html)

[12.1　Web项目的目录结构](https://calibre-internal.invalid/OEBPS/text00203.html)

[12.2　account-service](https://calibre-internal.invalid/OEBPS/text00204.html)

[12.2.1　account-service的POM](https://calibre-internal.invalid/OEBPS/text00205.html)

[12.2.2　account-service的主代码](https://calibre-internal.invalid/OEBPS/text00206.html)

[12.3　account-web](https://calibre-internal.invalid/OEBPS/text00207.html)

[12.3.1　account-web的POM](https://calibre-internal.invalid/OEBPS/text00208.html)

[12.3.2　account-web的主代码](https://calibre-internal.invalid/OEBPS/text00209.html)

[12.4　使用jetty-maven-plugin进行测试](https://calibre-internal.invalid/OEBPS/text00210.html)

[12.5　使用Cargo实现自动化部署](https://calibre-internal.invalid/OEBPS/text00211.html)

[12.5.1　部署至本地Web容器](https://calibre-internal.invalid/OEBPS/text00212.html)

[12.5.2　部署至远程Web容器](https://calibre-internal.invalid/OEBPS/text00213.html)

[12.6　小结](https://calibre-internal.invalid/OEBPS/text00214.html)

[第13章　版本管理](https://calibre-internal.invalid/OEBPS/text00215.html)

[13.1　何为版本管理](https://calibre-internal.invalid/OEBPS/text00216.html)

[13.2　Maven的版本号定义约定](https://calibre-internal.invalid/OEBPS/text00217.html)

[13.3　主干、标签与分支](https://calibre-internal.invalid/OEBPS/text00218.html)

[13.4　自动化版本发布](https://calibre-internal.invalid/OEBPS/text00219.html)

[13.5　自动化创建分支](https://calibre-internal.invalid/OEBPS/text00220.html)

[13.6　GPG签名](https://calibre-internal.invalid/OEBPS/text00221.html)

[13.6.1　GPG及其基本使用](https://calibre-internal.invalid/OEBPS/text00222.html)

[13.6.2　Maven　GPG　Plugin](https://calibre-internal.invalid/OEBPS/text00223.html)

[13.7　小结](https://calibre-internal.invalid/OEBPS/text00224.html)

[第14章　灵活的构建](https://calibre-internal.invalid/OEBPS/text00225.html)

[14.1　Maven属性](https://calibre-internal.invalid/OEBPS/text00226.html)

[14.2　构建环境的差异](https://calibre-internal.invalid/OEBPS/text00227.html)

[14.3　资源过滤](https://calibre-internal.invalid/OEBPS/text00228.html)

[14.4　Maven　Profile](https://calibre-internal.invalid/OEBPS/text00229.html)

[14.4.1　针对不同环境的profile](https://calibre-internal.invalid/OEBPS/text00230.html)

[14.4.2　激活profile](https://calibre-internal.invalid/OEBPS/text00231.html)

[14.4.3　profile的种类](https://calibre-internal.invalid/OEBPS/text00232.html)

[14.5　Web资源过滤](https://calibre-internal.invalid/OEBPS/text00233.html)

[14.6　在profile中激活集成测试](https://calibre-internal.invalid/OEBPS/text00234.html)

[14.7　小结](https://calibre-internal.invalid/OEBPS/text00235.html)

[第15章　生成项目站点](https://calibre-internal.invalid/OEBPS/text00236.html)

[15.1　最简单的站点](https://calibre-internal.invalid/OEBPS/text00237.html)

[15.2　丰富项目信息](https://calibre-internal.invalid/OEBPS/text00238.html)

[15.3　项目报告插件](https://calibre-internal.invalid/OEBPS/text00239.html)

[15.3.1　JavaDocs](https://calibre-internal.invalid/OEBPS/text00240.html)

[15.3.2　Source　Xref](https://calibre-internal.invalid/OEBPS/text00241.html)

[15.3.3　CheckStyle](https://calibre-internal.invalid/OEBPS/text00242.html)

[15.3.4　PMD](https://calibre-internal.invalid/OEBPS/text00243.html)

[15.3.5　ChangeLog](https://calibre-internal.invalid/OEBPS/text00244.html)

[15.3.6　Cobertura](https://calibre-internal.invalid/OEBPS/text00245.html)

[15.4　自定义站点外观](https://calibre-internal.invalid/OEBPS/text00246.html)

[15.4.1　站点描述符](https://calibre-internal.invalid/OEBPS/text00247.html)

[15.4.2　头部内容及外观](https://calibre-internal.invalid/OEBPS/text00248.html)

[15.4.3　皮肤](https://calibre-internal.invalid/OEBPS/text00249.html)

[15.4.4　导航边栏](https://calibre-internal.invalid/OEBPS/text00250.html)

[15.5　创建自定义页面](https://calibre-internal.invalid/OEBPS/text00251.html)

[15.6　国际化](https://calibre-internal.invalid/OEBPS/text00252.html)

[15.7　部署站点](https://calibre-internal.invalid/OEBPS/text00253.html)

[15.8　小结](https://calibre-internal.invalid/OEBPS/text00254.html)

[第16章　m2eclipse](https://calibre-internal.invalid/OEBPS/text00255.html)

[16.1　m2eclipse简介](https://calibre-internal.invalid/OEBPS/text00256.html)

[16.2　新建Maven项目](https://calibre-internal.invalid/OEBPS/text00257.html)

[16.3　导入Maven项目](https://calibre-internal.invalid/OEBPS/text00258.html)

[16.3.1　导入本地Maven项目](https://calibre-internal.invalid/OEBPS/text00259.html)

[16.3.2　从SCM仓库导入Maven项目](https://calibre-internal.invalid/OEBPS/text00260.html)

[16.3.3　m2eclipse中Maven项目的结构](https://calibre-internal.invalid/OEBPS/text00261.html)

[16.4　执行mvn命令](https://calibre-internal.invalid/OEBPS/text00262.html)

[16.5　访问Maven仓库](https://calibre-internal.invalid/OEBPS/text00263.html)

[16.5.1　Maven仓库视图](https://calibre-internal.invalid/OEBPS/text00264.html)

[16.5.2　搜索构件和Java类](https://calibre-internal.invalid/OEBPS/text00265.html)

[16.6　管理项目依赖](https://calibre-internal.invalid/OEBPS/text00266.html)

[16.6.1　添加依赖](https://calibre-internal.invalid/OEBPS/text00267.html)

[16.6.2　分析依赖](https://calibre-internal.invalid/OEBPS/text00268.html)

[16.7　其他实用功能](https://calibre-internal.invalid/OEBPS/text00269.html)

[16.7.1　POM编辑的代码提示](https://calibre-internal.invalid/OEBPS/text00270.html)

[16.7.2　Effective　POM](https://calibre-internal.invalid/OEBPS/text00271.html)

[16.7.3　下载依赖源码](https://calibre-internal.invalid/OEBPS/text00272.html)

[16.8　小结](https://calibre-internal.invalid/OEBPS/text00273.html)

[第17章　编写Maven插件](https://calibre-internal.invalid/OEBPS/text00274.html)

[17.1　编写Maven插件的一般步骤](https://calibre-internal.invalid/OEBPS/text00275.html)

[17.2　案例：编写一个用于代码行统计的Maven插件](https://calibre-internal.invalid/OEBPS/text00276.html)

[17.3　Mojo标注](https://calibre-internal.invalid/OEBPS/text00277.html)

[17.4　Mojo参数](https://calibre-internal.invalid/OEBPS/text00278.html)

[17.5　错误处理和日志](https://calibre-internal.invalid/OEBPS/text00279.html)

[17.6　测试Maven插件](https://calibre-internal.invalid/OEBPS/text00280.html)

[17.7　小结](https://calibre-internal.invalid/OEBPS/text00281.html)

[第18章　Archetype](https://calibre-internal.invalid/OEBPS/text00282.html)

[18.1　Archetype使用再叙](https://calibre-internal.invalid/OEBPS/text00284.html)

[18.1.1　Maven　Archetype　Plugin](https://calibre-internal.invalid/OEBPS/text00284.html)

[18.1.2　使用Archetype的一般步骤](https://calibre-internal.invalid/OEBPS/text00285.html)

[18.1.3　批处理方式使用Archetype](https://calibre-internal.invalid/OEBPS/text00286.html)

[18.1.4　常用Archetype介绍](https://calibre-internal.invalid/OEBPS/text00287.html)

[18.2　编写Archetype](https://calibre-internal.invalid/OEBPS/text00288.html)

[18.3　Archetype　Catalog](https://calibre-internal.invalid/OEBPS/text00289.html)

[18.3.1　什么是Archetype　Catalog](https://calibre-internal.invalid/OEBPS/text00290.html)

[18.3.2　Archetype　Catalog的来源](https://calibre-internal.invalid/OEBPS/text00291.html)

[18.3.3　生成本地仓库的Archetype　Catalog](https://calibre-internal.invalid/OEBPS/text00292.html)

[18.3.4　使用nexus-archetype-plugin](https://calibre-internal.invalid/OEBPS/text00293.html)

[18.4　小结](https://calibre-internal.invalid/OEBPS/text00294.html)

[附录A　POM元素参考](https://calibre-internal.invalid/OEBPS/text00296.html)

[附录B　Settings元素参考](https://calibre-internal.invalid/OEBPS/text00297.html)

[附录C　常用插件列表](https://calibre-internal.invalid/OEBPS/text00297.html)

# 第1章　Maven简介

本章内容

·何为Maven

·为什么需要Maven

·Maven与极限编程

·被误解的Maven

·小结

## 1.1　何为Maven

Maven这个词可以翻译为“知识的积累”，也可以翻译为“专家”或“内行”。本书将介绍Maven这一跨平台的项目管理工具。作为Apache组织中的一个颇为成功的开源项目，Maven主要服务于基于Java平台的项目构建、依赖管理和项目信息管理。无论是小型的开源类库项目，还是大型的企业级应用；无论是传统的瀑布式开发，还是流行的敏捷模式，Maven都能大显身手。

### 1.1.1　何为构建

不管你是否意识到，构建（build）是每一位程序员每天都在做的工作。早上来到公司，我们做的第一件事情就是从源码库签出最新的源码，然后进行单元测试，如果发现失败的测试，会找相关的同事一起调试，修复错误代码。接着回到自己的工作上来，编写自己的单元测试及产品代码，我们会感激IDE随时报出的编译错误提示。

忙到午饭时间，代码编写得差不多了，测试也通过了，开心地享用午餐，然后休息。下午先在昏昏沉沉中开了个例会，会议结束后喝杯咖啡继续工作。刚才在会上经理要求看测试报告，于是找了相关工具集成进IDE，生成了像模像样的测试覆盖率报告，接着发了一封电子邮件给经理，松了口气。谁料QA小组又发过来了几个bug，没办法，先本地重现再说，于是熟练地用IDE生成了一个WAR包，部署到Web容器下，启动容器。看到熟悉的界面了，遵循bug报告，一步步重现了bug……快下班的时候，bug修好了，提交代码，通知QA小组，在愉快中结束了一天的工作。

仔细总结一下，我们会发现，除了编写源代码，我们每天有相当一部分时间花在了编译、运行单元测试、生成文档、打包和部署等烦琐且不起眼的工作上，这就是构建。如果我们现在还手工这样做，那成本也太高了，于是有人用软件的方法让这一系列工作完全自动化，使得软件的构建可以像全自动流水线一样，只需要一条简单的命令，所有烦琐的步骤都能够自动完成，很快就能得到最终结果。