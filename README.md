# Tutorial: Instalação de Drivers NVIDIA no Kubuntu 25.04 para Data Science

**Criação:** Modelo LLM Opus 4.6 
**Testado em:** Dell Precision 7670 (Intel iGPU + NVIDIA RTX A3000 12GB)
**Sistema:** Kubuntu 25.04 (dual boot com Windows 11)
**Objetivo:** GPU híbrida (Intel para display, NVIDIA para computação CUDA)

---

## Links de Documentação Oficial (salve estes)

1. **Ubuntu — Instalação de drivers NVIDIA (Canonical)**
   https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/

2. **NVIDIA — Driver Installation Guide for Ubuntu**
   https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/ubuntu.html

3. **NVIDIA — CUDA Installation Guide for Linux**
   https://docs.nvidia.com/cuda/cuda-installation-guide-linux/

4. **PyTorch — Instalação oficial**
   https://pytorch.org/get-started/locally/

5. **Arch Wiki — PRIME (referência técnica sobre GPU híbrida)**
   https://wiki.archlinux.org/title/PRIME

6. **Ubuntu Wiki — NVIDIA Drivers Installation (community)**
   https://help.ubuntu.com/community/NvidiaDriversInstallation

---

## PARTE 1 — Preparação do sistema

### 1.1 Atualize tudo antes de começar

Abra o terminal (Ctrl+Alt+T) e execute:

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

**Por que:** atualizações de kernel pendentes podem causar conflito com o driver NVIDIA. Sempre parta de um sistema 100% atualizado.

### 1.2 Verifique se o sistema reconhece a GPU

Após o reboot, abra o terminal novamente:

```bash
lspci | grep -i nvidia
```

Você deve ver algo como:

```
01:00.0 3D controller: NVIDIA Corporation GA106GL [RTX A3000 Laptop GPU] (rev a1)
```

Se **não aparecer nada**, a GPU não está sendo detectada pelo sistema — verifique a conexão do SSD e se a BIOS do Dell está com a GPU habilitada.

### 1.3 Verifique também a Intel integrada

```bash
lspci | grep -i vga
```

Deve aparecer algo como:

```
00:02.0 VGA compatible controller: Intel Corporation [...]
```

Se ambas aparecem, você tem GPU híbrida e pode prosseguir.

---

## PARTE 2 — Instalação do driver NVIDIA (repositório Ubuntu)

### 2.1 Veja quais drivers estão disponíveis

```bash
ubuntu-drivers list
```

Isso mostra os drivers compatíveis com sua GPU. Exemplo de saída:

```
nvidia-driver-550
nvidia-driver-570
nvidia-driver-570-open
```

### 2.2 Instale o driver recomendado automaticamente

```bash
sudo ubuntu-drivers install
```

Este comando detecta sua GPU e instala o driver mais adequado. É a forma mais segura.

**Alternativa — instalar uma versão específica:**

Se por algum motivo quiser escolher manualmente:

```bash
sudo ubuntu-drivers install nvidia:570
```

### 2.3 Reinicie o computador

```bash
sudo reboot
```

### 2.4 Verifique se o driver foi instalado corretamente

Após o reboot, abra o terminal:

```bash
nvidia-smi
```

Você deve ver uma tabela mostrando:

- Driver Version (ex: 570.xx.xx)
- CUDA Version (ex: 12.x)
- Nome da GPU: RTX A3000
- Memória: 12288 MiB

Se aparecer erro "command not found" ou "NVIDIA driver not loaded", algo deu errado. Veja a seção de troubleshooting no final.

### 2.5 Teste complementar

```bash
# Verifica qual driver está em uso
cat /proc/driver/nvidia/version

# Lista módulos NVIDIA carregados no kernel
lsmod | grep nvidia
```

---

## PARTE 3 — Configurar modo híbrido (Intel display + NVIDIA compute)

Esta é a configuração ideal para pesquisadores: a Intel renderiza o desktop (menos calor, menos bateria), e os 12GB da RTX A3000 ficam 100% livres para CUDA.

### 3.1 Instale o pacote nvidia-prime (se não estiver instalado)

```bash
sudo apt install nvidia-prime
```

### 3.2 Configure o modo on-demand

```bash
sudo prime-select on-demand
```

### 3.3 Reinicie

```bash
sudo reboot
```

### 3.4 Verifique o modo ativo

```bash
prime-select query
```

Deve retornar: `on-demand`

