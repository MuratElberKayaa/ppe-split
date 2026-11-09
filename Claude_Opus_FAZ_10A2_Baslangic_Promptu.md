# Claude Code Opus — FAZ 10-A.2 Ürün ve İSG Gereksinimleri Başlangıcı

**MODEL:** Claude Opus  
**EFFORT:** High

PPE Safety Monitor projesinde FAZ 0–10-A geliştirmeleri raporlandı. FAZ 10-A.1 bellek A/B, güvenlik, platform ve uzun süreli test işleri Mac/Windows ortamlarında devam ediyor olabilir. Bu oturumun amacı bunları bozmak değil; **FAZ 10-A.2 ürün gereksinimi ve risk kabul dokümantasyonunu** somutlaştırmak.

## ÖNCE DENETLE

1. Bulunduğun gerçek proje kökünü ve platformu doğrula (Mac/Windows). `git status --short`, `git log -10 --oneline`, mevcut branch/HEAD, untracked belgeler, `.env` dosyalarının sadece varlığı.
2. İki cihazdaki kaynakların senkron olduğunu **varsayma**. Mevcut oturumda olmayan commit'leri uydurma; aktif test süreçlerini durdurma.
3. `docs/PHASE_10A_PREPILOT_REPORT.md`, varsa `docs/PHASE_10A1_REMEDIATION_REPORT.md`, `docs/PHASE_8_REPORT.md`, `docs/PHASE_9_REPORT.md`, `docs/MODEL_EVALUATION_PROTOCOL.md`, `docs/PPE_Saha_Hazirlik_Claude_Opus_Plani.md` ve eklediğim `docs/PRODUCT_ML_QUALITY_ROADMAP.md` dosyalarını incele.
4. Mevcut API/DB/UI özelliklerini yalnız dosya incelemesi ile envanterle, **rapora göre tahmin etme**.
5. Başka proje, Docker volume, gerçek kamera, ana DB, Windows süreçleri, model ve outbox'a müdahale etme. Aktif A/B veya soak testini bozabilecek komutlar çalıştırma.

## YALNIZ FAZ 10-A.2'Yİ UYGULA

A. Kullanıcı rollerini ve problem tanımını netleştir: şantiye İSG sorumlusu, operatör, saha yöneticisi, işletme admin'i, veri sorumlusu. Kullanım senaryoları, amaç dışı kullanımlar, insan kontrolü ve manuel güvenlik prosedürü.

B. `docs/product/PRD.md`, `PERSONAS.md`, `USER_STORIES.md`, `FEATURE_GAP_MATRIX.md`, `RISK_REGISTER.md`, `ACCEPTANCE_METRICS.md` oluştur. Her gereksinimde **mevcut / kısmi / eksik / doğrulanmamış**, öncelik, sahip, bağımlılık, kabul testi ve kullanıcı değeri olsun.

C. Mevcut kritik problem: yakın plan videoda track-frame'lerin %91'i analiz dışında kalabiliyor, bazı insan tespitleri yanlış, ByteTrack ID switch duplicate üretebiliyor, gerçek saha verisi yok. Bunların etkisini **güvenlik riski** olarak ele al; “insan analiz ediliyor/UNKNOWN” ile “uyumlu” durumlarını karıştırma.

D. Model ve sistem KPI sözlüğü oluştur: person/PPE detection recall, incident-level recall, false alarms/camera-hour, kapsam/UNKNOWN, alert latency, camera uptime, ID switch/duplicate, kalıcı outbox teslim gecikmesi, insan doğrulama süresi. Paydaları ve ölçüm kanıtını belirle. **Sayısal eşikleri İSG/işletme onayına bırak**, PASS uydurma.

E. FAZ 10-A.3–10-B arasındaki bağımlılıkları ve ürün backlog'unu sırala: izinli veri/etiketleme, bağımsız benchmark, kontrollü ML iyileştirmesi, backend/security, operatör triage UX, ticari değer araştırması, yazılı GO/NO-GO, en son izinli shadow-mode pilot.

F. KVKK ve Ultralytics/model lisans NO-GO'yu koru. Resmî yönlendirme için sadece karar için gereken kanıtları ve sorumluları belirle; hukuk kararı taklit etme, gerçek çalışan videosunu izin olmadan toplama.

G. Kullanıcı görüşmeleri henüz yapılmadıysa `docs/product/STAKEHOLDER_INTERVIEW_GUIDE.md` hazırla; cevapları uydurma.

## KABUL

- Dokümanların formatı ürün ekibi + İSG/işletme okuyucusuna anlaşılır olsun.
- Açık gereksinimlerin her biri kaynak/gerekçe, sahip, test ve GO/NO-GO bağımlılığı içersin.
- Backend/ML/UX'ün gerçekten eksik olduğunu yalnız repo doğrularsa 'eksik' yaz; değilse 'doğrulanmadı'.
- `docs/PHASE_10A2_PRODUCT_REPORT.md` içinde PASS/FAIL/SKIP/BLOCKED, değişen dosyalar ve açık kararlar yaz.
- Yalnız doğrulanmış dokümantasyon değişikliklerini, Git durumu uygunsa ve kullanıcı dosyaları korunuyorsa commit et; commit hash'ini paylaş.
- **FAZ 10-A.3'e, model eğitimine, yeni API/UI implementasyonuna veya gerçek saha pilotuna geçme.**
