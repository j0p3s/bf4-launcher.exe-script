English version modified by: TEGUH DWI iD

BF4 LAN BOT ALPHA 0.1 - In-match Control Launcher v1.0.4

Windows PowerShell 5.1 + built-in .NET Framework; no Python or extra installs.

This script is readable and editable. It does not modify any original DLL/EXE.

$ErrorActionPreference = 'Stop'
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing
[System.Windows.Forms.Application]
::EnableVisualStyles()

SendInput is permitted only when an actual BF4 game process owns the foreground window.

Never sends keystrokes to BF4Launcher, PowerShell, browsers, or an arbitrary application.

Add-Type -TypeDefinition @'
using System;
using System.Runtime.InteropServices;
using System.Threading;
public static class BF4ChatInput {
[DllImport("user32.dll")]
public static extern IntPtr GetForegroundWindow();
[DllImport("user32.dll")]
public static extern uint GetWindowThreadProcessId(IntPtr hwnd, out uint pid);
[DllImport("user32.dll")]
public static extern uint MapVirtualKey(uint code, uint mapType);


$$DllImport("user32.dll", SetLastError=true)$$


public static extern uint SendInput(uint count, Input

 input, int structSize);
[StructLayout(LayoutKind.Sequential)]
public struct Input {
public int type;
public InputUnion data;
}
[StructLayout(LayoutKind.Explicit)]
public struct InputUnion {
[FieldOffset(0)]
public MouseInput mouse;
[FieldOffset(0)]
public KeyInput keyboard;
[FieldOffset(0)]
public HardwareInput hardware;
}
[StructLayout(LayoutKind.Sequential)]
public struct MouseInput {
public int dx,dy;
public uint mouseData, dwFlags, time;
public UIntPtr dwExtraInfo;
}
[StructLayout(LayoutKind.Sequential)]
public struct KeyInput {
public ushort wVk, wScan;
public uint dwFlags, time;
public UIntPtr dwExtraInfo;
}
[StructLayout(LayoutKind.Sequential)]
public struct HardwareInput { public uint uMsg; public ushort wParamL, wParamH; }
public static bool IsBF4Foreground() {
IntPtr hwnd=GetForegroundWindow();
if(hwndIntPtr.Zero) return false;
uint pid;
GetWindowThreadProcessId(hwnd,out pid);
if(pid0) return false;
try {
string name=System.Diagnostics.Process.GetProcessById((int)pid).ProcessName.ToLowerInvariant();
return (name=="bf4" || name=="bf4_no_drm" || name=="bf4_x86" || name=="bf4_retail");
} catch { return false; }
}
private static bool Key(ushort vk, bool down) {
uint scan=MapVirtualKey(vk,0);
if(scan0) return false;
Input i=new Input(); i.type=1;
i.data.keyboard.wVk=0;
i.data.keyboard.wScan=(ushort)scan;
i.data.keyboard.dwFlags=0x0008u | (down?0u:0x0002u);
return SendInput(1,new Input

{i},Marshal.SizeOf(typeof(Input)))1;
}
private static bool Stroke(ushort vk) {
if(!IsBF4Foreground()) return false;
if(!Key(vk,true)) return false;
Thread.Sleep(62);
if(!Key(vk,false)) return false;
Thread.Sleep(56);
return IsBF4Foreground();
}
public static string SendCommand(int count) {
if(count<0 || count>63) return "Invalid bot count";
if(!IsBF4Foreground()) return "BF4 game window is not in the foreground; cancelled to prevent mistyping";
// BF4 default: J opens all-chat, then type !setbot N and Enter.
if(!Stroke(0x4A)) return "Failed to open chat with J, or the game lost focus";
Thread.Sleep(380);
string command="!setbot " + count.ToString(System.Globalization.CultureInfo.InvariantCulture);
foreach(char ch in command) {
if(!IsBF4Foreground()) return "BF4 lost focus during input; aborted";
if(ch'!') {
if(!Key(0x10,true)) return "Shift key input failed";
Thread.Sleep(45);
bool ok=Stroke(0x31);
Key(0x10,false);
if(!ok) return "Exclamation mark input failed";
} else {
ushort vk=(ushort)(ch' '?0x20:(int)char.ToUpperInvariant(ch));
if(!Stroke(vk)) return "Bot chat text input failed";
}
}
Thread.Sleep(120);
if(!Stroke(0x0D)) return "Chat Enter key could not be sent";
return "OK";
}
}
'@

Plain '-' and '=' keys: do NOT register them as global system hotkeys.

Only passively poll the physical main-keyboard keys while BF4 is in the foreground.

Otherwise RegisterHotKey would consume normal typing in Windows applications.

Add-Type -ReferencedAssemblies 'System.Windows.Forms' -TypeDefinition @'
using System;
using System.Windows.Forms;
public class BF4HotkeyForm : Form { }
'@
Add-Type -TypeDefinition @'
using System;
using System.Runtime.InteropServices;
public static class BF4KeyPoll {
[DllImport("user32.dll")]
private static extern short GetAsyncKeyState(int key);
public static bool Down(int vk) { return (((int)(ushort)GetAsyncKeyState(vk)) & 0x8000) != 0; }
public static bool Modified() {
return Down(0x10) || Down(0x11) || Down(0x12) || Down(0x5B) || Down(0x5C);
}
public static bool Minus() { return !Modified() && Down(0xBD); } // VK_OEM_MINUS main keyboard
public static bool Equals() { return !Modified() && Down(0xBB); } // VK_OEM_PLUS main keyboard, unshifted '='
}
'@

$script:mapChoices = @(
@{ Name = 'Siege of Shanghai'; Code = 'MP_Siege' },
@{ Name = 'Operation Locker'; Code = 'MP_Prison' },
@{ Name = 'Zavod 311'; Code = 'MP_Abandoned' },
@{ Name = 'Hainan Resort'; Code = 'MP_Resort' },
@{ Name = 'Operation Metro'; Code = 'XP0_Metro' },
@{ Name = 'Zavod: Graveyard Shift (Night)'; Code = 'XP5_Night_01' },
@{ Name = 'Propaganda'; Code = 'XP3_Prpganda' }
)

When started from the EXE, BF4_LAN_APPDIR is the folder that contains the EXE.

$script:appRoot = if ($env:BF4_LAN_APPDIR) { $env:BF4_LAN_APPDIR } elseif ($PSScriptRoot) { $PSScriptRoot } else { (Get-Location).Path }$script:settingsDir = Join-Path ([Environment]::GetFolderPath('LocalApplicationData')) 'BF4LANOneClick'
$script:settingsFile = Join-Path$script:settingsDir 'settings.json'
\$script:starting =\$false
$script:startTime = [DateTime]::MinValue$script:serverTime = [DateTime]::MinValue
\$script:serverStarted =\$false
\$script:gameStarted =\$false
\$script:gameLaunchAttempted =\$false
\$script:gameLaunchFailed =\$false
\$script:botDone =\$false
$script:botAttempts = 0$script:lastBotAttempt = [DateTime]::MinValue
\$script:runServer = ''$script:runGame = ''$script:runBots = 40$script:defaultTickets = 200$script:defaultTime = 200$script:runPort = 25210 $script:runPasswords = @()$script:activeRconPassword = ''
$script:lastProbe = [DateTime]::MinValue$script:firstPlayerSeen = [DateTime]::MinValue
$script:botsSentAt = [DateTime]::MinValue$script:botCommandAccepted = $false$script:reportedWaiting = $false$script:botLastError = ''
$script:probeFailures = 0$script:chatSent = $false$script:chatSentAt = [DateTime]::MinValue
$script:manualSendAt = [DateTime]::MinValue$script:chatSendError = ''
$script:lastHotkeyMinus = [DateTime]::MinValue$script:lastHotkeyEquals = [DateTime]::MinValue
\$script:minusCount = 0$script:equalsCount = 0 $script:lastHotkeySource = 'Not detected yet'$script:priorMinus = $false$script:priorEquals = $false$script:quickExpiry = [DateTime]::MinValue

Never call Test-Path -LiteralPath with a blank path: Windows PowerShell 5.1 throws.

