在Linux上通过Ollama部署DeepSeek模型可按以下步骤进行：

### 安装Ollama
在终端中运行以下命令：
```
curl -fsSL https://ollama.com/install.sh | sh
```
安装完成后，通过运行下面命令验证安装是否成功：
```
ollama --version
```

### 下载DeepSeek模型
根据硬件配置和需求，选择合适的DeepSeek模型版本，如`1.5B`、`7B`、`14B`、`70B`等。然后在终端中运行下载命令，以`7B`版本为例：
```
ollama run deepseek-r1:7b
```
也可以先使用`ollama pull deepseek-r1:7b`命令将模型拉取到本地，再运行`ollama run deepseek-r1:7b`启动模型。

### 测试模型
模型部署完成后，可以在终端中直接与模型进行交互。也可以搭配Open-WebUI等工具增强交互体验。若选择安装Open-WebUI，可参考以下步骤：
- **使用Docker安装**
    - 确保已安装Docker，若未安装，需先安装Docker。
    - 拉取Open-WebUI的Docker镜像：
```
docker pull ghcr.io/open-webui/open-webui:main
```
    - 运行Open-WebUI容器：
```
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```
    - 打开浏览器，访问`http://localhost:3000`，进入Open-WebUI界面，在设置中配置Ollama地址（默认为`http://localhost:11434`），即可选择DeepSeek模型进行对话。
- **直接安装**
    - 安装Python及虚拟环境管理包：
```
sudo apt install python3-venv -y
```
    - 创建虚拟环境：
```
python3 -m venv ~/open-webui-venv
```
    - 激活虚拟环境：
```
source ~/open-webui-venv/bin/activate
```
    - 安装Open-WebUI：
```
pip install open-webui
```
    - 运行Open-WebUI：
```
open-webui serve
```
    - 打开浏览器，访问`http://localhost:8080`，即可使用Open-WebUI与DeepSeek模型交互。
