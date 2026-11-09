# PPE Safety Monitor — Uçtan Uca Ürünleştirme, ML Kalitesi ve Saha Hazırlık Yol Haritası

**Sürüm:** Öneri v1.0 — 2026-11-02  
**Okuyucular:** Ürün/proje yönetimi, veri mühendisi, ML geliştirici, backend, frontend/UX, iş geliştirme, İSG ve işletme sorumluları.  
**Karar durumu:** Geliştirme planı; üretim/saha GO kararı değildir.  
**Dayanak:** Kullanıcının paylaştığı FAZ 6–10-A/10-A.1 Claude Code raporları ve önceki `PPE_Saha_Hazirlik_Claude_Opus_Plani.md`. Güncel repo doğrudan incelenmedi. Bu yüzden **eksik** görünen yeteneklerin kodda bulunup bulunmadığı her faz başında teyit edilmelidir.

## 0. Başlangıç durumu ve değişmez kurallar

- Proje: Python/Ultralytics YOLO26n (`best.pt`, 11 sınıf), ByteTrack, `PPEMonitor`, `IncidentTracker`, FastAPI, PostgreSQL 17, SQLite WAL durable outbox, WebSocket canlı panel, kullanıcı auth/RBAC, geçmiş olaylar/raporlar, operasyonel metrikler, Docker Compose.
- Son tamamlanmış ve raporlanmış FAZ 10-A commit: `d67854d`. FAZ 10-A.1 Mac ortamında geliştiriliyor; **son HEAD yeniden kontrol edilecek**, commit kimliği varsayılmayacak.
- Son raporlanan FAZ 10-A tabanı: 245/245 detector, 334/334 dashboard, 30/30 browser, 21/21 LAN TLS; FAZ 10-A.1 Mac raporunda 244 PASS + 1 Windows-only SKIP ve 338/338 dashboard; gerçek sonuçlar yeniden teyit edilmeli.
- Dashboard ana yığın yerel/localhost; Windows sahası ve Mac testi ayrıdır. Mac `MPS`, Windows RTX 3050 `CUDA`: platform testleri birbirinin yerine geçmez.
- Eğitim kökenli, kısmen sızıntı içeren açık veri setinde mAP50=0.547 bildirilmiş; saha doğruluğuna genellenmez. `no_helmet` recall=0.23; üretime otomatik eklenmez. Videolarda değerlendirilmeyen track-kare oranı bir örnekte %91; kritik kapsama riski.
- KVKK ve Ultralytics/model lisans uygunluğu **NO-GO**, etiketli gerçek şantiye verisi ve İSG tarafından imzalanmış kabul eşikleri yok; gerçek saha pilotu engellenir.
- Ana veritabanını, snapshotları, SQLite outbox'ları, eski migration'ları, gerçek gizli bilgileri, model dosyasını ve kullanıcı değişikliklerini koru. `git reset --hard`, `git clean`, `docker compose down -v`, `docker volume prune`, toplu `taskkill` yok. `LLM_Insight` ve diğer projelere dokunma.
- Her faz önce mevcut durum/Git/env/test incelemesi yapar. Her faz kendi raporuna PASS/FAIL/SKIP/BLOCKED, komut, kanıt, commit, risk ve sonraki koşulları yazar. Kritikte FAIL olduğunda faz tamamlandı sayılamaz.
- **Güvenlik ilkesi:** Bu çözüm İSG görevlisinin yerine geçmez, çalışanları yüzle tanımaz, otomatik disiplin/ceza vermez, tek başına iş durdurma veya güvenlik kararı almaz. “Kamera/analiz yok” durumunu “ihlal yok” veya “uygun” gibi göstermez.

## 1. Altı rolün sorumluluk sınırı