function IsExistingFolder([string]$path) {
if ([string]::IsNullOrWhiteSpace($path)) { return$false }
try { return [bool](Test-Path -LiteralPath $path.Trim().Trim('"') -PathType Container -ErrorAction Stop) }
catch { return $false }
}
function IsExistingFile([string]$path) {
if ([string]::IsNullOrWhiteSpace($path)) { return$false }
try { return [bool](Test-Path -LiteralPath $path.Trim().Trim('"') -PathType Leaf -ErrorAction Stop) }
catch { return $false }
}
function C([string]$hex) { [System.Drawing.ColorTranslator]::FromHtml($hex) }
function Say([string]$message) {
$script:log.AppendText(('

$$' + (Get-Date -Format 'HH:mm:ss') + '$$

 ' +$message + [Environment]::NewLine))
$script:log.SelectionStart =$script:log.TextLength
$script:log.ScrollToCaret()
$script:state.Text =$message
}
function Warn([string]$message) {
[System.Windows.Forms.MessageBox]
::Show($message, 'BF4 LAN Launcher', 'OK', 'Warning') | Out-Null
Say $message
}
function ServerRoot {
$val =$script:serverBox.Text.Trim().TrimEnd('')
if (-not (IsExistingFolder $val)) { throw 'Please select the full Server folder (not the preview folder inside a ZIP archive).' }
foreach ($name in @('BF4BlazeServer.exe','!StartServer.bat','Instance\Admin\MapList.txt')) {
if (-not (IsExistingFile (Join-Path $val $name))) { throw ('Server folder is missing ' +$name) }
}
return $val
}
function MapCode {
$i =$script:mapCombo.SelectedIndex
if ($i -lt 0 -or $i -ge$script:mapChoices.Count) { throw 'Please select a map first.' }
return [string]$script:mapChoices[$i].Code
}
function SavePreferences {
if (-not (Test-Path -LiteralPath $script:settingsDir)) { New-Item -ItemType Directory -Path $script:settingsDir -Force \vert{} Out-Null }$data = @{
Server = $script:serverBox.Text.Trim()
Game = $script:gameBox.Text.Trim()
Map = (MapCode)
Bots = [int]$script:botsBox.Value
TicketsPct = [int]$script:ticketBox.Value
TimePct = [int]$script:timeBox.Value
}
$json =$data | ConvertTo-Json -Compress
[System.IO.File]
::WriteAllText($script:settingsFile, $json, (New-Object System.Text.UTF8Encoding($false)))
}
function LoadPreferences {
if (Test-Path -LiteralPath $script:settingsFile -PathType Leaf) {
try {
$data = (Get-Content -LiteralPath$script:settingsFile -Raw -Encoding UTF8 | ConvertFrom-Json)
if ($data.Server) { $script:serverBox.Text = [string]$data.Server }
if ($data.Game) { $script:gameBox.Text = [string]$data.Game }
for($i=0; $i -lt $script:mapChoices.Count; $i++) {
if (\$script:mapChoices\[\$i].Code -eq$data.Map) { $script:mapCombo.SelectedIndex =$i; break }
}
if ($null -ne$data.Bots) { $script:botsBox.Value = [decimal][Math]::Max(0, [Math]::Min(63, [int]$data.Bots)) }
if ($null -ne$data.TicketsPct) { $script:ticketBox.Value = [decimal][Math]::Max(75, [Math]::Min(400, [int]$data.TicketsPct)) }
if ($null -ne$data.TimePct) { $script:timeBox.Value = [decimal][Math]::Max(50, [Math]::Min(300, [int]$data.TimePct)) }
} catch { Say ('Saved preferences could not be read; using defaults: ' + $_.Exception.Message) }
}
if ([string]::IsNullOrWhiteSpace($script:serverBox.Text)) {
foreach($root in @($script:appRoot, (Split-Path$script:appRoot -Parent))) {
$candidate = Join-Path$root 'Server'
if (Test-Path -LiteralPath (Join-Path $candidate 'BF4BlazeServer.exe') -PathType Leaf) {
$script:serverBox.Text =$candidate; break
}
}
}
if ([string]::IsNullOrWhiteSpace($script:gameBox.Text)) {
foreach($root in @($script:appRoot, (Split-Path$script:appRoot -Parent))) {
$candidate = Join-Path$root 'BF4Launcher.exe'
if (Test-Path -LiteralPath $candidate -PathType Leaf) {
$script:gameBox.Text =$candidate; break
}
}
}
}

BF4 RANKED server settings: these are percentages, NOT ticket or minute counts.

function SelectedRuleValues {
$ticket=[int]$script:ticketBox.Value
$duration=[int]$script:timeBox.Value
if ($ticket -lt 75 -or$ticket -gt 400) { throw 'Ticket multiplier must be between 75-400% (current RANKED config).' }
if ($duration -lt 50 -or$duration -gt 300) { throw 'Round time multiplier must be between 50-300% (current RANKED config).' }
return @{ Ticket=$ticket; Time=$duration }
}
function SaveRulesToStartup([string]$server) {
$file=Join-Path$server 'Instance\Admin\Startup.txt'
if (-not (IsExistingFile $file)) { throw 'Cannot find Instance\Admin\Startup.txt, so tickets and time cannot be saved.' }$values=SelectedRuleValues
$old=[System.IO.File]::ReadAllText($file)
$ticketPattern='(?m)^(\s*vars\.gameModeCounter\s+)\d+(\s*(?:\#[^\r\n]*)?)$'
$timePattern='(?m)^(\s*vars\.roundTimeLimit\s+)\d+(\s*(?:\#[^\r\n]*)?)$'
if ([regex]::Matches($old,$ticketPattern).Count -ne 1 -or [regex]::Matches($old,$timePattern).Count -ne 1) { throw 'Ticket or time setting in Startup.txt is missing or duplicated; writing stopped to avoid corrupting it.' }$text=[regex]::Replace($old,$ticketPattern,{
param($m)$m.Groups[1].Value + [string]$values.Ticket +$m.Groups[2].Value
})
$text=[regex]::Replace($text,$timePattern,{
param($m)$m.Groups[1].Value + [string]$values.Time +$m.Groups[2].Value
})
if ($text -ceq $old) { return$false }
$backup=Join-Path (Split-Path$file -Parent) 'Startup.before_launcher_rules.txt'
if (-not (IsExistingFile $backup)) { [IO.File]::Copy($file,$backup) }
$tmp=$file+'.launcher-rules.tmp'
try {

UTF-8 without BOM retains ASCII configuration and all readable Unicode comments.

[IO.File]
::WriteAllText($tmp,$text,(New-Object System.Text.UTF8Encoding($false)))
[IO.File]
::Copy($tmp,$file,$true)
} finally {
if (IsExistingFile $tmp) { Remove-Item -LiteralPath$tmp -Force }
}
return $true
}
function SaveMap([string]$server, [string]$code) {
$mapFile = Join-Path$server 'Instance\Admin\MapList.txt'
$old = [System.IO.File]::ReadAllText($mapFile)
$nonComments = @($old -split '\r?\n' | Where-Object { $_.Trim() -ne '' -and (-not$.Trim().StartsWith('#')) })
foreach($line in$nonComments) {
if ($line.Trim() -notmatch '^\w+\s+\w+\s+\d+$') { throw 'MapList.txt contains unrecognized content; writing stopped to avoid corruption.' }
}
$new = "$code ConquestLarge0 1rn"
if ($old.Trim() -eq $new.Trim()) { Say ('No map change needed: ' + $code); return }
$backup = Join-Path (Split-Path $mapFile -Parent) 'MapList.before_launcher.txt'
if (-not (Test-Path -LiteralPath $backup)) { 

$$System.IO.File$$

::Copy($mapFile, $backup) }
$tmp = $mapFile + '.launcher.tmp'
try {


$$System.IO.File$$

::WriteAllText($tmp, $new, 

$$System.Text.Encoding$$

::ASCII)


$$System.IO.File$$

::Copy($tmp, $mapFile, $true)
} finally {
if (Test-Path -LiteralPath $tmp) { Remove-Item -LiteralPath $tmp -Force }
}
Say ('Selected ' + $code + ' (Conquest Large); original map list backed up.')
}
function ReadServerVars(

$$string$$

$server) {
$text = 

$$System.IO.File$$

::ReadAllText((Join-Path $server '!StartServer.bat'))
$p = 

$$regex$$

::Match($text, '(?im)^\s*set\s+REMOTEADMINPORT\s*=\s*(\d+)\s*$')
$a = 

$$regex$$

::Match($text, '(?im)^\sset\s+ADMINPASSWORD\s*=\s*(

$$^\\r\\n$$

+)')
$port = 25210
if ($p.Success) { $port = 

$$int$$

$p.Groups

$$1$$

.Value }

BF4 may apply admin.password from Startup.txt after processing the BAT argument.

Avoid assuming which wins: try unique local credential candidates, never print them.

$passwords = New-Object 'System.Collections.Generic.List

$$string$$

'
$startup = Join-Path $server 'Instance\Admin\Startup.txt'
if (IsExistingFile $startup) {
$startupText = 

$$System.IO.File$$

::ReadAllText($startup)
foreach ($m in 

$$regex$$

::Matches($startupText, '(?im)^\sadmin.password\s+(

$$^\\r\\n#$$

+)')) {
$v = $m.Groups

$$1$$

.Value.Trim().Trim('"')
if ($v -and -not $passwords.Contains($v)) { $passwords.Add($v) }
}
}
if ($a.Success) {
$v = $a.Groups

$$1$$

.Value.Trim().Trim('"')
if ($v -and -not $passwords.Contains($v)) { $passwords.Add($v) }
}
return @{ Port=$port; Passwords=@($passwords.ToArray()) }
}
function IsPortOpen(

$$int$$

$port) {
$t = $null
try {
$t = New-Object System.Net.Sockets.TcpClient
$ar = $t.BeginConnect('127.0.0.1', $port, $null, $null)
if (-not $ar.AsyncWaitHandle.WaitOne(160)) { return $false }
$t.EndConnect($ar)
return $true
} catch { return $false } finally { if ($t) { $t.Close() } }
}
function ReadExactly($stream, 

$$int$$

$count) {
$out = 

$$byte\[$$

]::new($count)
$offset = 0
while ($offset -lt $count) {
$n = $stream.Read($out, $offset, $count - $offset)
if ($n -le 0) { throw 'Server closed the RCON connection' }
$offset += $n
}
return ,$out
}
function CallRcon($stream, 

$$int$$

$sequence, 

$$string\[$$

]$words) {
$buf = 

$$System.IO.MemoryStream$$

::new()
$bw = 

$$System.IO.BinaryWriter$$

::new($buf)
$bw.Write(

$$uint32$$

$sequence)
$bw.Write(

$$uint32$$

0)
$bw.Write(

$$uint32$$

$words.Length)
foreach($word in $words) {
$raw = 

$$System.Text.Encoding$$

::UTF8.GetBytes(

$$string$$

$word)
$bw.Write(

$$uint32$$

$raw.Length)
$bw.Write($raw)
$bw.Write(

$$byte$$

0)
}
$payload = $buf.ToArray()


$$BitConverter$$

::GetBytes(

$$uint32$$

$payload.Length).CopyTo($payload,4)
$stream.Write($payload,0,$payload.Length)
for($r=0; $r -lt 16; $r++) {
$hdr = ReadExactly $stream 12
$packetSeq = 

$$BitConverter$$

::ToUInt32($hdr,0)
$length = 

$$BitConverter$$

::ToUInt32($hdr,4)
$numWords = 

$$BitConverter$$

::ToUInt32($hdr,8)
if ($length -lt 12 -or $length -gt 1048576 -or $numWords -gt 256) { throw 'Invalid RCON packet' }
$data = ReadExactly $stream (

$$int$$

$length - 12)
$ms = 

$$System.IO.MemoryStream$$

::new($data)
$br = 

$$System.IO.BinaryReader$$

::new($ms)
$response = New-Object System.Collections.Generic.List

$$string$$


for($i=0; $i -lt $numWords; $i++) {
$len = 

$$int$$

$br.ReadUInt32()
if($len -lt 0 -or $len -gt 65536 -or $len -gt ($ms.Length - $ms.Position - 1)) { throw 'Corrupted RCON field' }
$value = 

$$System.Text.Encoding$$

::UTF8.GetString($br.ReadBytes($len))
if($br.ReadByte() -ne 0) { throw 'RCON field format error' }
$response.Add($value)
}
if (($packetSeq -band 

$$uint32$$

0x40000000) -ne 0 -and (($packetSeq -band 

$$uint32$$

0x3fffffff) -eq $sequence)) {
return ,$response.ToArray()
}
}
throw 'No RCON response received'
}
function InvokeRcon(

$$int$$

$port, 

$$string$$

$password, 

$$string\[$$

]$words) {
if (

$$string$$

::IsNullOrWhiteSpace($password)) { throw 'RCON admin password not set' }
$t = New-Object System.Net.Sockets.TcpClient
try {
$ar = $t.BeginConnect('127.0.0.1',$port,$null,$null)
if(-not $ar.AsyncWaitHandle.WaitOne(1300)) { throw 'RCON is not open yet' }
$t.EndConnect($ar)
$t.ReceiveTimeout=1700; $t.SendTimeout=1700
$stream=$t.GetStream()
$challenge=CallRcon $stream 1 @('login.hashed')
if ($challenge.Count -lt 2 -or $challenge

$$0$$

 -ne 'OK') { throw 'Server did not return a standard RCON login challenge' }
$hex=

$$string$$

$challenge

$$1$$


if ($hex.Length % 2 -ne 0) { throw 'RCON challenge format error' }
$salt=

$$byte\[$$

]::new(int)
for($i=0; $i -lt $salt.Length; $i++) { $salt

$$$i$$

=

$$Convert$$

::ToByte($hex.Substring($i2,2),16) }
$passBytes=

$$Text.Encoding$$

::UTF8.GetBytes($password)
$all=

$$byte\[$$

]::new(

$$int$$

($salt.Length + $passBytes.Length))


$$Array$$

::Copy($salt,0,$all,0,$salt.Length)


$$Array$$

::Copy($passBytes,0,$all,$salt.Length,$passBytes.Length)
$md5=

$$System.Security.Cryptography.MD5$$

::Create()
try { $digest=

$$BitConverter$$

::ToString($md5.ComputeHash($all)).Replace('-','') }
finally { $md5.Dispose() }
$login=CallRcon $stream 2 @('login.hashed',$digest)
if ($login.Count -lt 1 -or $login

$$0$$

 -ne 'OK') { throw 'RCON login verification failed' }
return ,(CallRcon $stream 3 $words)
} finally { $t.Close() }
}
function QueryRcon(

$$int$$

$port, 

$$string\[$$

]$words) {
$candidates=@()
if ($script:activeRconPassword) { $candidates=@($script:activeRconPassword) }
else { $candidates=@($script:runPasswords) }
if ($candidates.Count -eq 0) { throw 'No RCON password found in the BAT file or Startup.txt' }
$last=''
foreach($pw in $candidates) {
try {
$result=InvokeRcon $port $pw $words
$script:activeRconPassword=$pw
return ,$result
} catch {
$last=$.Exception.Message
if ($last -ne 'RCON login verification failed') { throw $last }
}
}
throw ('Neither server password passed RCON verification; please check the admin password actually in use.')
}
function GetServerPlayers(

$$int$$

$port) {
$words=QueryRcon $port @('serverInfo')
if ($words.Count -lt 4 -or $words

$$0$$

 -ne 'OK') { throw ('Abnormal serverInfo response: ' + ($words -join ' ')) }
$players=0
if (-not 

$$int$$

::TryParse(

$$string$$

$words

$$2$$

,

$$ref$$

$players)) { throw ('Cannot parse serverInfo player count: ' + 

$$string$$

$words

$$2$$

) }
return $players
}
function SetBotsByRcon(

$$int$$

$port, 

$$int$$

$bots) {
$result=QueryRcon $port @('PseudoPlayer.SpawnCount', 

$$string$$

$bots)
if ($result.Count -gt 0 -and $result

$$0$$

 -eq 'OK') { return 'OK' }
throw ('Server rejected PseudoPlayer.SpawnCount: ' + ($result -join ' '))
}
function TrySendBotChat(

$$int$$

$count) {
if ($count -lt 0) { return $false }
if (-not 

$$BF4ChatInput$$

::IsBF4Foreground()) { return $false }
try {
$result=

$$BF4ChatInput$$

::SendCommand($count)
if($result -ne 'OK') {
Say ('Could not auto-type chat command: ' + $result)
return $false
}
$script:chatSentAt=Get-Date
$script:chatSent=$true
Say ('Attempted to send !setbot ' + $count + ' (Win32 keyboard events). Verifying player count; if it did not take effect, check the in-game chat.')
return $true
} catch {
Say ('Simulated keypress failed: ' + $.Exception.Message)
return $false
}
}
function LaunchGame(

$$bool$$

$manual = $false) {

Mark the automatic attempt BEFORE asking Windows to start the app.

UAC denial (or a Windows launch error) MUST NOT trigger another attempt on the next timer tick.

if ((-not $manual) -and $script:gameLaunchAttempted) { return }
$script:gameLaunchAttempted = $true
$script:gameLaunchFailed = $false
$path = 

$$string$$

$script:runGame
try {
if (-not (IsExistingFile $path)) { throw 'BF4Launcher.exe not found; please check the launcher path.' }
$psi = New-Object System.Diagnostics.ProcessStartInfo
$psi.FileName = $path
$psi.WorkingDirectory = Split-Path $path -Parent
$psi.UseShellExecute = $true

Do NOT set Verb='runas': let the target EXE manifest decide whether a single UAC prompt is necessary.

$proc = 

$$System.Diagnostics.Process$$

::Start($psi)
if ($null -eq $proc) { throw 'Windows did not return game process information.' }
$script:gameStarted = $true
Say ('Step 3: BF4Launcher launch request submitted (PID ' + $proc.Id + '). Windows may show a normal UAC prompt once.')
Say 'Note: a successfully created process does not guarantee the game window appears. If no window shows up, first try double-clicking BF4Launcher.exe in the game folder.'
} catch {
$script:gameLaunchFailed = $true
$script:gameStarted = $false
$detail = $*.Exception.Message
Say ('Step 3: Game launch request failed: ' + $detail)
Say 'Automatic retry stopped to avoid repeated UAC prompts. The server is still running; you can click "Retry Game Manually" later.'
}
}
function FinishSequence {
$script:timer.Stop()
$script:starting = $false
$script:startButton.Enabled = $true
$script:retryGameButton.Enabled = $true
$script:state.ForeColor = C '#34D399'
Say 'Startup sequence complete. The server keeps running; closing this launcher will not close the server.'
}
function QueueManualChat {
if($script:runBots -lt 1) { Warn 'Please set a bot count greater than 0 and click One-Click Start first.'; return }
$script:manualSendAt=(Get-Date).AddSeconds(10)
Say ('Will try to send !setbot ' + $script:runBots + ' in 10 seconds. Switch back to the BF4 game now. Keys are only sent while BF4 is in the foreground, never to other windows.')
$script:manualTimer.Start()
}
function CopyBotChat {


$$Windows.Forms.Clipboard$$

::SetText('!setbot ' + 

$$int$$

$script:botsBox.Value)
Say ('Copied !setbot ' + 

$$int$$

$script:botsBox.Value + '; you can paste it into the game chat.')
}
function StartAll {
if ($script:starting) { return }
try {
$server = ServerRoot
$game = $script:gameBox.Text.Trim().Trim('"')
if (-not (IsExistingFile $game) -or (

$$IO.Path$$

::GetFileName($game) -ine 'BF4Launcher.exe')) { throw 'Please select the real BF4Launcher.exe file.' }

Never change a running server's MapList.

$existing = @(Get-Process -Name 'bf4_server_Final','bf4_server_Retail' -ErrorAction SilentlyContinue)
if ($existing.Count -gt 0) { throw 'A BF4 dedicated server is already running. Please end the old match first, then change the map and start again.' }
SaveMap $server (MapCode)


$$void$$

(SaveRulesToStartup $server)
SavePreferences
Say ('Tickets/time saved to Startup.txt: ' + 

$$int$$

$script:ticketBox.Value + '% / ' + 

$$int$$

$script:timeBox.Value + '%. ')
$creds = ReadServerVars $server
$script:runServer = $server
$script:runGame = $game
$script:runBots = 

$$int$$

$script:botsBox.Value
$script:TicketsPct = 

$$int$$

$script:ticketBox.Value
$script:TimePct = 

$$int$$

$script:timeBox.Value
$script:runPort = 

$$int$$

$creds.Port
$script:runPasswords = @($creds.Passwords)
$script:activeRconPassword = ''
$script:lastProbe = 

$$DateTime$$

::MinValue
$script:firstPlayerSeen = 

$$DateTime$$

::MinValue
$script:botsSentAt = 

$$DateTime$$

::MinValue
$script:botCommandAccepted = $false
$script:reportedWaiting = $false
$script:botLastError = ''
$script:probeFailures = 0
$script:chatSent = $false
$script:chatSentAt = 

$$DateTime$$

::MinValue
$script:manualSendAt = 

$$DateTime$$

::MinValue
$script:serverStarted = $false
$script:gameStarted = $false
$script:gameLaunchAttempted = $false
$script:gameLaunchFailed = $false
$script:botDone = $false
$script:botAttempts = 0
$script:lastBotAttempt = 

$$DateTime$$

::MinValue
$script:startTime = Get-Date
$script:starting = $true
$script:startButton.Enabled = $false
$script:retryGameButton.Enabled = $false
Start-Process -FilePath (Join-Path $server 'BF4BlazeServer.exe') -WorkingDirectory $server | Out-Null
Say 'Step 1: Blaze service started; preparing the BF4 dedicated server...'
Say 'Bot control: after your character has deployed, press = to send !setbot; press - to open the map/rules panel.'
Say 'Map changes use mapList RCON; bots only use the original mod chat command, not the ineffective PseudoPlayer.SpawnCount.'

$script:timer.Start()
} catch { Warn ('Cannot start: ' + $_.Exception.Message) }
}

--------------------- In-match map controls (experimental on ALPHA 0.1) ---------------------

function RuntimeRconReady {
# The launcher may have been reopened while an existing server stayed running.
$server=ServerRoot
if ($script:runServer -ne $server -or $script:runPort -le 0 -or $script:runPasswords.Count -eq 0) {
$creds=ReadServerVars $server
$script:runServer=$server
$script:runPort=

$$int$$

$creds.Port
$script:runPasswords=@($creds.Passwords)
$script:activeRconPassword=''
}
if (-not (IsPortOpen $script:runPort)) { throw 'The server RCON admin port is not ready yet; please start the server first.' }
return $server
}
function RconOK(

$$string\[$$

]$words) {
$response=QueryRcon $script:runPort $words
if ($response.Count -lt 1 -or $response

$$0$$

 -ne 'OK') {
throw ('Server rejected command ' + $words

$$0$$

 + ': ' + ($response -join ' '))
}
return ,$response
}

Rule SET commands return either OK (applied) or ExecutedOnNextRound

(accepted for the next round). Neither response is an error.

Do not relax RconOK's contract: map management and getters still require OK.

function SetRoundRule(

$$string$$

$name, 

$$int$$

$percent) {
$reply=QueryRcon $script:runPort @($name,

$$string$$

$percent)
if ($null -eq $reply -or $reply.Count -lt 1) {
throw ('Server did not reply to the ' + $name + ' setting command.')
}
$status=(

$$string$$

$reply

$$0$$

).Trim()
if ($status -ieq 'ExecutedOnNextRound') {
Say ('Server accepted ' + $name + '=' + $percent + '%: takes effect next round (normal acknowledgement).')
return 'next'
}
if ($status -ieq 'OK') {
Say ('Server accepted ' + $name + '=' + $percent + '%: returned OK; treated as taking effect next round.')
return 'applied'
}
throw ('Server did not accept ' + $name + ': ' + ($reply -join ' | '))
}
function ApplyRuntimeRules {
try {
$server=RuntimeRconReady
$values=SelectedRuleValues
$changed=SaveRulesToStartup $server
SavePreferences
if ($changed) { Say 'Backed up and updated Startup.txt (for the next server start).' }
Say ('Set Conquest ticket count=' + $values.Ticket + '%; round time=' + $values.Time + '%; sending RCON...')
$ticketStatus=SetRoundRule 'vars.gameModeCounter' (

$$int$$

$values.Ticket)
$timeStatus=SetRoundRule 'vars.roundTimeLimit' (

$$int$$

$values.Time)
$next=($ticketStatus -eq 'next' -or $timeStatus -eq 'next')
if ($next) {
Say ('Settings accepted: tickets and time take effect next round; the current round is unaffected.')
} else {
Say 'Both rule commands received OK. Treat the new rules as applying from the next round.'
}
try {
$a=RconOK @('vars.gameModeCounter')
$b=RconOK @('vars.roundTimeLimit')
$actualTicket=0; $actualTime=0
$aOK=($a.Count -ge 2 -and 

$$int$$

::TryParse(

$$string$$

$a

$$1$$

,

$$ref$$

$actualTicket))
$bOK=($b.Count -ge 2 -and 

$$int$$

::TryParse(

$$string$$

$b

$$1$$

,

$$ref$$

$actualTime))
if ($aOK -and $bOK -and $actualTicket -eq $values.Ticket -and $actualTime -eq $values.Time) {
Say ('RCON read back the selected values: tickets ' + $actualTicket + '%, time ' + $actualTime + '%. Still recommend checking the actual result next match.')
} elseif ($next) {
Say ('Current readback: tickets=' + ($a -join '|') + ', time=' + ($b -join '|') + '. Because the command is deferred to the next match, an unchanged readback is normal.')
} else {
Say ('Setting command received OK, but current readback is inconsistent: tickets=' + ($a -join '|') + ', time=' + ($b -join '|') + '. Please verify next round.')
}
} catch {
Say ('Server accepted the setting command, but readback failed: ' + $.Exception.Message + '. Please verify next round.')
}
$script:state.Text='Rules set successfully: effective next round. The current round is unchanged.'
$script:state.ForeColor=C '#34D399'
} catch {
Warn ('Rule setting not completed: ' + $.Exception.Message + "`nStartup.txt may already have been saved; if partially successful, check the log.")
}
}

BF4 RCON format: OK      ...

v0.8 incorrectly treated  (normally 3) as the first map name.

Read and validate the complete response before sending any state-changing map command.

function RuntimeMapEntries {
try { $result=RconOK @('mapList.list','0') }
catch { $result=RconOK @('mapList.list') }
$preview=($result -join ' | ')
if ($result.Count -lt 3) { throw ('mapList.list returned too few fields: ' +$preview) }
$count=0; $stride=0
if (-not [int]::TryParse([string]$result[1],[ref]$count) -or $count -lt 0 -or$count -gt 100) {
throw ('Cannot parse map count. Raw reply: ' + $preview)
}
if (-not [int]::TryParse([string]$result[2],[ref]$stride) -or$stride -ne 3) {
throw ('Server map entries are not in a 3-column format (map/mode/rounds); cannot safely change map. Raw reply: ' + $preview)
}
$required=3+$count*$stride
if (\$result.Count -lt$required) { throw ('Map list incomplete (need ' + $required + ' fields, got ' +$result.Count + '). Raw reply: ' + $preview) }$all=@()
for($i=0;$i -lt $count;$i++) {
$base=3+$i*$stride$map=[string]$result[$base]
$mode=[string]$result

$$\$base+1$$

$rounds=[string]$result

$$\$base+2$$


if ([string]::IsNullOrWhiteSpace($map) -or [string]::IsNullOrWhiteSpace($mode) -or $rounds -notmatch '^\d+$') {
throw ('Invalid map entry found (index ' + $i + '). Raw reply: ' + $preview)
}
$all+=@{ Index=$i; Map=$map; Mode=$mode; Rounds=$rounds }
}
return $all
}
function MapEntryIndex([object[]]$entries, [string]$code) {
foreach($entry in$entries) {
if ($entry.Map -ieq$code -and $entry.Mode -ieq 'ConquestLarge0') { return [int]$entry.Index }
}
return -1
}
function ApplyRuntimeMap([bool]$immediate) {
try {
$server=RuntimeRconReady
$code=MapCode$entries=@(RuntimeMapEntries)
if ($entries.Count -lt 1) { throw 'Server map list is currently empty. Operation stopped to avoid an endless loading screen.' }
Say ('Server map list: ' + (($entries | ForEach-Object { $_.Index.ToString() + '=' +$.Map + '/' + $.Mode }) -join ', '))$index=MapEntryIndex $entries$code
if ($index -lt 0) {
Say ('Server already has ' + $entries.Count + ' maps; adding ' +$code + '...')
[void]
(RconOK @('mapList.add',$code,'ConquestLarge0','1'))

Some custom server builds apply mapList changes asynchronously.

Never assume OK means the entry exists. Read it back before switching.

for($attempt=1; $attempt -le 3; $attempt++) {
Start-Sleep -Milliseconds 250
\$entries=@(RuntimeMapEntries)\$index=MapEntryIndex $entries$code
if (\$index -ge 0) { break }
}
if (\$index -lt 0) {
Say ('Map list after adding: ' + (($entries \vert{} ForEach-Object {$_.Index.ToString() + '=' + $_.Map + '/' +$*.Mode }) -join ', '))
throw 'RCON reply accepted mapList.add, but the server re-check still shows no target map in the list. The Alpha server may not support dynamic adding; stopped to avoid switching to the wrong map.'
}
Say ('New map now appears in the server list, index=' + $index)
} else {
Say ('Target map already exists in the server list, index=' + $index + '. No need to add it again.')
}
[void]
(RconOK @('mapList.setNextMapIndex',[string]$index))

Most BF4 servers return [OK,current,next]. Non-standard ALPHA builds may differ.

try {
$indices=RconOK @('mapList.getMapIndices')
$actualNext=0
if ($indices.Count -ge 3 -and [int]::TryParse([string]$indices[2],[ref]$actualNext)) {
if (*$actualNext -ne$*index) {
throw ('mapList.getMapIndices shows the next map index as ' + $actualNext + ', but the required index is ' +$index + '. ')
}
Say ('Next map index verified: ' + $actualNext)
} else {
Say ('Map index query returned a non-standard result: ' + ($indices -join ' | ') + '. Please still confirm in the actual game.')
}
} catch {

UnknownCommand here does not prove that setNextMapIndex failed.

Say ('Could not additionally verify the next map index: ' + $.Exception.Message)
}
Say ('RCON accepted the next-map setting: ' + $code + ' (index ' +$index + '). ')
try {
SaveMap $server$code # next fresh server startup
SavePreferences
} catch { Say ('Local MapList.txt could not be saved, but the server has handled the request for this run: ' + $.Exception.Message) }
if ($immediate) {
[void]
(RconOK @('mapList.runNextRound'))
Say ('Requested an immediate switch to ' + $code + '. After the map loads, deploy and press = to apply bots.')
} else {
Say ('Scheduled the next match to use ' + $code + '; keep fighting and check whether the map changed when the round ends.')
}
} catch { Warn ('Live map change failed: ' + $*.Exception.Message) }
}
function QueueRuntimeBot([int]$seconds) {
$script:runBots=[int]$script:botsBox.Value
try { SavePreferences } catch {}
$script:manualSendAt=(Get-Date).AddSeconds($seconds)
Say ('Queued: ' + $seconds + ' seconds until !setbot ' +$script:runBots + '. Make sure your character has deployed, the game window is in the foreground, and the chat box is not open.')
$script:manualTimer.Start()
}

------------------------- WinForms window -------------------------

Dark theme. Only the look was changed; control names, handlers and logic are the same.

$script:cBg='#0B1220'; $script:cCard='#131C31'; $script:cInput='#0A101D'
$script:cText='#E2E8F0'; $script:cMuted='#8CA0BC'; $script:cAccent='#38BDF8'

function NewLabel($parent,[string]$txt,[int]$x,[int]$y,[int]$w,[int]$h,[string]$color) {$l = New-Object System.Windows.Forms.Label
$l.Text =$txt; $l.SetBounds($x,$y,$w,$h);$l.ForeColor = C $color$l.BackColor = [System.Drawing.Color]::Transparent
$parent.Controls.Add($l)
return $l
}
function NewCard([string]$title,[int]$y,[int]$h) {$p = New-Object System.Windows.Forms.Panel
$p.SetBounds(24,$y,856,$h);$p.BackColor = C$script:cCard; $p.Anchor = 'Top,Left,Right'$script:form.Controls.Add($p)$t = NewLabel $p$title 18 8 500 20 $script:cAccent$t.Font = New-Object System.Drawing.Font('Segoe UI',8.5,[System.Drawing.FontStyle]::Bold)
return $p
}
function StyleInput($c) { $c.BackColor = C$script:cInput; $c.ForeColor = C$script:cText }
function StyleButton($b,[string]$bg,[string]$hover,[string]$fg) {$b.FlatStyle = 'Flat';$b.FlatAppearance.BorderSize = 0$b.UseVisualStyleBackColor =$false$b.BackColor = C $bg; $b.ForeColor = C $fg$b.FlatAppearance.MouseOverBackColor = C $hover$b.FlatAppearance.MouseDownBackColor = C $bg$b.Cursor = [System.Windows.Forms.Cursors]::Hand
}

$script:form = New-Object BF4HotkeyForm
$form.Text = 'BF4 LAN BOT - One-Click Launcher v1.0.4 \vert{} English version by TEGUH DWI iD'$form.StartPosition = 'CenterScreen'
$form.BackColor = C$script:cBg
$form.Font = New-Object Drawing.Font('Segoe UI',9.5)$form.ClientSize = New-Object Drawing.Size(904,980) $form.MinimumSize =$form.Size
$form.AutoScaleMode = 'Dpi'

---- Header ----

$header = New-Object System.Windows.Forms.Panel
$header.SetBounds(0,0,904,84);$header.BackColor = C '#0F1A33'; $header.Anchor = 'Top,Left,Right'$form.Controls.Add($header)$strip = New-Object System.Windows.Forms.Panel
$strip.SetBounds(0,0,6,84); $strip.BackColor = C$script:cAccent
$header.Controls.Add($strip)
$title = NewLabel$header 'BATTLEFIELD 4' 26 6 520 42 '#FFFFFF'
$title.Font = New-Object Drawing.Font('Segoe UI',20,[Drawing.FontStyle]::Bold)
$sub = NewLabel$header 'LAN BOT   |   One-Click Launcher   v1.0.4' 28 50 520 24 $script:cAccent$sub.Font = New-Object Drawing.Font('Segoe UI',10,[Drawing.FontStyle]::Bold)
$credit = NewLabel$header 'English version by TEGUH DWI iD' 600 28 280 28 '#BFDBFE'
$credit.BackColor = C '#1E3A8A'; $credit.TextAlign = 'MiddleCenter'; $credit.Anchor = 'Top,Right'$credit.Font = New-Object Drawing.Font('Segoe UI',9,[Drawing.FontStyle]::Bold)
$keysHint = NewLabel$form 'In BF4, press - on the main keyboard to open the panel; after deploying, press = to apply bots. You can also Alt+Tab to operate.' 26 92 850 20 $script:cMuted$keysHint.Anchor = 'Top,Left,Right'

---- Card 1: setup ----

$setup = NewCard 'SETUP' 118 146
$null = NewLabel $setup 'Server folder' 18 32 400 20$script:cText
$script:serverBox = New-Object Windows.Forms.TextBox
$serverBox.SetBounds(18,54,726,27); $serverBox.Anchor = 'Top,Left,Right'
$serverBox.BorderStyle = 'FixedSingle'; StyleInput $serverBox
$setup.Controls.Add($serverBox)
$serverBrowse = New-Object Windows.Forms.Button
$serverBrowse.Text='Browse...'; $serverBrowse.SetBounds(754,52,84,31);$serverBrowse.Anchor='Top,Right'
StyleButton $serverBrowse '#1F2A44' '#2B3A5C' '#E2E8F0'
$setup.Controls.Add($serverBrowse)
$serverBrowse.Add_Click({
$d=$null
try {
$d=New-Object Windows.Forms.FolderBrowserDialog
$d.Description='Select the extracted Server folder that contains BF4BlazeServer.exe'
$current=[string]$script:serverBox.Text
# On the first run this textbox is blank. Do NOT pass blank to -LiteralPath.
if (IsExistingFolder $current) { $d.SelectedPath=$current.Trim().Trim('"') }
if ($d.ShowDialog($script:form) -eq [System.Windows.Forms.DialogResult]::OK) {
$chosen=[string]$d.SelectedPath
if (IsExistingFolder $chosen) {
$script:serverBox.Text=$chosen
Say ('Selected server folder: ' + $chosen)
if (-not (IsExistingFile (Join-Path $chosen 'BF4BlazeServer.exe'))) {
Warn 'BF4BlazeServer.exe was not found in this folder. Please make sure you selected the fully extracted Server folder.'
}
} else { Warn 'The selected server folder does not exist. Please fully extract the package first.' }
}
} catch {
Warn ('A problem occurred while selecting the server folder: ' + $*.Exception.Message)
} finally {
if ($null -ne $d) {$d.Dispose() }
}
})
*$null = NewLabel$setup 'Game launcher (BF4Launcher.exe)' 18 86 510 20 $script:cText$*script:gameBox = New-Object Windows.Forms.TextBox
$gameBox.SetBounds(18,108,726,27); $gameBox.Anchor='Top,Left,Right'$gameBox.BorderStyle = 'FixedSingle'; StyleInput *$gameBox$setup.Controls.Add($gameBox)$gameBrowse=New-Object Windows.Forms.Button
$gameBrowse.Text='Browse...'; $gameBrowse.SetBounds(754,106,84,31);$gameBrowse.Anchor='Top,Right'
StyleButton $gameBrowse '#1F2A44' '#2B3A5C' '#E2E8F0'
$setup.Controls.Add($gameBrowse)$gameBrowse.Add_Click({
$d=$null
try {
$d=New-Object Windows.Forms.OpenFileDialog
$d.Title='Select BF4Launcher.exe'$d.Filter='BF4Launcher.exe|BF4Launcher.exe|All EXE|*.exe'
$current=[string]$script:gameBox.Text
if (IsExistingFile $current) { $d.FileName=$current.Trim().Trim('"') }
if ($d.ShowDialog($script:form) -eq [System.Windows.Forms.DialogResult]::OK) {
$script:gameBox.Text=$d.FileName
Say ('Selected game launcher: ' + $d.FileName)
}
} catch { Warn ('A problem occurred while selecting the game launcher: ' + $.Exception.Message) }
finally { if ($null -ne $d) {$d.Dispose() } }
})

---- Card 2: match settings ----

$match = NewCard 'MATCH SETTINGS' 274 190
$null = NewLabel $match 'Select map - Conquest Large' 18 32 440 20$script:cText
$null = NewLabel $match 'Total bots (both teams combined)' 508 32 330 20$script:cText
$script:mapCombo=New-Object Windows.Forms.ComboBox
$mapCombo.SetBounds(18,54,470,30); $mapCombo.DropDownStyle='DropDownList'
$mapCombo.FlatStyle='Flat'; StyleInput $mapCombo
foreach($m in $mapChoices){ [void]$mapCombo.Items.Add(($m.Name + '   \vert{}   ' +$m.Code)) }
$mapCombo.SelectedIndex=0; $match.Controls.Add($mapCombo)
$script:botsBox=New-Object Windows.Forms.NumericUpDown
$botsBox.SetBounds(508,54,330,30); $botsBox.Minimum=0; $botsBox.Maximum=63; $botsBox.Value=40 \$botsBox.BorderStyle='FixedSingle'; StyleInput$botsBox
$match.Controls.Add($botsBox)
$hint = NewLabel $match 'Conquest mode: ticket count and round time are adjusted as a percentage of the vanilla standard values (not absolute tickets or minutes).' 18 90 820 34$script:cMuted
$hint.Font = New-Object Drawing.Font('Segoe UI',8.5)
$null = NewLabel $match 'Ticket multiplier (\%)' 18 126 245 20$script:cText
$script:ticketBox=New-Object Windows.Forms.NumericUpDown
$ticketBox.SetBounds(18,148,240,30); $ticketBox.Minimum=75; $ticketBox.Maximum=400; $ticketBox.Increment=25; $ticketBox.Value=200 \$ticketBox.BorderStyle='FixedSingle'; StyleInput$ticketBox
$match.Controls.Add($ticketBox)
$null = NewLabel $match 'Round time multiplier (\%)' 278 126 215 20$script:cText
$script:timeBox=New-Object Windows.Forms.NumericUpDown
$timeBox.SetBounds(278,148,210,30); $timeBox.Minimum=50; $timeBox.Maximum=300; $timeBox.Increment=25; $timeBox.Value=200 \$timeBox.BorderStyle='FixedSingle'; StyleInput$timeBox
$match.Controls.Add($timeBox)
$script:applyRulesButton=New-Object Windows.Forms.Button
$applyRulesButton.Text='Apply tickets / time rules'; $applyRulesButton.SetBounds(508,146,330,34)
StyleButton $applyRulesButton '#0E7490' '#0891B2' '#FFFFFF'
$match.Controls.Add($applyRulesButton)
$applyRulesButton.Add_Click({ ApplyRuntimeRules })

---- Card 3: launch ----

$launch = NewCard 'LAUNCH' 474 150
$script:startButton=New-Object Windows.Forms.Button
$startButton.Text='START BF4   (Server + Game)'
$startButton.SetBounds(18,34,480,56)
StyleButton $startButton '#2563EB' '#3B82F6' '#FFFFFF'
$startButton.Font = New-Object Drawing.Font('Segoe UI',12,[Drawing.FontStyle]::Bold)
$launch.Controls.Add($startButton)
$startButton.Add_Click({ StartAll })
$copyButton=New-Object Windows.Forms.Button
$copyButton.Text='Copy !setbot command'; $copyButton.SetBounds(516,34,322,32)
StyleButton $copyButton '#1F2A44' '#2B3A5C' '#E2E8F0'
$launch.Controls.Add($copyButton)
$copyButton.Add_Click({try { CopyBotChat } catch { Warn $_.Exception.Message }})
$manualChatButton=New-Object Windows.Forms.Button
$manualChatButton.Text='Send bot command in 10 s'; $manualChatButton.SetBounds(516,72,322,32)
StyleButton $manualChatButton '#059669' '#10B981' '#FFFFFF'
$launch.Controls.Add($manualChatButton)
$manualChatButton.Add_Click({ QueueRuntimeBot 10 })

Manual retry is a user action, never a background timer retry.

$script:retryGameButton=New-Object Windows.Forms.Button
$retryGameButton.Text='Retry Game Manually (once only)'
$retryGameButton.SetBounds(516,110,322,32)
StyleButton $retryGameButton '#1F2A44' '#2B3A5C' '#E2E8F0'
$launch.Controls.Add($retryGameButton)
$retryGameButton.Add_Click({
if ($script:starting) { Warn 'The automatic startup sequence has not finished yet; please retry manually later.'; return }
$path=$script:gameBox.Text.Trim().Trim('"')
if (-not (IsExistingFile $path) -or ([IO.Path]::GetFileName($path) -ine 'BF4Launcher.exe')) {
Warn 'Please select the real BF4Launcher.exe first.'; return
}
$script:runGame=$path
LaunchGame $true
})
$script:state = New-Object Windows.Forms.Label
$state.Text='Standby: please select the paths and bot count.'; $state.ForeColor=C '#94A3B8'
$state.BackColor=[System.Drawing.Color]::Transparent
$state.SetBounds(18,98,480,44); $state.Anchor='Top,Left,Right'; $launch.Controls.Add($state)

---- Card 4: live match control ----

Runtime controls are usable even after startup; the launcher must remain open.

$live = NewCard 'LIVE MATCH CONTROL' 634 168
$runtimeTitle = NewLabel $live 'Match control    - open panel    \vert{}    = apply bots    \vert{}    change next map, ticket count and time' 18 30 820 22$script:cText
$runtimeTitle.Font=New-Object Drawing.Font('Segoe UI',9.5,[Drawing.FontStyle]::Bold)
$script:nextMapButton=New-Object Windows.Forms.Button
$nextMapButton.Text='Use this map next round'
$nextMapButton.SetBounds(18,58,266,42)
StyleButton $nextMapButton '#0E7490' '#0891B2' '#FFFFFF'
$live.Controls.Add($nextMapButton)
$nextMapButton.Add_Click({ ApplyRuntimeMap $false })
$script:nowMapButton=New-Object Windows.Forms.Button
$nowMapButton.Text='Switch to this map now'
$nowMapButton.SetBounds(294,58,266,42)
StyleButton $nowMapButton '#D97706' '#F59E0B' '#FFFFFF'
$live.Controls.Add($nowMapButton)
$nowMapButton.Add_Click({
if ([Windows.Forms.MessageBox]::Show('This will immediately end the current round and request loading the selected map. The Alpha version may error when changing maps. Continue?','Switch Map Now','YesNo','Warning') -eq 'Yes') {
ApplyRuntimeMap $true
}
})
$script:botRuntimeButton=New-Object Windows.Forms.Button
$botRuntimeButton.Text='Change bot count in 5 s'
$botRuntimeButton.SetBounds(570,58,268,42)
StyleButton $botRuntimeButton '#059669' '#10B981' '#FFFFFF'
$live.Controls.Add($botRuntimeButton)
$botRuntimeButton.Add_Click({ QueueRuntimeBot 5 })
$runtimeHint = NewLabel $live 'Ticket and time settings take effect from the next round; the current match will not change. - and = are only monitored while BF4 is in the foreground.' 18 108 820 34$script:cMuted
$runtimeHint.Font = New-Object Drawing.Font('Segoe UI',8.5)
$script:hotkeyState = NewLabel $live 'Hotkey self-check: detecting after startup...' 18 142 820 20 '#34D399'

---- Activity log + footer ----

$logTitle = NewLabel $form 'ACTIVITY LOG' 26 812 300 18 $script:cAccent
$logTitle.Font = New-Object System.Drawing.Font('Segoe UI',8.5,

$$System.Drawing.FontStyle$$

::Bold)
$script:log=New-Object Windows.Forms.TextBox
$log.Multiline=$true; $log.ReadOnly=$true; $log.ScrollBars='Vertical'; $log.BackColor=C '#060B16'
$log.BorderStyle='FixedSingle'
$log.ForeColor=C '#CBD5E1'; $log.Font=New-Object Drawing.Font('Consolas',9)
$log.SetBounds(24,834,856,100); $log.Anchor='Top,Left,Right,Bottom'
$form.Controls.Add($log)
$footer=New-Object Windows.Forms.Label
$footer.Text='English version by TEGUH DWI iD. No Python needed, no DLLs modified; running as administrator is recommended. Ticket and time multipliers only apply to the next round.'
$footer.ForeColor=C $script:cMuted; $footer.Font=New-Object Drawing.Font('Segoe UI',8.5)
$footer.SetBounds(24,940,856,32)
$footer.Anchor='Left,Right,Bottom'; $form.Controls.Add($footer)

$script:timer=New-Object Windows.Forms.Timer
$timer.Interval=1000
$timer.Add_Tick({
try {
if (-not $script:starting) { $script:timer.Stop(); return }
$elapsed=((Get-Date)-$script:startTime).TotalSeconds
if (-not $script:serverStarted -and $elapsed -ge 3) {
$batch=Join-Path $script:runServer '!StartServer.bat'
$cmdArgs='/d /c ""' + $batch + '""'
Start-Process -FilePath $env:ComSpec -ArgumentList $cmdArgs -WorkingDirectory $script:runServer | Out-Null
$script:serverStarted=$true; $script:serverTime=Get-Date
Say 'Step 2: BF4 dedicated server started; waiting for the admin port to be ready...'
}
if ($script:serverStarted -and -not $script:gameLaunchAttempted) {
$sinceServer=((Get-Date)-$script:serverTime).TotalSeconds
if (($sinceServer -ge 9 -and (IsPortOpen $script:runPort)) -or $sinceServer -ge 55) {
if ($sinceServer -ge 55) { Say 'RCON readiness not detected; still trying to open the game. Please check the server console.' }
# Even if launching fails (for example user clicks No on UAC), only one automatic attempt is allowed.
LaunchGame $false
}
}
if ($script:gameLaunchAttempted) {
$script:botDone=$true
FinishSequence
}
if ($script:gameLaunchAttempted -and -not $script:botDone) {
$sinceServer=((Get-Date)-$script:serverTime).TotalSeconds
if ($script:runBots -eq 0) {
$script:botDone=$true; Say 'Bots set to 0; no chat command needs to be sent.'; FinishSequence
} elseif ($sinceServer -ge 600) {
$script:botDone=$true
Say ('Waited over 10 minutes for a player to join; auto-typing stopped. You can manually use !setbot ' + $script:runBots)
FinishSequence
} elseif ($script:chatSent -and (((Get-Date)-$script:chatSentAt).TotalSeconds -ge 9)) {
try {
$players=GetServerPlayers $script:runPort
if ($players -ge ($script:runBots+1)) {
Say ('The server currently has ' + $players + ' players/bots; the bot count may have been applied. Please rely on the in-game scoreboard.')
} else {
Say ('The server currently shows ' + $players + ' players; could not confirm bots were spawned. DirectX/permissions may have blocked the simulated keys, or the chat hotkey is not J.')
Say ('Please check the game chat for !setbot ' + $script:runBots + '; if necessary, type it manually once.')
}
} catch { Say ('Player count verification after sending failed: ' + $.Exception.Message) }
$script:botDone=$true; FinishSequence
} elseif ((-not $script:chatSent) -and (IsPortOpen $script:runPort) -and (((Get-Date)-$script:lastProbe).TotalSeconds -ge 4)) {
$script:lastProbe=Get-Date
try {
$players=GetServerPlayers $script:runPort
$script:probeFailures=0
if ($players -ge ($script:runBots+1)) {
Say ('Server player count has reached ' + $players + '; no need to send the bot command again.')
$script:botDone=$true; FinishSequence
} elseif ($players -lt 1) {
if (-not $script:reportedWaiting) {
Say 'Waiting for a real player to enter the game map... The bot command will not be sent while the game is not loaded.'
$script:reportedWaiting=$true
}
} else {
if ($script:firstPlayerSeen -eq 

$$DateTime$$

::MinValue) {
$script:firstPlayerSeen=Get-Date
Say ('Detected ' + $players + ' players on the server. After waiting 12 seconds and confirming the BF4 game window is in the foreground, the chat command will be typed automatically.')
}
if (((Get-Date)-$script:firstPlayerSeen).TotalSeconds -ge 12) {
if(

$$BF4ChatInput$$

::IsBF4Foreground()) {
if(-not (TrySendBotChat $script:runBots)) {
Say 'Game key input did not complete; will not auto-repeat this round to avoid accidentally triggering chat.'
$script:botDone=$true; FinishSequence
}
} elseif (-not $script:reportedWaiting) {
Say 'Waiting for the BF4 game window to come to the foreground; please switch back to the game. The launcher never sends keys to other programs.'
$script:reportedWaiting=$true
}
}
}
} catch {
$script:probeFailures++
if ($script:probeFailures -eq 1 -or $script:probeFailures -eq 5) {
Say ('Failed to detect a real player entering the game: ' + $.Exception.Message)
}
if ($script:probeFailures -ge 5) {
Say 'Server status cannot be read; auto-typing stopped (to prevent mistakes).'
$script:botDone=$true; FinishSequence
}
}
}
}
} catch { Say ('A problem occurred in the startup steps: ' + $_.Exception.Message); FinishSequence }
})
$script:manualTimer=New-Object Windows.Forms.Timer
$manualTimer.Interval=250
$manualTimer.Add_Tick({
if ($script:manualSendAt -eq 

$$DateTime$$

::MinValue) { $script:manualTimer.Stop(); return }
if ((Get-Date) -lt $script:manualSendAt) { return }
$script:manualTimer.Stop()
$script:manualSendAt=

$$DateTime$$

::MinValue
if (-not 

$$BF4ChatInput$$

::IsBF4Foreground()) {
Say 'Manual send cancelled: BF4 was not the foreground window after 10 seconds. Please switch back to the game and try again.'
return
}


$$void$$

(TrySendBotChat $script:runBots)
})
$form.Add_FormClosing({
try { SavePreferences } catch {}
$script:timer.Stop()
$script:manualTimer.Stop()
})

$script:quickSendAt=

$$DateTime$$

::MinValue
$script:quickTimer=New-Object Windows.Forms.Timer
$quickTimer.Interval=120
$quickTimer.Add_Tick({
if ($script:quickSendAt -eq 

$$DateTime$$

::MinValue) { $script:quickTimer.Stop(); return }
if ((Get-Date) -lt $script:quickSendAt) { return }
# Do not send while '=' is still held; only type into an active BF4 client.
if (

$$BF4KeyPoll$$

::Equals()) {
if ((Get-Date) -ge $script:quickExpiry) {
$script:quickTimer.Stop(); $script:quickSendAt=

$$DateTime$$

::MinValue
Say 'The = key was never released; send cancelled. Release it and try again.'
}
return
}
$script:quickSendAt=

$$DateTime$$

::MinValue
$script:quickTimer.Stop()
if (-not 

$$BF4ChatInput$$

::IsBF4Foreground()) {
Say 'Detected =, but the foreground window is not BF4, so no chat was sent.'
return
}
$script:runBots=

$$int$$

$script:botsBox.Value


$$void$$

(TrySendBotChat $script:runBots)
})
function UpdateHotkeyState {
if ($null -eq $script:hotkeyState) { return }
$script:hotkeyState.Text = 'BF4 foreground key detection (no global hotkeys used): - count=' + $script:minusCount + '  = count=' + $script:equalsCount
}
function InvokeBF4Key(

$$int$$

$id) {
$now=Get-Date
if ($id -eq 1) {
if (($now - $script:lastHotkeyMinus).TotalMilliseconds -lt 800) { return }
$script:lastHotkeyMinus=$now; $script:minusCount++
$script:form.WindowState=

$$Windows.Forms.FormWindowState$$

::Normal
$script:form.TopMost=$true
$script:form.Show()
$script:form.Activate() | Out-Null
$script:form.BringToFront()
$script:form.TopMost=$false
Say ('Detected -; trying to open the admin panel. In exclusive fullscreen, if you cannot see it, press Alt+Tab.')
} elseif ($id -eq 2) {
if (($now - $script:lastHotkeyEquals).TotalMilliseconds -lt 800) { return }
$script:lastHotkeyEquals=$now; $script:equalsCount++
if (

$$BF4ChatInput$$

::IsBF4Foreground()) {
$script:quickSendAt=$now.AddMilliseconds(450)
$script:quickExpiry=$now.AddSeconds(5)
$script:quickTimer.Start()
Say ('Detected =; after release will try to send !setbot ' + 

$$int$$

$script:botsBox.Value + '. ')
}
}
UpdateHotkeyState
}

Passive foreground-only detection. Never consumes '-' or '=' when typing outside BF4.

$script:keysTimer=New-Object Windows.Forms.Timer
$keysTimer.Interval=60
$keysTimer.Add_Tick({
try {
$gameFront=

$$BF4ChatInput$$

::IsBF4Foreground()
$minus=$gameFront -and 

$$BF4KeyPoll$$

::Minus()
$equals=$gameFront -and 

$$BF4KeyPoll$$

::Equals()
if($minus -and -not $script:priorMinus) { InvokeBF4Key 1 }
if($equals -and -not $script:priorEquals) { InvokeBF4Key 2 }
$script:priorMinus=$minus
$script:priorEquals=$equals
} catch { }
})
$script:keysTimer.Start()

LoadPreferences
Say 'English version modified by TEGUH DWI iD.'
Say 'BF4 LAN BOT v1.0.4 loaded. If you see an old-version warning, close other launcher windows and start from this folder.'
Say 'Ready: select the Server folder first. The - and = keys on the main keyboard are only detected while BF4 is in the foreground and never trigger on the Windows desktop.'
Say 'Automatic flow: Blaze -> dedicated server -> BF4Launcher; after deploying, press = to set bots.'
$form.Add_Shown({
UpdateHotkeyState
Say 'Hotkeys enabled: in game, press - to show the panel; press = to apply bots. If the panel is not visible in exclusive fullscreen, use Alt+Tab.'
})


$$void$$

$$System.Windows.Forms.Application$$

::Run($form)


[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/j0p3s)