# Gno.land Pearl Testnet Validator Kurulum Rehberi

Bu rehber, `chain/pearl` branch'i üzerinde bir full node kurup validator adayı olarak kaydolma sürecini, gerçek bir kurulumda karşılaşılan sorunlar ve çözümleriyle birlikte anlatır. Resmi kaynak: [VALIDATOR.md](https://github.com/gnolang/gno/blob/chain/pearl/misc/deployments/pearl.gno.land/VALIDATOR.md)

> Bu rehber `gnoland v1.0.0-rc.0` (chain/pearl branch) ile test edilmiştir. Farklı sürümlerde flag isimleri değişebilir.

---

## Gereksinimler

- Go kurulu bir Linux sunucu (cloud/VPS, on-prem veya data-center)
- Dışarıya açık bir public IP ve `26656` (P2P) portu
- `git`, `make`, temel build araçları

---

## 1. Kaynak koddan binary derleme

```bash
git clone https://github.com/gnolang/gno.git gno-pearl
cd gno-pearl
git checkout chain/pearl
make -C gno.land install.gnoland install.gnokey
```

Bu, `gnoland` ve `gnokey`'i `$GOPATH/bin` (genelde `/root/go/bin`) altına kurar. Kurulumu doğrula:

```bash
which gnoland
which gnokey
```

Alternatif: Docker imajı derlemek için `docker build --target gnoland -t gnoland:pearl .` veya hazır binary'ler için [release sayfası](https://github.com/gnolang/gno/releases/tag/chain%2Fpearl) ve `ghcr.io/gnolang/gno/gnoland` container registry'si kullanılabilir.

---

## 2. Genesis dosyasını indirme ve doğrulama

```bash
wget -O genesis.json https://github.com/gnolang/gno/releases/download/chain/pearl/genesis.json
shasum -a 256 genesis.json
```

Çıktı şu hash ile **birebir eşleşmeli**:

```
c45fe60c8c8a1f859d9e4d5aad7ce4d100ff0eb78302e71318ba0de481a8dc91  genesis.json
```

Eşleşmiyorsa dosyayı kullanma, tekrar indir.

---

## 3. Node yapılandırması

> **Not:** Bu build'de `config init` / `secrets init` komutlarında `--home` flag'i **yoktur**. Bunun yerine her komut kendi dizin flag'ini alır (`-config-path`, `-data-dir`). `gnoland start` ise tek bir `-data-dir` kök dizini bekler ve içinde `config/` ve `secrets/` alt dizinlerini arar.

```bash
mkdir -p ~/pearl-node

gnoland config init -config-path ~/pearl-node/config/config.toml
gnoland secrets init -data-dir ~/pearl-node/secrets
```

### Config değerlerini ayarlama

**Zincir geneli — tüm node'larda birebir aynı olmalı:**

```bash
gnoland config set p2p.persistent_peers "g1m37xukfq6yl555k93fcyzns83qnmgyax9zm875@seed-1.pearl.testnets.gno.land:26656,g1ngukqd3khekaqjf90k45cglzm0l25wwzl2fkn2@seed-2.pearl.testnets.gno.land:26656" -config-path ~/pearl-node/config/config.toml
gnoland config set application.prune_strategy syncable -config-path ~/pearl-node/config/config.toml
gnoland config set consensus.timeout_commit 3s -config-path ~/pearl-node/config/config.toml
gnoland config set consensus.peer_gossip_sleep_duration 10ms -config-path ~/pearl-node/config/config.toml
gnoland config set p2p.flush_throttle_timeout 10ms -config-path ~/pearl-node/config/config.toml
```

**Node'a özel:**

```bash
# Public IP'ini öğren
curl -4 ifconfig.me

gnoland config set moniker "senin-node-adin" -config-path ~/pearl-node/config/config.toml
gnoland config set p2p.external_address "SUNUCU_PUBLIC_IP:26656" -config-path ~/pearl-node/config/config.toml
gnoland config set p2p.pex true -config-path ~/pearl-node/config/config.toml
```

