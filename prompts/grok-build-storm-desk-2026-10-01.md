
You are Grok Build in PowerShell on stp's Windows desk. This desk already mines Kaspa **Testnet-10 (TN10)** toward stp's
node (tn10-n0) through the bore tunnel. A large TN10 storm is planned. Your job: **watch the storm from this desk, keep
the desk miners healthy at about 75% CPU, check that stp's tx ids land, and write one log file stp can hand back.**
You do NOT send transactions and you do NOT drive the load. Load only comes from stp's node box.

## Hard rules (break none of them; if a step would, stop and ask stp)
1. **TN10 only. Never mainnet.** Only `kaspatest:` addresses, always `--testnet` on kaspa-miner, only `api-tn10.kaspa.org`.
   If anything shows a `kaspa:` (mainnet) address, mainnet port, or mainnet API, refuse and tell stp.
2. **No public posts, comments, issues, PRs, tweets or messages** of any kind. No GitHub writes. Read-only everywhere.
3. **Public APIs: read-only, at most one call every 5 minutes, and every call logged** with a timestamp in
   `public-api-calls.log`. The script enforces a 300 s floor. Do not add other public calls.
4. **Every claim needs evidence:** the command, its output, a timestamp, and the tx id where relevant. The script's JSONL log
   does this. Do not report numbers that are not in the log.
5. **Never ask for, type, paste or store a mnemonic, private key or wallet password.** Mining needs only the public address.
6. Payout address is fixed: `[address]`. Do not change it.
7. Only stop `kaspa-miner` processes that this desk started. Never kill other programs. Do not disable antivirus; if
   Defender quarantines the miner, stop and tell stp.
8. If the endpoint stops answering, **do not scan or guess [tunnel] ports.** Ask stp for the current port (stp reads it from
   `[local path]` on the box). Last known: `[ip]` (2026-09-27).

## Step 1: check prerequisites (read-only)
```powershell
$PSVersionTable.PSVersion
Test-Path $HOME\kaspa-tn10\kaspa-miner.exe
Get-Process kaspa-miner -ErrorAction SilentlyContinue | Select-Object Id,CPU,StartTime
Test-NetConnection [ip] -Port 3567 | Select-Object TcpTestSucceeded
```
If `kaspa-miner.exe` is missing, install **only** the official kaspanet/cpuminer **v0.2.7** release
(`kaspa-miner-v0.2.7-win64-amd64.exe`, expected SHA256 `[hash]`) into
`$HOME\kaspa-tn10\kaspa-miner.exe`, and check the hash with `Get-FileHash`. Do not use v0.2.5 or older.

