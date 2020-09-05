## jenkins插件 Active Choices Plugin

### 一、根据不同的项目，选择不同的子项目

![image-20200708104521117](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200708104521117.png)

配置过程：参数化构建过程

-->Active Choices Parameter

![img](https://upload-images.jianshu.io/upload_images/2377241-29cce198eee320cc.png?imageMogr2/auto-orient/strip|imageView2/2/w/523/format/webp)

![image-20200708104703146](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200708104703146.png)

**脚本内容**

```groovy
return[
"license-cloud-service",
"license-cloud-web",
"license-cloud-origin"
]

return["error PROJECT"]
```

--> Active Choices Reactive Reference

![](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200708104825997.png)

脚本内容：

```groovy
service=["license-cloud-company","license-cloud-core","license-cloud-quartz"]
web=["license-cloud-web-gateway","license-cloud-web-admin","license-cloud-web-front"]
origin= ["license-cloud-origin-intf","license-cloud-origin-job","license-cloud-origin-collection"]

if (PROJECT.equals("license-cloud-service")) {
  return service
} else if (PROJECT.equals("license-cloud-web")) {
  return web
} else if (PROJECT.equals("license-cloud-origin")) {
  return origin
} else {
  return ["Unknown MODULE"]
}

return["error MODULE"]
```

流水线语法：

```groovy
流水线语法：
properties([
  parameters([[
    $class: 'ChoiceParameter', 
    choiceType: 'PT_RADIO', 
    description: '请选择要打包的子项目', 
    filterLength: 1, filterable: false, 
    name: 'PROJECT', randomName: 'choice-parameter-1004535594101688', 
    script: [
      $class: 'GroovyScript', 
      fallbackScript: [
        classpath: [], 
        sandbox: false, 
        script: 'return["error PROJECT"]'], 
        script: [
          classpath: [], 
          sandbox: false, 
          script: '''return[
                      "license-cloud-service",
                      "license-cloud-web",
                      "license-cloud-origin"
                    ]'''
        ]
    ]
  ], 
  [
    $class: 'CascadeChoiceParameter', 
    choiceType: 'PT_CHECKBOX', 
    description: '', 
    filterLength: 1, 
    filterable: false, 
    name: 'MODULE', 
    randomName: 'choice-parameter-1004535596422808', 
    referencedParameters: 'PROJECT', 
    script: [
      $class: 'GroovyScript', 
      fallbackScript: [
        classpath: [], 
        sandbox: false, 
        script: 'return["error module"]'], 
        script: [
          classpath: [], 
          sandbox: false, 
          script: '''
            service=["license-cloud-company","license-cloud-core","license-cloud-quartz"]
            web=["license-cloud-web-gateway","license-cloud-web-admin","license-cloud-web-front"]
            origin= ["license-cloud-origin-intf","license-cloud-origin-job","license-cloud-origin-collection"]

            if (PROJECT.equals("license-cloud-service")) {
              return service
            } else if (PROJECT.equals("license-cloud-web")) {
              return web
            } else if (PROJECT.equals("license-cloud-origin")) {
              return origin
            } else {
              return ["Unknown module"]
            }'''
        ]
    ]
  ]
  ])
])

```

