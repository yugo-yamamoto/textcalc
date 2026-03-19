# CLAUDE.md — TextCalc 開発ノート

## WSL から Windows Python を実行してテスト・スクリーンショットを撮る

### 前提

- WSL 上のファイルを Windows Python で直接実行できる
- `python.exe` は WSL の PATH から呼び出し可能（Microsoft Store 版 Python で動作確認済み）
- ファイルパスは `wslpath -w` で Windows 形式に変換する

```bash
WIN_SCRIPT=$(wslpath -w /root/work/textcalc/textcalc.pyw)
python.exe "$WIN_SCRIPT"
```

---

### テストの基本パターン

クリップボードに入力値をセットしてアプリを起動し、Enter 送信後にクリップボードの結果を検証する。

```powershell
Add-Type -AssemblyName System.Windows.Forms

[System.Windows.Forms.Clipboard]::SetText('1 x 2 x 10000')

$proc = Start-Process -FilePath 'python.exe' -ArgumentList "$WIN_SCRIPT" -PassThru
Start-Sleep -Milliseconds 3000

[System.Windows.Forms.SendKeys]::SendWait('{ENTER}')
Start-Sleep -Milliseconds 1000

$result = [System.Windows.Forms.Clipboard]::GetText()
if ($result -like '*= 20,000*') { '[OK]' } else { '[NG]' }

$proc.Kill()
```

---

### スクリーンショット撮影

#### DPI に関する注意点

スクリーンショットを撮る側のプロセスも **DPI 宣言が必要**。
宣言しないと `CopyFromScreen` の座標が論理ピクセルになり、物理座標とずれて **黒画像** になる。

```powershell
Add-Type @'
using System;
using System.Runtime.InteropServices;
public class WinAPI {
    [DllImport("shcore.dll")] public static extern int SetProcessDpiAwareness(int v);
    [DllImport("user32.dll")] public static extern bool SetForegroundWindow(IntPtr hWnd);
    [DllImport("user32.dll")] public static extern bool GetWindowRect(IntPtr hWnd, out RECT rect);
    [DllImport("user32.dll")] public static extern bool SetWindowPos(IntPtr hWnd, IntPtr ins, int x, int y, int cx, int cy, uint f);
    public struct RECT { public int Left, Top, Right, Bottom; }
}
'@
[WinAPI]::SetProcessDpiAwareness(2) | Out-Null
```

#### ウィンドウを最前面に固定して撮影

`HWND_TOPMOST(-1)` + `SWP_SHOWWINDOW(0x40)` + `SWP_NOSIZE(0x01)` = フラグ `0x41`

```powershell
[WinAPI]::SetWindowPos($hwnd, [IntPtr](-1), 400, 300, 0, 0, 0x41) | Out-Null
[WinAPI]::SetForegroundWindow($hwnd) | Out-Null
Start-Sleep -Milliseconds 800

$rect = New-Object WinAPI+RECT
[WinAPI]::GetWindowRect($hwnd, [ref]$rect) | Out-Null
$w = $rect.Right - $rect.Left
$h = $rect.Bottom - $rect.Top

$bmp = New-Object System.Drawing.Bitmap($w, $h)
$g = [System.Drawing.Graphics]::FromImage($bmp)
$g.CopyFromScreen($rect.Left, $rect.Top, 0, 0, $bmp.Size)
$g.Dispose()
$bmp.Save("C:\path\to\output.png")
$bmp.Dispose()
```

#### SendKeys の注意点

`(` `)` `+` などは SendKeys の特殊文字のためそのまま送れない。
**日本語や特殊文字を含む入力はクリップボード経由で渡す**（アプリ起動前にセットすると起動時に自動入力される）。

---

## Python / tkinter の DPI 対応

### DPI 宣言（import 直後に実行）

```python
import ctypes

try:
    ctypes.windll.shcore.SetProcessDpiAwareness(2)  # Per-Monitor DPI Aware
except Exception:
    try:
        ctypes.windll.shcore.SetProcessDpiAwareness(1)  # System DPI Aware
    except Exception:
        ctypes.windll.user32.SetProcessDPIAware()
```

### ウィンドウサイズを DPI に合わせてスケール

```python
dpi_scale = root.winfo_fpixels("1i") / 96.0  # 96 は Windows の基準 DPI
w = round(520 * dpi_scale)
h = round(200 * dpi_scale)
root.geometry(f"{w}x{h}")
```

フォントサイズはポイント指定（例: `("Meiryo", 20)`）のままでよい。
ポイントは DPI 非依存であり、GDI が自動的に物理ピクセルに変換する。
