```
echo "========== check nvidia ==========="
lspci -nn | grep -iE 'NVIDIA|10de:20b0|10de:20c2|10de:2082'
echo "========== install soft ==========="
apt update
sudo apt install -y \
    build-essential \
    dkms \
    linux-headers-$(uname -r) \
    python3 \
    python3-yaml \
    git \
    curl \
    patch \
    pciutils \
    mokutil \
    ca-certificates \
    gnupg \
    tar \
    xz-utils \
    software-properties-common \
    jq \
    util-linux

echo "========== install docker ==========="
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

echo "========== install drivers nvidia ==========="
sudo add-apt-repository -y multiverse
sudo apt install -y nvidia-driver-615-open

echo "========== install CUDA nvidia ==========="
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.20.1-1
sudo apt install -y nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

echo "========== verification ==========="
[ -f /etc/os-release ] && echo "OS:               $(. /etc/os-release && echo "$PRETTY_NAME")"
echo "Kernel Version:   $(uname -r)"

echo -e "\n--- Software Versions ---"
command -v docker >/dev/null 2>&1 && echo "Docker Version:   $(docker --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -n 1)" || echo "Docker Version:   Not installed"
command -v nvidia-ctk >/dev/null 2>&1 && echo "NVIDIA Toolkit:   $(nvidia-ctk --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -n 1)" || echo "NVIDIA Toolkit:   Not installed"

{ command -v modinfo >/dev/null 2>&1 && modinfo nvidia >/dev/null 2>&1 && echo "NVIDIA Driver:    $(modinfo nvidia | grep ^version: | head -n 1 | awk '{print $2}')"; } || \
{ command -v nvidia-smi >/dev/null 2>&1 && echo "NVIDIA Driver:    $(nvidia-smi --query-gpu=driver_version --format=csv,noheader | head -n 1)"; } || \
echo "NVIDIA Driver:    Not loaded/installed"
sudo docker run --rm --gpus all ubuntu:24.04 nvidia-smi
echo "========== DONE NEED REBOOT ==========="

echo "========== Install cmpunlocker ==========="
cd ~
#git clone https://github.com/amoghmunikote/cmpunlocker.git
#cd cmpunlocker
#sudo ./install.sh --no-iommu --no-passthrough
#sudo shutdown -h now
#cd ~/cmpunlocker
#sudo ./verify.sh

nvidia-smi \
  --query-gpu=name,memory.total,pci.bus_id,pcie.link.gen.current,pcie.link.gen.max,pcie.link.width.current,pcie.link.width.max \
  --format=csv

echo "========== Install qwen38-flash-next-cmp170hx-recipe ==========="
cd ~
git clone https://github.com/Morrowmake/qwen38-flash-next-cmp170hx-recipe
cd qwen38-flash-next-cmp170hx-recipe
./start.sh

```
