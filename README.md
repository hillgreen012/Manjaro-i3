# Manjaro -i3的配置记录和相关文件
### 配置文件及其位置
1. i3's configure file--> ~/.i3/config, 
2. i3status's configure file --> /etc/i3status.conf,
3. URxvt‘s configure file --> ~/.Xresources
3. morc_menu的配置文件 --> ~/.config/morc_menu/morc_menu_v1.conf，可设置宽度等
4. 輸入法等配置文件--> ~/.profile
5. 引導菜單grub的配置文件 --> /etc/default/grub
6. 桌面组件conky，默认是i3配置文件中启动/usr/bin/start_conky_maia脚本，该脚本加载了/usr/share/conky目录下的两个配置文件，分别是右上角的conky_maia和conky1.10_shrotcuts_maia
7. $mod+9将lock screen，这是通过执行/usr/bin/blurlock脚本实现的，查看该脚本可见：先屏幕快照->模糊化->删除快照->执行i3lock，可以调整模糊的程度
----
### 安装过程
安装前，进入本本的win10，lenovo将自动对bios进行升级。使用dd/rufus制作启动盘并进行以下bios设置：
0. [arch linux 针对X1C6th的详细说明]（https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_X1_Carbon_(Gen_6)）
1. 在X1C6th（i7cpu8650/16G/512G）安装manjaro-i3 17.1.11, 因该本本预安装win10，需在bios中设置`Security -> Secure Boot - Set to "Disabled"`才能从U盘启动
2. X1C 的默认 BIOS 配置下 Thunderbolt BIOS Assist Mode 是 Disable 的，这会导致 Linux 在 s2idle 下的能耗特别高（温度平均46，平均功率约8.5w）。故需要进 BIOS 将其设置为 Enable，如此则温度降至40以下，功率也骤降至4-6w左右，办公续航轻松上11h
3. 另外将memory card reader、fingerprint、camera, bluetooth, wireless wan等禁用以节约功耗
4. enable S3 support, make sure you have at least BIOS version 1.30 installed. Then, go into the BIOS configuration, and `Config -> Power -> Sleep State - Set to "Linux"`。To check whether S3 is recognized and usable by Linux, run:`dmesg | grep -i "acpi: (supports"`
-----
安装后，因X1C分辨率为2560×1440，字体太小，故进行整体界面放大：
1. 打开~/.Xresources，修改`Xft.dpi`为140，值越大，字体越大，重新登录生效。bar的字体在`～/.i3/config`中可调节，右上角的字体在`/usr/share/conky/conky_maia`，右下角的字体在`usr/share/conky/conky1.10_shortcuts_maia`文件中调整
1. 设置更新。联网后，使用右下角的更新图标，使用preferences设置镜像位置为china，并刷新列表（自动找到最快的源），启用AUR（Arch User respoisity），然后进行更新
1. URvxt终端字体。（antialias为平滑，需要一个中文字体）
```sudo yaourt -S ttf-monaco wqy-microhei (或者install it by GUI)```
编辑~/.Xresource文件，对应位置修改如下
```
URxvt.font:xft:Monaco:pixelsize=16:antialias=true,xft:WenQuanYi Micro Hei Mono:pixelsize=16:antialias=true
URxvt.boldFont:xft:Monaco:pixelsize=16:Bold:antialias=true,xft:WenQuanYi Micro Hei Mono:pixelsize=16:Bold:antialias=true
```
执行xrdb -load ~/.Xresources生效(maybe need reboot)

4. 安装中文输入法
```sudo pacman -S fcitx
sudo pacman -S fcitx-im     ----全部安装，保证图形界面可用
sudo pacman -S fcitx-configtool   ----配置工具
sudo pacman -S googlepinyin    ----但不能进行模糊音设置
```
然后打开`～/.xprofile`文件，添加一下内容：
```export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
fcitx -d -r
```
注销用户重新登录，点击右下角图标进行配置（设置shift切换和< >翻页等）

