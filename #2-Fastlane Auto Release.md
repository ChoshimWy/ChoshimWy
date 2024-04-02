### - Fastlane 自动化打包
#### 1.Fastlane 环境参数设置
##### 在fastlane目录下创建一个`.env`文件, 使用方式在Fastfile中通过`"#{ENV['PGYER_API_KEY']}"`进行调用
```sh
APP_IDENTIFIER = "[你的应用BundleID]"
APP_ID = "[苹果开发者账号]"
TEAM_ID = "[开发者账号团队ID]"
APP_STORE_API_KEY_ID = "[APP Store api Key]"
APP_STORE_API_KEY_ISSUER_ID = "[APP Store api key issuer id]"
PGYER_API_KEY = "[蒲公英api key]"
```

#### 2.Fastfile
脚本中`[XXX]`,需要自己替换为对应内容.
```sh
default_platform(:ios)

platform :ios do |options|
  
  desc "测试环境自动化打包并上传蒲公英"
  # 测试
  lane :test do |options|
    buidl_lane(
      scheme:'[scheme]'
    )
    # 上传IPA到蒲公英 'update_description:描述内容'
    puts "上传到蒲公英: #{options[:update_description]}"
    update_description = "\n服务器: 测试环境\n更新描述:#{options[:update_description]}"
    pgyer(api_key: "#{ENV['PGYER_API_KEY']}", update_description: "#{update_description}")
  end
  
  
  desc "UAT环境自动化打包并上传TestFlight和蒲公英"
  # UAT
  lane :uat do |options|  
    buidl_lane(
      scheme:'[scheme]',
      export_method: 'ad-hoc'
    )

    update_description = "\n服务器: UAT环境\n更新描述:#{options[:update_description]}"
    puts "上传到蒲公英: #{options[:update_description]}"
    pgyer(api_key: "#{ENV['PGYER_API_KEY']}", update_description: "#{update_description}")
    
    export_app_store_ipa_lane(scheme:"[scheme]")
    puts "上传应用商店"
    upload_appstore_lane
  end

  desc "UAT环境自动化打包并上传TestFlight"
  # UAT
  lane :uat_only_testflight do |options|  
    buidl_lane(
      scheme:'[scheme]',
      export_method: 'app-store'
    )
    puts "上传应用商店"
    upload_appstore_lane
  end

  desc "UAT环境自动化打包并上传蒲公英"
  # UAT
  lane :uat_only_pgyer do |options|  
    buidl_lane(
      scheme:'[scheme]',
      export_method: 'ad-hoc'
    )

    update_description = "\n服务器: UAT环境\n更新描述:#{options[:update_description]}"
    puts "上传到蒲公英: #{options[:update_description]}"
    pgyer(api_key: "#{ENV['PGYER_API_KEY']}", update_description: "#{update_description}")
    
  end

  desc "生产环境自动化打包并上传TestFlight和蒲公英"
  # 线上
  lane :prod do |options|
    buidl_lane(
      scheme:'[scheme]',
      export_method: 'ad-hoc'
    )

    update_description = "\n服务器: 生产环境\n更新描述:#{options[:update_description]}"
    puts "上传到蒲公英: #{options[:update_description]}"
    pgyer(api_key: "#{ENV['PGYER_API_KEY']}", update_description: "#{update_description}")
    
    export_app_store_ipa_lane(scheme:"[scheme]")
    puts "上传应用商店"
    upload_appstore_lane
  end

  desc "生产环境自动化打包并上传蒲公英"
  lane :prod_only_pgyer do |options|
    buidl_lane(
      scheme:'[scheme]',
      export_method: 'ad-hoc'
    )

    update_description = "\n服务器: 生产环境\n更新描述:#{options[:update_description]}"
    puts "上传到蒲公英: #{options[:update_description]}"
    pgyer(api_key: "#{ENV['PGYER_API_KEY']}", update_description: "#{update_description}")
  end

  desc "生产环境自动化打包并上传TestFlight"
  lane :prod_only_testflight do |options|
    buidl_lane(
      scheme:'[scheme]',
      export_method: 'app-store'
    )
    puts "上传应用商店"
    upload_appstore_lane
  end

  # 打包封装
  lane :buidl_lane do |options|
    version_number = get_version_number(
      target: "#{options[:scheme]}"
    )
    build_number = get_build_number

    export_method = options[:export_method]
    if !export_method 
      if options[:scheme] == "[scheme]"
        export_method = 'ad-hoc'
      elsif options[:scheme] == "[scheme]"
        export_method = 'app-store'
      elsif options[:scheme] == "[scheme]"
        export_method = 'app-store'
      end
    end
    puts "版本: #{version_number}, build: #{build_number} export_method: #{export_method}"

    build_app(
      # 每次打包之前clean一下
      clean: true,
      # 打包出ipa文件路径
      output_directory: "./fastlane/build/#{version_number}/",
      # 打包的名称
      output_name: "#{options[:scheme]}.#{export_method}.#{version_number}(#{build_number}).ipa",
      # 项目的scheme
      scheme: options[:scheme],
      # 是否包含bitcode 根据自己项目的实际情况到buildsetting查看
      include_bitcode:true,
      # 打包导出方式，包含app-store, validation, ad-hoc, package, enterprise, development, developer-id and mac-application
      # 我们这里是上传蒲公英 所以就选择ad-hoc
      export_method: "#{export_method}",
      # 这个设置是为了设置xcode自动配置证书和配置文件，当然也可以手动配置
      export_xcargs: '-allowProvisioningUpdates'
    )
  end

  # 使用上一次的xcarchive再次导出IPA
  lane :export_app_store_ipa_lane do |options|
    build_app(
      scheme: "#{options[:scheme]}",
      skip_build_archive: true,
      export_method: "app-store",
      archive_path: lane_context[SharedValues::XCODEBUILD_ARCHIVE],
      output_directory: "./fastlane/build/#{lane_context[SharedValues::VERSION_NUMBER]}/",
      output_name: "#{options[:scheme]}.app-store.#{lane_context[SharedValues::VERSION_NUMBER]}(#{lane_context[SharedValues::BUILD_NUMBER]}).ipa",
      include_bitcode:true,
      export_xcargs: '-allowProvisioningUpdates'
    )
  end

  # 上传商店
  lane :upload_appstore_lane do |options|
    ipa = options[:ipa_path]
    puts "IPA路径: #{ipa}"
    
    api_key = app_store_connect_api_key(
      key_id: "#{ENV['APP_STORE_API_KEY_ID']}",
      issuer_id: "#{ENV['APP_STORE_API_KEY_ISSUER_ID']}",
      key_filepath: "./fastlane/AuthKey_#{ENV['APP_STORE_API_KEY_ID']}.p8",
      duration: 500,
      in_house: true
    )

    upload_to_testflight(
      api_key: api_key,
      # 跳过等待处理结果
      skip_waiting_for_build_processing:true,
      skip_submission: true
    )
  end


  # 所有lane开始之前
  # 解锁钥匙串
  before_all do |lane, options|
    password = options[:password]
    if !password 
      password = '[打包机器密码]'
    end
    # 解锁钥匙串
    sh "security -v unlock-keychain -p #{password}"
    

    # 版本号
    version_number = options[:version_number]
    build_number = options[:build_number]

    # 设置新的版本号时,重置BuildNumber
    if version_number
        if !build_number
          build_number = '1'
        end
    end

     # 不传入版号时版号自增
      # 手动传入的版本时进行项目内版本号更新
      increment_version_number(
        version_number: version_number
      )
    # end

    # Build号
    # if build_number
      # 手动传入时进行更新
      increment_build_number(
        build_number:build_number
      )
    # end

  end
  
  # 所有lane开始之后
  after_all do |lane|
    puts "结束"
  end
end
```

#### 3.使用方式
测试环境自动化打包并上传蒲公英
```sh
fastlane test
 ```
# 
UAT环境自动化打包并上传TestFlight和蒲公英
```sh
fastlane uat
 ```
# 
UAT环境自动化打包并上传TestFlight
```sh
fastlane uat_only_testflight
 ```
# 
UAT环境自动化打包并上传蒲公英
```sh
fastlane uat_only_pgyer
 ```
# 
生产环境自动化打包并上传TestFlight和蒲公英
```sh
fastlane prod
 ```
# 
生产环境自动化打包并上传蒲公英
```sh
fastlane prod_only_pgyer
 ```
# 
生产环境自动化打包并上传TestFlight
```sh
fastlane prod_only_testflight
 ```
# 

#### 4.可传入参数
* 打包机器密码:`password`
* 更新描述: `update_description`
* 版本号: `version_number`
* Build号: `build_number`

使用方式在命令后加入参数, 如`fastlane test password:1234 update_description: '本次更新为修复bug'`.

在Fastfile中通过`#{options[:update_description]}"`获取参数内容