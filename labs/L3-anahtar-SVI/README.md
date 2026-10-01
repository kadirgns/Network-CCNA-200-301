
# L3 Anahtar SVI — Yönlendirmeyi Switch'e Taşımak

**Platform:** GNS3 · 2 × Cisco IOSvL2 15.2 · 1 × Cisco 3745 (IOS 12.4) · 8 × VPCS
*

## Amaç

Lab 02'de VLAN'lar arası her paket SW1–R1 hattından iki kez geçiyordu — tek
fiziksel hat bütün trafiğin darboğazıydı. Bu labda yönlendirme switch'in kendisine
taşınıyor. R1 ağdan çıkmıyor; rolü değişiyor ve kenar yönlendirici oluyor.

Kanıtlanacak dört şey:

1. VLAN'lar arası trafik artık SW1 üzerinde dönüyor, R1'e hiç uğramıyor.
2. **R1 tamamen kapatıldığında bile VLAN'lar arası iletişim sürüyor** — lab 02'de
   bu imkânsızdı.
3. Dışarıya (R1'in loopback'i) erişim R1'e bağlı ve R1 kapanınca kesiliyor.
4. PC'lerde tek satır değişiklik gerekmiyor; gateway IP'leri aynı kalıyor,
   yalnızca o IP'ye cevap veren cihaz değişiyor.

## Topoloji

![Topoloji](topology-b04-03.png)

| Cihaz | Port | Bağlantı | Rol |
|---|---|---|---|
| SW1 | Gi2/0 | R1 Fa0/0 | **routed port**, `no switchport`, 10.0.0.1/30 |
| SW1 | Vlan10 | — | SVI, 192.168.10.1/24 |
| SW1 | Vlan20 | — | SVI, 192.168.20.1/24 |
| SW1 | Gi0/0 | SW2 Gi0/0 | 802.1Q trunk, native 999, allowed 10,20 |
| SW1 | Gi0/1–Gi0/3, Gi1/0 | PC1–PC4 | access |
| SW2 | Gi0/0 | SW1 Gi0/0 | 802.1Q trunk |
| SW2 | Gi0/1–Gi0/3, Gi1/0 | PC5–PC8 | access |
| R1 | Fa0/0 | SW1 Gi2/0 | 10.0.0.2/30 |
| R1 | Loopback0 | — | 203.0.113.1/32, internet simülasyonu |

### Adresleme

| Ağ | Maske | Gateway | Not |
|---|---|---|---|
| 192.168.10.0 | /24 | 192.168.10.1 (SW1 Vlan10) | VLAN 10 — PC1, PC3, PC5, PC7 |
| 192.168.20.0 | /24 | 192.168.20.1 (SW1 Vlan20) | VLAN 20 — PC2, PC4, PC6, PC8 |
| 10.0.0.0 | /30 | — | SW1 ↔ R1 nokta-nokta |
| 203.0.113.1 | /32 | — | R1 Loopback0 |

`/30` dört adresten ikisini kullanılabilir bırakır; nokta-nokta hatların standardı.
`203.0.113.0/24` ise RFC 5737'de dokümantasyon için ayrılmış bir blok — gerçek
internet adresi olmadığı için örneklerde güvenle kullanılır.

## Yapılandırma

Tam konfigürasyonlar `configs/` klasöründe. Kritik parçalar:

**SW1 — L3 anahtar**

```
ip routing
!
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
!
interface Vlan20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet2/0
 no switchport
 ip address 10.0.0.1 255.255.255.252
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.0.0.2
```

**R1 — kenar yönlendirici**

```
interface FastEthernet0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 203.0.113.1 255.255.255.255
!
ip route 192.168.10.0 255.255.255.0 10.0.0.1
ip route 192.168.20.0 255.255.255.0 10.0.0.1
```

Yönlendirme asimetrik ve bu kasıtlı: SW1 "bilmediğim her yer R1'e" diyor (varsayılan
rota), R1 ise iç ağları tek tek öğreniyor. Gerçek kampüs tasarımının aynısı — kenar
cihaz iç topolojiyi bilir, iç cihaz dış dünyayı bilmek zorunda değildir.

Lab 02'den sökülenler: R1'in `Fa0/0.10`, `Fa0/0.20`, `Fa0/0.999` alt arayüzleri ve
SW1 Gi2/0'ın trunk yapılandırması.

## Doğrulama

Tam çıktılar `verification.txt` dosyasında.

| Test | Beklenen | Sonuç |
|---|---|---|
| PC1 → PC2 (VLAN 10 → 20) | `ttl=63` | ✅ ~2.2 ms |
| PC1 → PC3 (VLAN 10 içi) | `ttl=64` | ✅ ~1.5 ms |
| PC1 → 203.0.113.1 | `ttl=254` | ✅ ~30 ms |
| `trace` PC1 → PC2 | tek atlama, `192.168.10.1` | ✅ |
| `trace` PC1 → 203.0.113.1 | `192.168.10.1` → `10.0.0.2` | ✅ |
| **R1 kapalı** → PC1 → PC2 | hâlâ geçer | ✅ `ttl=63` |
| **R1 kapalı** → PC1 → 203.0.113.1 | ölür | ✅ timeout |

