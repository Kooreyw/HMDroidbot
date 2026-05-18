# 环境与工程

  ┌────────────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 项         │ 说明                                                                                                                      │
  ├────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Python     │ 3.10～3.11（README 建议 3.11；勿用 3.13）                                                                                 │
  │ 工程       │ 仓库根目录执行 pip install -e .，可用 droidbot -h 或 python3 -m droidbot.start -h 自检                                    │
  │ hdc        │ 本仓库在 macOS/Linux 下固定调用 hdc 命令名（见 droidbot/adapter/hdc.py），需把 DevEco 工具链加入 PATH，而不是只记绝对路径 │
  │ config.yml │ 将 env 设为 macOS（你当前仓库里是 Linux，在 Mac 上应改掉，否则不影响 hdc 二进制名，但与文档/预期一致）                    │
  └────────────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

## hdc
  export PATH="/Applications/DevEco-Studio.app/Contents/sdk/default/openharmony/toolchains:$PATH"
  hdc version
  hdc list targets
  2PR6R23B17001517

  平板需打开开发者调试，USB（或网络）能出现在 hdc list targets 里；多设备时用 -t <序列号>。

## 输出目录 ./output ✅

  每次跑加 -o，建议按应用分子目录，避免 UTG/报告互相覆盖：

  -o ./output/notepad
  -o ./output/wechat
  -o ./output/shanbei
  -o ./output/tianxin

## 启动命令
主要是 -a 的参数如何设置？

  Harmony 路径下 AppHM 支持两种 -a 写法：

  1. 本地 .hap 路径 — 会从 HAP 解析 bundleName；若设备未装，会走 hdc install。
  2. 已安装应用的 bundleName（包名） — 通过 hdc shell bm dump -n <bundleName> 从设备取元数据；应用必须已安装。

  -a 不是 .hap 时，check_package 只会警告，不会要求磁盘上存在文件。

  务必加 -keep_app：默认结束后会尝试卸载被测应用（-keep_app 关闭卸载）。

  先在平板上查 bundleName。 在 Mac 上：hdc -t <你的设备序列号> shell bm dump -a

hdc -t 2PR6R23B17001517 shell bm dump -a | grep -i qi
	com.qianxin.inhouse.secid
	com.qianxin.ohs.inhouse.sslvpn
	com.qianxin.trustagent.harmonynext.inhouse.hap.bm


  在列表里对照应用名找到 完整 bundleName。
  运行模板

  cd /Users/mac/workspace/HuaWei/HMDroidbot
  python3 -m droidbot.start \
    -a '<bundleName或hap路径>' \
    -o ./output/notepad \
    -t '<hdc list targets 里的序列号>' \
    -count 1000 \
    -is_harmonyos \
    -keep_app \
    -debug


运行奇安信天信
cd /Users/mac/workspace/HuaWei/HMDroidbot
source .venv/bin/activate
.venv/bin/python3 -m droidbot.start \
    -a 'com.qianxin.trustagent.harmonynext.inhouse.hap.bm' \
    -o ./output/tianxin \
    -t '2PR6R23B17001517' \
    -count 1000 \
    -is_harmonyos \
    -keep_app \
    -debug



.venv/bin/python3 -m droidbot.start \
    -a 'com.huawei.hmos.hinote' \
    -o ./output/notepad \
    -t '2PR6R23B17001517' \
    -count 1000 \
    -is_harmonyos \
    -keep_app \
    -debug



# python 版本兼容3.13+ 的问题  ✅
实现基于 socket 的替代：telnetlib 在 Python 3.13+ 已移除。

# hdc 常用命令

## 设备管理
```bash
hdc version                                        # 查看 hdc 版本
hdc list targets                                   # 列出所有已连接设备
hdc -t <serial> target mount                       # 挂载设备文件系统（获取读写权限）
hdc -t <serial> target boot                        # 重启设备
hdc -t <serial> shell param get const.product.model   # 查看设备型号
hdc -t <serial> shell param get const.product.software.version  # 查看系统版本
```

## 应用管理
```bash
hdc -t <serial> shell bm dump -a                   # 列出所有已安装包名
hdc -t <serial> shell bm dump -n <bundleName>      # 查看某应用详细信息
hdc -t <serial> app install <path/to/app.hap>      # 安装 HAP
hdc -t <serial> app uninstall <bundleName>         # 卸载应用
hdc -t <serial> shell aa start -b <bundleName> -a <abilityName>  # 启动 Ability
hdc -t <serial> shell aa force-stop <bundleName>   # 强制停止应用
```

## 文件传输
```bash
hdc -t <serial> file send <本地路径> <设备路径>    # 推送文件到设备
hdc -t <serial> file recv <设备路径> <本地路径>    # 从设备拉取文件
```

## UI 自动化（uitest）
```bash
hdc -t <serial> shell uitest dumpLayout            # dump 当前页面布局（返回 JSON 路径）
hdc -t <serial> shell uitest uiInput click <x> <y>              # 点击坐标
hdc -t <serial> shell uitest uiInput longClick <x> <y>          # 长按坐标
hdc -t <serial> shell uitest uiInput swipe <x1> <y1> <x2> <y2> <speed>  # 滑动
hdc -t <serial> shell uitest uiInput inputText <x> <y> <text>   # 在坐标处输入文字
hdc -t <serial> shell uitest uiInput keyEvent Home              # 按 Home 键
hdc -t <serial> shell uitest uiInput keyEvent Back              # 按 Back 键
```

## 截图
```bash
hdc -t <serial> shell snapshot_display             # 截图到设备默认路径（返回文件路径）
hdc -t <serial> shell snapshot_display -f /data/local/tmp/shot.jpeg   # 截图到指定路径
```

## 日志
```bash
hdc -t <serial> hilog                             # 实时查看系统日志（类似 adb logcat）
hdc -t <serial> hilog -t 30                       # 最近 30 秒的日志
hdc -t <serial> hilog | grep <bundleName>         # 过滤特定应用日志
hdc -t <serial> hilog -x                          # 清空日志缓冲区
```

## Shell 调试
```bash
hdc -t <serial> shell                             # 进入交互式 shell
hdc -t <serial> shell ps -ef | grep <bundleName> # 查看应用进程 PID
hdc -t <serial> shell kill -9 <pid>              # 强制杀进程
hdc -t <serial> shell df -h                      # 查看存储空间
hdc -t <serial> shell cat /proc/meminfo          # 查看内存信息
```

## 本项目实际用到的命令
```bash
# 查 bundleName（然后用 grep 过滤关键字）
hdc -t 2PR6R23B17001517 shell bm dump -a | grep -i <关键字>

# 截图
hdc -t 2PR6R23B17001517 shell snapshot_display

# 从设备拉截图
hdc -t 2PR6R23B17001517 file recv /data/local/tmp/snapshot_xxx.jpeg ./output/temp/

# dump 布局
hdc -t 2PR6R23B17001517 shell uitest dumpLayout
```