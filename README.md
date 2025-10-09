# PluralisResearch на русском

Из документации:
```
PC/Server with Nvidia GPU:
Minimum 16GB GPU memory
Minimum 32GB RAM
Minimum 80GB disk space (required for Docker image)
```

Получить роль в дискорде:
В ветке join-run запускаем `!join-run`

Dashboard https://dashboard.pluralis.ai/

## Вариант1 с docker

Взял на https://cloud.datacrunch.io/ Tesla V100 с жестким диском 130Gb за 0.171$/h (Ubuntu 24.04 + CUDA 12.6 + Docker).
Но там RAM немного не хватает 23Gb.

Начинаем установку

Обновляем зависимости:
```
apt update -y && apt upgrade -y
```

Обновляем docker, на datacrunch оказался устаревшим
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo groupadd docker
sudo usermod -aG docker $USER

reboot
```

Установка с использованием docker:
```
git clone https://github.com/PluralisResearch/node0
cd node0
docker build . -t pluralis_node0
```

проверим работают ли ГПУ внутри докера
```
docker run --gpus all nvidia/cuda:13.0.0-base-ubuntu24.04 nvidia-smi
```

сгенерировать стартовый скрипт
подготавливаем HuggingFace token и email потом выполняем команду

```
python3 generate_script.py --use_docker
```

на вопрос 'Do you want to change anything? [Y/n]' отвечаем n

выйдет сообщение 'File start_server.sh is generated. Run ./start_server.sh to join the experiment.'

И запускаем сервер:
```
root@dull-sun-falls-fin-01:~/node0# ./start_server.sh
Container node0_251007130709 created.
Server is started. See run.out for the output.
```

проверим есть ли контейнер
```
docker ps -a
CONTAINER ID   IMAGE            COMMAND                  CREATED          STATUS          PORTS     NAMES
b5f483cfa828   pluralis_node0   "/opt/nvidia/nvidia_…"   57 seconds ago   Up 57 seconds             node0_251007130136
```

проверим потребление ГПУ

Проверяем логи в файле run.out

## Вариант2 runpod cli

Использую шаблон "Runpod Pytorch 2.8.0", беру RTX A4500 $0,28/hr

Возле шаблона нажать Edit и в "Expose TCP Ports" добавить 49200, Container Disk увеличиваю до 150Gb также отключил jupiter

```
apt update -y && \
apt install sudo -y && \
apt install nano -y && \
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3 1
```

Установка
```
git clone https://github.com/PluralisResearch/node0
cd node0

#установка conda
mkdir -p ~/miniconda3 && wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh && bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3 && rm -rf ~/miniconda3/miniconda.sh && ~/miniconda3/bin/conda init bash && ~/miniconda3/bin/conda init zsh
bash

#на вопрос о "Do you accept the Terms of Service (ToS)" пишем accept
conda create -n node0 python=3.11
conda activate node0

# Install node0
pip install .
```
Подготавливаем HuggingFace token и email и генерируем скрипт запуска
```
python3 generate_script.py
```

На вопрос "Do you want to change anything? [Y/n]" вводим n

На нашем pod в вебинтерфейсе рунпода смотрим "Direct TCP Ports". У меня `ip:14878 -> :49200`

Открываем start_server.sh и меняем в --announce_maddrs порт, например /ip4/213.175.107.143/tcp/14878`

И запускаем

```
apt install tmux -y && \
tmux new-session -s runnode

#внутри tmux
conda activate node0 && \
./start_server.sh
```

##Присоединение к узлам

Если выходит "[ERROR] [node0.security.authorization.authorize_with_pluralis:342] Authorization failed: Our servers are currently at full capacity. Exiting run." значит сейчас вход закрыт, нужно ждать когда откроют вход новым пользователям и попробовать снова