### 3.5 Confirme que o desktop está rodando na Intel

```bash
# Instale o utilitário se necessário
sudo apt install mesa-utils

# Verifique qual GPU está renderizando o desktop
glxinfo | grep "OpenGL renderer"
```

Deve mostrar algo como: `OpenGL renderer string: Mesa Intel(R) UHD Graphics`

### 3.6 Confirme que a NVIDIA está disponível para CUDA

```bash
nvidia-smi
```

A GPU deve aparecer com uso de memória mínimo (~5 MiB), pois não está renderizando nada. Isso significa que ela está livre para seus workloads.

### 3.7 (Opcional) Se precisar rodar algo gráfico na NVIDIA

Para rodar um programa específico na GPU NVIDIA (ex: visualização 3D):

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia nome_do_programa
```

Para CUDA, isso não é necessário — PyTorch e outras libs CUDA usam a NVIDIA automaticamente.

---

## PARTE 4 — Instalação do CUDA Toolkit

### 4.1 Instale o CUDA toolkit do repositório Ubuntu

```bash
sudo apt install nvidia-cuda-toolkit
```

### 4.2 Verifique a instalação

```bash
nvcc --version
```

Deve mostrar algo como: `Cuda compilation tools, release 12.x`

**Nota:** a versão do `nvcc` pode ser um pouco menor que a versão de CUDA mostrada no `nvidia-smi`. Isso é normal — o `nvidia-smi` mostra a versão máxima suportada pelo driver, o `nvcc` mostra a versão do toolkit instalado. O importante é que o toolkit seja igual ou menor que a versão do driver.

---

## PARTE 5 — Ambiente Python para Computer Vision

### 5.1 Instale dependências do sistema

```bash
sudo apt install python3-pip python3-venv python3-dev build-essential
```

### 5.2 Crie um ambiente virtual

```bash
python3 -m venv ~/cv
source ~/cv/bin/activate
```

**Dica:** adicione ao seu `~/.bashrc` para ativar automaticamente:

```bash
echo 'alias cv="source ~/cv/bin/activate"' >> ~/.bashrc
source ~/.bashrc
```

Agora basta digitar `cv` para ativar o ambiente.

### 5.3 Atualize o pip dentro do venv

```bash
pip install --upgrade pip
```

### 5.4 Instale o PyTorch com suporte CUDA

Vá em https://pytorch.org/get-started/locally/ e selecione:

- OS: Linux
- Package: Pip
- Language: Python
- Compute Platform: a versão CUDA compatível com seu driver

O comando típico será algo como:

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```

**Importante:** o PyTorch via pip já inclui CUDA runtime e cuDNN embutidos. Você NÃO precisa instalar cuDNN separadamente.

### 5.5 Verifique se o PyTorch enxerga a GPU

```bash
python3 -c "
import torch
print(f'PyTorch version: {torch.__version__}')
print(f'CUDA disponível: {torch.cuda.is_available()}')
print(f'CUDA version: {torch.version.cuda}')
print(f'cuDNN version: {torch.backends.cudnn.version()}')
if torch.cuda.is_available():
    print(f'GPU: {torch.cuda.get_device_name(0)}')
    print(f'VRAM: {torch.cuda.get_device_properties(0).total_mem / 1e9:.1f} GB')
"
```

Saída esperada:

```
PyTorch version: 2.x.x+cu12x
CUDA disponível: True
CUDA version: 12.x
cuDNN version: 90100
GPU: NVIDIA RTX A3000 Laptop GPU
VRAM: 12.0 GB
```

Se `CUDA disponível: False`, veja troubleshooting no final.

### 5.6 Instale bibliotecas para Computer Vision

```bash
# Processamento de imagem
pip install opencv-python-headless
pip install scikit-image
pip install Pillow

# Augmentação de dados (essencial para treinar modelos robustos)
pip install albumentations

# Modelos pré-treinados
pip install timm                    # ResNet, EfficientNet, ViT, Swin, etc.
pip install ultralytics             # YOLOv8/v11 (detecção, segmentação)

# Data science
pip install numpy pandas matplotlib seaborn

# Tracking de experimentos
pip install wandb                   # ou: pip install mlflow

# Treinamento organizado (opcional mas recomendado)
pip install pytorch-lightning

# Jupyter
pip install jupyterlab
```

### 5.7 Teste rápido de treinamento na GPU

