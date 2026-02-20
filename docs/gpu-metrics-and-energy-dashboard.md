# GPU metrik toplama analizi ve ayrı enerji dashboard'u

## Repoda mevcut durum

Bu repository içinde GPU metriklerini toplayan bir Helm values, ServiceMonitor veya exporter manifesti bulunmuyor. Repo şu an ağırlıklı olarak Rancher/Prometheus HA dokümantasyonu içeriyor.

- İncelenen ana dosya: `docs/rancher-2.13.1-prometheus-ha-best-practices.md`
- Sonuç: GPU metrikleri (DCGM/NVIDIA) ile ilgili bir toplama kuralı bu repoda tanımlı değil.

## GPU enerji hesabı için önerilen metrik

Ayrı dashboard için aşağıdaki metrik tercih edilmelidir:

- `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` (genellikle **mJ** cinsinden monoton artan sayaç)

Seçili zaman aralığındaki tüketim (kWh):

```promql
increase(DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION[$__range]) / 3.6e9
```

> Not: Eğer bu metrik yoksa `DCGM_FI_DEV_POWER_USAGE` (W) üzerinden integral yaklaşımıyla yaklaşık enerji hesaplanabilir; ancak sayaç metrik kadar doğru/kolay değildir.

## Eklenen dashboard

`dashboards/gpu-energy-consumption-dashboard.json` dosyası eklendi. Dashboard, seçili zaman aralığına göre enerji tüketimini şu kırılımlarda **tablo** olarak verir:

1. **Total** (tüm cluster)
2. **Node bazlı**
3. **GPU bazlı** (node + gpu + UUID)

Bu dashboard mevcut Grafana'ya import edilerek doğrudan kullanılabilir. Gerekirse label adları (`kubernetes_node`, `gpu`, `UUID`) ortamınızdaki gerçek label setine göre düzenlenmelidir.
