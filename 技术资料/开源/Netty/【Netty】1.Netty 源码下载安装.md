1. com.sun.tools 依赖无法找到， netty的pom.xml添加：
			<dependency>
				<groupId>com.sun</groupId>
				<artifactId>tools</artifactId>
				<version>1.7.0</version>
				<scope>system</scope>
				<systemPath>D:/Program Files/Java/jdk1.8.0_231/lib/tools.jar</systemPath>
			</dependency>
2.注释掉 netty的pom.xml的 tcnative.classifier 变更定义及使用














Netty 分享之动态生成重复性的代码: https://www.jianshu.com/p/9160684f134b