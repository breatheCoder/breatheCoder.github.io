由于obsidian开git插件定时扫描笔记同步到github和gitee上会让obsidian非常的卡

我就寻思能蹦呢像maven-search一样去定时同步, 结果发现windows就有这个功能.

所以我就写了一个脚本, 去让每周日十点半自动同步我的笔记到我的git仓库.

```bash
@echo off
setlocal

set "REPO_PATH=D:\note"

powershell -command "(Get-Date -f 'yyyy-MM-dd')" > date_tmp.txt
FOR /F "usebackq" %%i IN (date_tmp.txt) DO SET "CURRENT_DATE=%%i"
del date_tmp.txt

echo -----------------------------------
echo Starting Git auto push at %DATE% %TIME%
echo Repository path: "%REPO_PATH%"
echo Commit message will be: "%CURRENT_DATE%"

cd /d "%REPO_PATH%"
if %errorlevel% neq 0 (
    echo Error: Could not change to directory "%REPO_PATH%". Exiting.
    exit /b 1
)

git add .

set "CHANGES_DETECTED=false"
git status --porcelain | findstr /R ".*" >nul 2>&1

if %errorlevel% equ 0 (
    set "CHANGES_DETECTED=true"
)

if "%CHANGES_DETECTED%"=="false" (
    echo No changes detected. Skipping commit and push.
) else (
    git commit -m "%CURRENT_DATE%"
    if %errorlevel% neq 0 (
        echo Error: Git commit failed. Exiting.
        exit /b 1
    )
    echo Successfully committed with message: '%CURRENT_DATE%'

    git push gitee breathe


    git push github breathe

    
    echo Successfully pushed to remote.
)

echo Git auto push finished.
echo -----------------------------------

endlocal

```