| Rol | Esas sorumluluk | İnsan onayı / bağımlılık |
|---|---|---|
| Ürün sahibi / Proje yöneticisi | PRD, kullanıcı-persona, öncelik, backlog, release kararı, bütçe, risk sicili | İşletme ve İSG sorumlusu hedefleri imzalar |
| Veri mühendisi | Veri kaynakları, izin ve köken kaydı, annotation şeması, kalite, bölme, sürüm, saklama | Veri sorumlusu/KVKK onayı olmadan kişisel veri toplanmaz |
| AI / ML geliştirici | Baseline, bağımsız test, FP/FN, ID switch, model adayları, MLOps, model card | İSG hedefleri ve insan doğrulamalı ground truth |
| Backend geliştirici | API, PostgreSQL/outbox, RBAC, denetim kaydı, kamera/site kapsamı, health, retention | Güvenlik ve veri politikaları onayları |
| Frontend / mobil geliştirici | Operatör inceleme kuyruğu, alarm açıklaması, kapsama/UNKNOWN, erişilebilirlik, manuel doğrulama | İSG/operatör kullanıcı testleri |
| İş geliştirme & UX | Müşteri görüşmeleri, fiyat, değer önerisi, TCO, demo, ticari hipotezler | Yasal dağıtım ve gizlilik kararı |

**Ek zorunlu paydaşlar:** İSG uzmanı (kritiklik, kabul sınırı, insan süreci), veri sorumlusu/hukuk (KVKK/lisans), IT/BT işletme (ağ, yedek, cihaz ve bakım). Bu kararlar AI ajanına devredilmez.

## 2. Faz bağımlılık haritası

```text
10-A.1 teknik risk kapatma
       |
10-A.2 ürün/İSG gereksinimleri ve risk eşiği
       |
10-A.3 izinli veri hattı ve etiketleme ───> 10-A.4 bağımsız doğruluk benchmark'ı
                                                  |
                                         10-A.5 kontrollü ML iyileştirme
                                                  |
       +-------------------+----------------------+--------------------+
       |                   |                                           |
10-A.6 backend/ops     10-A.7 frontend/UX                         10-A.8 iş geliştirme
       +-------------------+----------------------+--------------------+
                                                  |
                                      10-A.9 GO/NO-GO kapısı
                                                  |
                                    10-B izinli shadow-mode pilot
                                                  |
                                   11 üretimleşme / 12 sürekli iyileştirme
```

10-A.6/7/8, 10-A.3/4 ile **paralel** geliştirilebilir; gerçek saha pilotu 10-A.9 tamamlanmadan asla başlatılamaz. Etiketli saha verisi yokken 10-A.4 altyapısı ve testleri geliştirilebilir, doğruluk PASS olamaz.

---

## FAZ 10-A.1 — Mevcut teknik açıkların tamamlanması (DEVAM EDİYOR)

**Sahip:** Backend / platform + ML, Windows ve Mac ortam sorumluları.  
**Hedef:** Mevcut mühendislik risklerinin ölçüm ve kanıtla kapatılması; 10-A.2'nin temiz başlangıcı.

**İşler:**
1. Mac üzerindeki `fixall90` A/B bellek testinin iki farklı image ID, eşit yük, RSS/heap, ısınma dönemi ve MB/saat eğimlerini doğrula; sonuç yoksa PASS deme.
2. 24 saatlik soak gerçek süreyle, GPU/detector ve gerekli sistem sağlığı gözetimiyle planla; koruma eşiği, disk/CPU/DB gözlemi ve operatör iptal yöntemi olsun. Aktif test sürerken aynı Mac'e ek ağır iş yükleme.
3. Windows CUDA GPU parity, Mac MPS parity ve cross-platform yeniden kurulum/doğrulama; her platform sonucu ayrı.
4. Pillow/filelock/setuptools ve tüm bileşenlerin pip-audit/uyumluluk/regresyonları.
5. Windows `.env`, snapshot, outbox, backup ACL inventarizasyonu; riskli değişiklikler için geri dönüş planı ve kullanıcı onayı. BitLocker ve yedek şifreleme operasyonu ayrı onaylı.
6. Ana dashboard admin oturumlu manuel kabul kontrol listesi; izole testlerle karıştırma.
7. Windows ana DB'deki orphan run'lar yalnız dry-run; `--apply` için güncel aday listesi + açık onay.
8. İzole eski-uygulama / yeni şema rollback testi, test kaynaklarının izolasyonu ve restore kanıtları.

