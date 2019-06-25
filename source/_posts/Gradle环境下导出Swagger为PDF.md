---
title: Gradle环境下导出Swagger为PDF
author: 鹏徙南冥
top: false
mathjax: false
categories:
  - java
tags:
  - gradle
  - swagger
  - swagger2markup
  - asciidoctor
date: 2019-06-25 14:14:50
updated: 2019-06-25 14:14:50
img:
summary:
---
## 说明
我个人是一直使用Swagger作为接口文档的说明的。但是由于在一些情况下，接口文档说明需要以文件的形式交付出去，如果再重新写一份文档难免有些麻烦。于是在网上看到了Swagger2Markup + asciidoctor导出PDF的方法，百度一番后感觉网上的文章还是有很多没有描述清楚的地方，遂还是硬着头皮把官方的英文文档大致浏览了一下，按照自己的思路整理出具体的步骤。

本文用到的工具：
- Gradle - 4.10.3
- SpringBoot - 2.1.6.RELEASE
- Swagger - 2.9.2
- Swagger2Markup - 1.3.3
- asciidoctor
- spring-restdocs-mockmvc

## 准备Swagger数据
SpringBoot中使用Swagger的过程就不再赘述了，下面是本文使用的范例：
```java
@Configuration
@EnableSwagger2
class SwaggerConfig {
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
            .apiInfo(apiInfo())
            .select()
            .apis(RequestHandlerSelectors.basePackage("com.jptangchina.gradle.controller"))
            .paths(PathSelectors.any())
            .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
            .title("Swagger2Markup Test Api")
            .version("1.0")
            .build();
    }
}
```

```java
@RestController
@RequestMapping("/user")
@Api(tags = "用户接口")
public class UserController {

    @ApiOperation("用户登录")
    @ResponseBody
    @PostMapping("/login")
    public Result<Void> login(
        @ApiParam(value = "用户名", example = "jptangchina", required = true) @RequestParam String username,
        @ApiParam(value = "密码", example = "jptangchina", required = true) @RequestParam String password) {
        return Result.ok();
    }
}
```
## 使用org.asciidoctor.convert生成PDF（个人不推荐使用）
> 官方教程地址：https://github.com/Swagger2Markup/spring-swagger2markup-demo

仅为了简单的导出PDF而言，本文针对官方案例均有所改动，去掉了部分没有用到的配置。

### 1. 获取Swagger json文件
Swagger页面本质上也就是对json文件进行解析。这里需要先编写单元测试访问`/v2/api-docs`接口并将json文件保存到本地。
```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
class SwaggerTest {
    @Autowired
    private MockMvc mockMvc;
    @Test
    public void generateAsciiDocsToFile() throws Exception {
        String outputDir = System.getProperty("io.springfox.staticdocs.outputDir");
        MvcResult mvcResult = this.mockMvc.perform(get("/v2/api-docs")
            .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andReturn();

        MockHttpServletResponse response = mvcResult.getResponse();
        String swaggerJson = response.getContentAsString();
        Files.createDirectories(Paths.get(outputDir));
        try (BufferedWriter writer = Files.newBufferedWriter(Paths.get(outputDir, "swagger.json"), StandardCharsets.UTF_8)){
            writer.write(swaggerJson);
        }
    }

}
```

> System.getProperty("io.springfox.staticdocs.outputDir");来自于build.gradle中的配置

### 2. 将json文件转换为adoc文件
转换json文件需要使用到`io.github.swagger2markup`插件的`convertSwagger2markup`方法。

引入相关依赖：
```java
buildscript {
    ...
    dependencies {
        ...
        classpath 'io.github.swagger2markup:swagger2markup-gradle-plugin:1.3.3'
    }
}
  
apply plugin: 'io.github.swagger2markup'
```

配置convertSwagger2markup：
```java
ext {
    asciiDocOutputDir = file("${buildDir}/asciidoc")
    swaggerOutputDir = file("${buildDir}/swagger")
}

test {
    systemProperty 'io.springfox.staticdocs.outputDir', swaggerOutputDir
}

convertSwagger2markup {
    dependsOn test
    swaggerInput "${swaggerOutputDir}/swagger.json"
    outputDir asciiDocOutputDir
    config = [
            'swagger2markup.pathsGroupedBy' : 'TAGS',
    ]
}
```

