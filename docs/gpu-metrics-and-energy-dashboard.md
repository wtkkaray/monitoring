# GPU metrik toplama analizi ve ayrı enerji dashboard'u

## Repoda mevcut durum

Bu repository içinde GPU metriklerini toplayan bir Helm values, ServiceMonitor veya exporter manifesti bulunmuyor. Repo şu an ağırlıklı olarak Rancher/Prometheus HA dokümantasyonu içeriyor.

- İncelenen ana dosya: `docs/rancher-2.13.1-prometheus-ha-best-practices.md`
- Sonuç: GPU metrikleri (DCGM/NVIDIA) ile ilgili bir toplama kuralı bu repoda tanımlı değil.

## 30 dakika için görülen değerler mantıklı mı?

Kısa cevap: **Hayır, paylaşılan büyüklükler büyük olasılıkla hatalı/şişkin.**

Örnek hızlı üst sınır kontrolü:
- 8 GPU x 700W x 0.5 saat ≈ **2.8 kWh / node**
- 10 node için bile ≈ **28 kWh / 30 dk**

`280921 kWh / 30 dk` ölçeği fiziksel olarak gerçekçi değildir. En yaygın nedenler:

1. Aynı GPU serisinin birden fazla kez scrape edilmesi (replica/federation/remote-read)
2. Yanlış metrik tipi veya birim varsayımı
3. Label kırılımında node etiketinin boş kalması

## Kullanılan metrik ve formül

- Metrik: `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` (genellikle mJ sayaç)
- Formül: `increase(DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION[$__range]) / 3.6e9` (kWh)

## Bu revizyonda conflict ve uyumluluk için yapılanlar

- Sorgularda gereksiz subquery kullanımı (`[$__range:]`) kaldırıldı, standart `[$__range]` kullanıldı.
- Tekilleştirme yaklaşımı korunarak `(node, UUID)` bazında `max by` ile mükerrer seri etkisi azaltıldı.
- Node label fallback sırası korundu: `kubernetes_node` -> `Hostname` -> `instance`.
- GPU panel legend alanı normalize `node` etiketine geçirildi.

## Dashboard kapsamı

`dashboards/gpu-energy-consumption-dashboard.json` aşağıdaki görünümü sunar:

1. Toplam enerji (Total)
2. Node bazlı tablo
3. GPU bazlı tablo (Node + GPU + UUID)

## Best-practice kontrol listesi

- Aynı UUID’nin aynı anda kaç seri ürettiğini doğrulayın.
- Önce tek job filtresi ile ölçüm alın, sonra genişletin.
- Büyük sapmada Prometheus UI’da job/instance kırılımı ile karşılaştırın.