**Çıktı:** `docs/PHASE_10A1_REMEDIATION_REPORT.md`, memory A/B CSV, soaks, güvenlik risk sicili.  
**Kabul:** Kritik testlerde PASS; tamamlanmamış 24 saat/Windows doğrulaması **INCOMPLETE** olarak kalabilir; hiçbir gerçek saha pilotu izni yaratmaz.

## FAZ 10-A.2 — Ürün tanımı, İSG risk modeli ve iş gereksinimleri

**Sahip:** Ürün sahibi; İSG, işletme, UX ortak.  
**Amaç:** “Baret/yelek tespit eden demo”dan iş süreçleri tanımlı ürüne geçmek.

**İşler:**
1. Müşteri türlerini ve kullanım bağlamını ayır: tek saha küçük işletme, çok kameralı yüklenici, OSGB/İSG ekibi, merkezden çoklu saha izleme. Öncelikli bir hedef persona seç.
2. Ürünün **yapacağı/yapmayacağı** işleri yaz: baret/yelek ihlali adayı, operatör doğrulaması, rapor, kamera sağlığı ve kapsam riski. Yüz tanıma, kişi puanlama, otomatik yaptırım kapsam dışı.
3. Başarı metrikleri için tanım ve payda hazırla: incident-level missing-helmet/vest recall, false alarms / camera-hour, alert latency (kamera olay zamanı → operatöre görünme), coverage, camera uptime, event delivery. UNKNOWN kapsama dışında saklanmaz; ayrıca raporlanır.
4. Kabul metriklerini **İSG ve işletme sorumlularının yazılı onayına** sun. Başlangıç sayıları öneri olabilir; geliştirici tek başına güvenlik eşiği belirlemez.
5. Risk sicili ve FMEA-benzeri öncelik: kaçırılan ihlal, sahte alarm, kamera kopması, stale frame, gizli veri, veri kaybı, yanlış durum etiketlemesi, hukuki engeller, insan faktörü.
6. Kullanıcı hikâyeleri ve rol matrisi: admin, supervisor, viewer, operator/reviewer gerekip gerekmediği; kamera/site kapsamı.
7. Mevcut endpoint, tablo ve ekranları gereksinimlere eşle; **gereksiz refactor yerine gap analysis** üret.
8. Saha pilotunu başlatmadan görüşülebilecek işletme/İSG paydaşları için 8–12 yarı yapılandırılmış soruluk görüşme rehberi hazırla; müşteri görüşmesi yapılmadıysa yapılmış gibi yazma.

**Çıktılar:** `docs/product/PRD.md`, `PERSONAS.md`, `USER_STORIES.md`, `ACCEPTANCE_METRICS.md`, `RISK_REGISTER.md`, `FEATURE_GAP_MATRIX.md`.  
**Kabul:** Her kritik gereksinim için sahibi, test yöntemi, risk ve onay durumu net; saha kabul eşikleri onaysız ise BLOCKED.

## FAZ 10-A.3 — Veri mühendisliği, etiketleme ve yönetişim

**Sahip:** Veri mühendisi; İSG, hukuk/veri sorumlusu, ML.  
**Amaç:** Güvenilir, izinli ve ayrıştırılmış veri kümesi oluşturmak.

**İşler:**
1. **Önce izin:** veri işleme amacı, hukuki dayanak, aydınlatma, minimizasyon, erişim yetkisi, saklama/imha, transfer, telif/lisans, ortam ve kamera görüş alanı kaydı. Onay yoksa gerçek çalışan görüntülerini toplama.
2. Veri şeması: `site_id`, `camera_id`, `source_video_id`, UTC/PTS timestamp, frame index, person bbox, PPE bbox, person–PPE association, helmet/vest status `PRESENT/MISSING/UNKNOWN`, occlusion, uzaklık/visibility bandı, camera scenario, incident span, annotator provenance.
3. Güvenilir örneklem tasarımı: gece/gündüz, iç/dış mekân, yakın/uzak, kask rengi, arka plan, makine, hareket bulanıklığı, yoğunluk, kadraj kenarı, occlusion, RTSP kesintisi. Eksik senaryoyu `NOT EVALUATED` göster.
4. En az iki bağımsız inceleyici gerektiren belirsizlik/itiraz prosedürü; uyuşmazlık çözümü, kalite örneklemesi ve inter-annotator agreement raporu.
5. Train/validation/test ayrımı **site/kamera/kaynak video/gün** gruplarına göre; yakın kareler ve sahneler arası leakage testleri. Nihai test kümesi dondurulur.
6. Dataset manifest + sha256 + lisans/izin referansı + annotation sürümü; public benchmark'ı sahanın yerine kullanma.
7. Dataset yönetimi Git dışında, erişim kontrollü ve şifreli saklama; veri tasnifi, geri çekme, silme, imha kanıtı. Otomatik imha ana veriye uygulanmadan önce onay ve test gerekir.
8. Hazır anotasyon araçlarından (örn. CVAT/Label Studio) birini ancak güvenlik/lisans/hosting uygunluğu görülürse öner; sırf araç edinmek için yeni servis açma.

