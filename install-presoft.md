
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
	software-properties-common
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
echo "========== DONE ==========="

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

cd ~
git clone https://github.com/Morrowmake/qwen38-flash-next-cmp170hx-recipe
cd qwen38-flash-next-cmp170hx-recipe
./start.sh 
