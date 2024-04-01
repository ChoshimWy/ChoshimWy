#### - 忽略第三方库版本低于项目版本
```Bash
# 在Podfile文件中添加以下内容
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings.delete 'IPHONEOS_DEPLOYMENT_TARGET'
    end
  end
end
```