**Çıktılar:** `docs/data/DATASET_SPEC.md`, `LABELING_GUIDE.md`, `DATA_GOVERNANCE.md`, `DATASET_CARD.md`, `SPLIT_PROTOCOL.md`, annotation validator, leak testleri ve örnek sentetik fixtures.  
**Kabul:** Gerçek saha verisi olmadan *pipeline PASS / data BLOCKED* mümkün; nicel saha doğruluğu PASS değil.

## FAZ 10-A.4 — Bağımsız saha benchmark'ı ve model değerlendirmesi

**Sahip:** ML geliştirici; veri mühendisi ve İSG.  
**Amaç:** Nesne tespiti ile olay kararı doğruluğunu birbirinden ayırmak.

**İşler:**
1. `best.pt` SHA256, paket/sürüm, inference config, fps/stride, CUDA/MPS/CPU, zaman bilgisi ve dataset hash kaydı.
2. Detector object-level: class AP, precision/recall, mAP50/mAP50-95, confusion, per-class `Person/helmet/vest/no_helmet`, confidence, konum ve occlusion kırılımları.
3. Person–PPE association-level: ekipmanın yanlış kişiye atanması, uzak/kenarda/örtülü kişi; `UNKNOWN` ve `NOT_EVALUATED` oranları ayrı.
4. Track/incident-level: olay eşleştirme penceresi yazılı; start/end doğruluğu, false event, missed event, duplicate, ID switch, alarm gecikmesi, false alarms/camera-hour; denominator ve kapsam açığa çıkarılır.
5. Regression: 91% değerlendirmeme görülen kırılım, kenar dokunuşu, koyu baretin örtülmesi, kırmızı makinenin Person algılanması, ID-switch duplicate ve yanlış end.
6. Kamera/koşul bazında sample size, belirsizlik aralıkları, insan yorumcuların uzlaşısı ve worst-case sonuçlar.
7. Kilitli test setinde threshold optimizasyonu yapma; candidate araştırmasında validation set kullan.
8. Model doğruluğu ve operasyonel delivery güvenilirliğini ayrı puanla.

**Çıktılar:** `docs/ml/MODEL_CARD.md`, `BENCHMARK_REPORT.md`, `ERROR_TAXONOMY.md`, `EVAL_ACCEPTANCE.md`, tam tekrarlanabilir script/test.  
**Kabul:** Etiketli, bağımsız, temsil gücü olan test kümesi + İSG eşiği olmadan saha doğruluğu BLOCKED.

## FAZ 10-A.5 — Kontrollü ML/temporal iyileştirme laboratuvarı

**Sahip:** ML geliştirici; backend review ve İSG.  
**Bağımlılık:** FAZ 10-A.4'te ölçülmüş gerçek hata örnekleri. Verisiz koşulda yalnız deney altyapısı.

