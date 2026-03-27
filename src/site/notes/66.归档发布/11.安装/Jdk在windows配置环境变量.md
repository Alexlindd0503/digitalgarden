---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/Jdk在windows配置环境变量/","dg-note-properties":{"时间":"2026-03-26"}}
---

#java #jdk #环境变量 #安装

```ad-summary
title: 总结

- JAVA_HOME 是其他变量的基础，配错了后面全白搭
- CLASSPATH 现在基本不用了，但加上没坏处
- Path 里加 bin 目录是为了在命令行直接敲 java/javac
- 装完一定验证：`java -version` 和 `javac -version` 都要有输出
```

## 1. 先说结论

装完 JDK 之后，必须配环境变量，不然命令行敲 `java` 会提示"不是内部命令"。一共要搞三个变量：`JAVA_HOME`、`CLASSPATH`、`Path`。

## 2. JAVA_HOME

新建系统变量，值填 JDK 的安装路径：

![1746514338050 a99da2d7 5cf3 4482 ac95 409c4ea5f377](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/1746514338050-a99da2d7-5cf3-4482-ac95-409c4ea5f377.png)

| 项 | 值 |
|----|-----|
| 变量名 | `JAVA_HOME` |
| 变量值 | `C:\Program Files\Java\jdk-17`（改成你实际的路径） |

> 路径里不要带 `bin`，只到 JDK 根目录就行。

这个变量后面两个都要用，**配错了后面全白搭**。很多工具（Maven、IDEA、Tomcat）也靠它找 JDK。

## 3. CLASSPATH

新建或修改系统变量：

```
.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar;
```

- 最前面的 `.` 表示当前目录
- `%JAVA_HOME%` 会自动展开成上面配的路径

> JDK 9 之后 classpath 机制改了，这个变量其实**可以不配**。但加上没坏处，老项目可能还依赖它。

## 4. Path

在系统变量 `Path` 里**追加**两个值：

```
%JAVA_HOME%\bin
%JAVA_HOME%\jre\bin
```

操作步骤：
1. 选中 `Path` → 点"编辑"
2. 点"新建"，分别粘贴上面两条
3. 一路"确定"关掉所有窗口

> 配完 Path 之后，**重新打开命令行**才能生效，已打开的窗口读不到新配置。

## 5. 验证

打开新的 CMD 窗口，执行：

```bash
java -version
javac -version
```

两条命令都输出版本号就说明配好了：

```
java version "17.0.2" 2022-01-18 LTS
javac 17.0.2
```

如果 `java` 有输出但 `javac` 报错，说明 Path 里的 `bin` 路径不对，检查一下。

## 6. 常见问题

### 6.1 装了多个 JDK 怎么办？

`JAVA_HOME` 指向哪个，就用哪个。想切换就改 `JAVA_HOME` 的值，不用动 Path。

### 6.2 命令行还是找不到 java？

- 确认用的是**新的**命令行窗口，旧窗口读不到新环境变量
- 检查 `JAVA_HOME` 路径是否正确，别多带 `bin` 或尾部空格
- 检查 Path 里是不是用了 `%JAVA_HOME%\bin` 而不是写死了绝对路径
