# Rancher 2.13.1 + HA Cluster için Prometheus Kurulumu (Best Practice)

Bu doküman, **Rancher 2.13.1** ortamında, HA (yüksek erişilebilir) Kubernetes cluster'larında Prometheus tabanlı izleme kurulumunda önerilen yaklaşımı özetler.

## 1) Sürüm uyumluluğu: en güvenli yol

Rancher ekosisteminde en güvenli ve desteklenebilir yöntem:

1. **Rancher UI > Apps/Marketplace** içinden gelen **`rancher-monitoring`** chart'ını kullanmak
2. Chart sürümünü, cluster'ın Kubernetes sürümüyle uyumlu olan seçenekten seçmek
3. Manuel olarak upstream `kube-prometheus-stack` chart'ına geçmeden önce operasyonel gereksinimi netleştirmek

> Özet: Rancher 2.13.1 ile "best practice", doğrudan Rancher'ın sunduğu ve test ettiği `rancher-monitoring` paketini kullanmaktır.

## 2) Hangi kurulum modeli?

### Model A (önerilen): Rancher Monitoring (`rancher-monitoring`)
- **Artıları:** Rancher ile entegre, RBAC/CRD/operatör akışı daha sorunsuz, yükseltme ve bakım kolay
- **Eksileri:** Upstream'e göre bazı özellikler chart sürümüne bağlı gelir

### Model B: Upstream `kube-prometheus-stack`
- **Artıları:** En güncel upstream özelliklere hızlı erişim
- **Eksileri:** Rancher ile destek matrisi/operasyon yükü daha kritik, CRD geçişlerinde dikkat gerekir

HA ve kurumsal işletim için Model A genellikle daha düşük risklidir.

## 3) HA tasarımı (Prometheus + Alertmanager)

Minimum HA önerisi:

- **Prometheus replicas:** `2`
- **Alertmanager replicas:** `3` (quorum için tek sayılar)
- **PodAntiAffinity:** aynı node'a düşmeyi engelle
- **Topology spread constraints:** zone/node dağılımı
- **PDB (PodDisruptionBudget):** bakım sırasında hizmet sürekliliği
- **Resources requests/limits:** OOM/CPU throttling riskini azalt

Not: Tek başına 2 Prometheus replikası, veri yazımı açısından çift ingest demektir. Uzun süreli saklama ve global sorgu için Thanos mimarisi düşünülmelidir.

## 4) PVC ve `block trident-ontap-san-ext4` kullanımı

Sizin senaryonuzda StorageClass olarak `trident-ontap-san-ext4` kullanılacaksa:

- **Prometheus ve Alertmanager için ayrı PVC** tanımlayın
- Access mode çoğunlukla **`ReadWriteOnce`** olur (SAN block için tipik)
- `volumeBindingMode: WaitForFirstConsumer` olması planlamayı iyileştirir (StorageClass tarafında)
- Disk performansı için IOPS/latency değerlerini kapasite planına dahil edin

Örnek values yaklaşımı (chart yapısına göre alan adları küçük fark gösterebilir):

```yaml
prometheus:
  prometheusSpec:
    replicas: 2
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: trident-ontap-san-ext4
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi

alertmanager:
  alertmanagerSpec:
    replicas: 3
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: trident-ontap-san-ext4
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi
```

## 5) Retention ve kapasite planı

- Başlangıç için `15d-30d` retention (lokal TSDB) pratik bir başlangıçtır
- Uzun süreli saklama gerekiyorsa:
  - Thanos sidecar + object storage (S3 uyumlu)
  - Prometheus yerel diskte daha kısa retention