**İşler:**
1. **Önce baseline:** üretim `best.pt`, PPEMonitor, ByteTrack ve eşikleri değişmeden kalsın; ayrı branch/candidate modeli/konfigürasyonu.
2. Hata kök nedeni: detection, PPE association, conf/calibration, yetersiz coverage gate, temporal smoothing, ID switch/track fragmentation; düzeltmeyi doğru katmanda öner.
3. Aday deneyleri ölçülebilir ve geri alınabilir yap: cropping/ROI ve kamera yerleşimi, cihaz/çözünürlük, düşük ışık, bbox ilişkilendirme, occlusion-aware UNKNOWN, daha kontrollü track association, deduplicate; üretime aktarım için onay.
4. Veri yeterliyse fine-tuning/transfer learning adayları; kökeni ve lisansı belli veriler, etiket dengesi, ayrı val/test, model registry, model card.
5. Ablation ve A/B: baseline vs candidate aynı test manifest ve koşullarda, incident recall/FP-h/coverage/latency metrikleri + olası regresyonlar.
6. `no_helmet` sınıfı düşük recall nedeniyle güvenilirliği kanıtlanana kadar PPEMonitor'a eklenmez.
7. Feature flag + rollback, onaysız yeniden eğitim/model değiştirme yok.

**Çıktılar:** `docs/ml/EXPERIMENT_REGISTRY.md`, `CANDIDATE_COMPARISON.md`, `MODEL_CHANGE_POLICY.md`, testler ve model checksum.  
**Kabul:** Adayın sahada fayda sağladığı etiketli bağımsız veriyle kanıtlanır, kabul eşikleri geçilir, İSG/ürün onayı vardır; değilse baseline kalır.

## FAZ 10-A.6 — Backend, DevSecOps ve operasyonel kalite

**Sahip:** Backend ve platform; veri sorumlusu, İSG.  
**Amaç:** Mevcut sağlam teknik omurgaya gerçek operasyon süreçlerini eklemek.

**İşler:**
1. Gözden geçirilmiş API sözleşmeleri: incident review/triage, insan doğrulaması, düzeltme notu, kategori ve audit; **algoritmik orijinal kararı değiştirmeden** ayrı review katmanı.
2. Multi-site/multi-tenant ihtiyacını PRD'ye göre değerlendir; gerekiyorsa `site_id`, kamera/site bazında RBAC, nesne düzeyi testler. Gereksiz çok-kiracılı mimariye geçme.
3. Data lifecycle: snapshot, outbox, yedekler, inceleme notları ve operatör kayıtları için saklama ve erişim politikasını yaz; yıkıcı imha önce izole doğrulama, sonra açık onay.
4. Release/CI: Python/Docker dependency lock, SBOM, güvenlik tarama, migration additive, secret scanning, test katmanları, signed/tagged model/config, backup/restore, rollback drill.
5. Linux/macOS/Windows portability: CPU/MPS/CUDA ayrımı, örnek `.env` ve platform startup, per-platform CI smoke; hazır Mac kopyasının canonical Git geçmişiyle farkı.
6. SLO ve alerting: camera uptime, image freshness, stale analysis, outbox pending age, ingest 5xx, unauthorized access, disk/DB faults; düzeltme runbook'ları.
7. Trafik/ağ güvenliği ve TLS/backup encryption/Windows ACL adımları sadece onaylı izole/yerel ortamda.
8. 24 saat soak ve gerçek RTSP/camera evidence gerekiyorsa BLOCKED olarak takip.

**Çıktılar:** `docs/engineering/API_CONTRACTS.md`, `DATA_RETENTION_DESIGN.md`, `RUNBOOK.md`, `RELEASE_CHECKLIST.md`, `SECURITY_REVIEW.md`, CI testleri.  
**Kabul:** Auth/RBAC/video/snapshot/WS regresyonları, event idempotency, crash replay, migration/backup restore, platform izolasyonu PASS; gizlilik kararı ve gerçek saha hardware doğrulaması ayrı gate.

## FAZ 10-A.7 — Operatör iş akışı, UX ve insan denetimi

**Sahip:** Frontend + UX; İSG.  
**Amaç:** Tespit panelini eyleme dönüştürülebilir ama insan kontrollü bir inceleme ürününe çevirmek.