5. GUI install proxychains-ng, then edit /etc/proxychains.conf file, replace "sock4..." at the end
5. `sudo pacman -S remmina freerdp`，远程登录工具remmina，支持各种协议（需另外安装freerdp），logout/reboot生效
5. google-chrome/firefox/nitrogen壁纸/shadowsocks/stardict/wps-office/ttf-wps-fonts/visual-studio-code-bin
6. morc_menu中动Arandr可进行双显示器设置，拖动即可
7. This did not work out of the box, but was easily fixed by doing the following (Assuming you are using GRUB):
Add `acpi.ec_no_wakeup=1` To your kernel parameters (/etc/default/grub -> GRUB_CMDLINE_LINUX_DEFAULT)

Then edit `/etc/systemd/logind.conf` and make sure you have an uncommented line like this: `HandleLidSwitch=suspend`

Now when you close the lid, the machine will sleep. When you open it, you'll just have to press the button to get it to come back. This might be annoying for some users, but that's how I'm used to waking up my machine.

10. 设置网络流量显示。新建如下内容文件`～/.i3/net-speed.sh`, 添加可执行权限`chmod +x ~/.i3/net-speed.sh`:
```
#!/bin/sh

# Authors:
# - Moritz Warning <moritzwarning@web.de> (2016)
# - Zhong Jianxin <azuwis@gmail.com> (2014)
#
# See file LICENSE at the project root directory for license information.
#
# i3status.conf should contain:
# general {
#   output_format = i3bar
# }
#
# i3 config looks like this:
# bar {
#   status_command exec ～/.i3/net-speed.sh
# }
#
# Single interface:
# ifaces="eth0"
#
# Multiple interfaces:
# ifaces="eth0 wlan0"
#

# Auto detect interfaces
ifaces=$(ls /sys/class/net | grep -E '^(eth|wlan|enp|wlp)')

last_time=0
last_rx=0
last_tx=0
rate=""

readable() {
  local bytes=$1
  local kib=$(( bytes >> 10 ))
  if [ $kib -lt 0 ]; then
    echo "? K"
  elif [ $kib -gt 1024 ]; then
    local mib_int=$(( kib >> 10 ))
    local mib_dec=$(( kib % 1024 * 976 / 10000 ))
    if [ "$mib_dec" -lt 10 ]; then
      mib_dec="0${mib_dec}"
    fi
    echo "${mib_int}.${mib_dec} M"
  else
    echo "${kib} K"
  fi
}

update_rate() {
  local time=$(date +%s)
  local rx=0 tx=0 tmp_rx tmp_tx

  for iface in $ifaces; do
    read tmp_rx < "/sys/class/net/${iface}/statistics/rx_bytes"
    read tmp_tx < "/sys/class/net/${iface}/statistics/tx_bytes"
    rx=$(( rx + tmp_rx ))
    tx=$(( tx + tmp_tx ))
  done

  local interval=$(( $time - $last_time ))
  if [ $interval -gt 0 ]; then
    rate="$(readable $(( (rx - last_rx) / interval )))↓ $(readable $(( (tx - last_tx) / interval )))↑"
  else
    rate=""
  fi

  last_time=$time
  last_rx=$rx
  last_tx=$tx
}

i3status | (read line && echo "$line" && read line && echo "$line" && read line && echo "$line" && update_rate && while :
do
  read line
  update_rate
  echo ",[{\"full_text\":\"${rate}\" },${line#,\[}" || exit 1
done)
```
注意：按该文件前面介绍修改`/etc/i3status.conf`和`～/.i3/config`两个文件

11. 安装oracle vbox后如果出现无virtualbox kernel driver，可使用先查看内核版本`uname -r`，然后`sudo pacman -S linux414-virtualbox-host-modules`，如果出现404错误，则refresh database
    但是, Vbox窗口非常难以调节, 我改用了VM. 通过GUI方式安装失败, 手动安装过程如下: 
    ```
     sudo mkdir /etc/init.d #VM需要, 确认有该目录
     sudo pacman -S linux414-headers  #安装header文件, 对应当前的Linux版本, 可用uname查看
     #官网下载VM Pro Workstration后
     chmod +x VMware-Player-VERSION-ID.ARCH.bundle  #添加可执行
     sudo ./VMware-Player-VERSION-ID.ARCH.bundle  #安装
     yaourt -S vmware-systemd-services  #安装服务
     sudo systemctl start vmware  #开始系统服务
     sudo systemctl enable vmware #设置开机启动
     vmware & #终端启动程序
    ```