> 更多config配置可以参考：http://swagger2markup.github.io/swagger2markup/1.3.3/#_swagger2markup_properties

### 3. 将adoc文件转换为PDF文件
转换PDF文件需要用到`org.asciidoctor.convert`插件的`asciidoctor`方法。
引入相关依赖：
```java
buildscript {
    ...
    dependencies {
        ...
        classpath 'org.asciidoctor:asciidoctor-gradle-plugin:1.5.3'
    }
}
apply plugin: 'org.asciidoctor.convert'
```

**手动编写**index.adoc文件，放置到${asciiDocOutputDir.absolutePath}中：
```
include::{generated}/overview.adoc[]
include::{generated}/paths.adoc[]
include::{generated}/definitions.adoc[]
include::{generated}/security.adoc[]
```

> {generated}默认值为${build}/asciidoc，参见：https://github.com/Swagger2Markup/swagger2markup-gradle-project-template

配置asciidoctor：
```java
asciidoctor {
    dependsOn convertSwagger2markup
    // sourceDir中需要包含有之前手动编写的index.adoc文件
    sourceDir(asciiDocOutputDir.absolutePath)
    sources {
        include "index.adoc"
    }
    backends = ['pdf']
    attributes = [
            doctype: 'book',
            toc: 'left',
            toclevels: '3',
            numbered: '',
            sectlinks: '',
            sectanchors: '',
            hardbreaks: '',
            generated: asciiDocOutputDir
    ]
}
```

### 4. 编写一个自定义task用来执行上述流程：
```java
task genPdf(type: Test, dependsOn: test) {
    include '**/*SwaggerTest.class'
    exclude '**/*'
    dependsOn(asciidoctor)
}
```

执行genPdf，就可以生成Swagger对应的PDF文件。

### 5. 小结
使用此方法步骤还是比较繁琐的，总体来讲就是json -> adoc -> pdf，并且使用此种方法目前有几个比较大的问题我仍然没有找到解决方案：
- 从官方文档中可以看到支持的语言默认有EN, DE, FR, RU。没错，不支持CN，从导出的文档也可以看到，部分中文无法显示，目前我也尚未找到是否有配置可以实现这个功能。网上的文章部分是通过替换源代码包里面的字体文件来实现，但是由于后面有更好的解决方案，这里就不再讨论。
- 从asciidoctorj-pdf的`1.5.0-alpha.16`版本以后（目前最新是1.5.0-alpha.18），这种方式生成文件会抛出异常，我个人并没有深究这个异常，有兴趣的读者可以通过修改版本号试一试。
- 生成的adoc文件一般包含overview.adoc、paths.adoc、definitions.adoc、security.adoc一共4个文件，这也是为什么要手动编写index.adoc进行整合的原因，感觉不太方便。
综上，我个人并不推荐采用此方式生成PDF。