**İşler:**
1. `New → In review → Confirmed violation / False alarm / Uncertain → Closed` gibi **önerilen** review workflow'u ihtiyaç ve mevcut incident state'inden ayrı tasarla. Mevcut `start/ongoing/end` DB semantiğini bozma.
2. Operatör geri bildirim ekranı: model çıktısı, zaman/kamera, kanıt görseli sadece yetkili rol, UNKNOWN nedeni, kamera sağlığı, insan kararı ve audit.
3. Varsayılan ekranda aşırı güven veren yeşil/uygun gösterimleri kaldır; veri yok ve değerlendirilemiyor durumları ayrışsın.
4. Review kuyruk filtreleri, `needs attention`, site/camera, status, zaman; mobil uyum ve WCAG temelli erişilebilirlik.
5. Kullanıcı araştırması: İSG/operatör için gözlem görevleri, ilk reaksiyon zamanı, yanlış alarm geri bildirimi, kullanıcı hatası; denek gerçek kişilerse izin.
6. Onaysız otomatik mail/SMS/WhatsApp veya üçüncü tarafa görüntü iletimi yok. Gerekirse yalnız lokal/izinli, içerik minimizasyonlu bildirim tasarımı.
7. Sahte/sentetik içerik içeren demo modu; gerçek veriyle karıştırılamaz `DEMO` ibaresi.

**Çıktılar:** `docs/ux/OPERATOR_JOURNEYS.md`, `WIREFRAMES.md`, `USABILITY_TEST_PLAN.md`, `REVIEW_WORKFLOW_SPEC.md`, frontend testleri.  
**Kabul:** Rol/kamera kapsamı, logout cleanup, false alarm düzeltme, UNKNOWN görünürlüğü ve veri gizliliği testleri PASS; gerçek operatör testi yapılmadıysa UX VALIDATION BLOCKED.

## FAZ 10-A.8 — İş geliştirme, maliyet ve satışa hazırlık

**Sahip:** İş geliştirme & UX + ürün sahibi.  
**Amaç:** Ürün talebini ve ekonomik uygulanabilirliğini kanıtlamak; henüz fiyat/kazanç uydurmamak.

**İşler:**
1. Müşteri keşif görüşmeleri: İSG profesyonelleri, şantiye müdürü, küçük/orta ölçekli yüklenici, IT/BT. Problem ve mevcut manuel süreç, yanlış alarm toleransı, kamera altyapısı, alım komitesi.
2. Rakip/alternatif karşılaştırması: manuel devriye, CCTV + insan inceleme, PPE AI ürünleri; kaynak ve tarih ile.
3. Değer önerisi: olay inceleme süresi ve görünürlük, kayıt kalitesi, operatör iş yükü; iş kazası azalması kanıtlanmadıkça vaat etme.
4. TCO: donanım (Mac/edge/Windows), kamera, bakım, depolama, yedekleme, lisans, kurulum, operatör zamanı, support, veri imhası, elektrik; site/kamera bazında.
5. Fiyat model hipotezleri: site başına, kamera başına aylık, on-prem lisans + bakım; gerçek maliyet ve müşteri görüşmelerinden sonra belirle.
6. Demo/sunum, ürün anlatımı, teknik mimari, sahada ne yapmadığı, doğruluk sınırlamaları, güvenlik ve lisans uyarısı.
7. Müşteri geri bildirimi için anonim/izinli CRM/backlog; ticari sözleşme öncesi hukuk ve lisans kararı.

**Çıktılar:** `docs/business/ICP_VALUE_PROPOSITION.md`, `CUSTOMER_INTERVIEW_GUIDE.md`, `COMPETITIVE_MAP.md`, `TCO_AND_PRICING.md`, `DEMO_SCRIPT.md`, `PRODUCT_ONE_PAGER.md`.  
**Kabul:** Görüşme yapılmadıysa “araştırma planı hazır” PASS, pazar doğrulaması PENDING; hayali satış veya müşteriye dair sayı yok.

## FAZ 10-A.9 — Pilot öncesi bağımsız GO/NO-GO kurulu

**Sahip:** Ürün sahibi; İSG, veri sorumlusu/hukuk, BT işletme ve teknik lider.  
**Amaç:** Teknik PASS ile etik/hukuki saha iznini kesin ayırmak.

