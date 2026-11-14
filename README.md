# Fix: "An Application Control policy has blocked this file" when running C++ on Windows 11

If you compile a C++ program on Windows 11 and Windows refuses to run the `.exe`, this guide shows two ways to get past it. It also covers setting up WSL (Linux inside Windows) so you can keep using VS Code, plus a bonus section on setting up Python (virtual environment, pandas, matplotlib and more) in WSL.

> **Note on how this was written:** I'm a first-year IT engineering student and a beginner with this. I got stuck on this error, worked through it with an AI assistant (Claude), and followed the steps on my own laptop. The explanation of the cause comes from that conversation, so I checked it against Microsoft's documentation: [Smart App Control overview on Microsoft Learn](https://learn.microsoft.com/windows/apps/develop/smart-app-control/overview). The steps below are what I actually did and what actually happened.

## The symptom

Compiling works, but running the program fails with an error like this in the VS Code PowerShell terminal:

```
Program 'main.exe' failed to run: An Application Control policy has blocked this file
...
CategoryInfo          : ResourceUnavailable: (:) [], ApplicationFailedException
FullyQualifiedErrorId : NativeCommandFailed
```

A Windows Security notification may also appear, saying part of the app was blocked because Windows can't confirm who published `main.exe`.

**Setup where this happened:** Windows 11, VS Code, `g++`, with output compiled to `$env:TEMP\main.exe`.

## Why it happens

Windows has a feature called **Smart App Control** (part of Windows' application control features). It blocks programs that are unsigned or have no reputation. A `.exe` you just compiled yourself is both, so Windows treats it as untrusted, even though you wrote it.

The compile step succeeds. Only *running* the result is blocked.

> In my case I did not open the Smart App Control settings to confirm it was on before choosing WSL. To check yours: **Windows Security → App & browser control → Smart App Control settings**. [UPDATE THIS LINE IF YOU CHECK.]

## Fix 1: Turn off Smart App Control

1. Open **Windows Security → App & browser control → Smart App Control settings**.
2. Set it to **Off**.

**Trade-offs:**
- On older Windows 11 builds, once it's off you can't turn it back on without resetting or reinstalling Windows. Newer builds (24H2 and 25H2 updates from build 26100.8116 / 26200.8116 onwards, as reported in Microsoft's release notes) let you switch it on and off without a reinstall, but some Microsoft pages still describe the old behaviour. **Check your own build** (run `winver`) and read Microsoft's documentation: [Smart App Control overview](https://learn.microsoft.com/windows/apps/develop/smart-app-control/overview).
- You lose a layer of protection. Microsoft Defender stays on, but you should avoid cracked software and unknown `.exe` files.
- If your laptop is managed by a college or employer, the setting may be greyed out.

## Fix 2: Use WSL (what I chose)

I chose this because I didn't want to turn off a security feature, and because Linux is useful for the AI/ML work I want to do. Programs compiled inside Linux are Linux binaries, so Windows' `.exe` check doesn't apply to them.

### Step 1: Install WSL

Open **PowerShell as administrator** and run:

```powershell
wsl --install
```

Restart when asked. (Windows installed updates during my restart, which took a while. Don't power off during that.)

**If Ubuntu doesn't open by itself after restarting**, run this in PowerShell:

```powershell
wsl --install -d Ubuntu
```

### Step 2: Create your Linux user

Ubuntu asks for a username and password.

- **When you type the password, nothing appears on screen, not even dots.** That's normal. It is still being typed. I thought my keyboard was broken at first.
- It asks about sharing anonymous usage data with Canonical. This is optional, and either answer is fine.

When you see a green prompt like `atharv@LAPTOP-NAME:~$`, you're inside Ubuntu.

### Step 3: Install the C++ compiler

```bash
sudo apt update
sudo apt install build-essential -y
g++ --version
```

### Step 4: Go to your project and open VS Code

Your Windows files are visible in Linux under `/mnt/c/`:

```bash
cd /mnt/c/Users/<YourName>/Documents/<YourProjectFolder>
ls
code .
```

- Linux is case-sensitive, so `Documents` and `documents` are different.
- Use `ls` to check you're in the right folder. If you get "No such file or directory", you're in the wrong one.
- Install the **WSL** extension in VS Code if it asks. The bottom-left corner should then say **WSL: Ubuntu**.

### Step 5: Compile and run

In the VS Code terminal (it should now be a Linux terminal):

```bash
g++ main.cpp -o main && ./main
```

There's no `.exe` on Linux. `./main` means "run the file named `main` in this folder."

## Bonus: Python with pandas, matplotlib and more in WSL

Once WSL works, Python works too, but Ubuntu handles it differently from Windows.

### 1. The command is `python3`, not `python`

My first "python not found" error happened because Ubuntu uses `python3`. Check yours:

```bash
python3 --version
```

Optional: make `python` work as well.

```bash
sudo apt install python-is-python3 -y
```

### 2. Install the tools for virtual environments

```bash
sudo apt update
sudo apt install python3-venv python3-pip -y
```

### 3. Create and activate a virtual environment

Ubuntu blocks `pip install` outside a virtual environment (you'll see `externally-managed-environment`). A virtual environment (venv) is a private folder of packages for your Python work.

```bash
mkdir -p ~/projects/myproject
cd ~/projects/myproject
python3 -m venv venv
source venv/bin/activate
```

When it's active, the prompt starts with `(venv)`.

### 4. Install packages

With `(venv)` showing:

```bash
pip install numpy pandas matplotlib opencv-python yfinance requests beautifulsoup4 plyer pillow
```

Names that differ from what you import:

| In your code | pip install |
|---|---|
| `import cv2` | `opencv-python` |
| `from bs4 import BeautifulSoup` | `beautifulsoup4` |
| `from PIL import Image` | `pillow` |
| `import turtle` | nothing (built in, but needs `python3-tk`) |

Extra system packages that some of these need on Ubuntu:

```bash
sudo apt install python3-tk libgl1 libglib2.0-0 libnotify-bin -y
```

### 5. Run a script

```bash
python test.py
```

If you get `ModuleNotFoundError: No module named 'pandas'`, check that `(venv)` is in your prompt. **The venv belongs to a terminal session, not to a folder or file.** You activate it once per terminal, and then any script in any folder can use the packages.

### 6. Activate it automatically in every new terminal

I wanted to reuse one venv everywhere, so I added this line to my shell startup file (run it once only):

```bash
echo 'source ~/projects/myproject/venv/bin/activate' >> ~/.bashrc
```

Open a new terminal and `(venv)` appears by itself. This doesn't affect C++, because `g++` ignores it. Don't move or delete the venv folder afterwards, because the path is hardcoded. To undo it, delete that last line from `~/.bashrc`.

### 7. VS Code Run button for Python

1. Install the **Python** extension, using the "Install in WSL: Ubuntu" button.
2. Press `Ctrl+Shift+P` → **Python: Select Interpreter**.
3. Choose `~/projects/myproject/venv/bin/python`.

### 8. Matplotlib plot didn't appear

My script ran with no errors, but the plot window never opened. Installing the Tk package fixed it:

```bash
sudo apt install python3-tk -y
```

Then rerun the script (it needs `plt.show()` at the end). If a window still won't open, `plt.savefig("plot.png")` saves the plot as an image you can open in VS Code. If no window opens at all, update WSL with `wsl --update`, then `wsl --shutdown`, and reopen Ubuntu.

### Python errors I hit

| Error | Cause and fix |
|---|---|
| `python: command not found` | Use `python3` |
| `No module named 'pandas'` | Venv not active: `source venv/bin/activate` |
| `can't open file ... No such file or directory` | Wrong folder, so use `ls` and `cd` to the folder that holds the file (quote paths that contain spaces) |
| Plot doesn't appear | Install `python3-tk` |

### Known limits (not tested by me)

- **plyer notifications** use Linux notifications, so they may not show as Windows popups from inside WSL.
- **Webcam access** with OpenCV (`cv2.VideoCapture(0)`) usually doesn't work from WSL by default. Reading image and video files works. [TEST AND UPDATE IF YOU TRY IT.]

## Things that confused me as a beginner

- **How to tell which terminal I'm in.** A green `user@computer:~$` prompt means Linux. `PS C:\...>` means Windows PowerShell.
- **The VS Code Run button** may still use Windows commands. The terminal command above is the reliable way.
- **Extensions installed on Windows don't automatically work inside WSL.** VS Code shows an "Install in WSL: Ubuntu" button.

## Summary

| Option | Effort | Downside |
|---|---|---|
| Turn off Smart App Control | Quick | Removes a security layer; on older builds it can't be turned back on without a reset |
| WSL | One-time setup | Learning a few Linux commands |

## Sources

- [Smart App Control overview (Microsoft Learn)](https://learn.microsoft.com/windows/apps/develop/smart-app-control/overview)
- [How to install Linux on Windows with WSL (Microsoft Learn)](https://learn.microsoft.com/windows/wsl/install)
- [Set up a WSL development environment (Microsoft Learn)](https://learn.microsoft.com/windows/wsl/setup/environment)

## Corrections welcome

I'm still learning. If something here is wrong or out of date for newer Windows versions, please open an issue or pull request.
