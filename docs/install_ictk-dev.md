## Install ICTK dev
```bash
curl -fsSL https://1Kaustubh122.github.io/industrial-control-toolkit/KEY.gpg \
  | gpg --dearmor | sudo tee /usr/share/keyrings/ictk.gpg >/dev/null

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/ictk.gpg] https://1Kaustubh122.github.io/industrial-control-toolkit jammy main" \
  | sudo tee /etc/apt/sources.list.d/ictk.list

sudo apt update && sudo apt install ictk-dev
````