12. 安装nginx，完成后使用`systemctl start/stop/restart nginx`， `systemctl enable nginx`设置开机启动，配置文件在`/etc/nginx/nginx.conf`。如果出现forbiden，则修改配置文件中`user qige`，qige为网站目录的所有者或nginx的运行者

13. 使用arandr设置双显示器后，重启就失效了。在arandr中将设置保存为`normal.sh`，添加执行权限，然后在`~/.i3/config`中写入`exec ~/.screenlayou/normal.sh`

14. 安装 rofi 代替 dmenu    安装 xfce4-terminal 代替urxvt   设置 conky 主题 关闭 i3status   (暂未执行)
    
15. chrome 支持命令行配置代理，在无法访问chrome 商店下载 SwitchyOmega 前，可以使用 google-chrome-stable --proxy-server="socks5://127.0.0.1:1080" www.google.com 命令启动进行登录谷歌账号(你需要先设置好ss)

16. TIM，先GUI安装deepin-wine-tim(按说明是不需要安装的，但估计AppImage缺少一些wine的库文件)，然后下载其AppImage文件（http://yun.tzmm.com.cn/index.php/s/5hJNzt2VR9aIEF2） AppImage 是一种把应用打包成单一文件的格式，只需要赋予可执行权限即可使用。(开始还能使用，后来只能sudo .TIM-x86_64.AppImage在终端使用了)

17. Tor-Browser，去官网下载压缩文件，解压后直接执行即可（AUR仓库由于GFW限制不能成功）

18. WPS Office安装后，对应的启动命令是（wps/et/wpp)，但由于版权原因，缺少中文字体，下载simsun/simfang/simhei/simkai等中文字体，`sudo mkdir /usr/share/fonts/WindowsFonts`，然后拷贝以上字体文件到该目录，然后`chmod 755 /usr/share/fonts/WindowsFonts/*`，最后重建字体缓存`fc-cache -f`。因为是英文系统故对其它程序没有影响，否则由于在系统中加入了宋体，而部分应用程序又将宋体作为默认字体，显示难看需要进一步调整。

19. 使用系统菜单对鼠标左右调整，但重启后失效，所以使用命令行进行。将以下命令添加到`～/.i3/config`文件的自启动部分：
`xinput list    #查看当前鼠标的id，如id为11,则
 xinput set-button-map 11 3 2 1   #调换，即可生效。改为1 2 3则换回
`

20. 系统默认浏览器设置为chrome，可从`mod+z-->settings-->preferred applications`中选择，如果不成功，也可编辑`.profile`文件，加入`export BROWSER=/usr/bin/google-chrome-stable`，同时编辑`~/.congfig/mimeapps.list`文件，修改为`google-chrome.desktop`

21. Manjaro默认已安装OpenSSH Server, 只需运行`systemctl enable/start/restart sshd.service`即可(开机运行/即可启动/重启)

22. 安装wireshark后运行不会有interface, 因为wireshark做了权限隔离, 也不推荐使用root运行, 需要将当前用户添加到wireshark组.`gpasswd -a yourusername wireshark`(似乎需要重启才生效?)

23. 在一次较久未升级后进行系统升级,中间在编译某软件(似乎是mongodb)太久了, 我终止了升级过程, 随后无论如何都不能更新, 出现`invalid crypto engine`等错误, 然后查询发现可以如下进行修正:
```
sudo rm -fr /etc/pacman.d/gnupg # 如果以下命令失败则先执行此命令
sudo pacman-key --init
sudo pacman-key --populate archlinux manjaro
sudo pacman-key --refresh-keys
sudo rm /var/cache/pacman/pkg/*
sudo pacman -Syu

```