build.gradle完整文件参考：
```java
buildscript {
    ext {
        springbootVersion = '2.1.6.RELEASE'
    }
    repositories {
        maven {
            url 'http://maven.aliyun.com/nexus/content/groups/public'
        }
    }
    dependencies {
        classpath "org.springframework.boot:spring-boot-gradle-plugin:${springbootVersion}"
        classpath 'org.asciidoctor:asciidoctor-gradle-plugin:1.5.3'
        classpath 'io.github.swagger2markup:swagger2markup-gradle-plugin:1.3.3'
    }
}

repositories {
    maven {
        url 'http://maven.aliyun.com/nexus/content/groups/public'
    }
}

apply plugin: 'java'
apply plugin: 'maven'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'
apply plugin: 'io.github.swagger2markup'
apply plugin: 'org.asciidoctor.convert'

group 'com.jptangchina'
version '1.0-SNAPSHOT'

sourceCompatibility = 1.8
targetCompatibility = 1.8

ext {
    asciiDocOutputDir = file("${buildDir}/asciidoc")
    swaggerOutputDir = file("${buildDir}/swagger")
    swaggerVersion = '2.9.2'
}

dependencies {
    compile 'org.springframework.boot:spring-boot-starter-web'
    compile "io.springfox:springfox-swagger2:${swaggerVersion}"
    compile "io.springfox:springfox-swagger-ui:${swaggerVersion}"
    compile 'io.github.swagger2markup:swagger2markup:1.3.3'
    asciidoctor 'org.asciidoctor:asciidoctorj-pdf:1.5.0-alpha.16'
    testCompile 'org.springframework.boot:spring-boot-starter-test'
    testCompile 'org.springframework.restdocs:spring-restdocs-mockmvc'
}

test {
    systemProperty 'io.springfox.staticdocs.outputDir', swaggerOutputDir
}

convertSwagger2markup {
    dependsOn test
    swaggerInput "${swaggerOutputDir}/swagger.json"
    outputDir asciiDocOutputDir
    config = [
            'swagger2markup.pathsGroupedBy' : 'TAGS',
    ]
}

asciidoctor {
    dependsOn convertSwagger2markup
    sourceDir(asciiDocOutputDir.absolutePath)
    sources {
        include "index.adoc"
    }
    backends = ['pdf']
    attributes = [
            doctype: 'book',
            toc: 'left',
            toclevels: '3',
            numbered: '',
            sectlinks: '',
            sectanchors: '',
            hardbreaks: '',
            generated: asciiDocOutputDir
    ]
}

task genPdf(type: Test, dependsOn: test) {
    include '**/*SwaggerTest.class'
    exclude '**/*'
    dependsOn(asciidoctor)
}

```
## 使用asciidoctor-gradle-plugin生成PDF（推荐）
asciidoctor-gradle-plugin也是官方推荐的使用方式。相对前面的方式，使用起来更加简单，也可以修改配置输出中文。

### 1. 引入插件
```java
plugins {
    id 'org.asciidoctor.jvm.pdf' version '2.2.0'
}
```

### 2. 编写测试类生成adoc
与第一中方法不同的是，不需要再将json文件保存到本地了。
```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class SwaggerTest {
    @Autowired
    private MockMvc mockMvc;
    @Test
    public void generateAsciiDocsToFile() throws Exception {
        String outputDir = System.getProperty("io.springfox.staticdocs.outputDir");
        MvcResult mvcResult = this.mockMvc.perform(get("/v2/api-docs")
            .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andReturn();

        Swagger2MarkupConfig config = new Swagger2MarkupConfigBuilder()
            .withMarkupLanguage(MarkupLanguage.ASCIIDOC)
            .withOutputLanguage(Language.ZH)
            .withPathsGroupedBy(GroupBy.TAGS)
            .withGeneratedExamples()
            .withoutInlineSchema()
            .build();

        MockHttpServletResponse response = mvcResult.getResponse();
        String swaggerJson = response.getContentAsString();
        Swagger2MarkupConverter.from(swaggerJson)
            .withConfig(config)
            .build()
            .toFile(Paths.get(outputDir + "/swagger"));
    }
}
```

> 有兴趣的读者可以阅读下toFile方法的源码，里面对第一种方法生成的4个文件进行了整合，这也是不再需要手动编写index.adoc文件的原因。

### 3. 配置asciidoctorPdf
```java

ext {
    asciiDocOutputDir = file("${buildDir}/asciidoc")
    // 创建字体与主题的文件夹
    pdfFontsDir = file("${buildDir}/fonts")
    pdfThemesDir = file("${buildDir}/themes")
    swaggerVersion = '2.9.2'
}

pdfThemes {
    local 'basic', {
        styleDir = pdfThemesDir
        // styleName会被程序用于匹配${styleName}-theme.yml，如default-styleName-theme.yml
        styleName = 'default'
    }
}
asciidoctorPdf{
    sourceDir(asciiDocOutputDir.absolutePath)
    sources {
        include "swagger.adoc"
    }
    fontsDir(pdfFontsDir.absolutePath)
    theme("basic")
}
```

