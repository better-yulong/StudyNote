MongoDB客户端 Studio 3T破解方法（方法一注册表方式验证OK): https://www.jianshu.com/p/7257f15e2620



- 问题1：
  <plugin>
        <groupId>com.github.veithen.alta</groupId>
        <artifactId>alta-maven-plugin</artifactId>
        <version>0.6.2</version>
        <executions>
          <execution> 
            <goals>
              <goal>generate-test-resources</goal>
            </goals>
            <configuration>
              <name>%bundle.symbolicName%.link</name>
              <value>%url%</value>
              <dependencySet>
                <scope>test</scope>
              </dependencySet>
            </configuration>
          </execution>
        </executions>
      </plugin>
      <plugin>