但以上命令需要libreadline.so.7文件, 执行以下操作即可: `sudo ln -s /usr/lib/libreadline.so.8 /usr/lib/libreadline.so.7`
另, 更新镜像排名, 选择国内比较快的官方镜像源, 勾选相应的镜像站 :
```
sudo pacman-mirrors -i -c China -m rank   #
```
24. GoLang开发配置. GUI安装go即可, 然后打开vscode, 安装go插件；新建一个hello.go文件, 将弹出安装相关工具对话框, 但由于有墙且VScode不支持sock5代理, 故安装不能成功. 进入`~/go`目录, 新建`golang.org/x`目录, 执行`git clone https://github.com/golang/tools.git`和`git clone https://github.com/golang/lint.git`命令, 然后在vscode中调出命令面板, 输入`Go: Install/Update Tools`安装即可

25. Cisco Packet Tracer安装：
```
 A. Download Packet Tracer 7.2.1 from official web site or https://www.sysnettechsolutions.com/en/ciscopackettracer/download-cisco-packet-tracer-7-2/
 B. git clone https://aur.archlinux.org/packettracer.git
 C. 将上面的安装文件复制到pachettracer目录，并进入该目录
 D. 运行命令即可安装：makepkg -sic 
 E. Start packet tracer by executing packettracer or packetraceer.sh file
``` 
26. 截屏：
```
Screenshot of whole screen - <Print>
Screenshot of focused container - mod+<Print>
Screenshot of selection - mod+<Shift>+<Print>
```

```
import i3.jpg //然后就让你选截屏的区域
sleep 5;import 213.png    //等待5秒截屏
import -frame    123.jpg     //截取鼠标点击的那个窗口
import -windows root   123.png  //截取全屏幕
```
后来添加了`deepin-screenshot`截图软件，可简单对图片进行标注等，在`～/i3/config`文件中添加了快捷方式

27. 在安装软件的过程中，如果某种原因未成功，再次运行时会出现等待另一个包管理器退出之类的信息，是因为有锁文件，删除即可（以前我会去进程中kill或重启）`sudo rm /var/lib/pacman/db.lock`

28. 当前使用了socks5代理, 但某些软件只支持http代理如vscode, 所以使用了privoxy进行转换
```
A. 安装privoxy
B. 在/etc/privoxy/config文件最后添加一行(注意最后的.): forward-socks5 127.0.0.1:1080 .
C. 在默认端口8118启动privoxy: systemctl start privoxy
D. 在vscode中配置http代理即可
```
29. 因需对pdf进行批注, 仓库安装FoxitReader报校验错, 因此去Foxit官网下载-->解压-->执行安装即可, 完毕后morc_menu菜单-->Office中即有FoxitReader

30. X1C按了键盘的静音键后, 再次开启则没有声音了. 执行`pavocontrol`进行调整

