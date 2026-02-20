# GPU metrik toplama analizi ve ayrı enerji dashboard'u

## Repoda mevcut durum

Bu repository içinde GPU metriklerini toplayan bir Helm values, ServiceMonitor veya exporter manifesti bulunmuyor. Repo şu an ağırlıklı olarak Rancher/Prometheus HA dokümantasyonu içeriyor.

- İncelenen ana dosya: `docs/rancher-2.13.1-prometheus-ha-best-practices.md`
- Sonuç: GPU metrikleri (DCGM/NVIDIA) ile ilgili bir toplama kuralı bu repoda tanımlı değil.

## 30 dakika için görülen değerler mantıklı mı?

Kısa cevap: **Hayır, paylaşılan büyüklükler büyük olasılıkla hatalı/şişkin.**

Örnek hızlı üst sınır kontrolü:
- 8 GPU x 700W (çok yüksek kabul) x 0.5 saat ≈ **2.8 kWh / node**
- 10 node için bile ≈ **28 kWh / 30 dk**

Sizde görülen `280921 kWh / 30 dk` seviyeleri fiziksel olarak gerçekçi değildir. Bu tip sapma genelde şu nedenlerden çıkar:

1. Aynı GPU serisinin birden fazla kez scrape edilmesi (replica/federation/remote-read tekrarları)
2. Yanlış metrik tipi veya birim varsayımı
3. Label kırılımında node etiketinin boş kalması ve yanlış gruplama

## GPU enerji hesabı için önerilen metrik

Ayrı dashboard için aşağıdaki metrik tercih edilmelidir:

- `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` (genellikle **mJ** cinsinden monoton artan sayaç)

Seçili zaman aralığındaki tüketim (kWh):

```promql
increase(DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION[$__range]) / 3.6e9
```

## Bu revizyondaki iyileştirmeler

Dashboard sorguları, mükerrer seri etkisini azaltmak için `(node, UUID)` bazında tekilleştirildi (`max by`).
Ayrıca node etiketi normalize edildi:

1. `kubernetes_node`
2. `Hostname`
3. `instance` (port kesilerek)

Böylece node isimlerinin boş gelmesi büyük oranda önlenir.

## Eklenen dashboard

`dashboards/gpu-energy-consumption-dashboard.json` dosyası ayrı dashboard olarak eklendi. Dashboard seçili zaman aralığına göre enerji tüketimini şu kırılımlarda **okunaklı tablo** olarak verir:

1. **Total** (tüm cluster)
2. **Node bazlı**
3. **GPU bazlı** (Node + GPU + UUID)

## Best-practice notları

- Önce ham metriği doğrulayın:
  - `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION`
  - aynı UUID’nin aynı anda kaç seri ürettiğini kontrol edin
- Tekilleştirme için `max by(node, UUID)` yaklaşımını kullanın.
- Grafana panelinde `kWh` birimini ve 2 ondalık gösterimi tercih edin.
- Büyük sapma varsa aynı sorguyu Prometheus UI’da job/instance kırılımında karşılaştırın.