⚠️ `p2p.external_address` alanına **yer tutucu değil, gerçek public IP'ni** yaz — yanlış/çözümlenemeyen bir değer node'un başlamasını engeller (`invalid p2p external address: unable to look up IP`).

**Önerilen:**

```bash
gnoland config set mempool.size 10000 -config-path ~/pearl-node/config/config.toml
gnoland config set p2p.max_num_outbound_peers 40 -config-path ~/pearl-node/config/config.toml
```

Sentry-node mimarisi için `gnoland` README'sindeki [Sentry-node architecture](https://github.com/gnolang/gno/blob/master/gno.land/cmd/gnoland/README.md#sentry-node-architecture) bölümüne bakılabilir.

---

## 4. Node'u başlatma

```bash
gnoland start \
  -data-dir ~/pearl-node \
  -gnoroot-dir ~/gno-pearl \
  -chainid pearl-1 \
  -genesis genesis.json \
  -skip-genesis-sig-verification
```

`-skip-genesis-sig-verification` **zorunlu**: bazı genesis işlemleri kasıtlı geçersiz/placeholder imza taşıyor (örn. `names.Enable`), bu flag olmadan node başlangıçta panikler.

Başarılı bir başlangıçta şu satırı görürsün:

```
Genesis replay report   {"mode": "strict", "total": 90, "ok": 90, "ok_gas_differs": 0, "failed": 0, "skipped_failed": 0}
```

Ve node henüz validator setinde olmadığı için:

```
This node is not a validator    {"addr": "g1...", "pubKey": "gpub1..."}
```

**Bu satırdaki `pubKey` değerini not al** — 6. adımda kayıt için gerekecek.

### Sync durumunu izleme

```bash
curl -s localhost:26657/status | jq '.result.sync_info'
```

`catching_up: false` olduğunda node tepe bloğa yetişmiştir. Sürekli izlemek için:

```bash
watch -n 2 'curl -s localhost:26657/status | jq ".result.sync_info.latest_block_height, .result.sync_info.catching_up"'
```

---

## 5. systemd servisi olarak çalıştırma

```bash
sudo tee /etc/systemd/system/gnoland-pearl.service > /dev/null <<'EOF'
[Unit]
Description=Gnoland Pearl Testnet Validator Node
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Environment=GNOROOT=/root/gno-pearl
WorkingDirectory=/root/gno-pearl
ExecStart=/root/go/bin/gnoland start \
  -data-dir /root/pearl-node \
  -gnoroot-dir /root/gno-pearl \
  -chainid pearl-1 \
  -genesis /root/gno-pearl/genesis.json \
  -skip-genesis-sig-verification
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gnoland-pearl
sudo systemctl start gnoland-pearl
```

⚠️ **`Environment=GNOROOT=...` satırı kritik.** Terminalde elle çalıştırırken kabuk GNOROOT'u otomatik algılayabiliyor, ama systemd temiz bir ortamda çalıştığı için bu değişkeni bilmez ve `panic: gno was unable to determine GNOROOT` hatası verir. `-gnoroot-dir` flag'i tek başına yeterli olmuyor, environment değişkeni de gerekiyor.

Durum ve log kontrolü:

```bash
sudo systemctl status gnoland-pearl
sudo journalctl -u gnoland-pearl -f
```

---

## 6. Validator adayı olarak kaydolma

### Consensus public key'i alma

> **Bilinen sorun:** `gnoland secrets get validator_key -data-dir ~/pearl-node` bu build'de bir reflect panic'i ile çöküyor (`reflect: call of reflect.Value.Interface on zero Value`), hem düz hem de `validator_key.pub_key -raw` ile denense de aynı sonucu veriyor.
>
> **Çözüm:** `gpub1...` değeri zaten node ilk başladığında loglara yazılıyor (`"This node is not a validator" {"pubKey": "gpub1..."}`) — servisi başlatıp logdan al:
>
> ```bash
> sudo systemctl restart gnoland-pearl
> sudo journalctl -u gnoland-pearl -f | grep -m1 "pubKey"
> ```
>
> Alternatif olarak `cat ~/pearl-node/secrets/priv_validator_key.json` ile ham anahtar dosyasını görebilirsin, ama oradaki `pub_key.value` alanı ham base64'tür — bech32 `gpub1...` formatına çevrilmesi gerekir, bu yüzden log satırından almak çok daha pratiktir.