**Existing miners:** if about 20 miners already run here (stp's current setup), leave them alone. The script counts all
`kaspa-miner` processes towards the CPU target. It only adds a miner below about 67% CPU and removes its newest one above
about 83%.

## Step 2: save the script
Save the script below as `$HOME\kaspa-tn10\storm\desk-storm-observer.ps1` (create the folder first):
```powershell
New-Item -ItemType Directory -Force $HOME\kaspa-tn10\storm | Out-Null
notepad $HOME\kaspa-tn10\storm\desk-storm-observer.ps1   # paste the script, save, close
```

```powershell
#Requires -Version 5.1
<#
  desk-storm-observer.ps1 — stp's Windows desk, Kaspa TESTNET-10 (TN10) storm observer + miner keeper.
  TN10 ONLY. NEVER MAINNET. Sends NO transactions. Posts NOTHING anywhere. Public API is READ-ONLY, <= every 5 min, logged.
  What it does, every loop:
    1. Keeps the desk kaspa-miner processes healthy at ~75% total CPU (adds/removes 1-thread miners; never kills foreign procs).
    2. Every 5 min: ONE cache-busted GET api-tn10 /info/health (acceptedTxBlockTimeDiff, blueScoreDiff) -> did reporting survive?
    3. Every 5 min: if tx ids are queued in txids-to-check.txt, ONE POST /transactions/acceptance for up to 50 of them.
    4. Measures desk-vantage timing: TCP connect latency to the mining endpoint, and miner "Block submitted" rate.
    5. Writes everything to a JSONL log + a human summary that stp hands back. Every line: timestamp, command, result.
  Usage (PowerShell, as the normal user, NOT admin):
    powershell -ExecutionPolicy Bypass -File .\desk-storm-observer.ps1 -DryRun          # one pass, changes nothing
    powershell -ExecutionPolicy Bypass -File .\desk-storm-observer.ps1 -Hours 24        # observe for 24 h
  Stop: Ctrl+C, or create the file  $HOME\kaspa-tn10\storm\STOP
#>
param(
  [double]$Hours = 24,
  [int]$TargetCpuPct = 75,
  [int]$MaxMiners = 24,
  [int]$ApiIntervalSec = 300,          # HARD FLOOR 300: public reads <= every 5 min
  [int]$LoopSec = 30,
  [string]$EndpointIp = "[ip]",
  [int]$EndpointPort = 3567,
  [string]$MinerExe = "$HOME\kaspa-tn10\kaspa-miner.exe",
  [switch]$DryRun
)
$ErrorActionPreference = "Continue"
# ---------------- HARD RULES (do not edit) ----------------
$PayoutAddr = "[address]"   # stp's TN10 pool address
$ApiBase    = "https://api-tn10.kaspa.org"                                                  # TN10 public API (read-only)
if (-not $PayoutAddr.StartsWith("kaspatest:")) { throw "REFUSE: payout is not a TN10 kaspatest: address" }
if ($ApiBase -notmatch "api-tn10\.") { throw "REFUSE: API base is not TN10" }
if ($ApiIntervalSec -lt 300) { Write-Warning "ApiIntervalSec raised to 300 (rule: public reads <= every 5 min)"; $ApiIntervalSec = 300 }
# ----------------------------------------------------------
$Dir = Join-Path $HOME "kaspa-tn10\storm"; New-Item -ItemType Directory -Force $Dir | Out-Null
$Stamp = Get-Date -Format "yyyyMMdd-HHmmss"
$Log = Join-Path $Dir "desk-storm-$Stamp.jsonl"; $ApiLog = Join-Path $Dir "public-api-calls.log"
$Summary = Join-Path $Dir "desk-storm-$Stamp-summary.md"; $TxFile = Join-Path $Dir "txids-to-check.txt"
$StopFile = Join-Path $Dir "STOP"; $MinerLogDir = Join-Path $Dir "miner-logs"; New-Item -ItemType Directory -Force $MinerLogDir | Out-Null
if (-not (Test-Path $TxFile)) { New-Item -ItemType File $TxFile | Out-Null }

function Now { (Get-Date).ToString("yyyy-MM-ddTHH:mm:ss.fffzzz") }
function W($obj) { $obj["t"] = Now; ($obj | ConvertTo-Json -Compress -Depth 6) | Add-Content -Path $Log -Encoding UTF8 }
function ApiLog($method, $url) { "[$(Now)] $method $url" | Add-Content -Path $ApiLog -Encoding UTF8 }

function Get-CpuPct {
  try { [math]::Round((Get-CimInstance Win32_Processor | Measure-Object -Property LoadPercentage -Average).Average, 0) } catch { -1 }
}
function Get-DeskMiners { @(Get-Process -Name "kaspa-miner" -ErrorAction SilentlyContinue) }
function Start-DeskMiner([int]$n) {
  $args1 = @("--testnet","-s",$EndpointIp,"-p",$EndpointPort,"-t","1","--mining-address",$PayoutAddr,"--user-agent-suffix","stp-desk")
  $out = Join-Path $MinerLogDir "miner-$n-$Stamp.log"
  if ($DryRun) { W @{ev="dryrun_would_start_miner"; cmd="$MinerExe $($args1 -join ' ')"; log=$out}; return }
  $p = Start-Process -FilePath $MinerExe -ArgumentList $args1 -RedirectStandardOutput $out -RedirectStandardError "$out.err" -WindowStyle Hidden -PassThru
  try { $p.PriorityClass = "BelowNormal" } catch {}
  W @{ev="miner_started"; pid=$p.Id; cmd="$MinerExe $($args1 -join ' ')"}
}
function Stop-OneDeskMiner {
  $m = Get-DeskMiners | Sort-Object StartTime -Descending | Select-Object -First 1
  if ($null -eq $m) { return }
  if ($DryRun) { W @{ev="dryrun_would_stop_miner"; pid=$m.Id}; return }
  Stop-Process -Id $m.Id -ErrorAction SilentlyContinue; W @{ev="miner_stopped"; pid=$m.Id; reason="cpu above target"}
}
function Test-Endpoint {
  $sw = [Diagnostics.Stopwatch]::StartNew(); $ok = $false
  try { $c = New-Object Net.Sockets.TcpClient; $iar = $c.BeginConnect($EndpointIp, $EndpointPort, $null, $null)
        $ok = $iar.AsyncWaitHandle.WaitOne(5000) -and $c.Connected; $c.Close() } catch {}
  @{ ok=$ok; ms=$sw.ElapsedMilliseconds }
}
function Count-Submitted {
  $n = 0; Get-ChildItem $MinerLogDir -Filter "*.log" -ErrorAction SilentlyContinue | ForEach-Object {
    $n += @(Select-String -Path $_.FullName -Pattern "Block submitted successfully" -SimpleMatch -ErrorAction SilentlyContinue).Count }
  $n
}
function Get-Health {
  $url = "$ApiBase/info/health?cb=$([DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds())"; ApiLog "GET" $url
  $sw = [Diagnostics.Stopwatch]::StartNew()
  try { $r = Invoke-WebRequest -Uri $url -UseBasicParsing -TimeoutSec 20 -Headers @{ "Cache-Control"="no-cache" }; $code = [int]$r.StatusCode; $body = $r.Content }
  catch { $code = -1; $body = $null; if ($_.Exception.Response) { $code = [int]$_.Exception.Response.StatusCode
            try { $sr = New-Object IO.StreamReader($_.Exception.Response.GetResponseStream()); $body = $sr.ReadToEnd() } catch {} } }
  $rec = @{ ev="api_health"; cmd="GET $url"; http=$code; ms=$sw.ElapsedMilliseconds }
  if ($body) { try { $j = $body | ConvertFrom-Json; $db = $j.database
      $rec.acceptedTxBlockTimeDiff = $db.acceptedTxBlockTimeDiff; $rec.blueScoreDiff = $db.blueScoreDiff; $rec.db_isSynced = $db.isSynced
      $rec.db_blueScore = $db.blueScore; $rec.stall_suspected = ($db.acceptedTxBlockTimeDiff -gt 120 -and $db.blueScoreDiff -lt 200) } catch { $rec.parse_err = "$_" } }
  W $rec; $rec
}
function Check-TxIds {
  $ids = @(Get-Content $TxFile -ErrorAction SilentlyContinue | ForEach-Object { $_.Trim().ToLower() } | Where-Object { $_ -match '^[0-9a-f]{64}$' } | Select-Object -First 50)
  if ($ids.Count -eq 0) { return }
  $url = "$ApiBase/transactions/acceptance"; ApiLog "POST" "$url (n=$($ids.Count))"
  try { $body = @{ transactionIds = $ids } | ConvertTo-Json -Compress
        $r = Invoke-RestMethod -Method Post -Uri $url -Body $body -ContentType "application/json" -TimeoutSec 30
        foreach ($x in @($r)) { W @{ ev="tx_acceptance"; cmd="POST $url"; txid=$x.transactionId; accepted=$x.accepted; acceptingBlueScore=$x.acceptingBlueScore; acceptingTimestamp=$x.acceptingTimestamp } }
        $done = @($r | Where-Object { $_.accepted } | ForEach-Object { $_.transactionId })
        $left = @(Get-Content $TxFile | Where-Object { $done -notcontains $_.Trim().ToLower() }); Set-Content -Path $TxFile -Value $left
  } catch { W @{ ev="tx_acceptance_err"; cmd="POST $url"; err="$_" } }
}

# ---------------- preflight ----------------
W @{ ev="start"; host=$env:COMPUTERNAME; hours=$Hours; targetCpu=$TargetCpuPct; endpoint="${EndpointIp}:$EndpointPort"; dryRun=[bool]$DryRun; payout=$PayoutAddr; ps=$PSVersionTable.PSVersion.ToString() }
if (-not (Test-Path $MinerExe)) { W @{ ev="warn"; msg="kaspa-miner.exe not found at $MinerExe (install kaspanet/cpuminer v0.2.7, official release only)" } }
$cores = [Environment]::ProcessorCount
$deadline = (Get-Date).AddHours($Hours); $lastApi = [datetime]::MinValue; $sub0 = Count-Submitted; $t0 = Get-Date
$healthSeen = @()
do {
  if (Test-Path $StopFile) { W @{ ev="stop_file"; path=$StopFile }; break }
  $cpu = Get-CpuPct; $miners = Get-DeskMiners; $ep = Test-Endpoint; $sub = Count-Submitted
  W @{ ev="tick"; cpu_pct=$cpu; miners=$miners.Count; cores=$cores; endpoint_ok=$ep.ok; endpoint_connect_ms=$ep.ms; blocks_submitted_total=$sub; blocks_submitted_since_start=($sub-$sub0) }
  # --- keep miners near target CPU (one change per loop, hysteresis +-8%) ---
  if ($ep.ok -and $cpu -ge 0) {
    if ($cpu -lt ($TargetCpuPct - 8) -and $miners.Count -lt [math]::Min($MaxMiners, $cores)) { Start-DeskMiner ($miners.Count + 1) }
    elseif ($cpu -gt ($TargetCpuPct + 8) -and $miners.Count -gt 0) { Stop-OneDeskMiner }
  } elseif (-not $ep.ok) { W @{ ev="endpoint_down"; msg="bore port may have changed; ask stp for the port in mining-endpoint.txt; do NOT scan ports" } }
  # --- public API, <= every 5 min ---
  if (((Get-Date) - $lastApi).TotalSeconds -ge $ApiIntervalSec) { $h = Get-Health; $healthSeen += ,$h; Check-TxIds; $lastApi = Get-Date }
  if ($DryRun) { break }
  Start-Sleep -Seconds $LoopSec
} while ((Get-Date) -lt $deadline)

# ---------------- summary for stp ----------------
$el = [math]::Round(((Get-Date) - $t0).TotalMinutes, 1); $subN = (Count-Submitted) - $sub0
$stalls = @($healthSeen | Where-Object { $_.stall_suspected }).Count
$acc = @(Get-Content $Log | Where-Object { $_ -match '"ev":"tx_acceptance"' }).Count
@"
# Desk storm observer summary ($env:COMPUTERNAME)
- Window: $($t0.ToString('yyyy-MM-dd HH:mm:ss zzz')) to $(Now)  ($el min). DryRun=$([bool]$DryRun). TN10 only.
- Desk blocks submitted during window: $subN (from miner logs, "Block submitted successfully").
- api-tn10 /info/health samples: $($healthSeen.Count); stall_suspected: $stalls  (stall = acceptedTxBlockTimeDiff>120 s while blueScoreDiff<200).
- Tx acceptance lookups logged: $acc  (see JSONL lines ev=tx_acceptance; each has txid + acceptingBlueScore + timestamp).
- Public API calls (all logged): $ApiLog
- Full log (every claim = command + output + timestamp): $Log
Hand back BOTH files to stp. Nothing was posted anywhere; no transactions were sent.
"@ | Set-Content -Path $Summary -Encoding UTF8
W @{ ev="end"; minutes=$el; desk_blocks=$subN; summary=$Summary }
Write-Host "Done. Log: $Log`nSummary: $Summary`nAPI log: $ApiLog"
```

## Step 3: dry run first (changes nothing, at most 1 public call)
```powershell
cd $HOME\kaspa-tn10\storm
powershell -ExecutionPolicy Bypass -File .\desk-storm-observer.ps1 -DryRun
Get-Content (Get-ChildItem desk-storm-*.jsonl | Sort-Object LastWriteTime | Select-Object -Last 1)
Get-Content .\public-api-calls.log
```
Show stp the output. Expected lines: `start`, `tick` (cpu_pct, miners, endpoint_ok, endpoint_connect_ms), `api_health`
(http 200, acceptedTxBlockTimeDiff, blueScoreDiff), `end`. A `dryrun_would_start_miner` / `dryrun_would_stop_miner` line
means it would have changed miner count.

## Step 4: when stp says the storm starts, run it for the window
```powershell
cd $HOME\kaspa-tn10\storm
powershell -ExecutionPolicy Bypass -File .\desk-storm-observer.ps1 -Hours 24 -TargetCpuPct 75
```
Leave the window open. To stop early: Ctrl+C, or `New-Item $HOME\kaspa-tn10\storm\STOP`.

## Step 5: confirm stp's tx ids land
stp gives you tx ids (64 hex chars) from the storm runner. Append them, one per line, to
`$HOME\kaspa-tn10\storm\txids-to-check.txt`. On each 5-minute API tick the script sends **one**
`POST /transactions/acceptance` for up to 50 ids, logs `txid`, `accepted`, `acceptingBlueScore`, `acceptingTimestamp`, and
removes the accepted ones from the file. If ids stay unaccepted for a long time, report that as a finding (with the log
lines). Do not query ids one by one.

## Step 6: what to watch and report (desk vantage)
- **Did reporting survive?** Each `api_health` line: `acceptedTxBlockTimeDiff` should stay small (seconds).
  `stall_suspected:true` (acceptedTxBlockTimeDiff > 120 s while blueScoreDiff < 200) is the 25-Sep stall signature.
  Report the **first** timestamp it appears, and whether it recovers.
- **Endpoint timing:** `endpoint_connect_ms` over time, and any `endpoint_down`.
- **Desk mining health:** `cpu_pct` near 75, `miners`, `blocks_submitted_since_start` still rising. If submits stop for
  more than 10 minutes while the endpoint is up, report it with log lines.

## Step 7: hand back
At the end the script prints three paths. Give stp **all three**, unchanged:
`desk-storm-<stamp>.jsonl`, `desk-storm-<stamp>-summary.md`, `public-api-calls.log`.
In your own reply, every number must quote the JSONL line it comes from (timestamp + command + value, plus the tx id where
relevant). Mark anything you could not verify as **unverified**. Do not post anything anywhere.

