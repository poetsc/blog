---
title: 终端代理配置
date: 2020-04-06 01:02:43
tags:
---
## MacOS ##
-  修改用户全局配置文件
```
vi ~/.zshrc
```
<!--more-->
-  然后在其里面添加如下内容
```
alias proxy='export all_proxy=socks5://127.0.0.1:1080'
alias unproxy='unset all_proxy'
```
-  最后执行如下命令使配置生效
```
source ~/.zshrc
```
* 开启代理模式
```
proxy
```
* 关闭代理模式
```
unproxy
```

------
## Windows ##
### wsl ###
>export https_proxy=http://127.0.0.1:7890
>export http_proxy=http://127.0.0.1:7890
>apt: https://askubuntu.com/questions/257290/configure-proxy-for-apt
<!--more-->
------
### cmd ###
> set http_proxy=127.0.0.1:7890
> set https_proxy=127.0.0.1:7890
> set http_proxy=
> set https_proxy=
------
### PowerShell ###
```
# NOTE: registry keys for IE 8, may vary for other versions
$regPath = 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings'

function Clear-Proxy
{
    Set-ItemProperty -Path $regPath -Name ProxyEnable -Value 0
    Set-ItemProperty -Path $regPath -Name ProxyServer -Value ''
    Set-ItemProperty -Path $regPath -Name ProxyOverride -Value ''

    [Environment]::SetEnvironmentVariable('http_proxy', $null, 'User')
    [Environment]::SetEnvironmentVariable('https_proxy', $null, 'User')
}

function Set-Proxy
{
    $proxy = 'http://example.com'

    Set-ItemProperty -Path $regPath -Name ProxyEnable -Value 1
    Set-ItemProperty -Path $regPath -Name ProxyServer -Value $proxy
    Set-ItemProperty -Path $regPath -Name ProxyOverride -Value '<local>'

    [Environment]::SetEnvironmentVariable('http_proxy', $proxy, 'User')
    [Environment]::SetEnvironmentVariable('https_proxy', $proxy, 'User')
}
```
> 相关链接
> https://zcdll.github.io/2018/01/27/proxy-on-windows-terminal/
> https://github.com/shadowsocks/shadowsocks-windows/issues/1489
> https://gist.github.com/famousgarkin/c5138b1e13ac41920d22
> https://www.youtube.com/watch?v=ZnTffdLnIqs