**Zorunlu imza/kabul listesi:**
- Etiketli bağımsız saha benchmark'ı, ölçümler, kamera bazlı worst-case, kabul metrikleri ve İSG imzası.
- Veri işleme amacı/dayanağı, aydınlatma, kamera görüş alanı, veri erişim/saklama/imha, gerekli hukuk/veri sorumlusu kararları.
- Ultralytics + model ağırlıkları + eğitim seti ve diğer bağımlılık lisanslarının kullanım senaryosu açısından değerlendirilmesi.
- Kamera/analiz kapsamı (UNKNOWN), yanlış negatif riski ve manuel fallback prosedürü.
- Gerçek RTSP cihazı/target hardware doğrulaması, 24 saat stabilite hedefi ve rollback.
- RBAC/TLS, backup şifreleme/erişim, secret ve yedek restore, audit, on-call/runbook.
- Operatör eğitimi, insan doğrulama, alarm yükü, şikâyet/itiraz süreci, test/kesinti planı.

**Çıktı:** `docs/decision/PILOT_GO_NO_GO.md` imza sahipleri ve kanıt bağlantıları; imzalanmamış maddeler BLOCKED.  
**Kabul:** Kritik maddelerden herhangi biri BLOCKED/FAIL = **NO-GO**. Bu fazın mühendislik dokümanı üretmesi saha erişim yetkisi oluşturmaz.

## FAZ 10-B — İzinli, insan onaylı shadow-mode saha pilotu (ŞİMDİ NO-GO)

**Sahip:** İSG + saha işletmesi; ürün, teknik, veri.  
**Önkoşul:** 10-A.9 resmî GO; onaylı saha ve kamera listesi.  
**Yöntem:** Kısa/ölçülebilir pilot takvimi ve örnekleme; yüz tanıma/otomatik yaptırım yok; insan yorumcu etiketleri ve günlük audit; kamera kopması, düşük coverage, yanlış alarm, kaçırma, tepki gecikmesi, bakım ve veri güvenliği takibi. Saha kararlarının yerini AI almaz.  
**Çıktılar:** `PILOT_EXECUTION_PLAN.md`, `PILOT_EVIDENCE.md`, `PILOT_EVALUATION.md`, tekrar GO/NO-GO.  
**Kabul:** Pilot gerçekten yapılmadıysa `PILOT NOT EXECUTED`; ticari kullanıma geçiş otomatik değil.

## FAZ 11 — Pilot sonrası üretimleşme ve dağıtım

**Koşul:** 10-B sonuçları ve ayrıca ticari dağıtım/hukuk onayı.  
**İşler:** cihaz kurulum paketleri, sürüm yönetimi, model/konfigürasyon release registry, güncelleme/rollback kanıtı, saha bazlı kullanıcı yönetimi, müşteri desteği, SLA/SLO, on-prem güvenli ağ/topoloji, test edilmiş backup/recovery, çoklu saha gerekiyorsa mimari, bakım planı, dokümantasyon ve operatör eğitimi.  
**Çıktı:** `docs/release/PRODUCTION_READINESS.md`, `SUPPORT_RUNBOOK.md`, `SLA_PROPOSAL.md`, `ROLLBACK_PROOF.md`.  
**Kabul:** Kanıtlanmış deployment ve yönetilebilir operasyon; model sertifikasyonu/iş güvenliği garantisi varsayılmaz.

## FAZ 12 — Sürekli kalite, MLOps ve ticari gelişim

**İşler:** izinli örnekleme ve drift izlemesi; dönemsel insan doğrulama; false alarm/false negative trendleri; sürümlü dataset/model registry; canary/rollback ve yeniden eğitim politikası; security patch ritmi; kullanıcı memnuniyeti ve support KPI; değişen saha koşullarına göre tekrarlanan kabul testleri; fiyatlandırma/müşteri kanıtı güncellemesi.  
**Çıktı:** `docs/operations/CONTINUOUS_MONITORING.md`, model/dataset değişim kayıtları, periyodik QA ve risk raporları.  
**Kabul:** Saha amaç, izin ve temsil koşulları değiştiğinde eskiden alınan GO geçersiz olabilir; yeniden değerlendirme gerekir.

---

## 3. Kalite ve kabul göstergelerinin sözlüğü

