# script
```shell
# ================================================
#   ADB 广告自动点击工具 - PowerShell 版
#   外星人加速器 自动看点击广告领取时长
#   支持 XML 精确解析 content-desc 和 text
# ================================================

$ADB = "adb"
$PACKAGE = "com.etalien.booster"
$REMOTE_XML = "/sdcard/window_dump.xml"
$LOCAL_XML = "window_dump.xml"
$KEYWORDS_END = "今日广告已看完，请明日再来"
$KEYWORDS_CLICK = "看广告 领时长"   # 可修改为你想点击的文字
$FILE_PATH = "window_Temp/image"
$FILE_NAME = Get-Date -Format "yyyy-MM-dd-hh-mm"
    
# 在当前脚本所在文件夹下创建文件夹（已存在则自动忽略，不报错）
New-Item -Path $FILE_PATH -ItemType Directory -Force | Out-Null
$Count = 0

function Invoke-Adb {
    param([string]$Command)
    
    # 使用 cmd.exe 执行完整命令，支持 > 重定向、二进制输出等
    cmd.exe /c "$ADB $Command"
}

function Get-Xml {
    Remove-Item $LOCAL_XML -ErrorAction SilentlyContinue
    Invoke-Adb "shell uiautomator dump $REMOTE_XML"
    Invoke-Adb "pull $REMOTE_XML $LOCAL_XML"
    
}

function Find-By-Xml {
    param([string]$SearchText)

    if (-not (Test-Path $LOCAL_XML)) { return $null }

    [xml]$xml = Get-Content $LOCAL_XML -Encoding UTF8 -ErrorAction SilentlyContinue
    if (-not $xml) { return $null }

    foreach ($node in $xml.SelectNodes("//node")) {
        $text = $node.GetAttribute("text")
        $desc = $node.GetAttribute("content-desc")
        $bounds = $node.GetAttribute("bounds")

        $full = "$text $desc".Trim()

        if ($full -like "*$SearchText*") {
            if ($bounds) {
                # 解析 bounds [x1,y1][x2,y2] → 计算中心点
                $nums = [regex]::Matches($bounds, "\d+").Value
                if ($nums.Count -eq 4) {
                    $x = ([int]$nums[0] + [int]$nums[2]) / 2
                    $y = ([int]$nums[1] + [int]$nums[3]) / 2
                    return [PSCustomObject]@{ X = [int]$x; Y = [int]$y; Text = $full }
                }
            }
        }
    }
    return $null
}

function Click {
    param([int]$X, [int]$Y)
    Write-Host "点击坐标: ($X, $Y)" -ForegroundColor Green
    Invoke-Adb "shell input tap $X $Y"
}

function Clear-Task {
    Invoke-Adb "shell input keyevent KEYCODE_HOME"
    Invoke-Adb "shell input keyevent 187"
    Start-Sleep -Seconds 1
    # Invoke-Adb "shell input tap 599 2464"
    # Dump 当前界面 XML
    Get-Xml

    $endResult = Find-By-Xml "清除" 
    if ($endResult) {
        Write-Host "清除" -ForegroundColor Green
        Click $endResult.X $endResult.Y
        Remove-Item $LOCAL_XML -ErrorAction SilentlyContinue
    }
    $endResult = Find-By-Xml "清理" 
    if ($endResult) {
        Write-Host "清理" -ForegroundColor Green
        Click $endResult.X $endResult.Y
        Remove-Item $LOCAL_XML -ErrorAction SilentlyContinue
    }
    $endResult = Find-By-Xml "全部任务"
    if ($endResult) {
        Write-Host "全部任务" -ForegroundColor Green
        Click $endResult.X $endResult.Y
        Remove-Item $LOCAL_XML -ErrorAction SilentlyContinue
    }
    Remove-Item $LOCAL_XML -ErrorAction SilentlyContinue
    Invoke-Adb "shell rm $REMOTE_XML"
    Invoke-Adb "shell input keyevent KEYCODE_HOME"
}

# ====================== 主循环 ======================
Write-Host "=== ADB 广告自动点击工具 (PowerShell) 已启动 ===" -ForegroundColor Cyan
while ($true) {
    $Count++
    $Now = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    Write-Host "`n第 ${Count} 轮启动  $Now" -ForegroundColor Yellow

    # 启动 App
    Invoke-Adb "shell monkey -p $PACKAGE -c android.intent.category.LAUNCHER 1"
    Start-Sleep -Seconds (Get-Random -Minimum 2 -Maximum 6)
    Invoke-Adb "exec-out screencap -p > $FILE_PATH/$FILE_NAME-start$Count.png"
    $currentFocus = adb shell dumpsys window | findstr mCurrentFocus
    # 显示变量内容
    Write-Host "当前焦点窗口: $currentFocus"
    # Dump 当前界面 XML
    Get-Xml

    # 检查是否今天已看完
    $endResult = Find-By-Xml $KEYWORDS_END
    if ($endResult) {
        Write-Host "`n【结束】今日广告已看完，请明日再来！" -ForegroundColor Red
        Write-Host "程序已停止。" -ForegroundColor Red
        Invoke-Adb "shell am force-stop $PACKAGE"
        Start-Sleep -Seconds 1
        Clear-Task
        # 加载 Windows Forms
        Add-Type -AssemblyName System.Windows.Forms
        # 弹出默认提示框
        [System.Windows.Forms.MessageBox]::Show("今日广告已看完，请明日再来！", "程序已停止。", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Information)
        # pause
        exit
    }
    # Dump 当前界面 XML
    # 查找并点击 “看广告 领时长”
    $clickResult = Find-By-Xml $KEYWORDS_CLICK

    if ($clickResult) {
        Write-Host "XML 命中: $($clickResult.Text)" -ForegroundColor Green
        Click $clickResult.X $clickResult.Y

        # 模拟观看广告（滑动 + 等待约 35-40 秒）
        Write-Host "正在观看广告（约 35 秒）..." -ForegroundColor Cyan
        $watchTime = 0
        while ($watchTime -lt 35) {
            $sleep = Get-Random -Minimum 5 -Maximum 10
            Start-Sleep -Seconds $sleep
            $watchTime += $sleep
            $currentFocus = adb shell dumpsys window | findstr mCurrentFocus
            # 这里加了 * 号，只要返回的字符串里包含这个 Activity 就算匹配成功
            if ($currentFocus -notmatch "com.etalien.booster") {
                Write-Host "检测到已离开目标 App，准备退出..."
                $sleep = Get-Random -Minimum 5 -Maximum 10
                Start-Sleep -Seconds $sleep
                # 3. 注意：break 只能在循环（while/for/foreach）中使用
                # 如果这段代码不在循环里，请使用 return 或 exit
                break 
            }
            # 这里加了 * 号，只要返回的字符串里包含这个 Activity 就算匹配成功 
            if ($currentFocus -match "com.etalien.booster/com.etalien.booster.mobile.MainActivity") { 
                Write-Host "检测到已离开目标 App，准备退出..." 
                # 3. 注意：break 只能在循环（while/for/foreach）中使用 
                # 如果这段代码不在循环里，请使用 return 或 exit 
                break 
            }
            # 显示变量内容
            Write-Host "当前焦点窗口:$watchTime  $currentFocus"
            Invoke-Adb "shell input swipe 300 1400 300 600 400"   # 向上滑动
        }
    }
    else {
        Write-Host "未找到 '看广告' 按钮，使用备用固定坐标..." -ForegroundColor Yellow
        # Click 540 1800   # ← 请根据你的手机分辨率修改这个坐标！
    }

    # 强制停止 App
    Invoke-Adb "shell am force-stop $PACKAGE"
    Start-Sleep -Seconds 1
    Clear-Task
    $currentFocus = adb shell dumpsys window | findstr mCurrentFocus
    # 显示变量内容
    Write-Host "当前焦点窗口: $currentFocus"
    # 随机间隔
    $wait = Get-Random -Minimum 1 -Maximum 3
    Write-Host "等待 ${wait} 秒后继续下一轮..." -ForegroundColor Gray
    Start-Sleep -Seconds $wait
}
# 结束
Invoke-Adb "exec-out screencap -p > $FILE_PATH/$FILE_NAME-end.png"
```