```bash
python3 -c "
import torch
import time

device = torch.device('cuda')
x = torch.randn(1000, 1000, device=device)

start = time.time()
for _ in range(1000):
    x = x @ x.T
torch.cuda.synchronize()
end = time.time()

print(f'1000 multiplicações de matrizes 1000x1000 na GPU: {end-start:.2f}s')
print(f'GPU memory usada: {torch.cuda.memory_allocated()/1e6:.0f} MB')
"
```

---

## PARTE 6 — Manutenção do dia a dia

### 6.1 Atualizações normais

```bash
sudo apt update && sudo apt upgrade -y
```

Os drivers NVIDIA do repositório Ubuntu são atualizados junto com o sistema. Não precisa de intervenção manual.

### 6.2 Após atualização de kernel

Se após um `apt upgrade` o kernel for atualizado, reinicie:

```bash
sudo reboot
```

O módulo NVIDIA será recompilado automaticamente (DKMS cuida disso).

Se após reboot o `nvidia-smi` der erro:

```bash
# Reinstala os headers do kernel atual
sudo apt install linux-headers-$(uname -r)

# Reconfigura o DKMS
sudo dkms autoinstall

sudo reboot
```

### 6.3 Upgrade de versão do Kubuntu (ex: 25.04 → 26.04 LTS)

```bash
sudo do-release-upgrade
```

Os drivers do repositório Ubuntu são migrados automaticamente. Após o upgrade, verifique com `nvidia-smi`.

---

## PARTE 7 — Configuração do Boot (EFI) no Dell

Como você não tem acesso à BIOS, pode ser necessário ajustar a ordem de boot.

### 7.1 Antes de mover o SSD de volta ao Dell

Ainda no ThinkPad, verifique se o boot entry do GRUB existe na EFI:

```bash
efibootmgr -v
```

### 7.2 Se o Dell não bootar no Linux

Se ao ligar o Dell ele for direto pro Windows, ligue com um live USB do Kubuntu e execute:

```bash
sudo efibootmgr --create \
    --disk /dev/nvme0n1 \
    --part 1 \
    --label "Kubuntu" \
    --loader "\EFI\ubuntu\shimx64.efi"
```

Ou, se conseguir acessar o Windows (mesmo sem admin), pode usar a tecla F12 no boot do Dell para selecionar o boot device manualmente.

---

## Troubleshooting

### nvidia-smi retorna erro

```bash
# Verifique se o módulo está carregado
lsmod | grep nvidia

# Se não estiver, tente carregar manualmente
sudo modprobe nvidia

# Se der erro de módulo, reconstrua
sudo apt install --reinstall nvidia-driver-570  # ou a versão que instalou
sudo reboot
```

### PyTorch não detecta a GPU

```bash
# Verifique se o driver está funcionando
nvidia-smi

# Se nvidia-smi funciona mas PyTorch não vê a GPU,
# provavelmente instalou a versão CPU do PyTorch. Reinstale:
pip uninstall torch torchvision torchaudio
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```

### Tela preta após instalar driver

1. Pressione `Ctrl+Alt+F2` para acessar o terminal (TTY)
2. Faça login
3. Remova o driver:

```bash
sudo apt remove --purge '^nvidia-.*'
sudo apt autoremove
sudo reboot
```

4. Após reboot (voltará ao driver nouveau), tente instalar novamente.

### Verificação geral do estado do sistema

```bash
# Status completo
echo "=== Driver ===" && nvidia-smi
echo "=== CUDA ===" && nvcc --version
echo "=== Prime ===" && prime-select query
echo "=== Módulos ===" && lsmod | grep nvidia | head -5
```

---

## Resumo dos comandos essenciais

| Ação | Comando |
|------|---------|
| Instalar driver | `sudo ubuntu-drivers install` |
| Verificar driver | `nvidia-smi` |
| Instalar CUDA toolkit | `sudo apt install nvidia-cuda-toolkit` |
| Verificar CUDA | `nvcc --version` |
| Modo híbrido | `sudo prime-select on-demand` |
| Consultar modo GPU | `prime-select query` |
| Instalar PyTorch (GPU) | `pip install torch torchvision --index-url https://download.pytorch.org/whl/cu124` |
| Testar PyTorch GPU | `python3 -c "import torch; print(torch.cuda.is_available())"` |
| Atualizar sistema | `sudo apt update && sudo apt upgrade -y` |

---

*Tutorial gerado em fevereiro de 2026. Verifique os links de documentação oficial para informações atualizadas.*
