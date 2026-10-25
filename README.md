
# BDD Support 0.3.2 — production kurulum ve işletim rehberi

Bu dosya müşteri Linux sunucusu için tek otoriter kurulum kaynağıdır. 0.3.2 ile
uygulama kaynak kodu müşteri paketinden ayrılmıştır. Next.js uygulaması GitHub
Actions içinde bir kez build edilir; müşteri sunucusu private registry'den SHA256
digest ile sabitlenmiş image'ı çeker. Host'a Node.js, npm, PostgreSQL, n8n veya
buildx kurulmaz.

Paket içindeki tüm domain, kullanıcı ve ID değerleri placeholder'dır. Gerçek
credential veya müşteri verisi pakette bulunmaz.

> **Secret uyarısı:** `settings.json`, `compose.json`, `deployed-settings.json`,
> Docker registry oturumu, backup arşivleri ve n8n veritabanı secret veya müşteri
> verisi içerir. Bunları destek kaydına, e-postaya ya da sohbet ekranına
> yapıştırmayın. Meta token, registry token, `service_role`, JWT veya parola
> paylaşılırsa ilgili credential'ı hemen rotate edin.

## 1. Dağıtım modeli

```text
Private GitHub kaynak kodu
          ↓ tag: v0.3.2
GitHub Actions: test + tek build + integration
          ├── private OCI image @sha256:digest
          └── kaynak kodsuz deployment arşivi + SHA256
                         ↓
                müşteri Linux sunucusu
```

Deployment arşivi yalnız yönetim aracı, migration'lar, n8n workflow'ları, Supabase
bootstrap dosyaları ve bu rehberi içerir. `src/`, `public/`, `Dockerfile`, npm
dosyaları, `.git`, developer testleri ve GitHub Actions tanımları müşteriye gitmez.

Her release değişmezdir. Aynı `0.3.2` içeriği yeniden üretilmez; düzeltme yeni bir
sürüm olur. Müşteriye özel değerler yalnız kalıcı state içinde tutulur.

```text
/opt/bdd-support/
  releases/0.3.2/               değişmez deployment paketi
  state/                        izin 0700; müşteriye ait kalıcı state
    settings.json               izin 0600; secret içerir
    deployed-settings.json      son başarılı müşteri ayarları
    current.json                sürüm, image digest ve source commit
    compose.json                üretilmiş Compose; secret içerir
    data/                       PostgreSQL, n8n, dosya ve TLS verileri
    backups/<UTC-zaman>/        cold backup
    failed-first-install-*/     yarım kurulum recovery arşivi
```

## 2. Ön gereksinimler

- x86_64/amd64 production Linux sunucu
- Python 3.10 veya üzeri
- Docker Engine
- Docker Compose 2.24 veya üzeri
- Registry ve vendor image'ları için internet erişimi
- En az 10 GiB boş disk
- TCP 80/443 ve UDP 443 boş ve dışarıdan erişilebilir
- Aynı sunucuya yönlenen üç DNS kaydı
- Private registry için yalnız pull/read yetkili müşteri credential'ı

Docker Desktop production yolu değildir. Müşteri hostunda Node.js, npm ve Docker
buildx gerekmez; build işlemi yalnız CI runner'da yapılır.

```bash
python3 --version
docker context show
docker context inspect
docker info
docker compose version
uname -m
df -h /opt
sudo ss -lntup | grep -E ':(80|443)\b' || true
```

**Beklenen:** Linux Engine context/socket, çalışan daemon, Compose >=2.24,
`x86_64`, yeterli disk ve boş portlar.

**Başarısızsa:** `desktop-linux`, kullanıcı home dizinindeki Docker Desktop socket'i
veya daemon bağlantı hatasını düzeltin. ARM host için bu release'i emülasyonla
zorlamayın; ayrı test edilmiş arm64 release gereklidir.

## 3. DNS

```bash
getent hosts panel.example.com
getent hosts api.example.com
getent hosts webhook.example.com
```

Üç kayıt production sunucusunun public IP adresini döndürmelidir. Caddy sertifikası
için internetten TCP 80/443 erişimi gerekir.

## 4. Private registry oturumu

Teknisyene yalnız image okuma yetkili ayrı credential verin. Tokenı komut satırı
argümanı veya shell history içine yazmayın:

