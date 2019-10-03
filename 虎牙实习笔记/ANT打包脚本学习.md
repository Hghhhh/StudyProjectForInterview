## 前言

实习的时候发布都是通过脚本把代码打包为一个压缩包，然后上传到服务器部署，每次打包都有点云里雾里的，因此有必要研究一下打包脚本的实现。

首先学习一下ant脚本

[Ant脚本教程（一）](https://zhuanlan.zhihu.com/p/26473584)

[Ant脚本教程（二）](https://zhuanlan.zhihu.com/p/26474875)



打包脚本

```ant
<?xml version="1.0" encoding="UTF-8"?>
<project name="java.app.ext_ems_statement" default="main" xmlns:artifact="urn:maven-artifact-ant">

    <property name="git.java.app.url" value="https://test:test@git.test.com/exc/test.git"/>

    <tstamp/>
    <property name="public.time" value="${DSTAMP}"/>

    <target name="init_env">
        <available property="env.properties.present" file="../${env}.properties" type="file" />
        <fail unless="env.properties.present" message="env config not exists" />
        <property file="../${env}.properties" />
        <property file="${config.version}.properties" />
        <property name="app.name" value="ext_ems_statement" />
        <property name="zipfile" value="java_${app.name}_${public.time}_${git.java.app.version}.${env}.zip"/>
        <!-- ant tools -->
        <path id="mylib">
            <fileset dir="${ant.libs}" includes="*.jar" />
        </path>
        <property name="work.source" value="${work.basedir}" />
        <typedef resource="bma/ant/commons/types.properties" classpathref="mylib" />
        <taskdef resource="bma/ant/commons/tasks.properties" classpathref="mylib" />
    </target>

    <target name="init_maven" depends="init_env">
        <!-- maven tools -->
        <typedef resource="org/apache/maven/artifact/ant/antlib.xml" uri="urn:maven-artifact-ant" classpathref="mylib" />
    </target>


    <target name="checkout_java_app" depends="init_env">
        <fail unless="git.java.app.url" message="SCM not set" />
        <mkdir dir="${work.source}/java_app" />
        <available property="appExists" file="${work.source}/java_app/.git/" type="dir"/>
        <antcall target="init_java_app"/>
        <antcall target="update_java_app"/>
        <antcall target="checkout_java_app_version"/>
    </target>
    <target name="init_java_app" unless="appExists">
        <exec executable="${git.bin}">
            <arg value="clone" />
            <arg value="-n" />
            <arg value="${git.java.app.url}" />
            <arg value="${work.source}/java_app" />
        </exec>
    </target>
    <target name="update_java_app" if="appExists">
        <exec executable="${hg.bin}">
            <arg value="pull" />
            <arg value="--tags" />
            <arg value="${git.java.app.url}" />
            <arg value="${git.java.app.version}" />
        </exec>
    </target>
    <target name="checkout_java_app_version">
        <echo message="checkout java_app ${git.java.app.version}"/>
        <exec executable="${git.bin}" dir="${work.source}/java_app">
            <arg value="checkout" />
            <arg value="--quiet" />
            <arg value="${git.java.app.version}" />
        </exec>
    </target>


    <target name="package_app" depends="init_env,checkout_java_app">
        <exec executable="${mvn.bat}">
            <arg line="-gs ${mvn.gs} -f ${work.source}/java_app/java_app/app_${app.name}/pom.xml clean" />
        </exec>
        <exec executable="${mvn.bat}">
            <arg line="-gs ${mvn.gs} -f ${work.source}/java_app/java_app/app_${app.name}/pom.xml -U install" />
        </exec>
    </target>

    <target name="cleanup" depends="init_env">
    </target>

    <target name="unpack" depends="init_maven">
        <artifact:pom id="maven.project" file="${work.source}/java_app/java_app/app_${app.name}/pom.xml" />
        <echo>unpack the ${maven.project.build.finalName}.jar</echo>
        <property name="appjarfile" value="${maven.project.build.directory}/${maven.project.build.finalName}.jar"/>
        <unzip src="${appjarfile}" dest="${work.source}/jar" />
    </target>

    <target name="writeConf" depends="init_env">
        <copy todir="${work.source}/deploy_jar" filtering="true" encoding="UTF-8">
            <fileset dir="${work.source}/jar" includes="">
                <include name="**/*.properties"/>
            </fileset>
            <filterset begintoken="${" endtoken="}">
                <filtersfile file="deploy_env.protest_common.properties" />
            </filterset>
            <filterset begintoken="${" endtoken="}">
                <filtersfile file="deploy_env.protest_a.properties" />
            </filterset>
        </copy>
        <copy todir="${work.source}/deploy_jar" encoding="UTF-8">
            <fileset dir="${work.source}/jar" />
        </copy>
    </target>

    <target name="repack" depends="init_maven">
        <jar destfile="${pack.source}/libs/${maven.project.build.finalName}.jar" basedir="${work.source}/deploy_jar"/>
    </target>

    <target name="build_workspace" depends="init_env">
        <copy todir="${work.basedir}/workspace" filtering="true" encoding="UTF-8">
            <fileset dir="${work.basedir}/java_app/java_app/app_${app.name}/">
                <include name="**/*.sh"/>
            </fileset>
            <filterset begintoken="$[" endtoken="]">
                <filtersfile file="deploy_env.protest_common.properties" />
            </filterset>
        </copy>
        <copy todir="${work.basedir}/workspace/libs" encoding="UTF-8">
            <fileset dir="${work.basedir}/java_app/java_app/app_${app.name}/target/libs" />
        </copy>
        <copy todir="${work.basedir}/workspace/classes" encoding="UTF-8">
            <fileset dir="${work.basedir}/deploy_jar" includes="**/*" />
        </copy>
        <touch file="${work.basedir}/workspace/version-${git.java.app.version}"/>
        <echo>${basedir}/${zipfile}</echo>
        <echo>${work.basedir}/workspace</echo>
        <zip destfile="${basedir}/${zipfile}" basedir="${work.basedir}/workspace" />
    </target>

    <target name="buildAll" depends="cleanup,package_app,unpack,writeConf,build_workspace"/>
    <target name="package" depends="cleanup,package_app,unpack,writeConf,build_workspace"/>
    <target name="packagetest" depends="cleanup,package_app,unpack,writeConf,build_workspace"/>

    <target name="main">
        <echo message="buildAll? package?" />
    </target>

</project>
```