### Operatör hesabının bakiyesini kontrol etme

```bash
gnokey query bank/balances/<g1-adresin> --remote https://rpc.pearl.testnets.gno.land
```

Bakiye boşsa faucet'ten talep et: **https://pearl.testnets.gno.land/faucet**

### Register işlemi

```bash
gnokey maketx call \
  --pkgpath gno.land/r/gnops/valopers \
  --func Register \
  --broadcast \
  --chainid pearl-1 \
  --args "<MONIKER>" \
  --args "<kısa açıklama>" \
  --args "<cloud|on-prem|data-center>" \
  --args "<operatör g1... adresin>" \
  --args "<gpub1... consensus pubkey>" \
  --gas-fee 1000000ugnot \
  --gas-wanted 100000000 \
  --remote https://rpc.pearl.testnets.gno.land \
  <gnokey-hesap-adın>
```

> `-home` flag'i gerekmedi çünkü varsayılan dizin (`/root/.config/gno`) zaten `gnokey list` ile bulunan hesabı içeriyordu. Farklı bir dizin kullanıyorsan `-home <dizin>` eklemen gerekir.

### Detaylı profil açıklaması ekleme (opsiyonel)

Daha uzun / çok satırlı bir validator profili eklemek için `UpdateDescription`:

```bash
gnokey maketx call \
  --pkgpath gno.land/r/gnops/valopers \
  --func UpdateDescription \
  --broadcast \
  --chainid pearl-1 \
  --args "<operatör g1... adresin>" \
  --args "$(cat <<'EOF'
<çok satırlı profil metni buraya>
EOF
)" \
  --gas-fee 1000000ugnot \
  --gas-wanted 15000000 \
  --remote https://rpc.pearl.testnets.gno.land \
  <gnokey-hesap-adın>
```

### Kaydı doğrulama

**https://pearl.testnets.gno.land/r/gnops/valopers**

---

## 7. Aktif validator setine girme

Register işlemi seni sadece **aday (candidate)** olarak listeler. Aktif validator setine (`r/sys/validators/v3`) eklenmen için bir GovDAO üyesinin bir öneri (proposal) açıp geçirmesi gerekir. Bu adım kendi kontrolünde değildir — kayıt sonrası GovDAO sürecini beklemek gerekir.

Mevcut aktif seti görüntülemek için: **https://pearl.testnets.gno.land/r/sys/validators/v3**

---

## Sık Karşılaşılan Sorunlar Özeti

| Sorun | Sebep | Çözüm |
|---|---|---|
| `flag provided but not defined: -home` | Bu build'de `config init`/`secrets init` `--home` desteklemiyor | `-config-path` / `-data-dir` kullan |
| `invalid p2p external address: unable to look up IP` | `p2p.external_address` yer tutucu/hatalı IP içeriyor | `curl -4 ifconfig.me` ile gerçek IP'yi al ve yaz |
| `panic: gno was unable to determine GNOROOT` (sadece systemd'de) | systemd ortamı kabuk env değişkenlerini miras almıyor | Servis dosyasına `Environment=GNOROOT=<repo-yolu>` ekle |
| `secrets get validator_key` reflect panic'i | Bu build'deki bilinen bir bug | `gpub1...` değerini node başlangıç logundan al |
| `unable to overwrite secret ... overwrite not enabled` | `secrets init` zaten var olan anahtarların üzerine yazmıyor | Zararsız — mevcut anahtarlar kullanılır; sıfırdan üretmek istersen `-force` ekle |

---

## Referanslar

- Resmi rehber: https://github.com/gnolang/gno/blob/chain/pearl/misc/deployments/pearl.gno.land/VALIDATOR.md
- Valoper listesi: https://pearl.testnets.gno.land/r/gnops/valopers
- Aktif validator seti: https://pearl.testnets.gno.land/r/sys/validators/v3
- Faucet: https://pearl.testnets.gno.land/faucet