```bash
read -rsp 'Registry read token: ' REGISTRY_TOKEN; echo
printf '%s' "$REGISTRY_TOKEN" | sudo docker login ghcr.io \
  --username REGISTRY_MACHINE_USER --password-stdin
unset REGISTRY_TOKEN
```

**Beklenen:** `Login Succeeded`.

**Başarısızsa:** Tokenın `read:packages` yetkisini, package erişimini, SSO
yetkilendirmesini ve image'ın bulunduğu organization/repository adını kontrol edin.
Registry credential'ını `settings.json` içine koymayın.

## 5. Paketi doğrulama ve açma

```bash
cd /paketin/bulundugu/dizin
sha256sum -c bdd-support-0.3.2.tar.gz.sha256
sudo install -d -m 0755 /opt/bdd-support/releases/0.3.2
sudo tar -xzf bdd-support-0.3.2.tar.gz \
  -C /opt/bdd-support/releases/0.3.2
cd /opt/bdd-support/releases/0.3.2
sudo python3 deploy/manage.py validate
```

**Beklenen:** checksum `OK`; validate çıktısında `0.3.2 customer artifact` görünür.
`deploy/release.json` içindeki bütün `images` değerleri `@sha256:` içermelidir.

**Başarısızsa:** Paketi çalıştırmayın. Arşiv/checksum çiftini onaylı GitHub
Release veya şirket dağıtım kanalından tekrar alın. Açılmış release'i düzenlemeyin.

## 6. Temiz kurulum

### Init

```bash
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py init \
  --state /opt/bdd-support/state
```

Komut panel, API ve webhook domain'lerini; ilk panel admin e-postası ve parolasını
sorar. `/opt/bdd-support/state/settings.json` oluşturur ve gerekli secret'ları
üretir. Dosya mevcutsa `init` tekrar çalıştırılmaz.

Meta hazır değilse `workflows_enabled=false` kalsın. Hazırsa şu alanları gerçek
değerlerle düzenleyin:

```json
{
  "whatsapp_access_token": "META_ACCESS_TOKEN_PLACEHOLDER",
  "whatsapp_phone_number_id": "META_PHONE_NUMBER_ID",
  "whatsapp_api_version": "vNN.0",
  "workflows_enabled": true
}
```

`whatsapp_phone_number_id` telefon numarası değil Meta'nın sayısal Phone Number
ID değeridir. Üretilmiş database/JWT/n8n secret'larını değiştirmeyin.

### Install

```bash
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py install \
  --state /opt/bdd-support/state
```

Araç Docker context/daemon, Compose, amd64 mimari, port, DNS, yazma izni ve disk
kontrolü yapar. Digest ile sabit image'ları kesintiden önce çeker; müşteri hostunda
uygulama build etmez. Ardından DB bootstrap, migration, ilk admin, workflow import/
publish, webhook registration, Schedule Trigger ve runtime env smoke kontrollerini
tamamlar.

**Başarılı son:** `Deployment complete: 0.3.2 https://panel.example.com`.

İlk kurulum yarıda kalırsa manuel restart ile geçiştirmeyin. Kısmi veri otomatik
reuse edilmez:

```bash
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py \
  reset-failed-install --state /opt/bdd-support/state
```

Bu işlem kısmi veriyi `failed-first-install-*` altına taşır ve `settings.json`
dosyasını korur. Gerçek müşteri verisi oluşmuş başarılı kuruluma uygulanmaz.

## 7. 0.2.8'den 0.3.2'ye geçiş

0.2.8 release dizinini ve state'i silmeyin. Registry login ve 0.3.2 paket
doğrulamasından sonra yeni release aracıyla:

```bash
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py update \
  --state /opt/bdd-support/state
```

Araç 0.2.8 kaynak-build formatından 0.3.2 image formatına izin verilen tek format
geçişini uygular. Önce 0.3.2 image'larını çeker, sonra 0.2.8 release ve state'inin
cold backup'ını alır. Bu eski backup, restore sırasında eski build context'ini de
korur.

**Beklenen:** backup yolu, managed workflow smoke sonucu ve `Deployment complete:
0.3.2`. `current.json` içinde app image digest'i ve source commit bulunur.

