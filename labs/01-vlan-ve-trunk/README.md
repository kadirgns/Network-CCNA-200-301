# VLAN ve 802.1Q Trunk


**Platform:** GNS3 · 2 × Cisco IOSvL2 15.2 · 4 × VPCS

### Labı kendin çalıştırmak

`lab-b04-01-vlan-trunk.gns3project` dosyasını indirip GNS3 ile aç.

## Amaç

İki switch arasında tek fiziksel hat üzerinden iki ayrı broadcast domain taşımak.
Aynı VLAN'daki hostların switch'ler arası haberleştiğini, farklı VLAN'dakilerin
yönlendirme olmadan haberleşemediğini kanıtlamak.

Bu labda bilinçli olarak router yok. VLAN'lar arası geçişin olmaması hedeflenen
sonuçtur; yönlendirme bir sonraki labın konusu.

## Topoloji


| Cihaz | Port | Bağlantı | Rol |
|---|---|---|---|
| SW1 | Gi0/0 | SW2 Gi0/0 | 802.1Q trunk |
| SW1 | Gi0/1 | PC1 | access, VLAN 10 |
| SW1 | Gi0/2 | PC2 | access, VLAN 20 |
| SW2 | Gi0/0 | SW1 Gi0/0 | 802.1Q trunk |
| SW2 | Gi0/1 | PC3 | access, VLAN 10 |
| SW2 | Gi0/2 | PC4 | access, VLAN 20 |

### Adresleme

| VLAN | Ad | Ağ | Hostlar |
|---|---|---|---|
| 10 | OFIS | 192.168.10.0/24 | PC1 `.11`, PC3 `.13` |
| 20 | MUHASEBE | 192.168.20.0/24 | PC2 `.12`, PC4 `.14` |
| 999 | NATIVE-KULLANILMIYOR | — | trunk native VLAN, hiçbir porta atanmaz |

Native VLAN'ın 1 yerine kullanılmayan bir VLAN'a (999) taşınması bilinçli bir
tercih: VLAN hopping saldırısının çift etiketleme varyantını zorlaştırır.

## Yapılandırma

Tam konfigürasyonlar `configs/` klasöründe. Özet:

- Her iki switch'te VLAN 10, 20, 999 tanımlı ve isimlendirilmiş
- Erişim portlarında `switchport mode access`, DTP kapalı, PortFast açık
- Trunk: statik (`mode trunk` + `nonegotiate`), native VLAN 999, izinli VLAN 10 ve 20
- Kullanılmayan portlar kapatılmış ve VLAN 999'a alınmış
- Temel sertleştirme: `enable secret`, `service password-encryption`, konsol ve VTY parolası, banner

## Doğrulama

Tam çıktılar `verification.txt` dosyasında.

| Test | Beklenen | Sonuç |
|---|---|---|---|
| PC1 → PC3 (aynı VLAN, trunk üzerinden) | Geçer | 
| PC2 → PC4 (aynı VLAN, trunk üzerinden) | Geçer | 
| PC1 → PC2 (farklı VLAN, yönlendirme yok) | no gateway found |

Trunk durumu:

```
Port        Mode             Encapsulation  Status        Native vlan
Gi0/0       on               802.1q         trunking      999

Port        Vlans allowed on trunk
Gi0/0       10,20

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/0       10,20
```

`Mode: on` statik trunk demek — DTP pazarlığı devre dışı.

## Ne öğrendim

**1. `switchport trunk encapsulation dot1q`, `switchport mode trunk`'tan önce gelmeli.**
ISL'i de destekleyen bir switch'te kapsülleme `auto` iken port trunk moduna alınamaz.
Bu satırı atladığımda `switchport mode trunk` sessizce reddedildi.

**2. IOS hata mesajı sonucu gösterir, sebebi bir önceki adımdadır.**

```
Command rejected: Conflict between 'nonegotiate' and 'dynamic' status on this interface: Gi0/0
```

Mesaj `nonegotiate`'i işaret ediyordu ama asıl sorun portun hâlâ dynamic olmasıydı;
yani ondan önceki `mode trunk` komutu uygulanmamıştı. Hatayı son komutta değil
zincirin başında aramak gerekiyor.

**3. Trunk portu `show vlan brief` listesinde görünmez.**
Sorunu ilk fark ettiğim yer burasıydı: Gi0/0, VLAN 1'in altında listeleniyordu ve
`show interfaces status` çıktısında Vlan sütununda `trunk` yerine `1` yazıyordu.



**4. Native VLAN uyumsuzluğunda CDP uyarır, trafiği durdurmaz.**

```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on
GigabitEthernet0/0 (1), with SW2 GigabitEthernet0/0 (999).
```

Link kapanmadı. Native VLAN'daki etiketsiz çerçeveler sessizce karşı tarafın
native VLAN'ına düşer — bu yüzden uyarı görmezden gelinecek bir şey değil.

**5. `switchport trunk allowed vlan <x>` listenin üstüne yazar.**
<!-- Kırma 2'yi yaptıktan sonra kendi gözlemini buraya yaz -->