- Disk boyutu hesabı için:
  - Hedef seri sayısı (cardinality)
  - scrape interval
  - label disiplininiz (high-cardinality label'ları sınırlayın)

## 6) Güvenlik ve operasyon

- **NetworkPolicy** ile Prometheus erişimini sınırlandırın
- **TLS** (ingress/internal) ve kimlik doğrulama ekleyin
- **kube-state-metrics/node-exporter** sürümlerini chart ile birlikte yönetin
- Upgrade öncesi CRD değişikliklerini release notes'tan kontrol edin

## 7) Uygulama adımları (özet runbook)

1. Rancher'da ilgili cluster'ı açın
2. Apps/Marketplace'ten `rancher-monitoring` chart'ını seçin
3. Kubernetes sürümünüzle uyumlu chart sürümünü seçin
4. Advanced values içine:
   - HA replica ayarları
   - `trident-ontap-san-ext4` PVC ayarları
   - resource requests/limits
5. Kurulumu tamamlayın
6. Doğrulama:
   - `kubectl get pods -n cattle-monitoring-system`
   - `kubectl get pvc -n cattle-monitoring-system`
   - Prometheus/Alertmanager target health

## 8) Sahada önerilen başlangıç profili

- Prometheus: `2 replicas`, `100Gi`, retention `15d`
- Alertmanager: `3 replicas`, `20Gi`
- Scrape interval: kritik olmayan işler için 30s
- Kural setleri: önce temel platform alarmları, sonra uygulama özel kurallar

## 9) Kaçınılması gerekenler

- Tüm workload'ları 15s scrape etmek (maliyet/cardinality patlatır)
- High-cardinality label kullanımı (`pod_uid`, `request_id` vb.)
- StorageClass/IOPS kapasitesini ölçmeden retention büyütmek
- Upstream chart'a direkt geçip Rancher operasyon modelini göz ardı etmek

## 10) Kısa karar

Rancher 2.13.1 + HA cluster için en uyumlu ve düşük riskli yaklaşım:

- **Rancher Monitoring (`rancher-monitoring`) chart'ı**
- **HA replica + anti-affinity + PDB**
- **PVC: `trident-ontap-san-ext4` (RWO)**
- **Retention'i kontrollü başlat, gerekiyorsa Thanos ile uzun süreli saklamaya geç**


## 11) Model A içinde Thanos dahili yapılandırma yapılabilir mi?

**Evet, yapılabilir.** `rancher-monitoring` (Prometheus Operator tabanlı) kurulumunda, chart sürümünüz destekliyorsa `prometheus.prometheusSpec.thanos` alanı ile sidecar yaklaşımı uygulanabilir.

Pratik akış:

1. Object storage bilgileri için Secret oluşturun (S3/MinIO vb.)
2. Values içinde `prometheus.prometheusSpec.thanos.objectStorageConfig` ile bu Secret'ı referanslayın
3. Global sorgu için ayrı bir **Thanos Query** bileşeni (Deployment) kurun
4. Retention'i Prometheus'ta kısa tutup (ör. 7-15 gün), uzun süreli veriyi object storage'a taşıyın

Örnek (alan adları chart sürümüne göre değişebilir):

```yaml
prometheus:
  prometheusSpec:
    thanos:
      objectStorageConfig:
        key: thanos.yaml
        name: thanos-objstore-secret
```

> Notlar:
> - Rancher'daki `rancher-monitoring` chart versiyonunda bu alanın bulunduğunu **Values/Questions** ekranında doğrulayın.
> - Sadece sidecar açmak yeterli değildir; merkezi sorgu için Thanos Query katmanı gerekir.
> - Ağ politikalarında Prometheus pod'ları ile Thanos Query arasındaki erişimi izinli hale getirin.

### 11.1 Yetkinlik ve gerçekçi sınırlar

- **Yetkinlik:** Evet, bu mimari uygulanabilir ve sahada yaygın bir pattern'dir (Prometheus Operator + Thanos sidecar + Query + object storage).
- **Sınır:** `rancher-monitoring` chart sürümüne göre alan adları/varsayılanlar değişebilir. Bu yüzden nihai doğrulama her zaman ilgili chart'ın **Values/Questions** çıktısında yapılmalıdır.

### 11.2 Model A + Thanos referans mimarisi

Model A içinde önerilen minimal uzun süreli saklama mimarisi:

1. `rancher-monitoring` içinde Prometheus HA (`replicas: 2`)
2. Her Prometheus pod'unda Thanos sidecar (`prometheus.prometheusSpec.thanos`)
3. S3 uyumlu object storage (ONTAP S3/MinIO/AWS S3)
4. Ayrı bir **Thanos Query** deployment (genellikle `monitoring` namespace)
5. (Opsiyonel ama önerilir) Store Gateway + Compactor + Ruler

> Üretimde sadece sidecar + query ile başlanabilir; veri hacmi ve sorgu sayısı arttıkça Store Gateway/Compactor eklenir.

### 11.3 Adım adım uygulama (tam akış)

#### Adım 1 — Object storage Secret hazırlığı

`thanos.yaml` içeriğini önce yerel oluşturun (S3 örneği):

```yaml
type: S3
config:
  bucket: thanos-metrics
  endpoint: s3.example.local
  region: us-east-1
  access_key: YOUR_ACCESS_KEY
  secret_key: YOUR_SECRET_KEY
  insecure: false
```

Secret oluşturun:

```bash
kubectl -n cattle-monitoring-system create secret generic thanos-objstore-secret \
  --from-file=thanos.yaml=./thanos.yaml
```

#### Adım 2 — Rancher Monitoring values güncellemesi

Advanced Values içinde (alanlar chart sürümüne göre farklı olabilir):

```yaml
prometheus:
  thanosIngress:
    enabled: false # dış erişim gerekiyorsa true + TLS + allowlist

  prometheusSpec:
    replicas: 2

    # Dedup ve çoklu Prometheus ayrımı için kritik
    replicaExternalLabelName: prometheus_replica

    externalLabels:
      cluster: prod-ha-1

    thanos:
      image: quay.io/thanos/thanos:v0.36.1
      version: v0.36.1
      objectStorageConfig:
        name: thanos-objstore-secret
        key: thanos.yaml
      resources:
        requests:
          cpu: 200m
          memory: 256Mi
        limits:
          cpu: "1"
          memory: 1Gi

    retention: 15d
```

#### Adım 3 — Thanos Query kurulumu

Thanos Query bileşenini ayrı deployment olarak kurun (Helm chart veya manifest). Örnek argümanlar:

```bash
--query.replica-label=prometheus_replica
--store=dnssrv+_grpc._tcp.prometheus-operated.cattle-monitoring-system.svc.cluster.local
```

> `--query.replica-label`, HA Prometheus verisinde dedup için kritik parametredir.

#### Adım 4 — (Önerilen) Store Gateway + Compactor

- **Store Gateway:** Object storage içindeki historical block'ları query katmanına açar.
- **Compactor:** Block birleştirme/downsampling yapar, maliyeti düşürür.

Retention politikası örneği:
- Prometheus local TSDB: `7-15d`
- Object storage: `180d+` (regülasyon ihtiyacına göre)

#### Adım 5 — NetworkPolicy ve güvenlik

Aşağıdaki trafiği açık bırakın:
- Thanos Query -> Prometheus sidecar gRPC
- Thanos Query/Store/Compactor -> Object storage endpoint

Ek güvenlik kontrolleri:
- S3 endpoint TLS doğrulama
- Secret rotasyonu
- Public ingress yerine internal LB/ServiceMesh tercih

### 11.4 Doğrulama checklist'i

1. Prometheus CR içinde thanos alanı işlenmiş mi?
```bash
kubectl -n cattle-monitoring-system get prometheus -o yaml | rg -n "thanos|replicaExternalLabelName|externalLabels"
```
2. Sidecar container pod'a eklenmiş mi?
```bash
kubectl -n cattle-monitoring-system get pod -l app.kubernetes.io/name=prometheus -o jsonpath='{range .items[*]}{.metadata.name}{" => "}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
```
3. Object storage'a block düşüyor mu?
- Bucket altında `01...` formatında TSDB block klasörleri görünmeli.
4. Query dedup çalışıyor mu?
- Aynı metrikte çift seri yerine tekleştirilmiş sonuç alınmalı (`query.replica-label`).

### 11.5 Sık yapılan hatalar ve çözüm

- **Hata:** `objectStorageConfig` secret key adı yanlış (`thanos.yaml` yerine başka key)
  - **Çözüm:** Secret key ile values'taki `key` birebir aynı olmalı.
- **Hata:** Sidecar var ama Query yok
  - **Çözüm:** Merkezi sorgu için Thanos Query deployment zorunlu.
- **Hata:** Dedup yapılmıyor
  - **Çözüm:** `externalLabels` + `replicaExternalLabelName` + Query `--query.replica-label` birlikte doğrulanmalı.
- **Hata:** Compactor yok, sorgu yavaş ve maliyet yüksek
  - **Çözüm:** Üretimde Store Gateway + Compactor katmanını ekleyin.

### 11.6 Operasyonel best-practice özeti (Model A + Thanos)

- Rancher Monitoring'i ana kontrol düzlemi olarak tutun.
- Thanos'u önce sidecar + query ile küçük başlatın.
- `trident-ontap-san-ext4` üzerinde Prometheus local retention'i kısa tutun.
- Uzun süreli saklamayı object storage'a verin.
- Düzenli olarak:
  - bucket growth,
  - compaction health,
  - query latency,
  - cardinality trendlerini izleyin.