31. 在看了v2ray的介绍后, 入手了一个v2ray机场(https://mk.nyuyu.net/aff.php?aff=742). 
  A. (请使用B方法)github上找到名为shadowray for v2ray客户端的一个python包, 可实现订阅, 还不错. 步骤如下:
```
sudo pip install shadowray // 安装
sudo vim /usr/lib/python3.7/site-packages/shadowray/subscribe/parse.py  //修改第50行, 将aid和level固定设置为2和0, 否则后面会出错 
shadowray --autoconfig  //会自动下载v2ray核心文件, 并在新建~/.shadowray目录中放置其相关文件. 升级则github下载v2ray-core文件解压到该目录即可
shadowray --subscribe-add 'haha,你的订阅url'  //添加订阅地址
shadowray --subscribe-update --port 1080  //生成服务器配置文件(resource/servers.json), 并指定本地端口(默认为1082)
shadowray --list  //查看当前服务器线路信息
shadowray --start 1 --daemon  //选择1号线路并后台运行
shadowray --stop  //终止后台进程
```
----
 B. github上找到一个可订阅并测速以及自动生成配置文件, 自动启动服务并方便切换的脚本, 果断用上:
 ```
 sudo pacman -S v2ray //安装v2ray, 并自动以服务方式后台启动, 注意如果没有启动成功, 留意v2ray的路径, 一般在/usr/bin下, 否则修改v2ray.service文件
 git clone https://github.com/akirarika/v2sub.git
 sudo cp v2sub/v2sub /usr/bin/    //拷贝到系统路径目录中
 sudo v2sub   //输入一次订阅地址即可, 然后选择线路, 
 ```
 
 因机场传输协议升级为WebSocket, 故修改`v2sub`源代码, 与`outbound->settings`平级输入:
 ```
 "streamSettings": {
   "network": "ws",
   "wsSettings": {
     "connectionReuse": True
   }
 }
 ```
 
 > 20191122, 脚本地址更新为`git clone https://github.com/xbblog95/v2sub.git`，是上一个脚本的更新，自动支持ws， 且支持测速即`sudo v2sub speed`。但请注意因其有其它需要的文件，故需要在其目录中运行或创建链接`sudo ln -s ~/v2sub/v2sub /usr/bin/v2sub`
 
 
> 每每解决了问题时, 才发现它在不起眼处!!!
> X1C安装了v2ray或shadowray都不能使用, 花了两天时间, 各种考虑, 几乎放弃, 才发现是manjaro的时钟没有同步. 而v2ray要求客户机与服务器的时间不能相差90s, 这就是悲剧之源!
> mod+ctrl+b 调出菜单, 安装timeset, 设置时间同步即可!
------------
以下为独立安装v2ray需要的手动配置, 使用shadowray后可不理会

`sudo pacman -S v2ray`

如果出现缺少文件, `sudo cp ~/.shadowray/v2ray/geo*.* /usr/bin/`, 然后配置`/etc/v2rayconfig.json文件的outbound部分， 添加如下vmess小节`:
```
{
  "protocol":"vmess",
  "settings": {
        "vnext": [
          {
            "address": "8.8.8.8", // 服务器的 IP
            "port": 443, // 服务器的端口
            "users": [
              {
                // id 就是 UUID，相当于用户密码
                "id": "7d4c4078-e129-416b-a483-cf5713a96a66",
                "alterId": 2,
                "security": "chacha20-poly1305"
              }
            ]
          }
        ]
      }
}
```
32. 蓝牙鼠标及无线网卡折腾记

**缘由:** PC机的无线鼠标按键不灵活, JD入手了一个100+的名为MS comfort的鼠标, 到货后发现是蓝牙的, 而PC无蓝牙适配器, 折腾开始

**折腾**

* JD上买个蓝牙适配器, 按照`arch bluetooth wiki`的介绍开始在PC上安装`bluez bluez-utils blueman`等, 总之命令行也罢, GUI也罢, 就是连接不成功, 期间重新启动还卡在`A stoped job is running, wait...`位置, 导致还不能正常重启!

  使用的命令如下:
  ```
  bluetoothctl  # 进入蓝牙配置
  power on      # 打开适配器
  scan on       # 进入扫描模式, 可扫描到新的蓝牙设备, 拷贝其mac
  trust mac
  pair mac
  connect mac
  ```
> 放弃, MS的鼠标不给Linux使用? :), 该蓝牙鼠标送与XXX, 在其WIN10上瞬间工作, :(

* 事情本来就此了. 某天清理机器, 觉得有关蓝牙的程序不需要了, 故搜索`bluetooth`, 将与之相关的删除. 期间`Manjaro`给了以下警告, 无非是可能有些程序需要(wireshark之类), 直接无视.

* 麻烦开始. PC上的USB无线网卡马上显示`Down`状态. **不能上网了** (删除期间真没涉及wireless方面的东东啊!)

* 重启1..., 2..., 3..., 仍然不能工作, 第一反应:无线网卡坏了(已用了9年)

* 拔出网卡, 插到X1C, 正常工作, 那就开始进行手动配置吧. **此时此处没有X1C是不可想象的! 我需要参考资料!**

* 以下的命令多费周章:
  ```
  ip addr 或 iw dev     # 查看网络设备接口名称, 以下称为wlp0s
  ip link show wlp0s    # 查看其状态是否up
  sudo ip link set wlp0s up  # up该接口
  iw wlp0s link         # 是否有连接
  sudo iw wlp0s scan    # 扫描AP, 找出要连接的SSID, 以下称为hahaha(WPA2加密)
  su                    # 切换为root
  wpa_passphrase hahaha >> /etc/wpa_supplicant/wpa.conf # 生成wpa连接配置文件, 该命令需输入wifi口令
  su XXXX   # 切换回一般用户
  sudo wpa_supplicant -i wlp0s -c /etc/wpa_supplicant/wpa.conf  # 连接指定AP
  iw wlp0s link         # 应看到连接成功
  sudo dhclient wlp0s   # 获取网络配置
  ```
  > sudo ip addr add 192.168.1.2/24 dev wls0s         # 指定IP
  > ip route add default via 192.168.1.1 dev wls0s    # 添加网关
  > /etc/resolv.conf为DNS配置

* 现在可以联网, 马上安装`networkmanager, nm-connection-editor, network-manager-applet`, 重启

**恢复如初**

33. 下载`kali`虚拟机时， chrome下载总是半途中断。安装了另外一个多线程下载工具`Axel`， 10分钟搞定。

```bash
yay -S axel
axel YOUR-URL -n 10 # 以10个线程下载文件在当前目录
```
34. 在Nova的提醒下，将vscode与github用SSH通过密钥关联起来。
```bash
# 生成SSH密钥
sshgen    # 默认在～/.ssh下生成密钥对（id_ras,id_ras.pub)，可以无需密码保护
#SSH连接测试：为了保险起见，我们应当测试一下是否能够通过SSH访问Github
ssh -T git@github.com # 看到如下信息即可：You've successfully authenticated, but GitHub does not provide shell access.
# 添加SSH Key到Github：登入github，修改帐号的setting中的ssk key，添加id_ras.pub公钥的内容即可
# 进入某仓库，获取ssh地址如：git@github.com:wang1/course.git
# 完善git配置信息
git config --global user.name "wang1"           #引号内替换成自己Github注册用户名
git config --global user.email "wang1@gmail.com"
# 关联本地和远程仓库
git remote add origin git@github.com:wang1/course.git
# 如果本地仓库已关联，可修改
git remote set-url origin git@github.com:wang1/course.git
# 查看远程关联情况
git remote -v
```
至此，在vscode中同步仓库就无需每次输入用户和密码了

35. 系统已内置了`webp`图片转换工具，生成以下脚本自动某目录下的`.jpg`和`.png`进行转换，文件名不变

```bash
# input/output in the same directory
for file in ./*
do
  if [ -f "$file" ] # only for file
  then
    cwebp "$file" -o "${file%.*}.webp" # default quality is 75
  fi
done
```

----
### Refrences:
> 1. http://www.php-master.com/post/258672.html
> 1. http://www.cnblogs.com/vachester/p/5649813.html
> 1. https://dev.to/lobo_tuerto/you-need-to-know-about-i3-363c  (fonts configure)
> 1. https://ohmyarch.github.io/2017/01/15/Linux下终极字体配置方案 
> 1. https://fontawesome.com/cheatsheet (开源图标字体，直接复制粘贴使用)
> 1. https://www.jianshu.com/p/cf14660d8af2 (#Manjaro-i3的配置)
> 1. https://www.jianshu.com/p/6e9eb98c0494 (Manjaro linux 安装笔记)
> 1. http://jimolonely.github.io/2017/12/27/linux/001-linux-manjaro/ (配置manjaro)

----

######################################################################
#                            oh-my-i3
# Date    : 12/12/2015
# Author  : Allen_Qiu
# Version : v1.1
#Mo
# 依赖:conky feh (可选推荐:xcompmgr freetype-infinality)
# 部分特性可能需要新版本支持，请更新至最新版本或自行修改相应部分
# 更多内容请参考:
# i3 User’s Guide: http://i3wm.org/docs/userguide.html
# oh-my-i3 github: https://github.com/ID1258/oh-my-i3.git
#
######################################################################

# => 开机自启（根据需要取消相应的注释#）
# 壁纸须先安装feh，并在此指定路径
exec --no-startup-id feh --bg-scale "$HOME/oh-my-i3/wallpaper.jpg"
# 设置透明须先安装xcompmgr，并在此指定软件和透明度（默认0.75），sleep保证transset在其所设置的软件之后启动，根据情况调节大小
#exec --no-startup-id xcompmgr &
#exec --no-startup-id sleep .2 && exec transset -n i3bar 0.75
# 建议另外指定一个脚本来启动通用的开机自启项目
exec --no-startup-id $HOME/bin/RC
# 总有一些奇葩无法正常用脚本启动
exec --no-startup-id kuaipan4uk &

# => 设定mod键与工作区名
set $mod  Mod4
set $WS1  1:?8?5
set $WS2  2:?8?3
set $WS3  3:?8?8
set $WS4  4:?8?5
set $WS5  5:?8?5
set $WS6  6:?8?7
set $WS7  7:?8?7
set $WS8  8:?8?1
set $WS9  9:?8?5
set $WS0 10:?8?4

# => 工作区切换&智能启动（添加智能启动脚本~/bin/st 自动且不重复打开工作区相应程序）
# 自动切换到新打开的窗口(需4.12版本支持)
#focus_on_window_activation focus
# 重复切换到当前工作区时会返回上一个所在工作区，有可能造成工作区错位
#workspace_auto_back_and_forth yes
# 注释部分请依据个人喜好更改并取消注释，且把i3-sensible-terminal替换为常用term
bindsym $mod+1 workspace $WS1, exec --no-startup-id ~/bin/st xfce4-terminal
bindsym $mod+2 workspace $WS2, exec --no-startup-id ~/bin/st google-chrome
bindsym $mod+3 workspace $WS3, exec --no-startup-id WizNote > /dev/null 2>&1
bindsym $mod+4 workspace $WS4, exec --no-startup-id ~/bin/st smplayer
bindsym $mod+5 workspace $WS5, exec --no-startup-id ~/bin/st thunar
bindsym $mod+6 workspace $WS6, exec --no-startup-id ~/bin/st virtualbox
bindsym $mod+7 workspace $WS7
bindsym $mod+8 workspace $WS8
bindsym $mod+9 workspace $WS9
bindsym $mod+0 workspace $WS0

# => 快捷启动
bindsym $mod+Return exec --no-startup-id xfce4-terminal
bindsym $mod+r      exec --no-startup-id dmenu_run -b -fn 'Monaco-12' -nb '#333333' -nf '#FFFFFF' -sb '#111111' -sf '#3399FF'
bindsym $mod+e      exec --no-startup-id thunar

# => 自定义窗口（支持定义边框类型，窗口布局，大小调整，自动归类工作区等等，多个定义用,隔开）
for_window [class="(?i)thunar"] move container to workspace $WS5, workspace $WS5, layout tabbed

# => 窗口边框类型（边框类型有normal正常/none无边框/pixel 1 自定义宽度）
# 默认普通窗口的边框类型
new_window none
# 默认浮动窗口的边框类型
new_float normal
# 取消工作区边缘的边框
hide_edge_borders both
# 在三种边框类型中切换
bindsym $mod+b border toggle

# => 新建窗口的分割方向
bindsym $mod+h split h
bindsym $mod+v split v

# => 移动窗口
bindsym $mod+Left  move left
bindsym $mod+Down  move down
bindsym $mod+Up    move up
bindsym $mod+Right move right

# => 调整窗口大小
bindsym $mod+Shift+Left  resize shrink width  10 px or 1 ppt
bindsym $mod+Shift+Down  resize grow   height 10 px or 1 ppt
bindsym $mod+Shift+Up    resize shrink height 10 px or 1 ppt
bindsym $mod+Shift+Right resize grow   width  10 px or 1 ppt

# => 关闭窗口
bindsym mod1+F4 kill

# => 焦点切换
# 焦点跟随鼠标移动
focus_follows_mouse yes
# 焦点切换到父窗口
bindsym $mod+q focus parent
# 焦点切换回子窗口
bindsym $mod+Shift+q focus child
# 焦点切换到浮动窗口
bindsym $mod+Shift+space focus mode_toggle
#
bindsym $mod+w focus up
bindsym $mod+s focus down
bindsym $mod+a focus left
bindsym $mod+d focus right

# => 布局切换
# 切换到堆叠布局
# bindsym $mod+z layout stacking
# 切换到标签布局
# bindsym $mod+x layout tabbed
# 切换到平铺布局（竖直/水平）
# bindsym $mod+c layout toggle split
# 在所有布局中轮回切换
bindsym $mod+x layout toggle all
# 窗口切换到全屏
bindsym $mod+f fullscreen toggle
# 窗口切换到浮动
bindsym $mod+space floating toggle
# 窗口切换为粘滞
bindsym $mod+g sticky toggle

# => 移动窗口到另一个工作区
bindsym $mod+mod1+Left  move container to workspace prev, workspace prev
bindsym $mod+mod1+Right move container to workspace next, workspace next
bindsym $mod+mod1+1 move container to workspace $WS1, workspace $WS1
bindsym $mod+mod1+2 move container to workspace $WS2, workspace $WS2
bindsym $mod+mod1+3 move container to workspace $WS3, workspace $WS3
bindsym $mod+mod1+4 move container to workspace $WS4, workspace $WS4
bindsym $mod+mod1+5 move container to workspace $WS5, workspace $WS5
bindsym $mod+mod1+6 move container to workspace $WS6, workspace $WS6
bindsym $mod+mod1+7 move container to workspace $WS7, workspace $WS7
bindsym $mod+mod1+8 move container to workspace $WS8, workspace $WS8
bindsym $mod+mod1+9 move container to workspace $WS9, workspace $WS9
bindsym $mod+mod1+0 move container to workspace $WS0, workspace $WS0

# => 暂存窗口（额外的可隐藏浮动窗口，取消浮动还原成普通窗口）
# 转换普通窗口为暂存窗口
bindsym $mod+Shift+minus move scratchpad
# 呼出/隐藏暂存窗口
bindsym $mod+minus scratchpad show

# => 重新载入（更改配置文件后只须重载即可生效，不包含自启部分）
bindsym mod1+Shift+r restart

# => 电源管理（Pause Break键呼出)
set $mode_system 系统:锁屏(L) 注销(O) 关机(S) 重启(R) 取消(Esc)
bindsym Pause mode "$mode_system"
mode "$mode_system" {
    bindsym l exec i3lock -c '#333333', exec sleep .1 && exec xset dpms force off, mode "default"
    bindsym o exec i3-msg exit
    bindsym s exec systemctl poweroff
    bindsym r exec systemctl reboot
    bindsym Escape mode "default"
}

############################# 主题相关 ###############################

# 字体
font pango:Monaco 10

# 窗口颜色                边框    背景    文字    提示
client.focused          #333333 #333333 #FFFFFF #333333
client.focused_inactive #999999 #999999 #FFFFFF #3399FF
client.unfocused        #999999 #999999 #FFFFFF #3399FF
client.urgent           #990000 #990000 #FFFFFF #990000
client.placeholder      #000000 #000000 #FFFFFF #000000
client.background       #FFFFFF

# i3bar
bindsym $mod+m bar mode toggle
bar {
    # i3bar调用
    status_command conky -c $HOME/.config/i3/conkyrc_bar
    # 显示位置
    #position top
    # 是否隐藏
    #mode hide
    # 显示/隐藏切换键
    #modifier $mod
    # 拆解工作区名（隐藏前面的1:2:3:……）
    strip_workspace_numbers yes
    # 定义分隔符（适用于i3status）
    #separator_symbol " "
    colors {
        background #333333
        statusline #FFFFFF
        #eparator  #3399FF
        # 工作区颜色         边框    背景    文字
        focused_workspace  #111111 #111111 #FFFFFF
        active_workspace   #FFFFFF #FFFFFF #FFFFFF
        inactive_workspace #333333 #333333 #FFFFFF
        urgent_workspace   #990000 #990000 #FFFFFF
        #binding_mode       #990000 #990000 #FFFFFF
    }
}
