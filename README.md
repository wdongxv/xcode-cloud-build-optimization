# Xcode Cloud Build “Could not resolve host” 错误解决
我相信你在用Xcode Cloud 时也经常遇到过“Could not resolve host”这个错误，非常讨厌👎，直接导致无法正确安装编译时所需要的必要文件。

测试发现，可能是因为距离导致网路延迟原因，特别有些cdn在中国的依赖库，在执行“pod install”时，许多域名就是无法成功解析，错误率非常高⚠️。

我一开始试用Xcode Cloud的时候苦于无法解决这个问题，当我正要放弃的时候，竟然编译成功了😺，所以，我相信一定是Xcode Cloud 底层的dns服务有着某些更严格的策略导致的域名解析失败，而不是完全无法解析某些域名。

所以，解决这个问题一下子有了思路💡，<b>在脚本里把无法解析的域名每隔5秒钟去查询解析一次，触发系统去查询dns，直到成功解析，然后再执行“pod install”。🫣</b>

所以有了第一版解决思路，把出现“Could not resolve host”的域名全部都ping一次，直到所有域名都成功，然后再执行“pod install”

但是这样执行的效率太低，所以进行优化🤯：<b>先执行“pod install”，如果失败，过滤出执行结果里的“Could not resolve host”错误，然后提取出域名后，再针对这个出错的域名进行循环查询。</b>

#### 于是 ci_pre_xcodebuild.sh 有了以下代码🧑‍💻：

```
#!/bin/sh

query_dns() {
    echo "#### ==== query dns： $@"
    local MAX_RETRIES=20
    local RETRY_COUNT=0
    local argsLenght=$#
    while true; do
        local OK_COUNT=0;
        for DOMAIN in "$@"; do
            # 执行 dig 命令查询 DNS
            ping -c 1 "$DOMAIN" > /dev/null 2>&1
            if [ $? -eq 0 ]; then
                # echo "---- DNS 查询成功！$DOMAIN "
                # ping -c 1 "$DOMAIN"
                ((OK_COUNT++))
            else
                # 查询失败时
                # echo "---- DNS 查询失败 $DOMAIN "
                dig $DOMAIN > /dev/null
                break;
            fi
        done
        if [ $OK_COUNT -eq $argsLenght ]; then
            echo "#### ==== 共 $argsLenght 个域名，成功查询 $OK_COUNT 个域名，成功！"
            return 1;
        fi
        
        ((RETRY_COUNT++))
        if [ $RETRY_COUNT -ge $MAX_RETRIES ]; then
            echo "#### ==== 达到最大重试次数 $RETRY_COUNT ，失败。"
            return 1  # 返回 1，表示失败
        fi

        echo "==== 成功查询 $OK_COUNT/$argsLenght 个域名，正在重试 $DOMAIN ...（第 $RETRY_COUNT / $MAX_RETRIES 次重试）"
        sleep 5
    done
}

retry_command() {
  local command="$1"
  local max_retries="$2"
  local retries=0

  until output=$($command 2>&1); do
    echo "=== $command ====: $output ====\n"
    errorHost=$(echo "$output" | grep "Could not resolve host" | head -n 1 | sed "s/.*Could not resolve host: \(.*\)/\1/")
    ((retries++))
    if [ $retries -ge $max_retries ]; then
      echo "==== 命令执行失败，已达到最大重试次数 ($max_retries)，退出。"
      return 1
    fi
    echo "==== 命令执行失败，正在重试...（第 $retries 次重试）:$errorHost"
    if [ -n "$errorHost" ]; then
      echo "---- errorHost: $errorHost"
      query_dns "$errorHost"
    else
      sleep 5
    fi
  done
  echo "==== $command 命令执行成功！"
}
export HOMEBREW_NO_AUTO_UPDATE=1 # disable homebrew's automatic updates.
export HOMEBREW_NO_INSTALL_CLEANUP=1
brew install cocoapods
retry_command "pod install" 10


```

#### 查看Xcode cloud 日志，可以看到在安装GDTMobSDK时无法解析qzs.gdtimg.com，然后触发重复尝试解析错误域名机制，在第二次重试的时候解析成功。😎

```

[!] Error installing GDTMobSDK

[!] /usr/bin/curl -f -L -o /Volumes/workspace/tmp/d20250225-5017-q36u7p/file.zip https://qzs.gdtimg.com/union/res/ios/sdk/GDT_iOS_SDK_4.15.22_ec9f2ef80e511308f3887ff2a4561a8b.zip --create-dirs --netrc-optional --retry 2 -A 'CocoaPods/1.16.2 cocoapods-downloader/2.1'

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current

                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (6) Could not resolve host: qzs.gdtimg.com

Warning: Problem : timeout. Will retry in 1 seconds. 2 retries left.

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (6) Could not resolve host: qzs.gdtimg.com

Warning: Problem : timeout. Will retry in 2 seconds. 1 retries left.

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (6) Could not resolve host: qzs.gdtimg.com ====

==== 命令执行失败，正在重试...（第 2 次重试）:qzs.gdtimg.com

---- errorHost: qzs.gdtimg.com

#### ==== query dns： qzs.gdtimg.com

#### ==== 共 1 个域名，成功查询 1 个域名，成功！

// ···

```

#### 然后脚本会继续执行一次pod install，发现GDTMobSDK已经正确安装，然后又出现第二个错误，自动解决 🤷：

```

// ···

Downloading dependencies

Installing Ads-Fusion-CN-Beta (6.6.1.5)

Installing Ads-Global (6.5.0.8)

Installing BUTTSDKFramework (1.45.1.8-premium)

Installing BaiduMobAdSDK (5.373)

Installing Flutter (1.0.0)

Installing GDTMobSDK (4.15.22)

Installing Google-Mobile-Ads-SDK (12.0.0)

Installing GoogleUserMessagingPlatform (2.7.0)

Installing KSAdSDK (3.3.74)

[!] Error installing KSAdSDK

[!] /usr/bin/curl -f -L -o /Volumes/workspace/tmp/d20250225-5075-hb0kk6/file.tgz https://ali2.a.yximgs.com/udata/pkg/KSAdSDKTarGz/KSAdSDK-framework-ad-3.3.74-98.tar.gz --create-dirs --netrc-optional --retry 2 -A 'CocoaPods/1.16.2 cocoapods-downloader/2.1'

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current

                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (6) Could not resolve host: ali2.a.yximgs.com

Warning: Problem : timeout. Will retry in 1 seconds. 2 retries left.

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (6) Could not resolve host: ali2.a.yximgs.com

Warning: Problem : timeout. Will retry in 2 seconds. 1 retries left.

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (6) Could not resolve host: ali2.a.yximgs.com ====

==== 命令执行失败，正在重试...（第 3 次重试）:ali2.a.yximgs.com

---- errorHost: ali2.a.yximgs.com

#### ==== query dns： ali2.a.yximgs.com

==== 成功查询 0/1 个域名，正在重试 ali2.a.yximgs.com ...（第 1 / 20 次重试）

==== 成功查询 0/1 个域名，正在重试 ali2.a.yximgs.com ...（第 2 / 20 次重试）

==== 成功查询 0/1 个域名，正在重试 ali2.a.yximgs.com ...（第 3 / 20 次重试）


```


# enjoy ！ 🤣
