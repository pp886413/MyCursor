---
name: youtube-clip-mp4
description: >-
  Download one time range from a YouTube URL and save it as an MP4 or MP3
  using yt-dlp and ffmpeg on this Windows PC. Use when the user gives a
  YouTube link and asks to clip, cut, trim, download a segment, or save an
  interval as mp4 or mp3（剪片、剪區間、下載片段、存成 MP4、轉成 MP3、只要聲音）.
---

# YouTube 區間剪成 MP4

只下載使用者指定的時間區間。使用者要 MP3、只要聲音、或轉成音檔時，輸出 MP3；其餘情況輸出 MP4。不要先抓整支影片再剪。

開始前先向使用者確認這三項。缺任何一項就先問，不要猜：

1. YouTube 連結
2. 開始時間（`1:23` 或 `0:01:23`）
3. 結束時間，或長度（例如「30 秒」）

存檔位置沒講就用桌面，不要再問。這台電腦的桌面是 `C:\Users\User\Desktop`。若使用者有指定其他資料夾，才改用該路徑。下指令前用這行取得實際桌面（含資料夾被重新導向的情況）：

```powershell
$desktop = [Environment]::GetFolderPath('Desktop')
```

年齡限制、會員限定、地區鎖定的影片可能失敗。失敗時把 yt-dlp 的錯誤原文告訴使用者，不要改用其他下載方式。

## 這台電腦的工具

在 **PowerShell** 操作。`python -m pip` 與目前工作目錄無關；當初是在 `C:\Users\User` 執行，cmd 裡下同一行也可以，只要 `python` 在 PATH 上。

| 工具 | 怎麼裝 | 執行檔 |
|---|---|---|
| yt-dlp | `python -m pip install -U yt-dlp` | `C:\Users\User\AppData\Local\Programs\Python\Python310\Scripts\yt-dlp.exe` |
| ffmpeg | `winget install --id Gyan.FFmpeg -e --accept-package-agreements --accept-source-agreements` | WinGet 套件 `Gyan.FFmpeg`，別名 `ffmpeg`、`ffprobe`、`ffplay` |

Python 是 `C:\Users\User\AppData\Local\Programs\Python\Python310\python.exe`。

winget 安裝後會改使用者 PATH。同一個 PowerShell 視窗要先重新讀取 PATH，否則找不到 `ffmpeg`：

```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable('Path','Machine') + ';' + [System.Environment]::GetEnvironmentVariable('Path','User')
```

確認：

```powershell
yt-dlp --version
ffmpeg -version
```

缺哪個就只裝哪個。兩者都已在 PATH 上時不要重裝。

## 剪片

時間寫成 yt-dlp 的區間：`*開始-結束`，例如 `*1:23-2:10`。使用者給的是長度時，自己換算成結束時間。

檔名只使用原始影片標題，再加副檔名（`.mp4` 或 `.mp3`）。不要加影片 ID、時間區間、章節、序號或其他字。已有同名檔時直接覆蓋，避免被改成 `標題 (1).mp3` 這類名字。`-o` 只含 `%(title)s.%(ext)s`。

### MP4

```powershell
yt-dlp --download-sections "*START-END" --force-keyframes-at-cuts --force-overwrites -f "bv*+ba/b" --merge-output-format mp4 -o "$desktop\%(title)s.%(ext)s" "URL"
```

- `--download-sections` 只抓該段。前面的 `*` 表示這是時間區間，不是章節名稱。
- `--force-keyframes-at-cuts` 讓切點對齊使用者給的時間。會重新編碼，比較慢，切點才準。
- `-f "bv*+ba/b"` 取最好的畫質加音訊，沒有分離串流時退回單一檔。
- `--merge-output-format mp4` 交給 ffmpeg 合成 MP4。

### MP3

使用者明確要 MP3 或只要聲音時才走這段。這台電腦的 ffmpeg 已有 `libmp3lame`，不需另外安裝。

```powershell
yt-dlp --download-sections "*START-END" -x --audio-format mp3 --audio-quality 0 --force-overwrites -o "$desktop\%(title)s.%(ext)s" "URL"
```

- `-x` 抽出音訊，`--audio-format mp3` 交給 ffmpeg 編成 MP3。
- `--audio-quality 0` 是 yt-dlp 的最高 MP3 音質（0 最好，9 最差）。
- 不要加 `--force-keyframes-at-cuts`，那是給畫面切點用的。

完成後用 ffprobe 確認檔案存在、時長接近指定區間：

```powershell
ffprobe -v error -show_entries format=duration,format_name -of default=nw=1 "輸出檔路徑"
```

把輸出的完整路徑與時長回報給使用者。