**Başarısızsa:** `FAILED` kaydını inceleyin. Migration/deployment başladıysa eski
state'i zorla açmayın; doğrulanmış pre-update backup'ı yeni/boş state'e restore edin.

## 8. Sürüm ve rollout politikası

Her müşteri atanmış release paketini kullanır. `latest` tag kullanılmaz. Rollout
sırası `internal → pilot → stable` şeklindedir. Promotion aynı image'ı yeniden
build etmez; mevcut GitHub Release'i onaylar. Müşterinin `current_version` kaydı
yalnız backup ve post-deploy smoke testleri geçtikten sonra güncellenir.

Yeni sürümü `/opt/bdd-support/releases/YENI_SURUM` altına açın, checksum/manifesti
doğrulayın ve yeni sürümün aracıyla update çalıştırın:

```bash
sudo python3 /opt/bdd-support/releases/YENI_SURUM/deploy/manage.py update \
  --state /opt/bdd-support/state
```

Aracın `minimum_upgrade_version` kontrolünü atlamayın. Çok eski müşteri için önce
belirtilen ara release kullanılır.

## 9. n8n ve Meta

n8n owner hesabı panel admin hesabından ayrıdır. Editör yalnız `127.0.0.1:5678`
dinler:

```bash
ssh -L 5678:127.0.0.1:5678 linux-kullanicisi@sunucu.example.com
```

Yerel tarayıcıda `http://localhost:5678` açın. Başarılı, hatalı ve manuel
execution'lar kaydedilir; pruning 24 saat ve 5.000 execution sınırının ilk
ulaşılanına göre çalışır.

Meta ayarları:

- Callback: `https://webhook.example.com/webhook/whatsapp`
- Verify token: `settings.json` içindeki `verify_token`
- Field: `messages`
- **Subscribe webhooks** açık
- Test numarası outbound yalnız allowed recipient listesine gönderir
- Production numarası için verify/register tamamlanmış olmalı
- Data-only eSIM genellikle SMS/voice OTP doğrulamasına uygun değildir

Phone Number ID veya token değişince:

```bash
sudoedit /opt/bdd-support/state/settings.json
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py configure \
  --state /opt/bdd-support/state
```

Configure sonunda secret yazdırılmadan Phone Number ID, API version, workflow
durumu ve webhook URL gösterilir. Yalnız container restart yeni env'i uygulamaz;
`configure` recreate işlemi gereklidir.

Graph kontrolünde tokenı process argümanında göstermeyin:

```bash
read -rsp 'Meta access token: ' META_TOKEN; echo
META_API_VERSION='vNN.0'
META_PHONE_NUMBER_ID='META_PHONE_NUMBER_ID'
curl -fsS --config - <<EOF
header = "Authorization: Bearer ${META_TOKEN}"
url = "https://graph.facebook.com/${META_API_VERSION}/${META_PHONE_NUMBER_ID}?fields=id,display_phone_number,verified_name"
EOF
unset META_TOKEN
```

## 10. Kabul smoke testleri

**Inbound:** Gerçek izinli telefondan benzersiz metin gönderin. n8n execution,
DB mesaj kaydı ve panel konuşması birlikte görünmelidir. Aynı
`whatsapp_message_id` tekrar gönderildiğinde ikinci kayıt oluşmaması beklenen
idempotency davranışıdır.

```bash
docker compose --project-name bdd-support \
  -f /opt/bdd-support/state/compose.json exec -T db \
  psql -X -U postgres -d postgres -c \
  "SELECT direction,status,whatsapp_message_id,created_at FROM messages ORDER BY created_at DESC LIMIT 5;"
```

**Outbound:** Panelden mesaj gönderin; `pending → sent`, `wamid...` oluşumu ve
hedef telefona teslim birlikte doğrulanır.

**Status:** Delivered/read event'leri aynı `messages` webhook field'i içinde
`statuses` payload'ı olarak gelir. Sorumlu akış:

```text
AB Support — WhatsApp Gelen Mesaj
WhatsApp Webhook → Durum mu Mesaj mı? → Statüsü Parse Et → Mesaj Durumunu Güncelle
```