> 本文字体与主题文件均来自于`asciidoctorj-pdf-1.5.0-alpha.18.jar`源码包，其路径位于：gems/asciidoctorj-pdf-1.5.0-alpha.18/data

### 4. 复制并修改default-theme.yml文件配置
为了解决中文无法显示的问题，首先需要自行下载一个支持中文的字体文件。

修改主题文件，将mplus1p-regular-fallback.ttf替换为自己下载的字体文件的名称。
```yml
M+ 1p Fallback:
  normal: your-font.ttf
  bold: your-font.ttf
  italic: your-font.ttf
  bold_italic: your-font.ttf
```

> 由于手动指定了字体文件的路径，所以除了中文以外，还**需要将源码中的其他字体文件一并复制到${pdfFontsDir}文件夹**。如果不愿意使用官方的字体，也可以考虑将default-theme.yml中其他的字体文件都修改为自己想要的文件。

保持task genPdf不变，再次运行即可生成PDF文件，生成的文件默认路径为${build}/docs/asciidocPdf

### 小结
asciidoctor-gradle-plugin的方式可以支持配置字体与主题，通过配置不仅规避了中文无法显示的问题，同时使用起来也更加简单。需要注意的是，采用此种方案生成出的文档会在封面写有项目的版本号，此版本号为build.gradle中的version，而非SwaggerConfig类中的version。

官方提供了很多配置，可以自行参考官方文档查看。

build.gradle完整文件参考：
```java
buildscript {
    ext {
        springbootVersion = '2.1.6.RELEASE'
    }
    repositories {
        maven {
            url 'http://maven.aliyun.com/nexus/content/groups/public'
        }
    }
    dependencies {
        classpath "org.springframework.boot:spring-boot-gradle-plugin:${springbootVersion}"
    }
}

plugins {
    id 'org.asciidoctor.jvm.pdf' version '2.2.0'
}

repositories {
    maven {
        url 'http://maven.aliyun.com/nexus/content/groups/public'
    }
}

apply plugin: 'java'
apply plugin: 'maven'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

group 'com.jptangchina'
version '1.0-SNAPSHOT'

sourceCompatibility = 1.8
targetCompatibility = 1.8

ext {
    asciiDocOutputDir = file("${buildDir}/asciidoc")
    pdfFontsDir = file("${buildDir}/fonts")
    pdfThemesDir = file("${buildDir}/themes")
    swaggerVersion = '2.9.2'
}

dependencies {
    compile 'org.springframework.boot:spring-boot-starter-web'
    compile "io.springfox:springfox-swagger2:${swaggerVersion}"
    compile "io.springfox:springfox-swagger-ui:${swaggerVersion}"
    compile 'io.github.swagger2markup:swagger2markup:1.3.3'
    testCompile 'org.springframework.boot:spring-boot-starter-test'
    testCompile 'org.springframework.restdocs:spring-restdocs-mockmvc'
}

test {
    systemProperty 'io.springfox.staticdocs.outputDir', asciiDocOutputDir
}

pdfThemes {
    local 'basic', {
        styleDir = pdfThemesDir
        styleName = 'default'
    }
}
asciidoctorPdf{
    sourceDir(asciiDocOutputDir.absolutePath)
    sources {
        include "swagger.adoc"
    }
    fontsDir(pdfFontsDir.absolutePath)
    theme("basic")
}

task genPdf(type: Test, dependsOn: test) {
    include '**/*SwaggerTest.class'
    exclude '**/*'
    dependsOn(asciidoctorPdf)
}
```
## 参考
[https://github.com/Swagger2Markup/swagger2markup](https://github.com/Swagger2Markup/swagger2markup)
[https://github.com/Swagger2Markup/spring-swagger2markup-demo](https://github.com/Swagger2Markup/spring-swagger2markup-demo)
[http://swagger2markup.github.io/swagger2markup/1.3.3](http://swagger2markup.github.io/swagger2markup/1.3.3)