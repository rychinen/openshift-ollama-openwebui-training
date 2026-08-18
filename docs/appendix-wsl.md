# Windows Server 2019 + WSL

WSL 内の Linux 用 `oc`、curl、jq、openssl を使用し、Windows の oc.exe と混在させません。

```powershell
wsl --status
wsl -l -v
wsl
```

```bash
sudo apt-get update && sudo apt-get install -y curl jq openssl
oc version --client
```