| KPI | Tam tanım | Onay/yorum |
|---|---|---|
| Incident recall (helmet/vest) | İnsan onaylı gerçek incident'ların kaçına zaman toleransında en az bir doğru alarm üretildiği | İSG eşik koyar; kamera ve koşul bazlı raporlanır |
| False alarm / camera-hour | İncelenmiş yanlış alarmlar / **gerçek aktif analiz saati** | Video süresi, kopan kamera süresi ve geçersiz kapsama ayrıca gösterilir |
| Coverage | Değerlendirilebilir track-frame / görülen tüm güvenilir person track-frame | “Güvenilir” payda kuralı açık; uzak/düşük confidence nedeniyle algılanmamış kişileri tek başına açıklamaz |
| Alert latency | İhlalin gerçek başlangıcı → operatör ekranında görünür alarm | Video zamanı ve wall-clock farkları, network/detector gecikmesi ayrı |
| Duplicate incidents | Aynı gerçek incident için fazladan oluşturulan incident sayısı | PID/ID switch ve yeniden bağlanma senaryoları |
| Lost persisted events | Outbox'a başarıyla kabul edilmiş, kabul SLA'sına rağmen DB'de olmayan olaylar | Diske kabul öncesi drop ayrı raporlanır; exactly-once network garantisi yok |
| Availability & health | Kamera video tazeliği, analizin çalışıyor oluşu, endpoint ve outbox sağlığı | “Çevrimdışı” kesinlikle “uyumlu” değildir |
| Review workflow | Alarmın insan tarafından inceleme süresi, confirmed/false/uncertain oranları | Operatör onayı orijinal model çıktısından ayrı tutulur |

**Sayısal kabul eşikleri bu planda bilinçli olarak boş:** İSG/işletme sorumluları hedef kamera ve kullanıcı yüküne göre yazılı onay vermelidir. Geliştirici tek başına `%99 güvenli` veya `0 false negative` gibi iddialar üretemez.

## 4. Öncelik ve ilk icra paketi

**P0 — Önce:** 10-A.1 checkpoint ve regresyon, veri hukuku/lisans, İSG kabul metrikleri, 91% analiz dışı kapsama sorunu, etiketli saha veri stratejisi.  
**P1 — Sonra:** 10-A.3 / 10-A.4 benchmark, 10-A.5 gerçek hata odaklı deneyler, 10-A.6/7 review workflow & security.  
**P2 — Paralel:** persona görüşmeleri, maliyet ve demo; mobil app, LLM, abonelik/tahsilat veya multi-tenant sadece doğrulanmış ihtiyaçla.  
**P3 — Son kapı:** 10-A.9 yazılı GO/NO-GO, 10-B shadow pilot, ancak ardından FAZ 11/12.

### İlk Claude Code oturumu için güvenli kapsam

- Henüz 10-A.1 bitmemiş veya commitlenmemiş olabilir. Önce Mac ve Windows **hangi depo güncel** belirle. Aktif A/B/soak testlerine dokunma.
- Çalışma ağacını, son HEAD'i, FAZ 10-A.1 rapor/ölçümlerini salt okunur incele.
- Bu yol haritasını `docs/PRODUCT_ML_QUALITY_ROADMAP.md` adına proje içine kullanıcı yerleştirdiyse oku, başka plana çelişiyorsa farkı raporla.
- **Yalnız FAZ 10-A.2** için mevcut özellik envanteri, PRD, gereksinim/rol matrisi, metrik tanımları, İSG onay formu, açık riskler ve önceliklendirilmiş backlog oluştur. Kod/DB/snapshot/model değiştirme. Kullanıcı görüşmelerini ve İSG/lisans/KVKK onaylarını uydurma.
- Raporu ve dokümanı doğrula, yalnız bu fazı commit et. Sonraki fazlara otomatik geçme.

## 5. Referanslar (hukuki görüş yerine geçmez)

- KVKK — İş yerlerinde güvenlik kamerası sistemleri: https://www.kvkk.gov.tr/Icerik/8770/is-yerlerinde-guvenlik-kamerasi-sistemi-kullaniminda-dikkat-edilecek-hususlara-dair-kamuoyu-duyurusu
- KVKK — Aydınlatma yükümlülüğü: https://www.kvkk.gov.tr/Icerik/2033/Aydinlatma-Yukumlulugu-
- Ultralytics — Lisans seçenekleri: https://docs.ultralytics.com/
- NIST AI RMF ölçüm ve insan gözetimi: https://airc.nist.gov/airmf-resources/playbook/measure/
