# Router-on-a-Stick — VLAN'lar Arası Yönlendirme

**Platform:** GNS3 · 1 × Cisco 3745 (IOS 12.4) · 2 × Cisco IOSvL2 15.2 · 8 × VPCS


## Amaç

Lab 01'de bilinçli olarak engellenen VLAN'lar arası trafiği, tek fiziksel router
arayüzü üzerinden 802.1Q alt arayüzlerle (subinterface) yönlendirmek.

Kanıtlanacak üç şey:

1. Farklı VLAN'daki hostlar artık haberleşiyor ve paket **router'dan geçiyor**
   (TTL 64 → 63, traceroute ilk atlama = gateway).
2. Aynı VLAN içindeki trafik router'a **uğramıyor** (TTL 64 sabit).
3. Switch'ler L2 cihaz olduğu için TTL'i düşürmüyor — iki switch aşan paket de
   yalnızca bir kez azalıyor.

Lab 01'deki topoloji aynen korundu, üzerine R1 ve 4 PC eklendi. Yani bu lab bir
öncekinin üstüne inşa ediliyor, sıfırdan kurulmuyor.

## Topoloji

![Topoloji](topology-b04-02.png)

| Cihaz | Port | Bağlantı | Rol |
|---|---|---|---|
| R1 | Fa0/0 | SW1 Gi2/0 | fiziksel, IP yok, `no shutdown` |
| R1 | Fa0/0.10 | — | alt arayüz, `encap dot1Q 10`, 192.168.10.1 |
| R1 | Fa0/0.20 | — | alt arayüz, `encap dot1Q 20`, 192.168.20.1 |
| R1 | Fa0/0.999 | — | alt arayüz, `encap dot1Q 999 native`, **IP yok** |
| SW1 | Gi2/0 | R1 Fa0/0 | 802.1Q trunk |
| SW1 | Gi0/0 | SW2 Gi0/0 | 802.1Q trunk |
| SW1 | Gi0/1 | PC1 | access, VLAN 10 |
| SW1 | Gi0/2 | PC2 | access, VLAN 20 |
| SW1 | Gi0/3 | PC3 | access, VLAN 10 |
| SW1 | Gi1/0 | PC4 | access, VLAN 20 |
| SW2 | Gi0/0 | SW1 Gi0/0 | 802.1Q trunk |
| SW2 | Gi0/1 | PC5 | access, VLAN 10 |
| SW2 | Gi0/2 | PC6 | access, VLAN 20 |
| SW2 | Gi0/3 | PC7 | access, VLAN 10 |
| SW2 | Gi1/0 | PC8 | access, VLAN 20 |

### Adresleme

| VLAN | Ad | Ağ | Gateway | Hostlar |
|---|---|---|---|---|
| 10 | OFIS | 192.168.10.0/24 | 192.168.10.1 (R1 Fa0/0.10) | PC1 `.11`, PC3 `.13`, PC5 `.15`, PC7 `.17` |
| 20 | MUHASEBE | 192.168.20.0/24 | 192.168.20.1 (R1 Fa0/0.20) | PC2 `.12`, PC4 `.14`, PC6 `.16`, PC8 `.18` |
| 999 | NATIVE-KULLANILMIYOR | — | — | trunk native VLAN, hiçbir porta atanmaz |

VPCS'te adres **tek satırda** verilir, maske ile gateway ayrı komut değildir:

```
PC1> ip 192.168.10.11/24 192.168.10.1
```

## Yapılandırma

Tam konfigürasyonlar `configs/` klasöründe. R1 özeti:

```
interface FastEthernet0/0
 no ip address
 no shutdown
!
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
!
interface FastEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
!
interface FastEthernet0/0.999
 encapsulation dot1Q 999 native
```

SW1 tarafında Gi2/0, lab 01'deki Gi0/0 ile birebir aynı sırayla trunk'a alındı:

```
interface GigabitEthernet2/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20
 switchport nonegotiate
```

## Doğrulama

Tam çıktılar `verification.txt` dosyasında.

