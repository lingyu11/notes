# Python Misc

## Mircosoft Python Tutorial

### 资料
* 视频：✅[初级](https://www.bilibili.com/video/BV1nE41127zQ?spm_id_from=333.788.videopod.episodes&vd_source=2a33d03ec3e67e46971208a7faa0dcda)，📌[进阶](https://www.bilibili.com/video/BV1WT4y137cD/?spm_id_from=333.788.video.desc.click&vd_source=2a33d03ec3e67e46971208a7faa0dcda)，📌[高阶](https://www.bilibili.com/video/BV1qa4y1Y7CD/?spm_id_from=333.788.video.desc.click&vd_source=2a33d03ec3e67e46971208a7faa0dcda)
* 代码：[Git repo](https://github.com/microsoft/c9-python-getting-started)

### Installing packages
1. `pip install colorama` or `pip install -r requirements.txt`

2. Virtual environment
    * By default, packages are installed globally
        - Version manangement becomes a challenge
    * Virtual environments can be used to contain and manage package collections
        - Really just a folder behind the scenes with all your packages

3. Creating a virtual environment:
    * `pip install virtualenv` # install virtual environment
    * Windows system
        * `python -m venv <folder_name>`
    * Linux system
        * `virtualenv <folder_name>`

4. Using virtual environment:
    * Windows system
        * cmd.exe: `<folder_name>\Scripts\activate.bat`
        * Powershell: `<folder_name>\Scripts\activate.ps1`
        * bash shell: `. ./<folder_name>/Scripts/activate`
    * Linux system
        * `<folder_name>/bin/activate`
    
5. Decorators:
    * Adjectives
    * Add additional functionality to code


## Python 100 Days

### 资料
* 代码：[jackfrued python 100 days](https://github.com/jackfrued/Python-100-Days)
* 视频：[jackfrued](https://www.bilibili.com/video/BV13t4y1a7TV?spm_id_from=333.788.videopod.sections&vd_source=2a33d03ec3e67e46971208a7faa0dcda)

### 安装
1. Windows下常用命令
    * `python --version`
    * `pip --version`
    * `python -m pip install -U --user pip` # 更新 pip
    * `pip config set global.index-usr https://pypi.doubanio.com/simple` # 换国内源
    * `pip install jupyter` # 安装 jupyter
    * `pip install -U jupyter` # 升级 jupyter
    * `jupyter notebook` # 启动 jupyter
    * `pip install pyzmq==20.0.0` # 安装指定版本的 pyzmq