Son iki satır bu labın tamamı. Lab 02'de R1 kapandığında VLAN'lar arası iletişim
de ölürdü; artık ölmüyor.

SW1'in rota tablosu:

```
Gateway of last resort is 10.0.0.2 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 10.0.0.2
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.0.0/30 is directly connected, GigabitEthernet2/0
L        10.0.0.1/32 is directly connected, GigabitEthernet2/0
      192.168.10.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.10.0/24 is directly connected, Vlan10
L        192.168.10.1/32 is directly connected, Vlan10
      192.168.20.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.20.0/24 is directly connected, Vlan20
L        192.168.20.1/32 is directly connected, Vlan20
```

Bu çıktıda `L` satırları var. R1'in aynı komutu `L` üretmiyor — local route kodu
IOS 15.x ile geldi, R1 ise 12.4. İki platformu yan yana çalıştırmanın yan faydası.

## Ne öğrendim

**1. TTL'i mutlak değerinden değil, başlangıç değerinden eksilme olarak okumak gerekiyor.**
`203.0.113.1`'e ping attığımda `ttl=254` geldi; ben `62` bekliyordum. Hata benim
varsayımımdaydı: cevabı R1'in kendisi üretiyor ve IOS kendi ürettiği paketleri
TTL **255** ile gönderiyor. Dönüşte yalnızca SW1'i aşıyor → 254.

| Görünen | Cevabı üreten | Aşılan router |
|---|---|---|
| `ttl=64` | host (VPCS) | 0 |
| `ttl=63` | host (VPCS) | 1 |
| `ttl=255` | router | 0 |
| `ttl=254` | router | 1 |

Doğru soru "TTL kaç" değil, **"bu cevabı kim üretti"**. Sonra eksilmeyi sayarsın.

**2. IOS, bir sonraki atlamanın kendi arayüzü olmasına izin vermiyor.**

```
SW1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.2
%Invalid next hop address (it's this router)
```

Mesaj rotayı işaret ediyordu ama hata arayüzdeydi: Gi2/0'a `.1` yerine `.2`
yazmıştım. IOS "kendi kendine paket gönderemezsin" diyordu. Bu mesajı gördüğünde
iki yerden biri yanlıştır — ya next-hop ya da arayüzün kendi adresi.

**3. `no switchport` arayüzü resetliyor.**

```
SW1(config-if)# no switchport
*Oct  1 10:23:19: %LINK-3-UPDOWN: Interface GigabitEthernet2/0, changed state to up
```

Port L2'den L3'e geçerken down/up yapıyor. Beklenen davranış, panik gerektirmiyor —
ama aynı anda o portun trunk yapılandırması da siliniyor, yani karşı taraf
(bu labda R1'in alt arayüzleri) o anda kopuyor.

**4. SVI'lar kapalı doğar.**
Her `interface Vlan<x>` için ayrı `no shutdown` gerekiyor. Lab 02'de alt arayüzler
durumu fiziksel arayüzden miras alıyordu; SVI'larda böyle bir miras yok.

**5. VPCS ayarları kendiliğinden kalıcı değil.**
Testlerin ortasında PC1 gateway'ine ulaşamaz oldu. Sebep ARP önbelleği değildi —
node yeniden başlayınca VPCS konfigürasyonunu tamamen kaybetmişti. Çözüm adresi
yeniden vermek, kalıcı çözüm her PC'de `save` komutunu çalıştırmak.

**6. Trace çıktısındaki "Destination port unreachable" hata değil, başarı işareti.**

```
PC1> trace 192.168.20.12
 1   192.168.10.1   1.667 ms  0.862 ms  1.101 ms
 2   *192.168.20.12   4.804 ms (ICMP type:3, code:3, Destination port unreachable)
```

VPCS trace'i UDP paketi yollar; hedefe vardığında karşı taraf "o portta dinleyen
yok" der. Bu mesaj gelmişse yol tamamlanmıştır.

**7. Trace, hedefin kendi adresini değil, cevabın çıktığı arayüzü gösterir.**
`trace 203.0.113.1` çıktısında son atlama `10.0.0.2` yazıyor, `203.0.113.1` değil.
R1 cevabı gönderene en yakın arayüzünden kaynaklandırıyor. Loopback'lere yapılan
trace'lerde sık görülür ve arıza sanılıp zaman kaybettirir.

.

## Labı kendin çalıştırmak

`configs/` altındaki dosyalar düz IOS konfigürasyonlarıdır; GNS3'te topolojiyi
kurup konsoldan yapıştırman yeterli.
