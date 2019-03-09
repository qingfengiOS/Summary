#FastLane命令
s1:
 fastlane init 

s2:
fastlane add_plugin pgyer

s3:
vim ./fastlane/Fastfile

s4:
build_app(export_method: "ad-hoc")
pgyer(api_key: "0cc5b8f7162750b13f2a4982c18cc7d5", user_key: "7e78a93092d38fa3d43227a4bed897b2")

s5:
fastlane beta




xcode-select – 更改默认Xcode
xcode-select -switch xcode_folder_path(xcode_folder_path表示应用程序路劲，需要使用最新的xcode9.2)
xcode-select --print-path 完成后可以打印路劲

解决安装gem没有权限问题sudo gem install fastlane -NV -n /usr/local/bin