```bash
docker compose --project-name bdd-support \
  -f /opt/bdd-support/state/compose.json exec -T db \
  psql -X -U postgres -d postgres -c \
  "SELECT whatsapp_message_id,status,delivered_at,read_at FROM messages WHERE direction='outbound' ORDER BY created_at DESC LIMIT 10;"
```

## 11. Backup, restore ve original sisteme dönüş

```bash
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py backup \
  --state /opt/bdd-support/state
```

Cold backup kısa maintenance kesintisi oluşturur ve başarıda servisleri geri açar.
Varsayılan en yeni 7 backup tutulur; `failed-first-install-*` backup'a alınmaz.

```bash
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py prune-backups \
  --state /opt/bdd-support/state --keep-backups 14
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py prune-backups \
  --state /opt/bdd-support/state --keep-backups 14 --apply
```

Restore yalnız yeni/boş state'e yapılır. Original ve restored sistem aynı port ve
Compose project adını kullandığından aynı anda çalıştırılmaz:

```bash
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py stop \
  --state /opt/bdd-support/state
sudo install -d -m 0700 /opt/bdd-support/restored-state
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py restore \
  --state /opt/bdd-support/restored-state \
  --backup /opt/bdd-support/state/backups/UTC_BACKUP_DIZINI
```

Restore'un yazdırdığı `restored-release/deploy/manage.py` yolu ile `status`, DB ve
panel kontrolü yapılır. Sonra original sisteme dönülür:

```bash
sudo python3 /opt/bdd-support/restored-state/restored-release/deploy/manage.py stop \
  --state /opt/bdd-support/restored-state
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py start \
  --state /opt/bdd-support/state
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py status \
  --state /opt/bdd-support/state
```

## 12. Troubleshooting

```bash
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py status \
  --state /opt/bdd-support/state
docker compose --project-name bdd-support \
  -f /opt/bdd-support/state/compose.json logs --tail=200 \
  db auth rest realtime storage gateway n8n app proxy
```

- `pull access denied`: registry login/read permission veya package erişimi hatalı.
- `manifest unknown`: release metadata/image silinmiş ya da yanlış registry.
- `no matching manifest`: host mimarisi release ile uyumlu değil.
- Authentication Error veya OAuth 190/463: Meta token geçersiz/süresi dolmuş.
- `(#131030)`: Meta test recipient allowlist sınırı; kod hatası değildir.
- Inbound var, outbound yok: container Phone Number ID değerini kontrol edin.
- Graph 200, runtime auth hatası: ayarı düzeltip `configure` çalıştırın.
- Webhook 200, panelde kayıt yok: duplicate `whatsapp_message_id` kontrol edin.
- n8n execution yok: publish, webhook registration ve retention'ı kontrol edin.
- Secret destek kaydında göründüyse ilgili credential'ı rotate edin.

## 13. Cleanup

Önce listeleyin, sonra yalnız exact path/ID ile silin:

```bash
sudo find /opt/bdd-support/state -maxdepth 1 -type d -name 'failed-first-install-*' -print
sudo find /opt/bdd-support/state/backups -mindepth 1 -maxdepth 1 -type d -print | sort
sudo python3 /opt/bdd-support/releases/0.3.2/deploy/manage.py cleanup \
  --state /opt/bdd-support/state \
  --path /opt/bdd-support/state/failed-first-install-UTC_DIZINI
```

Dry-run çıktısını kontrol edip aynı komuta `--apply` ekleyin. Test workflow ve
sahte konuşmalarda önce exact UUID'yi listeleyin, backup alın ve yalnız seçilen
ID'yi silin. Wildcard ile müşteri verisi silmeyin.

## 14. Release doğrulaması

Developer kaynak ağacında `manage.py pack` bilinçli olarak reddedilir. Müşteri
paketi yalnız tag pipeline'ında app ve vendor digest'leri çözüldükten sonra üretilir.
Pipeline; Python/Node testleri, production build, clean Docker install, update,
backup/restore, n8n Schedule Trigger, inbound duplicate, mock outbound,
delivered/read, kaynak kodsuz arşiv, manifest, checksum ve provenance kontrollerini
geçmeden GitHub Release yayınlamaz.

İzole Docker Linux runner'da manuel kabul testi:

```bash
python3 deploy/tests/integration.py
```

Gerçek müşteri sunucusunda integration testini çalıştırmayın.
