# Flock Script
## Flock'u servise çevirme
İlk önce eğer screen ile çalıştırıyorsanız servisli çalıştırmaya döndürmeniz gerekiyor. Screen'e girip `CTRL + C` ile durdurun ve `exit` yazarak screen'i kapatın.

alttaki kod opsiyonel garanti olsun diye attım

    cd
    source ~/.bashrc
    cd llm-loss-validator
    conda activate llm-loss-validator

alttaki zorunlu servis yapmak için :D

    sudo tee /etc/systemd/system/flockd.service > /dev/null << EOF
    [Unit]
    Description=Flock Validator Service
    After=network.target
    
    [Service]
    User=root
    WorkingDirectory=/root/llm-loss-validator/src
    Environment="PATH=/root/anaconda3/envs/llm-loss-validator/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ExecStart=/bin/bash -c 'source /root/anaconda3/bin/activate llm-loss-validator && bash start.sh --hf_token BURAYA-HUGGİNG-KEY-YAZ --flock_api_key BURAYA-FLOCK-API-KEY-YAZ --task_id BURAYA-ID-YAZ --validation_args_file validation_config_cpu.json.example --auto_clean_cache True'
    Restart=on-failure
    
    [Install]
    WantedBy=multi-user.target
    EOF
başlatalım

    sudo systemctl daemon-reload
    sudo systemctl enable flockd
    sudo systemctl start flockd
    sudo journalctl -u flockd -f
## Flock Script'ini yükleme (kendi servisiyle)

    cd
    nano flockd_update.sh

aşağıdakini direkt yapıştırıp kaydedip çıkın

```
#!/bin/bash

LOG_FILE="/var/log/flockd-updater.log"

log() {
    local level="$1"
    local message="$2"
    local timestamp
    timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    echo "$timestamp [$level] $message" | tee -a "$LOG_FILE"
}

check_flockd_status() {
    log "INFO" "Flockd hizmeti kontrol ediliyor..."
    if sudo journalctl -u flockd --since "10 minutes ago" | grep -q "exit code"; then
        log "ERROR" "Exit code bulundu. Güncelleme başlatılıyor..."
        return 0
    else
        log "INFO" "Flockd düzgün çalışıyor."
        return 1
    fi
}

update_flockd() {
    log "INFO" "Flockd durduruluyor..."
    sudo systemctl stop flockd || { log "ERROR" "Flockd durdurulamadı."; return 1; }

    log "INFO" "Güncellemeler yapılıyor..."

    cd /root/llm-loss-validator || { log "ERROR" "Proje dizinine erişilemedi. Güncelleme başarısız oldu."; sudo systemctl start flockd; return 1; }

    git fetch origin && git pull origin main || { log "ERROR" "Git pull başarısız oldu."; sudo systemctl start flockd; return 1; }

    if ! /root/anaconda3/bin/conda env list | grep -w "llm-loss-validator" > /dev/null; then
        log "INFO" "Conda ortamı kuruluyor..."
        /root/anaconda3/bin/conda create -n llm-loss-validator python==3.10 -y || { log "ERROR" "Conda ortamı oluşturulamadı."; sudo systemctl start flockd; return 1; }
    else
        log "INFO" "Conda ortamı zaten mevcut."
    fi

    source /root/anaconda3/bin/activate llm-loss-validator || { log "ERROR" "Conda ortamı aktifleştirilemedi."; sudo systemctl start flockd; return 1; }
    
    pip install -r /root/llm-loss-validator/requirements.txt || { log "ERROR" "Gereksinimler yüklenemedi."; sudo systemctl start flockd; return 1; }

    log "INFO" "Flockd yeniden başlatılıyor..."
    sudo systemctl restart flockd || { log "ERROR" "Flockd yeniden başlatılamadı."; return 1; }

    log "INFO" "Güncelleme tamamlandı."
    return 0
}

check_and_update_repo() {
    log "INFO" "Depoda güncelleme kontrol ediliyor..."

    cd /root/llm-loss-validator || { log "ERROR" "Proje dizinine erişilemedi. Güncelleme kontrolü başarısız oldu."; return 1; }

    git fetch origin

    LOCAL=$(git rev-parse @)
    REMOTE=$(git rev-parse @{u})

    if [ "$LOCAL" != "$REMOTE" ]; then
        log "INFO" "Güncelleme mevcut, repo güncelleniyor..."
        git pull origin main || { log "ERROR" "Git pull başarısız oldu."; return 1; }

        if ! /root/anaconda3/bin/conda env list | grep -w "llm-loss-validator" > /dev/null; then
            log "INFO" "Conda ortamı kuruluyor..."
            /root/anaconda3/bin/conda create -n llm-loss-validator python==3.10 -y || { log "ERROR" "Conda ortamı oluşturulamadı."; return 1; }
        else
            log "INFO" "Conda ortamı zaten mevcut."
        fi

        source /root/anaconda3/bin/activate llm-loss-validator || { log "ERROR" "Conda ortamı aktifleştirilemedi."; return 1; }
        
        pip install -r /root/llm-loss-validator/requirements.txt || { log "ERROR" "Gereksinimler yüklenemedi."; return 1; }

        log "INFO" "Repo güncellemesi tamamlandı."
    else
        log "INFO" "Repo zaten güncel."
    fi
    return 0
}

while true; do
    if check_flockd_status; then
        update_flockd
    fi
    
    check_and_update_repo
    
    sleep 600
done;
```

 ```nano /etc/systemd/system/flockd-updater.service```

aşağıdakini direkt yapıştırıp kaydedip çıkın

    [Unit]
    Description=Flock Validator Auto-Updater
    After=network.target
    
    [Service]
    User=root
    WorkingDirectory=/root
    ExecStart=/bin/bash /root/flockd_update.sh
    Restart=always
    RestartSec=10
    StandardOutput=append:/var/log/flockd-updater.log
    StandardError=append:/var/log/flockd-updater-error.log
    KillMode=control-group
    TimeoutStopSec=30
    
    [Install]
    WantedBy=multi-user.target

    sudo chmod +x /root/flockd_update.sh
    sudo systemctl daemon-reload
    sudo systemctl enable flockd-updater
    sudo systemctl start flockd-updater

scriptte hata varsa tespit logu👇

    sudo tail -f /var/log/flockd-updater-error.log -n 100

tüm detayıyla log👇

    sudo tail -f /var/log/flockd-updater.log -n 100

Log boyutu kontrolü için öneriyorum

    sudo nano /etc/logrotate.d/flockd-updater
alttakini yapıştırıp kaydedip çıkın

    su root root
    
    /var/log/flockd-updater.log /var/log/flockd-updater-error.log {
        size 50M
        rotate 5
        compress
        missingok
        notifempty
        create 0640 root root
    }