| Test | Beklenen | Sonuç |
|---|---|---|
| PC1 → 192.168.10.1 (kendi gateway'i) | Geçer, `ttl=255` | ✅ |
| PC1 → PC2 (VLAN 10 → 20, aynı switch) | Geçer, `ttl=63` | ✅ |
| PC1 → PC3 (VLAN 10 içi, aynı switch) | Geçer, `ttl=64` | ✅ |
| PC1 → PC4 (VLAN 10 → 20) | Geçer, `ttl=63` | ✅ ilk paket ARP yüzünden timeout |
| PC1 → PC8 (VLAN 10 → 20, iki switch aşıyor) | Geçer, `ttl=63` | ✅ |
| PC1 → PC7 (VLAN 10 içi, iki switch aşıyor) | Geçer, `ttl=64` | ✅ |
| `trace` PC1 → PC2 | İlk atlama 192.168.10.1 | ✅ |

TTL okuması bu labın bütün hikâyesi:

```
ttl=255   →  hedef router'ın kendisi (IOS kendi ürettiği yanıtı 255 ile gönderir)
ttl=64    →  paket hiç router'a uğramadı, L2'de kaldı
ttl=63    →  tam bir router atlaması yapıldı
```

İki switch aşan trafikte de `ttl=63` çıkması, switch'lerin TTL alanına
dokunmadığının doğrudan kanıtı.

## Ne öğrendim

**1. Alt arayüz sırası katı: `encapsulation` → `ip address`.**
Kapsülleme tanımlamadan IP vermeye çalıştığımda IOS doğrudan reddetti:

```
% Configuring IP routing on a LAN subinterface is only allowed if that
  subinterface is already configured as part of an IEEE 802.1Q vlan.
```

Alt arayüz, hangi VLAN etiketini taşıyacağını bilmeden IP taşıyamaz.

**2. Alt arayüzün durumu fiziksel arayüzden miras alınır.**
Her şeyi doğru yazdığım hâlde `show ip interface brief` çıktısında hepsi
`administratively down` görünüyordu. Sebep: dynamips router'lar arayüzleri kapalı
açılıyor ve ben Fa0/0'da `no shutdown` yapmamıştım. Alt arayüzlerde ayrıca
`no shutdown` gerekmez — fiziği açınca hepsi birden kalkar.

**3. Native VLAN alt arayüzüne IP verilmez.**
`ip address native` diye bir komut yok. `native` anahtar kelimesi
`encapsulation dot1Q 999 native` satırının parçası. Bu alt arayüz yalnızca
etiketsiz çerçevelerin hangi VLAN'a ait sayılacağını belirtmek için var,
IP taşımaz.

**4. Gateway adresi ağ adresi olamaz.**
`ip address 192.168.20.0 255.255.255.0` yazıp arayüzün neden `unassigned`
kaldığını aradım. `.0` ağın kendisinin adresi, bir arayüze atanamaz —
kullanılacak ilk host adresi `.1`.

**5. İlk ping'in timeout olması normal.**
PC4'e ilk ping'de bir timeout, sonrasında temiz yanıtlar aldım. Gateway ARP
çözümlemesi ilk paketin süresini aşıyor; bu bir arıza değil.

**6. Bu tasarımın darboğazı tek hat.**
VLAN 10'dan VLAN 20'ye giden her paket SW1–R1 hattından **iki kez** geçiyor:
bir kez yukarı, bir kez aşağı. 8 host ile sorun değil ama trafik arttığında
bu hat doyar — bir sonraki labın (L3 switch SVI) var oluş sebebi tam olarak bu.

## Kırmalar

<!-- Üç kırmayı da yaptın ve geri aldın. Her birinin gözlemini buraya tek
     cümleyle yaz, sonra bu yorum satırını sil. Çıktılar verification.txt'de. -->

| Kırma | Yapılan | Gözlem |
|---|---|---|
| 1 | | |
| 2 | | |
| 3 | | |

## Sınavda

- Router-on-a-stick'te fiziksel arayüze **IP verilmez**; IP'ler alt arayüzlerdedir.
  Sorularda fiziksel arayüzde IP gören şıkkı eleyeceksin.
- `encapsulation dot1Q <vlan>` komutu `ip address`'ten **önce** gelmek zorunda.
- `native` anahtar kelimesi `encapsulation` satırındadır; native VLAN alt arayüzü
  IP taşımaz.
- Switch tarafındaki port **trunk** olmalı, access değil. Router-on-a-stick'in
  en sık sorulan arıza senaryosu budur.
- TTL okuması: `255` = router'ın kendisi, `64` = L2'de kaldı, `63` = bir router
  atlaması. Switch TTL'i düşürmez.
- Bu tasarımın sınırı tek fiziksel hat; CCNA bunu "router-on-a-stick darboğazı"
  olarak sorar ve çözüm olarak L3 switch SVI'yi bekler.

## Labı kendin çalıştırmak

`configs/` altındaki dosyalar düz IOS konfigürasyonlarıdır; GNS3'te topolojiyi
kurup konsoldan yapıştırman yeterli. Depoda IOS imajı **yok** ve olmayacak —
Cisco IOS imajları Cisco'nun lisansına tabidir, dağıtılamaz. İmajları kendi
lisanslı kaynağından (Cisco VIRL/CML, DevNet, GNS3 marketplace'teki açık
cihazlar) temin etmen gerekir.
